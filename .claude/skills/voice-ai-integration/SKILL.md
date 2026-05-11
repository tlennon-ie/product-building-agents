---
name: voice-ai-integration
description: How to architect a conversational voice AI pipeline — ephemeral per-session agents with injected context, streaming STT, post-call memory/goal extraction, and audio failure handling. Vendor-agnostic.
---

This skill describes how to build a production conversational voice surface for an AI product. It assumes you have already chosen a voice provider, an STT provider, and an LLM provider, and you need to assemble them into a pipeline that feels like a real conversation.

The patterns here are vendor-neutral. They apply whether your voice stack is a single integrated provider (one vendor doing STT+LLM+TTS) or a composed stack (three different vendors stitched together).

## When to use this skill

Reach for it when you are:

- Building a voice-call surface in a product that already has text chat.
- Replacing a "call a persistent agent" architecture with per-session agents.
- Designing the post-call processing pipeline (transcripts → durable memory).
- Debugging voice quality issues that the LLM team is blaming on "the model."
- Tightening latency on an existing voice flow.

## Persistent vs. ephemeral voice agents

Most voice providers let you create either a persistent agent (a named, long-lived configuration) or an ephemeral session agent (created fresh per call and discarded). The tradeoff:

| Concern | Persistent agent | Ephemeral session agent |
|---|---|---|
| Setup latency on call start | Lower (already exists) | Slightly higher (~100-300ms to create) |
| Context per user | Generic — same for everyone | Personalized — injected memories, goals, summary |
| Privacy isolation | Risk of context bleed across users if misconfigured | Isolated by construction |
| A/B testing prompts | Hard — affects live calls | Trivial — each session is its own variant |
| Mid-iteration prompt updates | Risky — live calls may be in-flight | Safe — next call picks up new template |
| Cost | Lower per call | Marginal extra cost per creation |

For any product where the agent should know the user, ephemeral is the only sane choice. The setup cost is paid back the moment a returning user hears the agent reference what they talked about last time.

Treat the ephemeral agent as cattle, not pets.

## High-level pipeline

```
[client]                 [server]                   [providers]
  |                         |                          |
  | tap "start call" ---->  |                          |
  |                         | load user context        |
  |                         | compose system prompt    |
  |                         | create_ephemeral_agent ->| voice_provider
  |                         |        <-- connection_token
  | <-- token --------------|                          |
  |                                                    |
  | open ws / webrtc -----------> voice_provider       |
  |                                                    |
  |  [streaming audio + transcripts during call]       |
  |                                                    |
  | tap "end call" ----->   |                          |
  |                         | fetch transcript ------->| voice_provider
  |                         | run extraction --------->| llm_provider
  |                         | dedup + store            |
  | <-- toast: "saved N" ---|                          |
```

## Context injection at session start

The composed system prompt is the single most important artifact in the voice pipeline. It is built from four blocks:

```
def compose_system_prompt(persona, user_ctx, prior_summary):
    return f"""
{persona.base_prompt}

{persona.voice_specific_rails}

# What we know about this user
{format_memory_block(user_ctx.memories, max_tokens=1500)}

# Active goals
{format_goals(user_ctx.goals)}

# Last conversation
{prior_summary or "This is your first conversation with this user."}

# Behavioral rules for this call
- Speak conversationally; never read lists or markdown.
- Keep responses short — voice tolerates 1-3 sentences, not paragraphs.
- End most turns with one question, not several.
- If the user pauses, do not jump in immediately.
"""
```

### Memory block

Do not dump every memory you have on the user. Rank and cap:

- Recency: weight memories from the last 30 days higher.
- Relevance: if the call has a topic hint (e.g., the user opened a "work stress" persona), boost memories tagged with that topic.
- Cap the total block at ~1500 tokens. The LLM degrades on long system prompts.

### Goals

Include active goals, not historical ones. A goal the user completed three weeks ago is noise.

### Prior summary

If this is a resumed session, include a 100-200 word summary of the prior session — not the raw transcript. Raw transcripts blow your token budget and bury the signal.

### Dynamic greeting

Generate the opening line dynamically. A returning user should hear something like "Hey, last week you said the presentation was on Thursday — how did it land?" Generate it from the prior summary at call-start time, and pass it as the agent's `first_message`.

## Streaming STT handoff patterns

Streaming STT emits two kinds of events:

- **Interim transcripts** — partial guesses, updated as the user speaks. Use for UI feedback ("you're saying..."), not for LLM input.
- **Final transcripts** — committed text, emitted after VAD detects end-of-utterance.

The handoff to the LLM happens on `final`. Common patterns:

### Pattern 1: Turn-based (simplest)

```
on_stt_final(text):
    transcript_buffer.append(user_turn(text))
    response = llm_provider.complete(transcript_buffer)
    tts_provider.speak(response)
    transcript_buffer.append(agent_turn(response))
```

Works for most products. Fails when the user adds a clarification right after stopping.

### Pattern 2: Debounced turn

```
on_stt_final(text):
    pending_text += " " + text
    debounce(500ms, then submit_to_llm(pending_text))
```

Waits a beat in case the user is taking a breath. Adds latency but reduces interruptions.

### Pattern 3: Barge-in (advanced)

```
on_stt_interim(text):
    if tts_currently_playing and len(text.split()) > 2:
        tts_provider.stop()
        # let the user finish, then process
```

Required for natural-feeling voice. Without it, the user has to wait through the agent's full response before being heard.

## Latency budget breakdown

End-to-end target: under 1500ms from user-stops-talking to agent-starts-speaking.

```
STT finalization (after VAD):     200-300ms
Server: parse + compose request:    20-50ms
LLM first token:                  300-500ms
TTS first audio byte:             200-300ms
Network + client buffer:          100-200ms
                       total:    ~820-1350ms
```

Each provider lies a little about its latency. Measure it yourself:

```
def instrument_turn(session_id, turn_id):
    metrics = {
        "stt_final_at":      None,
        "llm_request_at":    None,
        "llm_first_token_at": None,
        "tts_first_byte_at":  None,
        "client_playback_at": None,
    }
    # record at each stage, emit on turn complete
```

Plot p50/p95/p99 per stage. The slowest stage is your bottleneck; do not optimize the others.

## The post-call extraction pipeline

After the call ends, the transcript becomes a durable artifact. The extraction pipeline turns it into structured memory.

```
def post_call_pipeline(session_id, user_id):
    transcript = fetch_transcript(session_id)

    # 1. Gate on minimum content
    user_words = count_user_words(transcript)
    if user_words < MIN_EXTRACTION_WORDS:  # e.g., 15
        log_event("extraction_skipped_too_short", session_id, user_words)
        return

    # 2. Run extraction LLM call
    extraction = llm_provider.extract(
        prompt=EXTRACTION_SYSTEM_PROMPT,
        user_message=transcript,
        response_schema={
            "summary":  "string, 100-200 words",
            "memories": "array of {fact: string, category: string}",
            "goals":    "array of {description: string, target_date: string?}",
        },
        max_output_items=7,  # 5 memories + 2 goals ceiling
    )

    # 3. Dedup semantically against existing
    new_memories = semantic_dedup(extraction.memories, existing_for(user_id))
    new_goals    = semantic_dedup(extraction.goals,    existing_for(user_id))

    # 4. Persist
    store_memories(user_id, new_memories)
    store_goals(user_id, new_goals)
    store_summary(session_id, extraction.summary)

    # 5. Surface to user
    notify_client(user_id, {
        "memories_saved": len(new_memories),
        "goals_saved":    len(new_goals),
    })
```

### Why a minimum-words threshold

Short sessions are noise. A user who opened a call, said "hi" to the agent, and hung up has not given you data worth extracting. Below the threshold (15 user words is a sane default; tune by product), skip extraction entirely. This avoids:

- LLM extraction cost on garbage input.
- Polluting the user's memory store with hallucinated facts.
- Surfacing a misleading "we saved 3 things" toast on an essentially empty call.

### Why semantic dedup matters

String-equality dedup fails immediately. The user says "I'm stressed about my job" today and "work has been overwhelming" next week — these are the same fact. Run an embedding-based similarity check (cosine > 0.85 → duplicate) before insert. Without dedup, the memory store balloons with redundant entries and the next prompt's memory block becomes useless.

### Cap output per session

Limit extraction to, say, 5 memories and 2 goals per call. One chatty session should not single-handedly fill the user's profile. The cap forces the extraction model to prioritize.

## Handling partial and failed audio

Voice transport fails in ways text chat does not. Build for it.

| Failure | What you observe | What to do |
|---|---|---|
| User's mic permission revoked | No audio frames after start | Surface clear UI, end call, no charge |
| Network drop mid-call | Websocket heartbeat misses | Auto-reconnect with `session_id`; resume from last completed turn |
| STT returns garbage | Low confidence scores, nonsense words | Continue but flag the session for review; do not retry mid-call |
| LLM provider 500 mid-turn | No tokens within 8s | Play a pre-recorded "give me a second" TTS clip, retry once, then end |
| TTS returns silence | No audio bytes within 3s of request | Fall back to secondary TTS provider if configured, else end gracefully |
| Transcript incomplete on end | Final transcript missing the last user turn | Run extraction on what you have; flag low completeness |
| User said nothing for 60s | Silence with no interim transcripts | End call, no charge, no extraction |

The reconnect-with-resume case is worth special attention. On reconnect, the client passes the original `session_id`. The server looks up the session, restores the conversation state up to the last completed agent turn, and creates a fresh agent connection with that history pre-loaded as the conversation context. The user should not have to re-explain anything.

## Observability for voice flows

Instrument from the start. Voice quality regressions are invisible without metrics.

Required dashboards:

- **Latency**: p50/p95/p99 per stage (STT, LLM first token, TTS first byte) over time, sliced by provider.
- **Extraction yield**: average memories and goals per session, % of sessions skipped for low word count.
- **Completion**: % of started calls that reached `on_call_end` cleanly (vs. dropped, timed out, errored).
- **STT confidence**: distribution per session; sessions in the bottom decile should be auto-flagged for review.
- **Provider error rates**: per provider, per route, with alerting on > 1% sustained.
- **Dedup hit rate**: % of extracted memories rejected as duplicates. Too low = dedup is broken. Too high = extraction prompt is regurgitating context instead of new signal.

Log the composed system prompt for each session (or at least its hash + the input components). When a user reports a bad call, the first question is "what did the agent know going in?" — and you need to be able to answer it.

## Anti-patterns

- **Persistent agent shared across users.** Privacy disaster waiting to happen.
- **Sending the raw prior transcript in the system prompt.** Use a summary. Transcripts are not knowledge.
- **No barge-in.** The user feels talked-over by the agent. They will not come back.
- **Synchronous extraction on the call-end response.** Run it async; let the client return immediately and toast when extraction lands.
- **String-equality dedup on memories.** Will not survive contact with real users.
- **Extracting from every session regardless of length.** Trash in, trash out.
- **Treating STT confidence as binary.** Use the score; sessions in the low decile are corrupt data.
- **Hardcoding one provider.** Always have a fallback path for TTS and LLM at minimum. Voice providers go down.
