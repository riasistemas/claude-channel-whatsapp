# Changelog

All notable changes to this plugin are documented here. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and the project
adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.7] — 2026-04-29

Metadata sync release. No behavioral changes.

### Changed

- `.claude-plugin/plugin.json` was previously stuck at `0.1.5` while
  `package.json`, the git tag `v0.1.6`, and the CHANGELOG had moved on.
  Per the [plugin manifest schema](https://code.claude.com/docs/en/plugins-reference#plugin-manifest-schema),
  if `version` is set in `plugin.json`, that value wins over the
  marketplace entry — so users on `0.1.5` could miss patch updates
  including the `0.1.6` security release. This bump realigns all
  three sources at `0.1.7`.
- Enriched `.claude-plugin/plugin.json` with the canonical metadata
  block (`$schema`, `author`, `homepage`, `repository`, `license`).
  These were already present in the marketplace catalog
  (`riasistemas/claude-plugins`) but missing from the plugin
  manifest itself, so installs that bypass the marketplace lacked
  attribution and license info.

## [0.1.6] — 2026-04-29

Security hotfix release. Closes the v0.1.6 blockers from the external
security review on 2026-04-27 (issues #1–#5, #15) plus one hardening
follow-up (#6). See PR #16 for the program shape.

### Security

- **#1 / PR #19** — `uploadMedia` no longer shells out to `curl` via
  `execSync`. Replaced with native `fetch` + `Bun.file` + `FormData`.
  Two leaks closed: command injection through shell-concatenated file
  paths, and `WHATSAPP_ACCESS_TOKEN` exposure in `argv` (visible to
  any process running `ps -ef`).
- **#2 / PR #20** — WhatsApp document filenames are now sanitized
  before write. `path.basename` strips traversal components; null
  bytes / dot-only / empty names fall back to `${mediaId}.${ext}`;
  resolved path is asserted under `realpathSync(MEDIA_DIR)`. Closes
  arbitrary FS write via `[Document: ../../.ssh/authorized_keys]`.
- **#4 / PR #21** — Outbound `reply` and `react` now call
  `assertSendablePhone(phone)` at handler entry. Accepts only
  `SELF_PHONE`, entries in `access.allowFrom`, or chats that messaged
  us within the last 600s (aligned with `lastInboundByChat`
  eviction). The previously-empty `if (!allowed && ...) {}` block is
  gone.
- **#5 / PR #21** — `reply.files` paths now pass through
  `assertSendable(f)` (canonical pattern from
  `anthropics/claude-plugins-official` telegram L131-145 / imessage
  L225-239). Blocks paths under `STATE_DIR` from being shipped as
  documents; `STATE_DIR/media` is carved out so re-forwarding inbound
  media still works.
- **#15 / PR #22** — Permission relay body no longer renders
  `input_preview` (which carried the raw `tool_input` JSON for Bash,
  including secrets in env-var assignments) or the
  `permissionPattern` literal (which captured
  `STRIPE_KEY="sk_live_..."` as the first token via whitespace
  tokenization). Body now shows `description` (Claude's prose) only,
  with a generic `🔁 *Always* = auto-approve este tipo de
  solicitação` line. A new `sanitizeSecrets` helper masks 12
  secret/PII pattern families (Stripe, GitHub, AWS, Slack, Bearer,
  Anthropic, env-var assignments, high-entropy fallback, CPF, CNPJ)
  before send. Defense-in-depth in case `description` itself names
  a secret. Fail-closed if a regex throws on adversarial input.
- **#6 / PR #23** — `activeTask` now expires 5 minutes after the
  inbound message that set it. Permission attribution (`👤 During
  conversation with X`) was previously sticky until the next `reply`
  for the same phone — a `permission_request` arriving an hour later
  was still attributed to the original conversation. After expiry,
  attribution falls back to `⚙️ Internal work`. A 60s `setInterval`
  cleans up the stale entry.

### Changed

- **#3 / PR #17** — `bun.lock` is now committed. The `start` script
  uses `bun install --frozen-lockfile --no-summary`, and CI's
  pre-existing `--frozen-lockfile` step now actually enforces (it
  was a no-op while the lockfile was gitignored). Caret-ranged deps
  (`@modelcontextprotocol/sdk ^1.0.0`, `zod ^3.23.8`,
  `@types/bun ^1.3.10`) are now pinned via the lockfile; updates
  go through `bun update` + commit.

### Notes

- Existing in-memory `sessionAllowPatterns` ("Always" approvals)
  invalidate on plugin restart due to PR #22 — same restart
  semantics as before, just one extra reload right after upgrade.
  Re-tap "🔁 Always" once and the new pattern is back.
- `actions/checkout@v4` Node 20 deprecation surfaced as a warning
  on CI runs and is tracked in #18 for a v0.2.x bump.

## [0.1.5] — 2026-04-26

### Fixed

- README still described the Quick Setup pairing flow as the default, but
  the actual default has been `allowlist` since 0.1.1. Quick Setup now
  shows `/whatsapp:access allow <phone>` as the recommended path and
  documents `pairing` as an opt-in customer-support flow.
- README's "Access control" quick reference said `Default policy is
  pairing`. Updated to `allowlist`.
- Replaced a hardcoded example phone number in a `phoneFromChatId` code
  comment with a generic `<E164>` placeholder.

### Removed

- Roadmap-style "What you don't get (yet)" section — collapsed into a
  shorter, neutral "What you don't get" list. The ambition statements
  (Embedded Signup / hosted v0.2) didn't belong in a current-version
  README.

## [0.1.4] — 2026-04-26

### Added

- **`unhandledRejection` and `uncaughtException` handlers**. The server
  now logs unhandled errors instead of dying silently, matching the
  pattern used by the telegram channel plugin.
- **`SIGHUP` shutdown handler**.
- **Orphan watchdog** that polls every 5 s for parent-PID change or a
  destroyed/ended stdin pipe and self-terminates. Catches the case where
  Claude Code dies hard without sending SIGTERM (the existing PID
  lockfile only handles new sessions; the watchdog handles long
  idle gaps where no new session arrives).

### Changed

- `plugin.json` description and keywords trimmed to match the telegram
  pattern. Marketing copy moved to the README.
- README "About" section reduced to a neutral "Maintainer" line.

## [0.1.3] — 2026-04-26

### Removed

- **Groq Whisper inbound transcription**. The plugin no longer auto-
  transcribes inbound audio. Audio messages are still downloaded and
  the local file path is forwarded to the assistant via `meta.file_path`,
  so a downstream tool or skill can transcribe if needed. Rationale:
  channel plugins (telegram, discord, imessage) do not process content
  upstream — they forward raw and let the assistant decide. Removing
  Groq aligns this plugin with that pattern, drops a third-party
  dependency, and reduces configuration surface. The `GROQ_API_KEY`
  env var is no longer recognized.

### Changed

- Audio inbound is now downloaded to `MEDIA_DIR` like other media types,
  so `meta.file_path` is populated and the assistant can `Read` it.

## [0.1.2] — 2026-04-26

### Documentation

- **`allowProspects` flag**: documented in `ACCESS.md` (escape-hatch section
  + JSON schema) and in the `/whatsapp:access` skill. Toggle with
  `/whatsapp:access set allowProspects true`. When true, unknown senders
  pass through tagged `relationship: "prospect"`, bypassing both
  `allowlist` drop and `pairing` codes. Useful for inbound sales / lead
  capture.
- **Permission relay**: documented as opt-in in `README` via
  `WHATSAPP_PERMISSION_TARGET` env var. Forwards Claude Code's permission
  prompts to a configured phone as a WhatsApp interactive message with
  three buttons (✅ Allow / 🔁 Always / ❌ Deny). Text fallback
  (`yes XXXXX` / `always XXXXX` / `no XXXXX`) also accepted. Replies
  honored only from the configured target. Validated in production
  internally before this release.

## [0.1.1] — 2026-04-26

### Added

- **Pairing flow**: `gate()` now returns a `pair` action when `dmPolicy` is
  `pairing` and an unknown sender writes in. The server generates a 6-char
  code, persists it in `access.pending` with a 1h TTL, and sends a message
  back to the sender with the code so the operator can run
  `/whatsapp:access pair <code>` to approve them.
- **`checkApprovals()`**: 5s poll over `~/.claude/channels/whatsapp/approved/`.
  When `/whatsapp:access pair` drops a marker, the server sends the new
  contact a "✅ Paired!" confirmation and removes the marker.
- **File-type dispatch in `reply` tool**: attachments are routed by
  extension. `.ogg`/`.opus` → voice note (`voice: true` forced — required
  for waveform/play-button rendering). `.jpg`/`.jpeg`/`.png`/`.webp` →
  inline image. Everything else → document.
- **Webhook receive log**: every inbound `POST /webhook` now logs
  `webhook received (N bytes)` to aid debugging.

### Changed

- **Default `dmPolicy` is now `allowlist`** instead of `pairing`. WhatsApp
  uses the sender's phone number as the ID (unlike Telegram/Discord opaque
  IDs), so operators already know who to allow. Pairing remains available
  but is opt-in to avoid the per-stranger outbound message cost.

### Fixed

- **`parseWebhookPayload`** expected `payload.body.entry` (legacy CF Worker
  envelope shape). Meta sends `payload.entry` directly, so the parser
  silently returned `[]` and no inbound messages were processed. Bug
  surfaced when migrating from R2-polling inbound to direct HTTP webhook.

## [0.1.0] — 2026-04-26

### Added

- Initial public release.
- HTTP webhook receiver (`Bun.serve` on `WHATSAPP_PORT`, default `3789`) with
  Meta verification (`GET /webhook`) and HMAC-SHA256 signature validation
  (`POST /webhook`).
- Outbound via WhatsApp Cloud API (Graph v24.0): text, images, documents,
  audio (voice notes), reactions, replies-to.
- Three MCP tools: `reply`, `react`, `chat_messages`.
- Two skills: `/whatsapp:configure` (credentials, status, lockdown guidance),
  `/whatsapp:access` (pairing, allowlist, group policy).
- SQLite-backed message history at `~/.claude/channels/whatsapp/messages.db`.
- Optional inbound audio transcription via Groq Whisper (`GROQ_API_KEY`).
- Brazilian DDD9 matching heuristics behind `WHATSAPP_PHONE_REGION=BR`.
- Configurable timezone for `local_time` annotations (`WHATSAPP_TIMEZONE`).
- PID lockfile to prevent zombie instances across session restarts.
