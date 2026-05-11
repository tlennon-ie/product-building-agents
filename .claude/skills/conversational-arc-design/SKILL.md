---
name: conversational-arc-design
description: How to design multi-phase conversational AI flows that feel psychologically coherent — phase definition, transition signals, persona/voice consistency, and avoiding tone-bleed between sibling products on shared infrastructure.
---

## Why arcs matter

A conversation without an arc is a feature; a conversation with an arc is a product. The difference is structural, not lexical — you cannot fix a flat conversation by adding warmer words.

LLMs default to a "helpful assistant" register: medium-length, balanced, lightly empathetic, mildly advisory. This register is fine for utility tools (search, summarization, code help) and disastrous for any product where the *shape* of the conversation is the value proposition.

An arc is an explicit, named, multi-phase structure encoded in the system prompt that tells the model where the conversation is going and how to behave at each step. Arcs solve four problems at once:

1. They give the model somewhere to go, so it stops repeating the same register turn after turn.
2. They produce a felt sense of progress for the user, even in short sessions.
3. They make the product distinctive — two products with the same infra but different arcs feel like different products.
4. They make prompts diffable — when you change "phase 2 should be more reflective", you can verify it on fixtures.

If your product's pitch is "talk to an AI about X", you need an arc. If it's "ask an AI to do X for you", you probably don't.

## When NOT to use an arc

Arcs are overhead. Skip them when:

- The user's job is transactional (lookup, generation, transformation).
- Sessions are sub-30-seconds and single-turn dominant.
- The product's value is the model's knowledge, not the conversation shape.

Forcing an arc onto a transactional product makes it feel slow and theatrical.

## Anatomy of an arc

Every arc has four ingredients:

- **Phases** — named segments with distinct behaviors. Usually 3 or 4. Never more than 5 without a very strong reason.
- **Transition signals** — observable cues that move the conversation from one phase to the next. Detected by the model from the user's latest turn(s).
- **Phase constraints** — what the model is allowed and forbidden to do in each phase.
- **Exit conditions** — what ends the arc and what state the conversation lands in.

If any of these four is implicit, the model will not reliably follow the arc. Name them all.

## Two archetypes

### The 3-phase decompression flow

Used when the user's primary need is to feel heard, not advised. The arc resolves emotional pressure rather than producing a decision.

```
PHASE 1 — RELEASE
Goal: Make space for the user to put it all down.
Behavior:
  - Receive. Do not redirect.
  - Two sentences max.
  - End with one open invitation question.
  - No advice. No reframes. No silver linings.
Transition cue:
  User's pace slows. Sentences get shorter. They ask a reflective
  question or say something self-aware ("I guess I'm just...").

PHASE 2 — EXPLORATION
Goal: Help the user hear their own pattern.
Behavior:
  - Reflect back what you're hearing as a pattern, not a verdict.
  - Two to three sentences.
  - End with one reflective question.
  - Still no advice.
Transition cue:
  User signals winding down ("yeah", "thanks for listening",
  longer pauses, lower intensity language).

PHASE 3 — GROUNDING
Goal: Close the loop. Leave the user steadier than they arrived.
Behavior:
  - One concrete grounding observation.
  - Acknowledge the work they just did.
  - No new threads. No new questions.
  - Optional: one small, low-effort next step IF the user asked.
Exit:
  The session ends here. Do not loop back to Phase 1.
```

The hard part of this arc is not Phase 1. It's enforcing "no advice" in Phase 2 when the user explicitly asks for advice. The model wants to help. You have to tell it that withholding advice *is* the help.

### The 4-phase guidance flow

Used when the user wants to leave with a next step. Resolves uncertainty into commitment.

```
PHASE 1 — UNDERSTANDING
Goal: Establish what's actually being asked.
Behavior:
  - Ask clarifying questions before diagnosing.
  - Two to three sentences.
  - End with one specific clarifying question.
Transition cue:
  You have enough context to name the actual problem.

PHASE 2 — DIAGNOSIS
Goal: Reframe the situation in a way the user hadn't seen.
Behavior:
  - State the pattern you see, plainly.
  - Three to four sentences.
  - End with one question that tests the reframe.
Transition cue:
  User agrees with the reframe, or refines it.

PHASE 3 — GUIDANCE
Goal: Offer a path forward, not a lecture.
Behavior:
  - One concrete recommendation. Not three.
  - Explain the trade-off honestly.
  - Three to five sentences.
  - End with one question about feasibility.
Transition cue:
  User signals readiness to commit, or names a constraint.

PHASE 4 — COMMITMENT
Goal: Convert intent into a specific next action.
Behavior:
  - Restate the action in the user's own words.
  - Name the smallest version of it the user can do this week.
  - End with one accountability question.
Exit:
  The session ends with a specific, scoped, time-bound action.
```

The hard part of this arc is Phase 3 — the model wants to give three recommendations. Force it to pick one. Three options is a dodge; one option is a stance.

## Transition cues

Phases don't transition on turn count. They transition on observable signals in the user's language. List the cues explicitly in the prompt so the model can pattern-match.

Useful cue families:

- **Pace shift** — sentence length dropping, more pauses, "I guess", "yeah", "right".
- **Self-reference** — user shifts from describing the situation to describing themselves.
- **Question type** — user moves from open ventilation to specific questions.
- **Energy** — intensity words drop ("furious" → "frustrated" → "tired").
- **Closing language** — "anyway", "so yeah", "I should probably...".

Forbid the model from transitioning on turn count alone. A user who is still dumping on turn 8 should still be in Phase 1.

## Keeping sibling arcs from collapsing

When two products share scaffolding — same memory injection format, same call architecture, same extraction pipeline, same persona schema — their arcs will drift toward each other unless you actively differentiate.

The four levers, in order of effectiveness:

1. **Different phase counts.** A 3-phase product and a 4-phase product cannot accidentally feel the same — the rhythm is different.
2. **Different ending shapes.** One product ends with grounding (no question). The other ends with a commitment question. Same input, different exit feel.
3. **Different default lengths.** Specify in the frame: "Default response: 2 sentences" vs. "Default response: 4 sentences". This single constraint creates an enormous felt difference.
4. **Different question styles.** One product asks open invitations ("what's that like?"). The other asks action-forcing questions ("what would you do first?"). Codify per phase.

Test for collapse by reading three turns from each product side by side with the persona labels stripped. If you can't tell which is which, you have collapse. The fix is in the frame, not in the persona file.

## Voice consistency inside an arc

A persona's voice must hold across phases. The phases change what the model does; the persona governs how it sounds.

In practice this means the persona block goes in the frame (above the arc) and the arc references but does not override voice. If you find yourself writing voice rules inside a phase ("in Phase 2, be more warm"), you have the layering wrong — pull voice up into the frame and let the phase only govern behavior.

Voice constraints worth specifying:

- Sentence length distribution (e.g. "mostly short, occasional longer").
- Allowed and banned phrasings (concrete lists, not vibes).
- Reference points the persona pulls from (e.g. "speaks like someone who has worked in X for fifteen years").
- Failure mode ("if you find yourself sounding like a generic assistant, you've drifted").

## Testing arcs

Arcs regress silently. You will not notice that Phase 2 has started giving advice until a user complains. Build a regression harness before you ship the second prompt change.

### Fixture set

20–50 input scripts covering:

- Cold start, no memory.
- Returning user with rich memory.
- User who blasts through Phase 1 in one turn.
- User who refuses to leave Phase 1.
- Off-topic drift.
- Jailbreak attempt mid-arc.
- Ambiguous request that could fit either Phase 1 or Phase 2.
- Very short messages.
- Very long messages.
- User explicitly asking for advice in a no-advice phase.

### Rubric

Per output, score:

- **Phase adherence** — did the model behave according to the named phase? 0/1.
- **Voice match** — does this sound like the persona? 0–3.
- **Length compliance** — within the specified default? 0/1.
- **Ending shape** — did it end the way the phase requires? 0/1.
- **Guardrail compliance** — no banned phrasings, no out-of-scope content? 0/1.

Score by hand on every prompt change for the first month. After that, run an LLM-as-judge with a separate, larger model scoring against the rubric. Track scores over time. A drop in one dimension across many fixtures means the latest prompt change broke something.

### A/B prompt evals

When you have two candidate frames or two candidate arc structures, run them against the same fixture set and have a judge model pick the better output per fixture. Tally wins. Don't trust a single fixture — trust the distribution.

### Live signal

Once shipped, watch:

- Session length distribution. A flatlined distribution means the arc isn't producing felt progress.
- Turn count before exit. If users leave mid-Phase-2, the transition cues are wrong.
- Memory extraction yield. Low yield means the arc isn't producing substantive content.
- User-initiated topic switches. Frequent switches mean Phase 1 isn't doing its job.

## Red flags

Signs your arc is broken:

- The model gives advice in a no-advice phase. → Frame is too soft. Add an explicit forbidden-actions list to the phase.
- The model skips phases. → Transition cues are missing or the model is treating turn count as a signal.
- Every session feels the same length. → The arc is hard-coded to time rather than reading cues.
- Users complain it feels "scripted". → Phase constraints are too rigid; loosen voice inside phases, not behavior.
- Sibling products feel identical. → Tone-bleed. Fix in the frame using the four levers above.
- Memory extraction yield is low. → Phase 2 isn't producing substantive user reflection. The arc is moving too fast.
- The model loops back to Phase 1 after Phase 3. → No explicit exit condition. Add one.

## A working frame

A compact template that holds the layers correctly:

```
[FRAME]
You are <persona name>, a <persona role>.
Voice: <2–4 lines of voice constraints>.
Banned phrasings: <list>.
Default response length: <N sentences>.

You will not: <scope refusals>.
You will: <scope pursuits>.

[ARC]
You move through <N> phases. Detect the current phase from
the user's latest turn(s) using these cues:

PHASE 1 — <NAME>
  Goal: ...
  Behavior: ...
  Transition cue: ...

PHASE 2 — <NAME>
  ...

[CONTEXT]
Memories about this user:
<injected>

Goals this user is working on:
<injected>

Recent session summary:
<injected>

[INSTRUCTION]
Respond to the user's latest message according to the phase
you detect. Stay in voice. Honor the phase constraints.
```

Keep the sections delimited and ordered. The model reads structure.

## Final notes

Arcs are not a substitute for a good persona, and a persona is not a substitute for a good arc. You need both. The persona makes each turn sound right; the arc makes the sequence of turns mean something.

When in doubt, fewer phases. Three sharp phases beat five blurry ones. A blurry phase is one where you can't say in a single sentence what the model is forbidden from doing.
