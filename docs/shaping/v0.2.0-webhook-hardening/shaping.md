# Shaping — Cluster A: Webhook hardening (#7 + #8 + #12)

Bundle three issues that all touch the same ~50 lines of `Bun.serve` boot + webhook handler at `server.ts:1639-1687`. Single MF, hard cutover.

## §0 Status TOC

| # | Section | Status | Notes |
|---|---|---|---|
| 1 | Micro feature | ✓ | webhook server hardening — body cap + hostname pin + zod validation |
| 2 | Problem | ✓ | 3 vectors at the webhook entry point: DoS, exposure, untyped payload |
| 3 | Outcome | ✓ | each issue closed with citable fix; observable at HTTP surface |
| 4 | Why now | ✓ | last security-flavored cluster of v0.2.0 hardening; surface is small + hot |
| 5 | Monorepo scan | ✓ | single file (`server.ts`); 1 inbound webhook surface; 0 external consumers |
| 6 | Requirements | ✓ | per-issue acceptance below |
| 7 | Candidate shapes | ✓ | 1 chosen — bundle with shared diff |
| 8 | Selected shape | ✓ | one PR, ~50 LOC, atomic |
| 9 | Appetite | ✓ | 1 day inside cycle |
| 10 | In/Out scope | ✓ | webhook entry only; payload-validation-everywhere out |
| 11 | Rabbit holes | ✓ | 4 risks, 3 with pre-bet evidence |
| 12 | S-DROP-G | ✓ | all dimensions decided |
| 13 | Cutover | ✓ | A (narrow fits) |
| 14 | Pair shaping | ✓ | async — Reinaldo PR review |
| 15 | Breadboard | ✓ | inbound HTTP → cap → sig verify → zod parse → enqueue |
| 16 | Build scopes | ✓ | single scope, 3 sub-changes |
| 17 | Gate verification | ✓ | passes |
| 18 | Bet | ○ | awaiting Reinaldo go-ahead |

**Siblings:** evidence.md · spikes/

---

## §1 Micro feature

Harden the webhook server boot in `server.ts:1639-1687`:

1. Add a JSON body size cap to reject `> 1 MB` payloads early (issue **#7**).
2. Pin `Bun.serve` hostname to `127.0.0.1` explicitly (issue **#8**).
3. Validate the webhook payload with a zod schema and check `phone_number_id` per `entry[]` element (issue **#12**) — currently only checked on the first entry.

## §2 Problem

`server.ts:1639-1679` is the entry point for every Meta webhook. Three weaknesses:

### #7 — no body size cap (server.ts:1664)

```ts
if (req.method === 'POST' && url.pathname === '/webhook') {
  const rawBody = await req.text()   // ← unbounded
  const sig = req.headers.get('x-hub-signature-256')
  if (!verifyHubSignature(rawBody, sig)) { ... }
```

A malicious caller pointing at the public tunnel can stream gigabytes into `req.text()`. The Bun process buffers the whole body in memory before HMAC verification runs, so the rejection at signature-mismatch time happens *after* allocation. Tunnel ingress + memory pressure → OOM kill.

The HMAC check after the fact is irrelevant for DoS — the cost is paid before the check.

### #8 — Bun.serve binds 0.0.0.0 by default (server.ts:1639-1687)

```ts
const httpServer = Bun.serve({
  port: WEBHOOK_PORT,
  async fetch(req) { ... },
})
```

No `hostname` field. Bun's default is `0.0.0.0` (all interfaces). On a multi-tenant host (or one with a routable IPv6), the webhook listener is reachable beyond the tunnel. README claims the plugin binds to `localhost`; it does not. The HMAC check on every POST contains exposure but the listener itself is reachable.

### #12 — webhook payload typed via `as any` (server.ts:1671 + processWebhookPayload)

```ts
const payload = JSON.parse(rawBody)
void processWebhookPayload(payload)
```

`payload: unknown` flows through `processWebhookPayload`, which casts liberally. `parseWebhookPayload(payload)` (downstream) does manual property access without validation. Two concrete consequences:

- **Inconsistent `phone_number_id` check** — `processWebhookPayload` validates the metadata `phone_number_id` only for `entry[0]`, ignoring entries 1+. Meta does send batched webhooks with multiple `entry[]` items in pricing-tier scenarios.
- **Type erosion** — every downstream `(payload as any).entry?.[0]?.changes?.[0]?.value?.metadata?.phone_number_id` is a chain of `unknown` access. Bugs from missing fields surface as silent drops, not loud failures.

## §3 Outcome

Observable success metrics:

1. **Body cap probe**: `curl -X POST -d "$(head -c 2000000 /dev/urandom | base64)" $WEBHOOK_URL` returns `413 payload too large` within 100ms; plugin memory does not climb.
2. **Hostname probe**: `lsof -i :3789` after plugin start shows `127.0.0.1:3789` only — no IPv6 or 0.0.0.0 binding.
3. **Schema rejection probe**: a valid HMAC signed POST with malformed JSON (e.g., `{"entry":[{"changes":[]}]}` missing `value`) returns `200` (Meta retry contract preserved) but plugin logs `webhook payload schema rejected: <zod issue>` and does not invoke `processWebhookPayload`.
4. **Per-entry phone_number_id**: a valid POST with two entries (`entry[0]` for our number, `entry[1]` for another WABA) processes only the matching entry; the other is logged-and-skipped, not silently merged.
5. Issues #7, #8, #12 closed via PR-merge.

## §4 Why now

Last "security-flavored" cluster of the v0.2.0 cycle. The webhook surface is *the* entry point — every threat model that touches inbound starts here. Body-cap + hostname-pin in particular are 5-line changes that should have been in v0.1.6 if the appetite had allowed (per parent v0.1.6 §10 Out-of-scope). #12 is bigger but groups naturally because it touches the same handler block.

## §5 Monorepo scan

Single file (`server.ts`). Touched surfaces:

### Surface 1: `Bun.serve` boot (server.ts:1639-1687)
Consumers: 0 (the server is the producer, not a consumer of anything observable).
External consumer: Meta Graph API (sends webhooks). No contract change visible to Meta (we still return 200 on accepted POSTs).

### Surface 2: webhook payload parser (`processWebhookPayload`, `parseWebhookPayload`)
Locations: `processWebhookPayload` at server.ts:1238+, `parseWebhookPayload` (downstream).
Consumers (call graph from L1674 `void processWebhookPayload(payload)`):
- inbound message handling at `parseWebhookPayload`
- `lastInboundByChat.set` at L1576 (post-PR4 numbering)
- `activeTask` set at L1567+ (#6 expiresAt now too)
- HMAC reply handling for permission relay (L1336)
- All inbound database writes (`stmtInsertMsg`)

Single producer (this webhook handler), single in-process consumer chain. No cross-service.

### Surface 3: env var addition
New: `WHATSAPP_WEBHOOK_HOSTNAME` (optional, default `127.0.0.1`) — escape hatch for operators who run the plugin in unusual network setups (e.g., Docker bridge requiring `0.0.0.0`).
New: `WHATSAPP_WEBHOOK_BODY_LIMIT_BYTES` (optional, default `1048576` = 1 MB).

### Constraints
- WhatsApp Cloud API webhook payloads in practice are < 50 KB (single message + metadata). 1 MB cap leaves 20× headroom for batched deliveries.
- Meta's retry contract: HTTP 200 within 20s = "delivered"; non-200 or timeout = retried up to 24h. Schema-rejected payloads should still return 200 (logged and dropped) rather than 4xx, otherwise Meta retry-storms a malformed entry forever.
- HMAC check at L1666 must run BEFORE schema validation — skipping it would let unsigned junk reach the parser.

## §6 Requirements

### #7 — body cap

- Add `WHATSAPP_WEBHOOK_BODY_LIMIT_BYTES` env (default `1_048_576` = 1 MB).
- Read `Content-Length` header; if present and exceeds cap, return `413` immediately.
- For `Content-Length` absent or unreliable, read body via streaming and abort if cumulative bytes exceed cap. Bun supports `req.body` as a `ReadableStream` — drain with size guard.
- Cap rejection happens **before** HMAC verification (the body isn't trustworthy enough to spend signature-compute on).

### #8 — hostname pin

- Add `WHATSAPP_WEBHOOK_HOSTNAME` env (default `127.0.0.1`).
- Pass `hostname` in the `Bun.serve` config: `Bun.serve({ port, hostname: WEBHOOK_HOSTNAME, fetch })`.
- README "Quick Setup" step 4 already says "expose localhost" — update if needed for accuracy (it implies localhost binding which now becomes truth).

### #12 — zod payload validation + per-entry phone_number_id

- Define a zod schema that mirrors the Meta webhook payload shape (subset we use): `entry[]` with `changes[]`, each change with `value.metadata.phone_number_id` + optional `value.messages[]` + optional `value.contacts[]`.
- After HMAC verify and JSON.parse, run `WebhookPayload.safeParse(payload)`. On failure: `log` the zod issue + return 200 (Meta contract). On success: continue.
- Inside `processWebhookPayload`, replace the manual `as any` access with the typed result. Iterate **per entry**, validating `phone_number_id` for each — drop entries that don't match `PHONE_NUMBER_ID`, process those that do. Log `webhook ignored (phone_number_id mismatch): <id>` per skipped entry.

## §7 Candidate shapes

### Shape A: Single PR, single bundle (recommended)
- All three changes in one PR (~50 LOC net).
- One review round, atomic ship.
- Pros: same surface, same review context, no cross-PR coordination.
- Cons: one fix's regression blocks all three.

### Shape B: 3 separate PRs (#7, #8, #12)
- Open three small PRs.
- Pros: each can ship independently if one needs more iteration.
- Cons: 3 review cycles, 3× CI runs, plus the schema validation in #12 makes the type story across the file inconsistent until all three land. Adds ceremony for a 50-LOC change.

**Selected: Shape A.**

## §8 Selected shape

One PR, branch `fix/v0.2.0-webhook-hardening`. Three sub-changes ordered:

1. **Body cap** (env + cap check + 413 path) — closes #7
2. **Hostname pin** (env + Bun.serve hostname field) — closes #8
3. **Zod schema + per-entry validation** — closes #12

Total ~50 LOC + maybe 25 LOC for the schema. PR body groups changes by issue with line-cited evidence.

## §9 Appetite

- **Time-box**: 1 day.
- **Circuit breaker**: if the streaming body-cap path proves harder than expected (Bun stream API quirks), drop the streaming guard and rely on `Content-Length` header only — log a warning if header is absent and fall through to the existing unguarded path. Ledger a follow-up to add streaming guard when bun:test harness is in place to prove it.
- **Why time-box is enough**: surface is ~50 LOC in one block; zod schema is the only design call; existing code does manual access we can typecheck against.
- **Must-fit**: #7 (body cap) — even header-only is enough for v0.2.0; #8 (1-line); #12 (zod schema + per-entry).
- **First cut**: streaming body-cap (header-only stays).

## §10 In-scope / Out-of-scope

### In-scope
- Body size cap on `/webhook` POST
- `Bun.serve` hostname pin
- Zod schema for webhook payload
- Per-entry `phone_number_id` validation
- README "Quick Setup" tweak if the hostname change requires it (probably not — it already says "localhost")
- Two new env vars + their docs in README

### Out-of-scope
- [no-go] **Validation everywhere** — only the inbound webhook payload gets the schema. Outbound Graph API responses, MCP tool args, etc. are separate MFs.
- [no-go] **Restructure error handling** — non-zero return codes for various failure modes is a separate UX/observability MF.
- [no-go] **Add new tools** — hardening only.
- [no-go] **Change HMAC compute** — `verifyHubSignature` stays as-is (constant-time compare is already correct).
- TLS termination at the plugin (Meta requires HTTPS but operators use tunnels) — out of scope; tunnel handles TLS.

## §11 Rabbit holes

### R-1: streaming body-cap on Bun has quirks
- **Category**: external integration (Bun's request body API)
- **Evidence (pre-bet)**: spike at `spikes/body-cap-probe.ts` proves `req.body` (ReadableStream) can be drained chunk-by-chunk with a running byte counter, aborting via `controller.error(...)` when limit exceeded. **Status: complete (designed)**.
- **Mitigation**: if the stream guard misbehaves, fall back to `Content-Length` header check only. Header-only is ~80% of the threat surface anyway (most malicious POSTs do send a length header).

### R-2: zod schema misses a Meta payload variant
- **Category**: contract drift
- **Evidence (pre-bet)**: schema based on actual `processWebhookPayload` access paths + Meta's documented webhook reference. Real-world payload sample captured during PR1 manual probe (Reinaldo's number, voice note + document) matches the schema.
- **Mitigation**: schema marks unknown fields as `passthrough()` (zod default for `z.object`) so adding fields Meta later sends doesn't break us. If a known-required field is missing, we log and 200 (Meta retry behavior unchanged from current).
- **Layer**: contract / unit fixture

### R-3: per-entry filtering changes behavior for batched webhooks
- **Category**: regression risk
- **Per-consumer impact**: `parseWebhookPayload` currently iterates entries already; adding the per-entry `phone_number_id` gate means entries from other WABAs (which we shouldn't see in practice — Meta routes by configured WABA — but theoretically possible) get dropped instead of silently processed alongside ours.
- **Evidence**: the current code at `processWebhookPayload` (server.ts:1238) reads `inboundPhoneId = (payload as any)?.entry?.[0]?.changes?.[0]?.value?.metadata?.phone_number_id` and rejects the WHOLE webhook if `entry[0]`'s phone_number_id doesn't match. The new behavior is *strictly safer* — we now process matching entries and skip non-matching ones instead of accepting them via off-by-one. If there's any production payload that today processes correctly because `entry[0]` is the one we want and `entry[1]` happens to also be ours, the new behavior still processes both. The only behavior change is when `entry[0]` is OURS and `entry[1]` is NOT — old code processed both (bug), new code processes only the first.
- **Layer**: unit fixture

### R-4: 413 status code surprises tunnel/proxy stacks
- **Category**: ops
- **Evidence**: `cloudflared tunnel` documented to forward arbitrary status codes through. `ngrok` ditto. No known proxy that re-codes 413 to something else. Mitigation: log every 413 path so we can correlate if Meta complains.
- **Layer**: observability

### Anti-pattern lens (TPG)
- ✅ "Live dependency without synthetic check" — applies to Meta webhooks. Mitigated by zod schema + the existing HMAC check + manual probe at PR review.
- ✅ "All tests pass" with no integration proof — applies (still no `bun:test` harness; tracked under #28). Per-PR fixture script in PR body.
- ❌ Migration testing — N/A, no schema change.
- ❌ Retry/idempotency — N/A, Meta retries are observed not changed.

## §12 S-DROP-G

- **Security**: addressed — three explicit hardening changes. AuthZ unchanged (HMAC stays). New env vars rotation: not applicable (config, not credentials).
- **Data**: N/A — no schema, no migration.
- **Resilience / Rollback**: addressed. Single revert. **Kill switch**: env vars allow operator to disable the new behavior partially: set `WHATSAPP_WEBHOOK_BODY_LIMIT_BYTES=0` to disable cap, set `WHATSAPP_WEBHOOK_HOSTNAME=0.0.0.0` to revert to all-interface bind. These are escape hatches, not compat path — single-path gate satisfied.
- **Observability**: addressed. Three new log lines: `webhook rejected (size > N bytes)`, `webhook bound to <hostname>:<port>` (already in startup log), `webhook payload schema rejected: <zod issue>` and `webhook ignored (phone_number_id mismatch): <id>`. All structured `log()` calls.
- **Platform**: addressed — no infra change. Two env vars added with safe defaults; existing operators see no behavior change unless they set the override.
- **Governance**: owner = Reinaldo. Kill-the-bet view: not doing it leaves the public tunnel reachable by 0.0.0.0 + DoS-able + untyped — acceptable for v0.1.x but not for marketplace v0.2.0.

## §13 Cutover decision

**A (narrow fits)**. Triggers: zero. All three sub-changes are atomic, single-file, no compat shim. Hostname change has the env-var escape hatch but that's an operational kill switch, not a compat path.

## §14 Pair shaping

Async — Reinaldo PR review. No live session needed; design space is constrained by Bun's API surface + Meta's documented payload shape.

## §15 Breadboard

```
Meta → POST /webhook
  │
  ├─ [#7 NEW] Content-Length check  →  if > limit: 413
  │
  ├─ [#7 NEW] streaming drain with size guard  →  if exceeded: 413
  │
  ├─ HMAC verify (server.ts:1666 — unchanged)
  │
  ├─ [#12 NEW] WebhookPayload.safeParse(JSON.parse(body))
  │     │
  │     ├─ on fail: log + 200 (Meta retry preserved)
  │     │
  │     └─ on success: validated payload
  │
  ├─ [#12 NEW] for each entry in payload.entry:
  │     │
  │     ├─ if entry.changes[0].value.metadata.phone_number_id !== PHONE_NUMBER_ID:
  │     │     log + skip
  │     │
  │     └─ else: processWebhookPayload(entry)
  │
  └─ 200 OK

Bun.serve hostname [#8 NEW] = WHATSAPP_WEBHOOK_HOSTNAME (default 127.0.0.1)
```

## §16 Build scopes

Single scope, three sub-changes:

### Scope: webhook-hardening

- **change_type**: `new-rule` (cap + zod) + `new-rule` (hostname pin) — bundled
- **required_test_layers**: `[static, unit (fixture), manual probe at review]`
- **evidence_plan**:
  - static: typecheck via existing CI
  - unit fixture: spike that exercises the cap logic + the zod schema against representative payloads + the per-entry filter
  - manual probe at review: Reinaldo confirms `lsof -i :3789` shows 127.0.0.1; sends an oversized payload via curl and observes 413; sends a real WhatsApp message and confirms the typed flow still works
- **deferred_layers**: integration → tracked under #28 (bun:test harness)
- **V1**: yes (entire scope)

## §17 Gate verification (pre-bet)

- SSOT: pass — N/A (single-file repo).
- Single path: pass — env-var kill switches are operational, not compat. Cutover §13 = A.
- Type safety: pass — zod schema replaces `as any` chains; `z.infer` provides typed downstream access.
- S-DROP-G: addressed.
- Monorepo scan: every consumer mapped.
- Cutover: A.
- Evidence-backed: R-1, R-2, R-3 have spike or design evidence; R-4 is observability mitigation.
- Test matrix: scope has change_type + required_test_layers + evidence_plan.

All gates pass.

## §18 Bet

- **Decision**: pending — awaiting Reinaldo go-ahead on shape.
- **Approver**: Reinaldo via PR review on this shape commit.
- **Timestamp**: 2026-04-30 (shape produced)
- **shaping_commit_sha**: filled at commit.

### Context payload
- Blast radius: 1 file, ~75 LOC, 0 cross-service consumers
- Risk class: **medium** (security surface change, but well-bounded; not high since no SSOT change, no cross-tenant, no data migration)
- Cutover: A
- Appetite: 1 day; circuit breaker = drop streaming guard, header-only fallback
- Pair shaping: async
