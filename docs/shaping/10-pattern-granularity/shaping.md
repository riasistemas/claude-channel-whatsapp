# Shaping — #10 permission "Always" pattern granularity

## §0 Status TOC

| # | Section | Status | Notes |
|---|---|---|---|
| 1 | Micro feature | ✓ | refine `permissionPattern()` so "Always" doesn't auto-allow more than the user signed up for |
| 2 | Problem | ✓ | `Bash:git` matches `git status` AND `git push origin main --force` |
| 3 | Outcome | ✓ | re-prompt on dangerous variant after auto-allow on benign one |
| 4 | Why now | ✓ | UX flaw in v0.1.6 `sessionAllowPatterns`; safe by default before plugin grows |
| 5 | Monorepo scan | ✓ | single function (`permissionPattern`); 2 call sites |
| 6 | Requirements | ✓ | granularity rule per chosen shape |
| 7 | Candidate shapes | ◐ | **3 options — Reinaldo picks at bet** |
| 8 | Selected shape | ○ | pending pick |
| 9 | Appetite | ✓ | 0.5–1 day, depends on shape |
| 10 | In/Out scope | ✓ | `permissionPattern` only |
| 11 | Rabbit holes | ✓ | 3 risks |
| 12 | S-DROP-G | ✓ | all decided |
| 13 | Cutover | ✓ | A (narrow fits) |
| 14 | Pair shaping | ✓ | async — but bet decision needs Reinaldo's pick on §7 |
| 15 | Breadboard | ✓ | one function rewrite |
| 16 | Build scopes | ✓ | single scope, depends on §7 pick |
| 17 | Gate verification | ✓ | passes per shape |
| 18 | Bet | ○ | awaiting Reinaldo's §7 pick |

**Siblings:** evidence.md · spikes/

---

## §1 Micro feature

Refine the `permissionPattern()` function at `server.ts:809-830` so the "🔁 Always" auto-allow scope is closer to user intent. Today, tapping "Always" on a permission for `git status` auto-approves *all* `git` invocations including destructive ones.

## §2 Problem

Current implementation (post-v0.1.6, server.ts:809-830):

```ts
function permissionPattern(
  tool_name: string,
  description: string,
  input_preview: string,
): string {
  if (tool_name === 'Bash') {
    // First token of the command: `ls`, `git`, `curl`, ...
    const cmdMatch = /^\s*([^\s|;&]+)/.exec(input_preview)
    return `Bash:${cmdMatch?.[1] || '*'}`
  }
  // ... non-Bash: extract longest path's dir
}
```

Bash patterns are derived from the **first token** only. Concrete consequences:

| First request | Always-tap auto-approves later |
|---|---|
| `git status` | `git push origin main --force` |
| `npm install` | `npm publish` |
| `curl https://api.example.com/health` | `curl -X POST $UNTRUSTED_URL` |
| `rm /tmp/scratch.log` | `rm -rf ~/.ssh` |

For non-Bash tools the pattern is the dir of the longest extracted path — also coarse. `Edit` on `/Users/luoarch/notes/today.md` auto-approves all future `Edit` operations under `/Users/luoarch/notes/`, which is mostly fine but still surprises operators when they tap Always on a notes edit and find Claude later editing config there.

The leak surface from #15 is closed (PR #22 stripped the pattern from the WhatsApp body) but the **scope of consent** the operator is granting via "Always" is still coarser than the visible request. The body shows `git status`; the auto-allow rule that gets stored is `Bash:git`.

## §3 Outcome

Observable success metric (per chosen shape — see §7):

- **Shape A (specific)**: tapping Always on a `git status` permission later re-prompts on `git push --force` (different second token).
- **Shape B (granular by safe-flag list)**: tapping Always on `git status` auto-approves `git status -s`, `git diff`, `git log` but re-prompts on `git push`/`git reset`/`git rm`.
- **Shape C (drop "Always")**: every permission is single-shot; nothing to test about scope.

Probe per shape lives in `spikes/`.

## §4 Why now

This is the residual UX flaw from the v0.1.6 permission relay redesign. Surfaced in PR #22 review but explicitly deferred (parent shape §10) to v0.2.x. Now is the time before:
- Plugin gets distributed widely (marketplace) and the coarse-Always behavior surprises new users.
- Anyone bumps into a real near-miss (operator taps Always on `npm install` to skip approvals, Claude later runs `npm publish`).

## §5 Monorepo scan

Single function (`permissionPattern`), two call sites:
- `server.ts:914` — handler at permission_request notification: computes pattern, checks `sessionAllowPatterns.has(pattern)` for auto-allow, stores in `pendingPermissions`.
- `server.ts:1334` — reply handler: when user taps "Always", reads `pendingPermissions.get(request_id).pattern` and adds to `sessionAllowPatterns`.

No external consumer of the pattern string (closed by PR #22 — pattern stays in-process). No persistence (in-memory Set; clears on plugin restart).

Constraints:
- Pattern strings are opaque to the user — they don't see them in the body, just behavior.
- `sessionAllowPatterns` is a `Set<string>` — exact-match lookup. Any change to the format changes lookup semantics; no per-existing-entry compatibility shim needed since restarts clear the set.

## §6 Requirements

Depend on §7 pick. Common to all shapes:
- The function rewrite must keep the `permissionPattern(tool_name, description, input_preview): string` signature so call sites at L914 and L1334 don't change.
- Lookup behavior unchanged: same string → same allow.
- v0.1.6 in-memory state invalidates on next restart (no migration).

Per-shape:
- **Shape A (specific)**: Bash pattern includes first AND second token (or as many tokens as are non-flag, non-quoted). E.g., `git status` → `Bash:git status`; `git push origin main --force` → `Bash:git push`.
- **Shape B (granular by safe-flag list)**: Bash pattern carries the verb + a safe/unsafe classification token. E.g., `git status` → `Bash:git:safe`; `git push --force` → `Bash:git:unsafe`. Maintains a small allowlist of safe verbs per command (`git: status, diff, log, show; npm: list, view, info; curl: GET-only`).
- **Shape C (drop "Always")**: remove `sessionAllowPatterns`, drop "Always" button from interactive buttons, simplify reply handler. Single-shot every time. Permission body shows 2 buttons (Allow / Cancelar).

## §7 Candidate shapes — REINALDO PICKS

### Shape A — first 2 tokens
- **Pattern format**: `Bash:<verb> <subverb>` (e.g., `Bash:git status`, `Bash:git push`); fall back to `Bash:<verb>` if there's only one token; non-Bash: pattern = `<tool>:<dir>` UNCHANGED.
- **LOC**: ~5 in `permissionPattern` body. Trivial.
- **UX**: tap Always on `git status` → re-prompts on `git push`. Tap Always on `npm install` → re-prompts on `npm publish`. Generally matches user intent.
- **Failure modes**:
  - Two-token still too coarse for `curl` (different URLs are different intent but both might be "GET"). Acceptable for now — `curl` is one example, not the rule.
  - Subverb after `--` (e.g., `npm run -- test`) makes pattern weird. Edge case.
- **Recommended if**: Reinaldo wants a one-step UX improvement without changing the mental model.

### Shape B — verb + safe/unsafe classification
- **Pattern format**: `Bash:<verb>:<safe|unsafe>` (e.g., `Bash:git:safe`, `Bash:git:unsafe`); subverb classified via per-verb safe-list.
- **Safe-list (illustrative; tune at impl time)**:
  ```ts
  const SAFE_VERBS: Record<string, Set<string>> = {
    git: new Set(['status', 'diff', 'log', 'show', 'branch', 'remote']),
    npm: new Set(['list', 'view', 'info', 'outdated', 'audit']),
    bun: new Set(['list', 'pm']),
    docker: new Set(['ps', 'inspect', 'logs', 'images']),
    // ...
  }
  ```
- **LOC**: ~30 (the safe-list + the classifier).
- **UX**: tap Always on `git status` → auto-approves `git diff`, `git log`, `git branch`, but re-prompts on `git push`. Operator effectively grants "all read-only git ops" in one tap. Higher utility than A.
- **Failure modes**:
  - Maintenance burden — the safe-list is opinionated, can drift. New tools (e.g., a custom CLI) default to "everything unsafe" until added.
  - Pattern category leakage — adversarial user could find a safe verb that has destructive flags (`git diff --output=/etc/passwd`? probably not but illustrates the principle).
- **Recommended if**: Reinaldo wants more thoughtful auto-allow scoping and is OK maintaining the safe-list.

### Shape C — drop "Always" entirely
- **Pattern format**: N/A; `permissionPattern` deleted; `sessionAllowPatterns` deleted.
- **LOC**: ~50 LOC removed; ~5 LOC added (button list shrinks).
- **UX**: every permission re-prompts. Aligns with `anthropics/claude-plugins-official` telegram + imessage (neither has "Always"). Eliminates the entire scope-of-consent concern.
- **Failure modes**:
  - Operator fatigue — every `git status` re-prompts. In a long Claude session this gets annoying fast.
- **Recommended if**: Reinaldo concluded the marketplace audience values security-by-default over UX convenience, and the absence of "Always" matches what other plugins do.

### Recommendation

**Shape A** as the default — minimal change, clear UX win, maintains the "Always" ergonomics. Shape B if the maintenance burden of a small safe-list feels worth it for the marketplace polish. Shape C only if the design philosophy has shifted entirely (which would warrant its own discussion before bet).

## §8 Selected shape

Pending Reinaldo's §7 pick. Until that's recorded, §18 Bet stays open.

## §9 Appetite

- **Shape A**: 0.5 day. Single function rewrite + spike.
- **Shape B**: 1 day. Function rewrite + safe-list curation + spike + manual probe.
- **Shape C**: 0.5 day. Mostly deletion + button-list trim.

## §10 In-scope / Out-of-scope

### In-scope
- `permissionPattern()` rewrite (server.ts:809-830)
- Spike updating `sessionAllowPatterns` lookup matching the new format
- Per-shape changes in `pendingPermissions` storage if needed (probably none — value is a string either way)

### Out-of-scope
- [no-go] **Persist `sessionAllowPatterns` across plugin restarts** — separate MF (data migration concerns)
- [no-go] **User-visible pattern rendering** — closed by PR #22; we don't surface patterns in the body anymore
- [no-go] **"Allow this command for X minutes" TTL on patterns** — separate MF
- Mention of patterns in plugin.log — out of scope (operator-internal)

## §11 Rabbit holes

### R-1: Shape B safe-list curation drift
- **Category**: maintenance
- **Mitigation**: ship a v0.2.x version with a conservative safe-list (only obviously read-only ops); evolve via small PRs. The safe-list lives in source, so reviewable per-change.
- **Layer**: code review (not test)

### R-2: Existing in-memory state invalidates
- **Category**: minor user-visible change at upgrade
- **Evidence**: `sessionAllowPatterns` is `Set<string>` in-memory; plugin restart clears it. v0.2.0 launch = fresh restart = fresh set. Operator re-taps Always once after upgrade. Documented in CHANGELOG.
- **Layer**: docs

### R-3: Spike fixture covers all 3 shapes consistently
- **Category**: shape evidence
- **Evidence**: `spikes/pattern-fixture.ts` defines 12 representative request inputs and asserts the expected pattern per shape. Run with `bun run spikes/pattern-fixture.ts -- A` (or `B`/`C`). Status: spike skeleton committed; per-shape outputs filled when shape is selected.

## §12 S-DROP-G

- **Security**: addressed — narrowing auto-allow scope is purely security-positive. AuthZ unchanged.
- **Data**: N/A — in-memory only.
- **Resilience**: revert via single PR.
- **Observability**: existing `log()` line at server.ts:939 (`auto-allow ${request_id} (session pattern: ${pattern})`) keeps working with the new format.
- **Platform**: no infra change.
- **Governance**: owner = Reinaldo. Kill-the-bet: defer to v0.3 if appetite tightens; current behavior is "works but coarse," not broken.

## §13 Cutover decision

**A (narrow fits)**. Single function, no compat path, in-memory state that resets on restart.

## §14 Pair shaping

Async — Reinaldo's §7 pick is the only synchronous decision needed; can be a comment on this shape's PR or a verbal "go with A".

## §15 Breadboard

```
Claude → permission_request → handler
  │
  ├─ pattern = permissionPattern(tool_name, description, input_preview)
  │     [SHAPE A]: Bash → first 2 tokens
  │     [SHAPE B]: Bash → verb + safe/unsafe class
  │     [SHAPE C]: function deleted; pattern logic removed
  │
  ├─ if [shape ≠ C] sessionAllowPatterns.has(pattern):
  │     auto-allow
  │
  └─ else:
        send body with N buttons
        (N=3 for A/B; N=2 for C — Allow / Cancelar)
```

## §16 Build scopes

Single scope. Spec-first via spike (`spikes/pattern-fixture.ts`).

- **change_type**: `new-rule`
- **required_test_layers**: `[static, unit (fixture)]`
- **evidence_plan**: 12 representative input cases per shape; assert pattern output matches expected; lookup round-trip (`Set.add` → `Set.has`) validates exact-match semantics.

## §17 Gate verification (pre-bet)

All gates pass per any selected shape — single function, single-path, type-safe. Differs only in code volume.

## §18 Bet

- **Decision**: pending — needs Reinaldo §7 pick.
- **Approver**: Reinaldo via comment on shape PR.
- **Recommended default**: A.
- **Once §7 picked**: this shape locks the §16 build scope per the chosen shape, then `/build` runs.
