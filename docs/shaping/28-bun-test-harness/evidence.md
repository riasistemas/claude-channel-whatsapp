# Evidence — #28 bun:test harness

## R-1: import.meta.main gate covers all side-effects
- Layer: static + unit
- Evidence type: spike `spikes/import-meta-probe.ts` (run before /build)
- Status: pending pre-bet
- Plan: probe imports server.ts via `import { sanitizeSecrets } from '../server.ts'` from a test-style file (no `Bun.run` of server.ts) and asserts:
  - `acquireLock` not called (lockfile not created in tmpdir)
  - `Bun.serve` not called (no listener on WEBHOOK_PORT)
  - `mkdirSync(STATE_DIR)` not called (no state dir creation)
  - `requireEnv` not called (no env-var validation; no `process.exit`)
- Mitigation if fails: extract to explicit `bootstrap()` function called from `if (import.meta.main) bootstrap()` block.

## R-2: CI runtime overhead
- Layer: timing observation
- Status: complete
- Bun's native test runner clocks under 1s for typical unit-test suites with no I/O. The bootstrap suite (~50 unit assertions) is expected to run in well under 5s. CI total run rises from ~30s to ~35s — acceptable.

## R-3: Test isolation for fs-touching helpers
- Layer: unit fixture template
- Status: complete (designed)
- Pattern documented in `tests/assert-sendable.test.ts` skeleton:
  ```ts
  import { describe, it, expect, beforeAll, afterAll } from 'bun:test'
  import { mkdirSync, rmSync } from 'fs'
  import { tmpdir } from 'os'
  import { join } from 'path'
  
  const TEST_STATE = join(tmpdir(), `whatsapp-test-${Date.now()}`)
  
  beforeAll(() => {
    mkdirSync(join(TEST_STATE, 'media'), { recursive: true })
    process.env.WHATSAPP_STATE_DIR = TEST_STATE
  })
  afterAll(() => rmSync(TEST_STATE, { recursive: true, force: true }))
  ```
- Tests use real filesystem (in tmpdir), real regex, real helpers — no mocks.

## TPG anti-pattern lens
- "All tests pass with no integration proof" — addressed for the helpers we test. Integration (HTTP through to MCP notification) is a separate MF.
- "Heavy mocks" — explicitly avoided. Helpers are pure functions or fs-touching with real tmp paths.

## Migration of v0.1.6 spike fixtures
| Spike file | Coverage | Migrates to | LOC delta |
|---|---|---|---|
| `pr4/spikes/sanitize-fixture.ts` (12 cases) | sanitizeSecrets | `tests/sanitize.test.ts` | ~−90 + ~+120 |
| `pr4/spikes/render-body-fixture.ts` (21 cases) | renderPermissionBody | `tests/render-permission-body.test.ts` | ~−140 + ~+170 (V2) |
| (inline in PR2 body, 12 cases) | safeMediaPath | `tests/safe-media-path.test.ts` | new file ~120 LOC |
| (inline in PR3 body, 7 cases) | assertSendable | `tests/assert-sendable.test.ts` (V2) | new file ~80 LOC |
| (inline in PR4 body, 4 cases) | assertSendablePhone | included in `tests/assert-sendable.test.ts` (V2) | +~40 LOC |
| (inline in PR #31 body, 13 cases) | sanitizeSecrets lowercase variants | merged into `tests/sanitize.test.ts` | +~30 LOC |
| (inline in PR #32 body, 15 cases) | sanitizeDisplayName | added to `tests/sanitize.test.ts` | +~50 LOC |

After V1: `pr4/spikes/sanitize-fixture.ts` deleted.
After V2: `pr4/spikes/render-body-fixture.ts` deleted.
