# Product Building Agents

A reference fleet of Claude Code agents and skills for shipping AI SaaS products with a small team — or one founder.

This repo is the practitioner-level extract of what I ran day-to-day while building three sibling AI products in parallel. The agents and skills here are deliberately **vendor-agnostic** so they're useful as a starting point for your own stack. Swap in your auth provider, your payment provider, your LLM, your voice provider — the patterns hold.

## What's in here

```
.claude/
├── agents/
│   ├── prompt-engineer.md          # Designs conversational arcs and persona prompts
│   ├── backend-architect.md        # Auth, RLS, payments, credit billing
│   ├── voice-pipeline-engineer.md  # Voice AI, STT, post-call extraction
│   ├── ui-designer.md              # Multi-brand design system, glassmorphic UI
│   ├── security-reviewer.md        # OWASP + AI-specific threat review
│   ├── master-orchestrator.md      # Cross-product backporting and fleet consistency
│   └── ad-campaign-manager.md      # In-product ad ops + AI-generated creative
└── skills/
    ├── conversational-arc-design/
    ├── multi-tenant-saas-backend/
    ├── voice-ai-integration/
    ├── multi-brand-design-system/
    ├── ai-saas-security-review/
    ├── cross-repo-backporting/
    └── multi-platform-ad-ops/
```

Each agent has a companion skill. The agent is the persona you invoke; the skill is the deeper how-to reference the agent (or you) loads when working on that domain.

## Agents at a glance

| Agent | When to invoke |
|-------|----------------|
| **prompt-engineer** | Designing system prompts, persona definitions, multi-phase conversation arcs, dynamic greetings, memory/goal extraction prompts, jailbreak guardrails. |
| **backend-architect** | Auth integration, RLS policy design, payment provider integration, credit-based billing, webhook handlers, rate limiting, admin guards. |
| **voice-pipeline-engineer** | Conversational voice AI, ephemeral per-session agents with injected context, streaming STT, post-call memory/goal extraction, audio debugging. |
| **ui-designer** | Component aesthetics, design tokens, multi-brand systems, dark glassmorphic patterns, motion language, accessibility floors. |
| **security-reviewer** | After code touching auth, payments, user input, AI prompts, file uploads, or admin surfaces. Reviews for OWASP + AI-specific threats (prompt injection, jailbreaks, system-prompt extraction). |
| **master-orchestrator** | Cross-product changes — backporting a fix from one sibling product to others, coordinating schema migrations, deciding what's core vs product-specific. |
| **ad-campaign-manager** | In-product ad operations — abstracting multiple ad platform APIs behind one interface, generating AI video/image creative programmatically, channel budget allocation, creative rotation. |

## Skills at a glance

Skills are loadable how-to references. They're not personas — they're playbooks the agents (or you) consult while working.

| Skill | Covers |
|-------|--------|
| **conversational-arc-design** | Multi-phase conversation flows, transition signals, tone differentiation between sibling products on shared infrastructure. |
| **multi-tenant-saas-backend** | Three-client auth pattern (anon / user-scoped / service-role), RLS policy patterns, webhook idempotency, credit ledger with auto-refund. |
| **voice-ai-integration** | Ephemeral vs persistent voice agents, context injection, streaming STT handoff, post-call extraction pipeline, latency budgeting. |
| **multi-brand-design-system** | Token layer architecture, brand differentiation via hue/accent/motion, glassmorphic do's and don'ts, accessibility at the token layer. |
| **ai-saas-security-review** | Six-layer threat model, classical web checklist plus AI-specific threats (direct/indirect injection, jailbreaks, system-prompt extraction), severity definitions. |
| **cross-repo-backporting** | Core vs product-specific classification, canonical-first backport workflow, drift reconciliation, the consolidation-vs-duplication call. |
| **multi-platform-ad-ops** | Channel abstraction contract, AI creative generation pipeline, three-phase budget allocation (discovery → exploit → defend), telemetry schema, safety rails. |

## How to use these agents

The files in `.claude/agents/` and `.claude/skills/` follow Claude Code's standard format. Anywhere Claude Code skills and subagents work, these will too.

### Claude Code (terminal CLI)

Project-scoped — drop into a repo you're working on:

```bash
cd your-project
git clone https://github.com/tlennon-ie/product-building-agents.git tmp-fleet
cp -r tmp-fleet/.claude/agents/* .claude/agents/
cp -r tmp-fleet/.claude/skills/* .claude/skills/
rm -rf tmp-fleet
```

User-scoped — make them available across every project:

```bash
mkdir -p ~/.claude/agents ~/.claude/skills
cp -r .claude/agents/* ~/.claude/agents/
cp -r .claude/skills/* ~/.claude/skills/
```

Inside a Claude Code session, invoke an agent with the Task tool (or just ask Claude to use it):

> "Use the backend-architect agent to design the credit ledger schema."

Skills are auto-discovered. Claude will load them when relevant, or you can ask for one explicitly:

> "Use the multi-tenant-saas-backend skill and walk me through the RLS audit."

### VS Code (Claude Code extension)

The VS Code extension reads the same `.claude/` directory in your workspace root. Clone or copy the files into your project's `.claude/` directory and the agents and skills appear automatically. Reload the window if they don't show up immediately.

### Claude Desktop

Claude Desktop reads skills from `~/.claude/skills/` (user scope). Copy this repo's skills there:

```bash
mkdir -p ~/.claude/skills
cp -r .claude/skills/* ~/.claude/skills/
```

Agents in `.claude/agents/` are a Claude Code feature; Claude Desktop's primary unit is skills. The agent files here are still useful as a reference — open them and paste the system prompt into a Project's custom instructions if you want that persona behavior in Desktop.

### Cursor

Cursor doesn't read `.claude/` natively, but the skill bodies translate cleanly. Two options:

1. **Project Rules** — open Cursor → Settings → Project Rules. Create a rule per agent, paste the body of the `.claude/agents/<agent>.md` file as the rule content. Cursor will apply it as context.
2. **Custom modes** — for the heavier personas (prompt-engineer, backend-architect, security-reviewer), create a Cursor custom mode and paste the agent body as the system prompt.

### Codex CLI / other agent harnesses

Most modern agent harnesses accept a system prompt and a tool allowlist. The agent files are markdown with YAML frontmatter — strip the frontmatter, paste the body into your harness's system prompt slot. The `tools:` field in the frontmatter tells you what tool surface the agent expects.

## Design principles behind the fleet

A few opinions that shape every file in here:

1. **Non-coding leverage matters more than coding leverage.** The prompt engineer, security reviewer, UI designer, ad ops agent — those are the hats a solo builder can't credibly wear. Replacing those well is what changes the math, not faster code generation.

2. **Brief agents like job descriptions, not like prompts.** The agents that produced the best work were the ones with clear scope, mental model, and quality bars. Treating an agent like autocomplete gets autocomplete-quality work.

3. **Agents can't hold architectural coherence — you can.** Three apps, shared backbone, divergent personalities — keeping that from drifting requires a human who holds the whole thing in their head. The `master-orchestrator` agent helps, but the call is still yours.

4. **Reviewing agent output critically is the actual job.** Agents are fast, confident, and wrong in ways that look right. Every agent file in here is meant to be invoked by someone who'll verify the output, not approve it on sight.

## Customizing for your stack

Every file is vendor-neutral. Find-and-replace to fit:

| Generic term in this repo | Replace with your stack's |
|---------------------------|---------------------------|
| `auth provider` | Clerk, Auth0, Supabase Auth, NextAuth, etc. |
| `BaaS with RLS` | Supabase, Firebase, PlanetScale + custom RLS, etc. |
| `payment provider` | Stripe, Paddle, Lemon Squeezy, etc. |
| `LLM provider` | OpenAI, Anthropic, Google, etc. |
| `voice provider` | ElevenLabs, OpenAI Realtime, Vapi, etc. |
| `STT provider` | Deepgram, AssemblyAI, Whisper, etc. |
| `ad channel` | Meta Ads, Google Ads, TikTok Ads, LinkedIn Ads, X Ads, etc. |
| `video generation service` | Higgsfield, Runway, Sora, Veo, etc. |

The patterns survive the swap. That's the point.

## Contributing

This is a personal reference repo first, a public example second. PRs that keep the vendor-neutral framing and the practitioner-grade tone are welcome. Vendor-specific forks are encouraged — fork this and replace the generics with your stack to make it concrete for your team.

## License

MIT. See [LICENSE](./LICENSE).
