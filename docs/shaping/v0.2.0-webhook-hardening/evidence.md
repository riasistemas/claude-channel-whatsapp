# Evidence — Cluster A (#7 + #8 + #12)

## R-1: Streaming body-cap on Bun
- Layer: unit fixture
- Evidence type: spike `spikes/body-cap-probe.ts`
- Status: complete (designed)
- Notes: drains `req.body` (ReadableStream) chunk-by-chunk with running counter. Aborts via `controller.error()` when limit exceeded. Bun returns the in-flight promise as rejected, which the handler catches and returns 413.

## R-2: Zod schema vs Meta payload variants
- Layer: contract / unit fixture
- Evidence type: spike `spikes/webhook-schema-probe.ts`
- Status: complete (designed)
- Notes: schema covers the subset `processWebhookPayload` actually reads. `passthrough()` on `z.object` lets Meta add fields without breaking. Real payload sample (PR1 manual probe captured `+5561985598585` voice note + document) parses cleanly.

## R-3: Per-entry filtering regression risk
- Layer: unit fixture (logical proof)
- Evidence type: code reading + behavior delta analysis
- Status: complete
- Notes: current behavior at `processWebhookPayload` (server.ts:1238) reads `entry[0]`'s `phone_number_id` and rejects whole webhook on mismatch. New behavior is *strictly safer*. Concrete cases:

| current state of webhook | old behavior | new behavior | change |
|---|---|---|---|
| 1 entry, OURS | process | process | none |
| 1 entry, NOT OURS | reject all | reject all | none |
| 2 entries, both OURS | process both | process both | none |
| 2 entries, OURS + NOT OURS | process both (bug — OURS check only on `entry[0]`) | process OURS, log+skip NOT OURS | **bug fixed** |
| 2 entries, NOT OURS + OURS | reject all (bug — `entry[0]` check rejects) | log+skip first, process OURS | **bug fixed** |

## R-4: 413 status code path
- Layer: observability
- Evidence type: doc references (cloudflared, ngrok pass-through behavior)
- Status: complete
- Notes: log every 413 path so we can correlate if Meta's webhook stats show new "delivery failed" entries. If cloudflared collapses 413 to 502 for some reason, we'll see it in the plugin log first.

## TPG anti-pattern lens
- "Live dependency without synthetic check" — applies (Meta webhooks). Mitigated by zod schema + manual probe.
- "All tests pass" with no integration proof — applies (no bun:test). Per-PR fixture script.

## Follow-ups to file post-build
- If zod schema turns out to need `passthrough` removed (some Meta variant has fields that look like ours but mean something else), follow-up under #25 v0.2.x.
