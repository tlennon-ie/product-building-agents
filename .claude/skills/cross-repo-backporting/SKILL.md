---
name: cross-repo-backporting
description: How to keep a fleet of sibling SaaS products consistent — what counts as "core" vs "product-specific", how to backport bug fixes and infrastructure changes cleanly, schema migration coordination, and the consolidation-vs-duplication call.
---

## The fleet pattern

A fleet is a small set of sibling SaaS products that share an identical backbone and differ only on the surface. Think of it as one engineered system wearing several brands.

A typical fleet has:

- **2–4 products** (more than that and the coordination cost overwhelms the synergy)
- **One shared backbone**: auth, sessions, billing/credits, rate limiting, database access patterns, streaming pipelines, voice/audio, moderation, observability
- **Divergent surfaces**: personas, prompt arcs, brand, copy, pricing, domain-specific features, target audience

Why this pattern exists:

- Reuse infrastructure across multiple bets
- Test market positioning with parallel brands
- Keep each product simple by refusing to bolt every feature into one app
- Let each product develop its own personality without forking the engine

Why it's hard:

- Every backbone change must reach every product, or drift sets in
- Schema migrations multiply by the number of products
- Dependency upgrades become coordinated waves
- The temptation to extract a shared package arrives early and is usually wrong

This skill is how you keep the fleet honest without prematurely consolidating it.

## Classifying changes: core vs product-specific

The single most important call is whether a change is core or product-specific. Get this wrong in either direction and you pay for it.

### Core (must stay in sync)

These belong to the backbone. If they diverge across products, you have a bug factory.

- Authentication, middleware, session handling
- Authorization rules, row-level security, tenant isolation
- Credit/quota accounting, refund-on-failure logic
- Rate limiting (strategy and thresholds)
- Request pipeline: validation, error envelopes, retry policies
- Streaming: chunking, backpressure, abort handling
- Voice/audio pipeline: TTS/STT routing, session lifecycle
- Moderation, jailbreak detection, prompt-injection guards
- Memory/context extraction pipeline
- Admin guards and admin route shape
- Webhook signature verification
- Logging shape, telemetry events, error reporting
- Shared schema: users, sessions, credits, audit logs

### Product-specific (intentionally divergent)

These are where the products earn their individual identity. Forcing these into sync destroys the point of running a fleet.

- Persona definitions, system prompts, prompt arcs
- Greetings, microcopy, error message tone
- Brand colors, themes, typography
- Landing pages, marketing pages, legal pages
- Pricing tiers, feature gates, plan copy
- Domain features unique to one product (e.g., goal tracking in one product only)
- Persona libraries and metadata
- Product-specific tables (only-here data)
- App icons, OG images, social previews
- Email template content (delivery is core, content is product)

### Gray zone

Decide deliberately. Defaulting either way creates technical debt.

- UI primitives (toast, modal, button): core if visually identical, product if styled
- Onboarding flow: core if the steps match, product if order/content diverges
- Analytics event names: strongly prefer core — analytics dashboards depend on it
- Feature flags: core for the flag system, product for which flags exist

### The default

When unsure, classify as **core**. Duplication of core code is cheap to maintain with a backport workflow. Silent drift in code you assumed was product-specific is expensive — it usually surfaces as a bug only one product has.

## Canonical-product-first workflow

Every cross-product change has a canonical product. The canonical product is wherever the change was first written, verified, and shipped. It is not necessarily the largest or oldest product — it is the one where the change is known to work.

### Step 1: Land it in the canonical

Treat the canonical change as a normal feature/bug PR:

- Write the code in one product
- Add tests
- Verify locally
- Ship to staging, then production
- Watch for regressions for at least a release cycle

Do not begin backporting until the canonical change has burned in.

### Step 2: Diff against each sibling

For each sibling product, diff the touched files against the canonical:

```bash
# from a fleet directory containing all product repos
for sibling in product-b product-c; do
  echo "=== $sibling ==="
  diff product-a/src/lib/credits.ts $sibling/src/lib/credits.ts
done
```

You will see three kinds of regions:

1. **Identical**: backbone code that matches byte-for-byte. The canonical patch applies cleanly.
2. **Adapted**: backbone code that legitimately references product-specific identifiers (env var names, table prefixes, brand strings). The patch applies with name substitution.
3. **Drifted**: backbone code that has silently diverged. Investigate why. If there's no good reason, reconcile (separate PR — see below). If there is a reason, document it and decide whether the canonical change still makes sense here.

### Step 3: Apply to each sibling

For each sibling, in parallel:

- Branch: `backport/<short-description>`
- Apply the patch, adapting product-specific identifiers only
- Run the sibling's full test suite
- Open a PR that references the canonical PR by URL in the description
- Keep the PR scoped to the backport — no opportunistic refactoring

### Step 4: Track until merged

Maintain a short ledger (in an issue, a Notion doc, a tracker — wherever the team works):

```
Canonical: Product A PR #423 (merged)
Backports:
  - Product B PR #387 (merged)
  - Product C PR #201 (in review)
```

The change is not done until every sibling is merged or explicitly excluded with a written reason.

## Branch and PR strategy

### One PR per repo

Each sibling gets its own PR in its own repo. Do not try to bundle backports into a single multi-repo PR — review, CI, and merge timing should be independent per product.

### Parallel vs sequential

**Parallel (default)**: Open all sibling PRs at the same time. Each sibling reviews and merges on its own schedule. Use this for the vast majority of backports.

**Sequential**: Use only when the canonical change is risky and you want one more sibling to validate before propagating to the rest. Land in Product B, watch for a few days, then propagate to Product C, etc.

Default to parallel. Sequential is for genuinely risky changes (auth, payments, anything irreversible).

### Branch naming

Use the same branch name across all sibling repos:

```
backport/credit-refund-on-stream-error
backport/upgrade-runtime-major
backport/fix-rls-policy-on-sessions
```

Predictable naming makes the ledger trivial to maintain and grep-friendly six months later.

### PR description template

```
Backport of <canonical-repo>#<pr-number>: <one-line summary>

Canonical PR: <url>
Why: <one-paragraph explanation of the original change>

Adaptations made for this product:
- <e.g., renamed `PRODUCT_A_API_KEY` → `PRODUCT_B_API_KEY`>
- <e.g., adjusted import path>

Drift found: <none, or "see follow-up PR #X">

Verification:
- [ ] Tests pass
- [ ] Smoke test against staging
- [ ] No new lint errors
```

## Handling drift

Drift is when backbone code has silently diverged across siblings. It is the single biggest threat to a fleet, because it accumulates invisibly.

### Causes of drift

- A bug was fixed in one product but never backported
- A feature was prototyped in the backbone of one product and forgotten
- An auto-formatter ran with different settings
- A developer "improved" the code without realizing it was shared
- A backport was adapted too aggressively

### Detection

Run a periodic fleet-wide diff (monthly is reasonable). For each shared backbone file:

```bash
diff product-a/src/lib/auth.ts product-b/src/lib/auth.ts
diff product-a/src/lib/auth.ts product-c/src/lib/auth.ts
```

Expected output: empty, modulo product-specific identifiers.

Anything else gets logged in a drift catalogue.

### Reconciliation

For each drifted file:

1. Determine which version is canonical. Usually the most recent verified change wins, but sometimes the older version is correct and the newer version was a bad change.
2. Open a `chore: reconcile drift in <file>` PR in each sibling that needs to converge.
3. Keep the PR strictly about reconciliation. No new features, no refactors.
4. Cross-link the reconciliation PRs so future archaeologists can trace the convergence.

### What not to do

- Do not bundle drift reconciliation into a feature backport PR. They are different changes with different risks.
- Do not "reconcile" by deleting a file in one product because the others don't have it. That's not reconciliation; that's a regression.
- Do not let drift sit. The longer it persists, the more future changes will pile on top of it, making reconciliation harder.

## Schema migration coordination

Migrations on shared tables are the highest-stakes cross-product change. Drift here corrupts data.

### Rules

1. **Author once, in the canonical product.** Write the migration, run it against canonical staging, verify behavior.
2. **Copy the migration file verbatim** into each sibling's migrations directory. The SQL is identical.
3. **Number sequentially per product.** Each product has its own migration numbering. The canonical might be at 015, a sibling at 013. The sibling gets two migrations to catch up: the new one becomes 014 in that product. Maintain a mapping in a fleet-wide ledger.
4. **Apply in order, per product.** Each product's database runs its own migrations in its own order. The constraint is that the *shipped* schemas converge, not that the migration numbers match.
5. **Never edit a shipped migration.** If a shared migration was wrong, write a follow-up migration. Edit-in-place will corrupt environments that already ran the old version.
6. **Test rollback.** Or document a forward-only fix path before shipping.
7. **Roll out within a tight window.** Apply to all production databases within hours, not days. A schema divergence that lingers becomes a release blocker for the entire fleet.

### Migrations that mix shared and product-specific tables

Split them. A migration that touches both a shared `users` table and a product-specific `goals` table should be two migrations: one core, one product-local. The core one backports. The product-local one stays put.

### Pre-flight checklist

Before applying a shared migration to production:

- [ ] Migration ran on canonical staging without errors
- [ ] Migration ran on canonical production without errors
- [ ] Migration file copied to every sibling repo
- [ ] Migration applied to every sibling staging
- [ ] Rollback path documented (or forward-only fix path)
- [ ] Rollout window scheduled across all production databases

## Dependency upgrade orchestration

Framework majors, runtime upgrades, and SDK majors must roll out as a coordinated wave.

### Workflow

1. **Pick the canonical product.** Upgrade it first.
2. **Burn it in.** At least one release cycle in production. Watch for deprecation warnings, error rate changes, latency shifts.
3. **Write upgrade notes.** Document every code change required: API shape changes, config flags, breaking imports, polyfills, replaced packages.
4. **Roll out to siblings in parallel.** Use the notes as a checklist. Each sibling gets its own PR.
5. **Hold the fleet at the same major.** Do not let one product lag a major behind — that is the path to a permanent fork.

### Minor and patch upgrades

Batch them. Run a fleet-wide bump on a fixed cadence (weekly or biweekly). Don't update dependencies ad hoc per product — that drifts versions across the fleet.

### When to refuse an upgrade

If you cannot commit to upgrading every sibling within a defined window (typically 2–4 weeks for majors), do not upgrade the canonical. A single-product major upgrade in a fleet is technical debt taken on by everyone else.

## The consolidation decision

Sooner or later someone proposes extracting a shared package. The instinct is correct in the long run and almost always premature in the short run.

### Costs of a shared package

- A versioning surface every product depends on
- A release cadence (who publishes, when, how, against which test matrix)
- Coupling that makes per-product experimentation harder
- A new repo and toolchain to maintain
- A hard fork point if one product needs to deviate

### When extraction is the right call

All of these must be true:

- The code has been duplicated and kept in sync for at least several months
- Backport friction is measurable and recurring (you've backported the same module 5+ times)
- The API surface has stabilized — you are not still discovering its shape
- Every product genuinely needs the same behavior right now
- No product is currently bending the code for product-specific reasons
- The team can commit to operating a versioned internal package with proper releases

If any of these is false, keep duplicating and use the backport workflow.

### How to extract

When you do extract, do it in one wave:

1. Cut the package from the current canonical version of the code
2. Publish (private registry, git submodule, or whatever your toolchain supports)
3. Migrate every product in the same PR cycle
4. Delete the duplicated copies in every product
5. Run the full test suite in every product

Do not leave a half-extracted state where one product uses the package and others still have local copies. That is the worst of both worlds: you have the package overhead and the duplication risk.

## Audit checklists

### Monthly fleet audit

- [ ] Run diff on all backbone files across all products
- [ ] Catalogue any drift; open reconciliation PRs
- [ ] Verify dependency versions across products; flag any majors that have drifted
- [ ] Verify schema state across products; flag any missing migrations
- [ ] Review the backport ledger; close out finished entries, escalate stuck ones

### Pre-release audit (per product)

- [ ] No backbone files diverge from canonical (except for documented adaptations)
- [ ] All applicable backports are merged or explicitly excluded
- [ ] All shared schema migrations have been applied
- [ ] Dependency versions match canonical (or have a documented exception)

### Post-incident audit

When something breaks in one product:

- [ ] Was the broken code core or product-specific?
- [ ] If core: are siblings affected? Patch them.
- [ ] If product-specific: confirm siblings aren't silently running similar code.
- [ ] Update this skill or the orchestrator agent with any new lesson.

## Anti-patterns

- **"Just for now"**: applying a fix to one product and promising to backport later. Later is never. Backport now or document why not.
- **Drive-by improvements**: tweaking adjacent code during a backport. Keep the diff minimal so review stays cheap.
- **Premature shared package**: extracting on the third occurrence of duplication. Wait until the API has stabilized.
- **Skipping the canonical burn-in**: backporting before the canonical change has proven itself. You'll spread bugs.
- **Mega-PRs across repos**: trying to coordinate multi-repo merges atomically. One PR per repo, tracked in a ledger.
- **Editing shipped migrations**: changing a migration after it has run anywhere. Always follow up with a new migration.

## Output discipline

When operating on a cross-product change, produce:

- A classification: core or product-specific
- The canonical product, named explicitly
- A per-sibling plan: files touched, adaptations needed, drift found
- One PR per sibling, cross-linked to the canonical
- A ledger entry recording the change and its propagation status

Keep it tight. Coordination overhead is the fleet pattern's main cost; minimize the meta-work so the engineering work can dominate.
