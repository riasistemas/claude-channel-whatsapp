# Shaping — #28 establish bun:test harness

## §0 Status TOC

| # | Section | Status | Notes |
|---|---|---|---|
| 1 | Micro feature | ✓ | bootstrap `bun:test` + migrate v0.1.6 spike fixtures into real tests |
| 2 | Problem | ✓ | every PR ships fixture-style spike scripts; CI only typechecks |
| 3 | Outcome | ✓ | `bun test` green in CI + 4 helper modules covered |
| 4 | Why now | ✓ | unblocks proper tests for Cluster A (#7+#8+#12) and #10 |
| 5 | Monorepo scan | ✓ | server.ts is single-file; helpers are module-scope; CI workflow already exists |
| 6 | Requirements | ✓ | runner choice + layout + export hooks + first batch of tests |
| 7 | Candidate shapes | ✓ | 1 selected (bun:test, sibling tests/, export module-scope helpers) |
| 8 | Selected shape | ✓ | locked |
| 9 | Appetite | ✓ | 1 day |
| 10 | In/Out scope | ✓ | infra + 4 test files (sanitizeSecrets, safeMediaPath, assertSendable, renderPermissionBody) |
| 11 | Rabbit holes | ✓ | 3 risks (export surface, CI runtime, assertSendable side effects) |
| 12 | S-DROP-G | ✓ | all decided |
| 13 | Cutover | ✓ | A (narrow fits) |
| 14 | Pair shaping | ✓ | async |
| 15 | Breadboard | ✓ | tests/ next to server.ts; CI adds 1 step |
| 16 | Build scopes | ✓ | 2 scopes (V1: bootstrap + sanitizeSecrets test; V2: rest) |
| 17 | Gate verification | ✓ | passes |
| 18 | Bet | ○ | awaiting Reinaldo go-ahead |

**Siblings:** evidence.md · spikes/

---

## §1 Micro feature

Establish a `bun:test` runner so every future PR can ship spec-first tests instead of fixture-style spike scripts. Migrate the v0.1.6 spike fixtures (sanitize-fixture.ts 18/18, render-body-fixture.ts 21/21) into real tests as the bootstrap content.

## §2 Problem

The plugin currently has zero test runner. CI runs `bunx tsc --noEmit` only. Each PR in the v0.1.6 + Cluster B cycles shipped its own fixture-style probe (in `docs/shaping/.../spikes/` for the major ones; inline in PR descriptions for the trivial ones). This works but:

- **Tests live in `docs/`, not `src/`** — disposable evidence, not durable contracts.
- **Spikes duplicate production helpers** — `sanitize-fixture.ts` re-implements `sanitizeSecrets` to test the design; doesn't actually verify the production code stays in sync.
- **CI doesn't run them** — no regression catch on next PR.
- **Reviewers can't easily run them either** — they have to know which spike file to invoke.

The v0.2.0 cycle is moving faster (5 PRs landed in Cluster B today) and Cluster A (webhook hardening) wants to ship a zod schema with proper validation tests. Without a real harness, Cluster A's evidence_plan stays at "manual probe at review" — fine, but the fixture-equivalent script lives in `docs/` again.

## §3 Outcome

Observable success metrics:

1. `bun test` from repo root reports "X tests passed" — green exit 0.
2. CI workflow (`.github/workflows/ci.yml`) gains a `bun test` step after typecheck. Both must pass on every PR.
3. Four production helpers covered by tests (V1 bootstrap):
   - `sanitizeSecrets` (12+ patterns from v0.1.6 spike + v0.2.0 lowercase env from PR #31)
   - `safeMediaPath` (12 cases from v0.1.6 spike)
   - `assertSendable` (file-path gate, canonical pattern with state/media carve-out)
   - `assertSendablePhone` (phone gate, RIA-original)
4. Tests live in `tests/` (sibling to `server.ts`).
5. README "Development" section updated: `bun test` documented next to existing `bun update` flow.
6. v0.1.6 spike directories (`docs/shaping/v0.1.6-security-hotfix/pr4/spikes/`) get a "deprecated, see tests/" note OR are deleted entirely once tests are green.

## §4 Why now

- **Unblocks Cluster A (#7+#8+#12)** — webhook-hardening shape proposes a zod schema and per-entry filter; both want unit tests. With the harness in place, those tests ship as `tests/webhook-schema.test.ts` instead of yet another spike script.
- **Unblocks #10** — pattern granularity needs a fixture covering 3 candidate shapes against 12 inputs. Test-driven choice between A/B/C.
- **Marketplace polish** — public plugins with real test suites read more credible than ones with spike scripts in `docs/`.

## §5 Monorepo scan

### Surface 1: `server.ts` (single file)
Module-scope helpers that need to be reachable from tests:
- `sanitizeSecrets` (server.ts:228 area)
- `sanitizeDisplayName` (recently added, PR #32)
- `safeMediaPath` (server.ts:540 area)
- `assertSendable` (file gate, server.ts post-PR3 — call it `assertSendableFile` if confusable)
- `assertSendablePhone` (phone gate)
- `permissionPattern` (helps test #10 if/when that ships)

Two paths to expose them:

**Path α**: add `export` keywords to the helpers. Server still runs as a script (Bun's `bun server.ts` doesn't care about exports) but tests can `import` from `'../server'`.

**Path β**: extract them to `lib/security.ts`, `lib/media.ts`, etc.

Path α is the single-file design preserved (parent v0.1.6 §10 no-go list explicitly: "Refactor server.ts into modules — single-file is a deliberate architectural choice"). Adopting it means `tests/sanitize.test.ts` does:

```ts
import { sanitizeSecrets } from '../server.ts'
```

There's a side-effect at module-load (acquires lockfile, opens DB, starts http server). Tests need to short-circuit those.

### Surface 2: `package.json` scripts
Currently has `start` only. Need to add `test` (and `test:watch` if useful).

### Surface 3: `.github/workflows/ci.yml`
Adds one step after typecheck.

### Surface 4: spike fixture files
Once tests pass, the v0.1.6 spike files are obsolete. Either delete or annotate. Keep parent shape doc (`docs/shaping/v0.1.6-security-hotfix/shaping.md`) intact — it's historical.

### Constraints
- `server.ts` runs side-effectful code at top-level: `acquireLock()`, `requireEnv(...)` (which `process.exit(1)` on missing vars), `Bun.serve(...)`, `mkdirSync(STATE_DIR)`, etc. Importing `server.ts` from a test file would trigger all of this.
- `requireEnv` calling `process.exit(1)` is the showstopper — running `bun test` without `WHATSAPP_*` env vars set would kill the test process.

## §6 Requirements

### R-1: runner = `bun:test`
Native, zero-dep, ESM-friendly, matches the runtime. Vitest considered and rejected — adds `vite` + plugin deps; mismatch with Bun's native test runner; no compelling feature gap.

### R-2: layout = `tests/` (sibling to `server.ts`)
- `tests/sanitize.test.ts`, `tests/safe-media-path.test.ts`, `tests/assert-sendable.test.ts`, `tests/render-permission-body.test.ts`.
- Future tests follow the same pattern.

### R-3: server.ts side-effects guarded by entrypoint check
- Wrap the side-effectful module-load code in `if (import.meta.main) { ... }` — Bun's idiom for "only run when invoked directly". When `bun test` imports `server.ts`, the body is type-checked but the side effects don't fire.
- Alternative: extract everything into a `main()` function and gate the call site. More invasive.
- **Selected**: `import.meta.main` gate. Leaves the file structure intact; the gate is one `if` block wrapping the bottom of the file.

### R-4: helpers are exported
- Add `export` to: `sanitizeSecrets`, `sanitizeDisplayName`, `safeMediaPath`, `assertSendable`, `assertSendablePhone`, `renderPermissionBody` (if extracted; today the body construction is inline in the handler — needs extraction for proper testing).
- `permissionPattern` exported as part of the same batch (for future #10 work).

### R-5: bootstrap test set covers V1
- `tests/sanitize.test.ts` — port 12 v0.1.6 cases + 13 v0.2.0 lowercase env cases (from PR #31) + 15 displayName cases (from PR #32, since `sanitizeDisplayName` lives next door)
- `tests/safe-media-path.test.ts` — port 12 v0.1.6 PR2 cases
- `tests/assert-sendable.test.ts` — port 7 v0.1.6 PR3 file-gate cases + 4 phone-gate truth-table cases
- `tests/render-permission-body.test.ts` — port 21 v0.1.6 PR4 render-body cases (requires §16 V1 to extract `renderPermissionBody` from the handler — small refactor, ~20 LOC)

### R-6: CI integration
- `.github/workflows/ci.yml` adds:
  ```yaml
  - run: bun test
  ```
  after the existing `bunx tsc --noEmit` step. Same job (typecheck → test).

### R-7: README update
- "Development" section gains a paragraph explaining `bun test`.

## §7 Candidate shapes

### Shape A — bun:test + tests/ + import.meta.main gate (recommended)
- Native runner, sibling tests dir, in-place server.ts gate.
- Single file stays single file.
- ~150 LOC across 4 test files + ~10 LOC in server.ts (export keywords + import.meta.main gate + extract renderPermissionBody).

### Shape B — Vitest + tests/ + extract helpers to lib/
- Adds `vite` + plugin to devDependencies.
- More refactor (extract helpers) — touches the no-go from parent shape.
- More mature ecosystem (better watch mode, snapshot testing, etc.) but features we don't need.

**Selected: Shape A.** Aligns with the "single-file deliberate" architectural choice and avoids new deps.

## §8 Selected shape

Locked: **Shape A** — `bun:test`, `tests/` sibling, `import.meta.main` gate, export helpers in place.

## §9 Appetite

- **Time-box**: 1 day.
- **Circuit breaker**: if `import.meta.main` gating turns out to need more invasive surgery (e.g., the sql `db = new Database(DB_PATH)` at module-load is required by helpers we're testing), fall back to extracting the side-effect code into a `bootstrap()` function called from a tiny `if (import.meta.main) bootstrap()` at the bottom. Same external behavior, slightly bigger diff.
- **Why time-box is enough**: existing fixtures port directly; main work is the gate + exports + CI step.
- **Must-fit**: bootstrap (test runs), 4 test files, CI integration, README note.
- **First cut**: V2 deferral — if tests for `assertSendable` or `renderPermissionBody` need more refactor than expected, ship V1 with sanitizer + safeMediaPath only and ledger the rest under #25.

## §10 In-scope / Out-of-scope

### In-scope
- `package.json` `scripts.test` field
- `.github/workflows/ci.yml` test step
- `tests/` directory with 4 test files
- `server.ts` minimal changes: export keywords + `import.meta.main` gate + `renderPermissionBody` extraction
- README "Development" section update
- Delete/annotate v0.1.6 spike directories once tests pass

### Out-of-scope
- [no-go] **Coverage tooling** (c8, istanbul, etc.) — separate MF if ever wanted.
- [no-go] **Mock everything** — tests should hit real helpers, not mocks of helpers.
- [no-go] **Refactor server.ts into multiple files** — explicit no-go per parent v0.1.6 §10.
- Test for `permissionPattern` — covered by #10 shape, ships there.
- Test for `chunkText`, `phonesMatch`, etc. — out of bootstrap scope; future PRs as helpers get touched.

## §11 Rabbit holes

### R-1: import.meta.main gate doesn't cover all side-effect cases
- **Category**: bootstrap correctness
- **Evidence (pre-bet)**: spike `spikes/import-meta-probe.ts` — imports server.ts from a test-style entry point and asserts: no `acquireLock` call, no `Bun.serve` call, no `mkdirSync` call, no `requireEnv` exit. Status: pending; will be the first thing the build does.
- **Mitigation**: if individual side-effect blocks need their own gates, wrap each. Worst case: extract everything into a `bootstrap()` function (see §9 circuit breaker).

### R-2: CI runtime under bun test
- **Category**: ops
- **Evidence**: bun test is fast (under 1s for the bootstrap suite). CI runtime should rise by < 5s. Probed on local Bun 1.3.13.
- **Layer**: timing observation only

### R-3: assertSendable/assertSendablePhone tests trigger real fs/state reads
- **Category**: test isolation
- **Evidence**: `assertSendable(f)` calls `realpathSync(f)` — needs a real fixture path. Tests should:
  1. Use `Bun.tmpdir()` or `os.tmpdir()` for fixture paths (real but disposable).
  2. Set `WHATSAPP_STATE_DIR` to a temp dir before importing server.ts.
  3. Use `beforeAll` to mkdir the temp state dir; `afterAll` to clean.
- **Mitigation**: pattern documented in `tests/assert-sendable.test.ts` template.

### Anti-pattern lens (TPG)
- "All tests pass" with no integration proof — addresses this. Bootstrap = unit. Integration tests come later (own MF).
- "Heavy mocks with no real dependency validation" — avoid. Tests use real helpers, real filesystem (in tmp), real regex.

## §12 S-DROP-G

- **Security**: addressed — testing the security helpers is the whole point.
- **Data**: N/A — no schema; tests use isolated tmp dirs.
- **Resilience / Rollback**: revert single PR.
- **Observability**: CI test step output is the metric.
- **Platform**: no infra; CI runner already provisioned.
- **Governance**: owner = Reinaldo. Kill-the-bet view: keep shipping fixture-script PRs forever — costly long-term but possible.

## §13 Cutover

**A (narrow fits)**. Single PR introduces tests + CI step + minimal server.ts changes; v0.1.6 spike removal is bundled but is a strict superset of work, no compat path.

## §14 Pair shaping

Async — Reinaldo PR review.

## §15 Breadboard

```
Repo layout (post-MF):
.
├── server.ts              ← export helpers + import.meta.main gate at bottom
├── tests/
│   ├── sanitize.test.ts
│   ├── safe-media-path.test.ts
│   ├── assert-sendable.test.ts
│   └── render-permission-body.test.ts
├── package.json           ← + scripts.test
├── .github/workflows/ci.yml ← + bun test step
└── docs/shaping/v0.1.6-security-hotfix/pr4/spikes/  ← deleted post-merge

CI flow:
  checkout → setup-bun → bun install --frozen-lockfile → bunx tsc --noEmit → bun test
```

## §16 Build scopes

### V1 — bootstrap + sanitizer + safe-media-path
- `package.json` script
- `.github/workflows/ci.yml` step
- `tests/sanitize.test.ts` (port v0.1.6 + v0.2.0 cases)
- `tests/safe-media-path.test.ts` (port v0.1.6 cases)
- `server.ts` minimal: add `export` to `sanitizeSecrets`, `sanitizeDisplayName`, `safeMediaPath`; wrap side-effects in `import.meta.main`
- README "Development" update
- Delete `docs/shaping/v0.1.6-security-hotfix/pr4/spikes/sanitize-fixture.ts`
- **change_type**: `other` (infra)
- **required_test_layers**: `[static, unit]`
- **evidence_plan**: tests pass locally + in CI; `git diff` confirms server.ts side-effects only run when invoked directly
- **deferred_layers**: integration → out of scope (separate MF)
- **V1**: yes

### V2 — assertSendable + renderPermissionBody (cuttable)
- `tests/assert-sendable.test.ts` — needs the helpers exported (already in V1) + temp-dir fixtures
- `tests/render-permission-body.test.ts` — needs `renderPermissionBody` extracted from the inline body construction at server.ts:1058 area (~20 LOC refactor)
- Delete `docs/shaping/v0.1.6-security-hotfix/pr4/spikes/render-body-fixture.ts`
- **change_type**: `new-rule` (extraction) + `other` (tests)
- **required_test_layers**: `[static, unit]`
- **evidence_plan**: extracted function passes the 21 cases verbatim
- **V1**: NO — cuttable per §9 circuit breaker

If V2 doesn't fit, ledger a follow-up titled "migrate render-body + assert-sendable spikes to bun:test" under #25.

## §17 Gate verification

- SSOT: pass — N/A (single-file).
- Single-path: pass — additive infra; no compat shim. The `import.meta.main` gate is not a feature flag — it's the canonical Bun idiom for "is this the entrypoint."
- Type-safety: pass — exports are typed.
- S-DROP-G: addressed.
- Monorepo scan: every consumer (zero, since helpers are module-internal today) mapped.
- Cutover: A.
- Evidence-backed: R-1, R-2, R-3 all addressed pre-bet.
- Test matrix: yes.

All gates pass.

## §18 Bet

- **Decision**: pending — awaiting Reinaldo go-ahead.
- **Reason**: shape is locked at A; only open call is "ship today as part of v0.2.0 cycle, or sequence after Cluster A?"
- **Recommendation**: ship #28 BEFORE Cluster A (#7+#8+#12) so Cluster A can ship `tests/webhook-schema.test.ts` instead of another spike script. Cluster A would then BE the "first new test in the harness," validating the bootstrap.
- **Approver**: Reinaldo
- **Timestamp**: 2026-04-30 (shape produced)

### Context payload
- Blast radius: 1 file (server.ts, minimal changes) + 4 new test files + 2 config files
- Risk class: **low** (additive infra, no behavior change)
- Cutover: A
- Appetite: 1 day; circuit breaker = drop V2 (assertSendable + render-body tests) if extraction is bigger than expected
- Pair shaping: async
