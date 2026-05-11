---
name: multi-tenant-saas-backend
description: Patterns for the backend foundation of multi-tenant AI SaaS — auth provider + JWT to DB handoff, row-level security policy patterns, payment-provider webhook idempotency, credit-based billing with auto-refund, and admin allowlists. Vendor-agnostic.
---

This skill is a reference for the backbone of a multi-tenant AI SaaS backend. It assumes a managed Postgres-style database with row-level security (RLS), an external auth provider that issues JWTs, and an external payment provider that emits signed webhooks. Substitute your chosen vendors as needed — the patterns are the same.

## The three-client pattern

You need three distinct database clients in your server code. Mixing them up is the most common source of multi-tenant data leaks.

### 1. Anonymous client

Used for public reads (marketing pages, public profiles, sign-up flows). No JWT attached. RLS treats every query as unauthenticated, so only rows explicitly marked public via policy are visible.

```ts
const anonClient = createDbClient(DB_URL, DB_ANON_KEY);
```

### 2. User-scoped client (JWT attached)

Used inside authenticated API routes. The auth provider's JWT is passed through; RLS policies use `auth.uid()` (or equivalent) to filter rows.

```ts
function createUserClient(jwt: string) {
  return createDbClient(DB_URL, DB_ANON_KEY, {
    global: { headers: { Authorization: `Bearer ${jwt}` } },
  });
}
```

This is the default client for almost every authenticated route. If you find yourself reaching for the service-role client, ask why first.

### 3. Service-role admin client (bypasses RLS)

Used only for: admin endpoints, background jobs, webhook handlers, and cross-tenant maintenance. Bypasses RLS entirely — so every query MUST filter by `user_id` manually.

```ts
let _adminClient: DbClient | null = null;
function getAdminClient() {
  if (!_adminClient) {
    _adminClient = createDbClient(DB_URL, DB_SERVICE_ROLE_KEY);
  }
  return _adminClient;
}
```

Treat the service-role key like a root password. It must never touch client bundles, logs, or error reports.

## RLS policy patterns

Enable RLS on every user-owned table. The presence of a `user_id` column without an enforced policy is a latent data leak.

### Owner-only access (most common)

```sql
alter table memories enable row level security;

create policy "owner can select"
  on memories for select
  using (auth.uid()::text = user_id);

create policy "owner can insert"
  on memories for insert
  with check (auth.uid()::text = user_id);

create policy "owner can update"
  on memories for update
  using (auth.uid()::text = user_id)
  with check (auth.uid()::text = user_id);

create policy "owner can delete"
  on memories for delete
  using (auth.uid()::text = user_id);
```

Notes:

- `using` controls visibility (SELECT, UPDATE, DELETE).
- `with check` controls what may be written (INSERT, UPDATE).
- Both clauses are needed on UPDATE to prevent users from "moving" rows to another user's id.

### Public read, owner write

```sql
create policy "anyone can read public profiles"
  on profiles for select
  using (is_public = true or auth.uid()::text = user_id);
```

### Tenant-scoped (org-style)

If your tenancy unit is an organization rather than a user, add a `memberships` table and join:

```sql
create policy "members can read org docs"
  on documents for select
  using (
    exists (
      select 1 from memberships
      where memberships.org_id = documents.org_id
        and memberships.user_id = auth.uid()::text
    )
  );
```

### Gotchas

- **Service-role queries bypass RLS.** You MUST filter by `user_id` in code. RLS is not a safety net for admin routes.
- **`auth.uid()` returns `uuid`; many auth providers issue text ids.** Cast consistently (`auth.uid()::text`) and keep the column type uniform across tables.
- **No JWT means `auth.uid()` is null.** That's not an error — RLS quietly returns zero rows. Test for it.
- **Joins respect RLS.** A join from table A (user-readable) to table B (locked down) returns nothing for the locked rows. This can mask bugs in queries that look like they "just return empty."
- **Functions and views inherit the caller's RLS by default.** If you need to bypass for a specific aggregation, mark the function `security definer` and own the access logic inside it.

### Testing RLS

For every new policy, write a smoke test:

```ts
test("user A cannot read user B's memories", async () => {
  const a = createUserClient(jwtForUserA);
  const b = createUserClient(jwtForUserB);

  await b.from("memories").insert({ user_id: "user_b", content: "secret" });
  const { data } = await a.from("memories").select("*");

  expect(data).toEqual([]);
});
```

Run this on every PR that touches schema. RLS regressions are silent and catastrophic.

## Webhook signature verification and idempotency

External webhooks are unauthenticated by default — anyone with the URL can POST. Two protections are mandatory.

### Signature verification

The payment provider signs each webhook payload with a secret known only to you and them. Verify before doing anything else.

```ts
export async function POST(req: Request) {
  const sig = req.headers.get("payment-signature");
  const raw = await req.text(); // raw body, not parsed JSON
  let event;
  try {
    event = paymentProvider.verifyWebhook(raw, sig, WEBHOOK_SECRET);
  } catch {
    return new Response("invalid signature", { status: 400 });
  }
  // ... continue
}
```

Two non-obvious points:

- Verify against the **raw body**, not the re-serialized JSON. Re-serializing changes whitespace and breaks the signature.
- Return 400 on signature failure, not 401. Webhooks aren't "auth"; they're a tamper check.

### Replay protection (idempotency)

Webhooks may be delivered multiple times. Your handler must produce the same DB state whether called once or ten times.

```sql
create table webhook_events (
  event_id text primary key,
  provider text not null,
  type text not null,
  received_at timestamptz default now(),
  processed_at timestamptz
);
```

```ts
const { error } = await admin
  .from("webhook_events")
  .insert({ event_id: event.id, provider: "payment", type: event.type });

if (error?.code === "23505") {
  // already seen — return 200 so the provider stops retrying
  return new Response("ok", { status: 200 });
}

await handleEvent(event);

await admin
  .from("webhook_events")
  .update({ processed_at: new Date().toISOString() })
  .eq("event_id", event.id);
```

The unique primary key is the dedupe lock. If two deliveries race, exactly one inserts; the other gets a unique-violation and skips.

### Out-of-order delivery

Don't trust event sequence. For state-shaped events ("subscription updated"), fetch the current entity from the provider before writing:

```ts
const current = await paymentProvider.subscriptions.retrieve(event.data.object.id);
await admin.from("subscriptions").upsert(mapToRow(current));
```

This makes the handler idempotent AND order-independent.

## Credit ledger design

Credits are the unit of consumption for AI SaaS (a chat turn, a voice minute, a generation). Two tables:

```sql
create table credit_balance (
  user_id text primary key,
  balance integer not null default 0 check (balance >= 0),
  updated_at timestamptz default now()
);

create table credit_log (
  id bigserial primary key,
  user_id text not null,
  delta integer not null,           -- negative = deduct, positive = grant/refund
  reason text not null,             -- 'chat', 'voice', 'refund:chat', 'grant:subscription'
  route text,
  metadata jsonb,
  created_at timestamptz default now()
);
create index on credit_log(user_id, created_at desc);
```

### Atomic deduct

Use a single UPDATE with a WHERE clause that enforces sufficient balance:

```sql
update credit_balance
   set balance = balance - :cost,
       updated_at = now()
 where user_id = :user_id
   and balance >= :cost
returning balance;
```

If no row is returned, the user lacked credits — return 402. Never proceed to the AI call without a successful deduct.

### Grant (subscription renewal)

Process in the webhook handler. Idempotent because the webhook event id is already deduplicated:

```ts
await admin.rpc("apply_credit_delta", {
  p_user_id: userId,
  p_delta: planMonthlyCredits,
  p_reason: "grant:subscription",
});
```

Where `apply_credit_delta` is a SQL function that updates `credit_balance` and inserts into `credit_log` atomically.

## Auto-refund on streamed AI failure

Streaming makes the deduct-before-call pattern tricky: the call might fail after one token, after fifty, or right at the end. The rule: if the user didn't get a usable response, they don't pay.

```ts
async function streamWithRefund({ userId, cost, route, runAiCall }) {
  await deductOrThrow(userId, cost, route);

  let tokensDelivered = 0;
  let failed = false;

  try {
    const stream = await runAiCall();
    const reader = stream.getReader();
    while (true) {
      const { value, done } = await reader.read();
      if (done) break;
      tokensDelivered += value.tokenCount ?? 1;
      yield value;
    }
  } catch (err) {
    failed = true;
    throw err;
  } finally {
    if (failed || tokensDelivered < MIN_USABLE_TOKENS) {
      await refund(userId, cost, route, { reason: "stream_failed", tokensDelivered });
    } else {
      await logUsage(userId, cost, route, { tokensDelivered });
    }
  }
}
```

Define `MIN_USABLE_TOKENS` per route. For a chat turn, ten tokens is not a usable answer; refund. For a one-line voice response, the threshold is lower. Pick numbers, document them, revisit when complaints arrive.

Refunds are credit movements like any other — they write to `credit_log`. The audit trail must show: deduct, refund, net zero.

## Admin allowlist via env vars

Admin gating belongs in environment variables, not in a DB column you might forget to flip back after testing.

```ts
const ADMIN_IDS = (process.env.ADMIN_USER_IDS ?? "")
  .split(",")
  .map((s) => s.trim())
  .filter(Boolean);

export function requireAdmin(userId: string | null) {
  if (!userId || !ADMIN_IDS.includes(userId)) {
    throw new HttpError(403, "forbidden");
  }
}
```

Every admin route starts with `requireAdmin(userId)`. The allowlist is short (typically just the founders); changes require a deploy, which is exactly the friction you want for privilege grants.

Pair with an audit log:

```sql
create table audit_log (
  id bigserial primary key,
  actor_user_id text not null,
  action text not null,
  target_type text,
  target_id text,
  metadata jsonb,
  created_at timestamptz default now()
);
```

Every admin mutation writes a row. This is your forensics layer.

## Integration testing approach

Unit tests don't catch the failures that hurt in this layer. Three integration tests pay for themselves many times over:

1. **RLS isolation.** Two users, two rows, four CRUD operations each. Cross-tenant access must fail.
2. **Webhook idempotency.** POST the same signed event twice. The second call must be a no-op (no duplicate credit grant, no duplicate row).
3. **Credit auto-refund.** Force the AI call to fail mid-stream. Assert that balance returns to its pre-call value and the log shows deduct + refund.

Run these against a real test database with real RLS enabled, not against mocks. The whole point is that the database enforces the rules.

## Quick checklist before merging backend changes

- [ ] Every new table has RLS enabled.
- [ ] Every policy has SELECT + INSERT + UPDATE + DELETE clauses (or an explicit reason for omitting one).
- [ ] Every service-role query filters by `user_id` or `tenant_id` in code.
- [ ] Every webhook handler verifies signature against raw body.
- [ ] Every webhook handler deduplicates by event id.
- [ ] Every credit-spending route wraps the AI call in try/finally with refund-on-failure.
- [ ] Every admin route calls `requireAdmin` first.
- [ ] No new `NEXT_PUBLIC_*` env var contains a secret.
- [ ] Required env vars are validated at boot.
