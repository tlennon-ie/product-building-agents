---
name: multi-platform-ad-ops
description: How to build in-product ad operations — abstracting multiple ad platform APIs behind one interface, programmatic AI video/image creative generation, channel-level budget allocation, creative rotation on performance signals, and admin-dashboard telemetry. Vendor-agnostic.
---

## The in-dashboard ad-ops pattern

The pattern is simple: your product owns the ad console. Every ad platform is reduced to a remote API behind a common interface. An operator runs all channels from one admin screen built into your product. No tab-hopping between vendor dashboards.

Why this pattern wins:

- One operator can run many channels.
- Every action is logged in your own database, not someone else's UI.
- Channel additions are integrations, not retraining.
- Safety rails and audit are first-class, not bolted on.
- The same backend that runs the product runs the ads.

When this pattern is wrong:

- You are spending under a trivial monthly threshold across one channel. Do it by hand.
- You have a dedicated media buyer who already runs a stack you cannot replace.
- The channel does not expose an API of sufficient quality for the actions you need.

Everything below assumes you are committed to the pattern.

## The channel-abstraction layer

One interface. Every ad platform implements it. The orchestrator only talks to the interface.

Contract:

```pseudocode
interface AdChannel {
  create_campaign(spec: CampaignSpec) -> campaign_id
  update_campaign(campaign_id, patch) -> void
  set_budget(campaign_id, daily_budget_cents) -> void
  pause_campaign(campaign_id) -> void
  resume_campaign(campaign_id) -> void
  upload_creative(asset: Asset, metadata: AssetMeta) -> creative_id
  attach_creative(campaign_id, creative_id) -> void
  detach_creative(campaign_id, creative_id) -> void
  fetch_metrics(campaign_id, window: TimeWindow) -> MetricsSnapshot
  health_check() -> Status
}
```

`CampaignSpec` is your normalized shape, not any platform's shape. It includes:

- `name`, `objective` (acquisition, retargeting, awareness)
- `audience` (your internal segment id; mapped per channel)
- `budget_daily_cents`, `budget_lifetime_cents` (optional)
- `start_at`, `end_at`
- `creative_ids` (initial attachments)
- `target_cpa_cents`, `target_roas` (where the channel supports it)

`MetricsSnapshot` is also normalized:

- `window_start`, `window_end`
- `spend_cents`, `impressions`, `clicks`, `conversions`
- `ctr`, `cpc_cents`, `cpa_cents`, `roas`
- `raw` — the original platform payload, kept for forensics

Each channel adapter handles:

- Auth, token refresh, credential storage.
- Mapping your normalized spec to the platform's actual API.
- Rate-limit handling and backoff.
- Retries on idempotent operations only.
- Error translation into a small, common error taxonomy.

The orchestrator never sees a platform-specific error code. Channel A and Channel B both raise `RateLimitedError`, `TransientError`, `PermanentError`, or `PolicyError`. That is enough.

## Creative-generation pipeline

The pipeline turns a brief into many tagged, moderated, storable assets.

Stages:

1. **Brief intake.** Operator (or another agent) submits a brief with audience, channel, message, format constraints, and brand rules.
2. **Prompt construction.** A prompt builder turns the brief into prompts for the relevant generation services. Templates per channel, per format, per audience.
3. **Generation call.**
   - Motion: call `video_generation_service.generate(prompt, aspect_ratio, duration_seconds, seed)`.
   - Stills: call `image_generation_service.generate(prompt, aspect_ratio, style_tags, seed)`.
4. **Asset landing.** Returned asset is downloaded and written to the asset store with a content-hash filename. Never trust the generation service to host long-term.
5. **Tagging.** Write a row to `ad_creatives` with: asset url, content hash, brief id, channel, audience segment, prompt text, prompt hash, seed, model id, generation cost cents, aspect ratio, duration, created at.
6. **Moderation pass.** Automated check for brand violations, regulated-category content, banned visual elements. Failed assets are flagged, not deleted.
7. **Approval gate** (optional). For sensitive briefs, a human must approve before the asset can be attached.
8. **Ready for attach.** Only assets in state `approved` can be passed to `upload_creative` on any channel.

Rules:

- One brief, many variants. Five to ten is a reasonable default.
- Prompt cache: `(prompt_hash, model_id, seed)` is unique. Do not regenerate.
- Cost ledger: every generation call records its cost. Generation budget is a real budget.
- Keep prompts, seeds, and model ids forever. You will want to regenerate variants of winners six months later.
- Aspect ratios per channel: do not produce a square asset for a channel that only serves vertical. Generate per channel.

Asset store schema:

```pseudocode
ad_creatives {
  id, content_hash, asset_url,
  brief_id, channel_id, audience_segment_id,
  prompt_text, prompt_hash, seed, model_id,
  aspect_ratio, duration_seconds, format,
  generation_cost_cents,
  moderation_status, moderation_notes,
  approval_status, approved_by, approved_at,
  state: draft | approved | live | demoted | retired | flagged,
  created_at, updated_at
}
```

## Budget allocation strategies

Default is explore then exploit. Do not allocate by intuition.

### Equal-split start (phase 1)

For each newly activated channel:

- Daily budget at `floor_cents` (the smallest meaningful daily spend for that channel).
- Run until per-channel sample size is large enough to read CPA with reasonable confidence. "Large enough" is domain-specific; pick a conversions-per-channel threshold and commit to it before launch.
- No reallocation during this phase except safety pauses.

### Explore/exploit (phase 2)

Once per day (or your chosen cadence):

```pseudocode
for each channel:
  cpa = blended_cpa(channel, window=last_7_days)
  roas = roas(channel, window=last_7_days)
  if cpa <= target_cpa * (1 - margin):
    # winner
    new_budget = current_budget * (1 + scale_step)
    new_budget = min(new_budget, channel_ceiling)
  elif cpa >= target_cpa * (1 + margin):
    # loser
    new_budget = current_budget * 0.5
    if new_budget < floor_cents:
      pause(channel)
  else:
    # neutral
    new_budget = current_budget
  set_budget(channel, new_budget)
  log_decision(channel, old, new, reason, inputs)
```

Constraints:

- `scale_step` no greater than twenty-five percent per day. Larger steps break the platform's learning.
- `channel_ceiling` is the per-channel hard cap. Above it requires manual approval.
- Pauses are recorded with cause. Auto-resume is allowed only after a configurable cool-off.

### Defend (phase 3)

For mature winners:

- Hold budget. Watch for saturation: rising CPA at flat-or-rising spend over a rolling window.
- On saturation, step budget down (not up) and reallocate the freed budget to next-best channels.
- Refresh creative aggressively on saturating channels — saturation often is creative fatigue, not audience exhaustion.

All allocator decisions are logged to `ad_allocator_decisions` with full inputs. No invisible changes.

## Creative rotation and fatigue detection

Per active creative on each campaign, track over a rolling window:

- `ctr_window`
- `cpa_window`
- `frequency_window` (where available)
- `downstream_conversion_rate`

Default rotation policy (tune the thresholds per product):

- **Promote** when `ctr_window > median_campaign_ctr * (1 + promote_margin)` for `N` consecutive windows. Increased share of impressions for the next period.
- **Demote** when `ctr_window < median_campaign_ctr * (1 - demote_margin)` for `N` consecutive windows. Reduced share.
- **Retire** when `ctr_window < ctr_peak * (1 - decay_pct)` and the creative is older than `min_age`. Move state to `retired`.
- **Refresh trigger** when the count of `approved` creatives on a campaign drops below `min_pool_size`, or when the only above-median creative is one. Kick a new brief into the generation pipeline.

Fatigue is real and fast. A creative that crushed for a week will often decay sharply in the second week. Always have at least one challenger live against any winner.

## Cross-channel attribution caveats

You will be lied to. Build accordingly.

- Every ad platform overcounts. Summing platform-reported conversions across channels will exceed reality.
- Single source of truth: your own backend, keyed by a click identifier (`ad_click_id`) that you pass through the destination URL and store on first landing.
- Platform-reported CPA is a ranking signal, not an accounting figure. Use it inside the allocator; do not report it to finance.
- Blended CPA for accounting: `total_spend_cents / your_system_conversions`.
- View-through conversions: default off. They are noise for most products.
- Multi-touch attribution: nice idea, expensive in practice. Default to last-click on your own click identifier and revisit only when volume justifies the modeling effort.
- Channels report against different windows and aggregation lags. Compare like-for-like windows; never compare a channel's day-zero number against another channel's day-seven number.

## Telemetry schema

Minimum tables (names indicative; shape is the point):

```pseudocode
ad_channels {
  id, key, display_name, status, credentials_ref,
  created_at, updated_at
}

ad_campaigns {
  id, channel_id, platform_campaign_id,
  name, objective, audience_segment_id,
  state, daily_budget_cents, started_at, ended_at,
  target_cpa_cents, target_roas,
  created_at, updated_at
}

ad_campaign_metrics {
  id, campaign_id, window_start, window_end,
  spend_cents, impressions, clicks, conversions,
  ctr, cpc_cents, cpa_cents, roas,
  source: 'platform' | 'internal',
  raw_payload_ref,
  fetched_at
}

ad_creative_metrics {
  id, creative_id, campaign_id,
  window_start, window_end,
  impressions, clicks, conversions,
  ctr, cpa_cents,
  fetched_at
}

ad_allocator_decisions {
  id, channel_id, campaign_id,
  decided_at, old_budget_cents, new_budget_cents,
  reason, inputs_json, actor: 'system' | user_id
}

ad_platform_calls {
  id, channel_id, method, endpoint,
  request_ref, response_ref, status_code,
  latency_ms, outcome, error_class,
  created_at
}
```

Snapshot, do not overwrite. The history table is what makes decay detectable.

Admin dashboard reads from these tables only. The dashboard never calls an ad platform live for a page render — it reads what the polling worker has already pulled.

## Safety rails

Build these first. They are not features; they are preconditions.

1. **Daily spend ceiling.** Per channel and total. The allocator cannot exceed it. A worker enforces it: if approaching ceiling, pause campaigns proactively.
2. **Anomaly halt.** Hourly check: if spend, CPA, or impressions deviate from trailing baseline by more than a defined threshold, pause and emit alert.
3. **Manual-approval gate.** Any budget change above `manual_approval_step_cents` or above `manual_approval_absolute_cents` queues for human approval rather than executing.
4. **Kill-switch.** One action — one row update or one CLI command — that flips all campaigns on all channels to paused. Test it monthly against a real but tiny live campaign.
5. **Dry-run mode.** Allocator and rotation runs compute decisions, write them to the decisions log with `actor='dry_run'`, and skip execution. Run dry for a sustained period before flipping to live.
6. **Credential isolation.** Per-channel credentials in the secret store. Scoped, rotated, never logged.
7. **Rate-limit handling.** Every adapter respects platform rate limits and backs off exponentially. A naive retry loop will get the account suspended.
8. **Idempotency.** Retries are only allowed on idempotent calls. Mutating calls require an idempotency key the channel adapter generates and stores.

## Failure modes

You will hit all of these. Plan for them.

- **Channel API outage.** Treat as transient up to a defined window, then alert. Do not pile retries.
- **Channel policy rejection.** A creative or campaign rejected on policy grounds. Mark creative as `flagged`, do not auto-resubmit, escalate.
- **Account suspension.** Disable the channel adapter, halt all in-flight operations on that channel, escalate immediately. Do not retry.
- **Metric lag and revisions.** Platforms revise historical numbers for days. Always re-fetch a sliding trailing window, not just yesterday.
- **Currency and timezone drift.** Normalize at ingest: store all money in cents in a single reporting currency; store all timestamps in UTC.
- **Cost runaway from generation pipeline.** Generation is metered. If a brief generates a thousand variants in a loop, you have a bug, not a feature. Cap variants per brief at the allocator level.
- **Attribution drift.** Your own conversion count and platform-reported conversions will diverge. Trust your own; flag the channel when divergence exceeds a threshold.
- **Operator footgun.** A human typing a four-figure daily budget into a field built for two-figure budgets. The manual-approval gate exists for this.
- **Stale creative pool.** All approved creative on a campaign retires and no refresh has been triggered. The refresh trigger must be a worker, not a vibe.

## Putting it together

The minimum viable in-product ad-ops stack:

1. One channel adapter behind the common interface. Working end-to-end with safety rails.
2. One brief → generation → moderation → approve → upload path. Working end-to-end.
3. Allocator running daily in dry-run for at least a week.
4. Telemetry tables and a basic dashboard reading from them.
5. Kill-switch tested against a tiny live campaign.

Only then add the second channel. Adding channels before the rails are proven is how operators lose four-figure sums to a single bad loop.
