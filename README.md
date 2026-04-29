# WhatsApp

Connect a WhatsApp Business number to your Claude Code via the official WhatsApp Cloud API.

The MCP server receives WhatsApp webhooks from Meta over HTTP, forwards inbound messages to your Claude Code session, and replies through the Cloud API. Built on the official Meta Graph API — uses your WABA tokens, not WhatsApp Web reverse-engineering.

## Prerequisites

- [Bun](https://bun.sh) — the MCP server runs on Bun. Install with `curl -fsSL https://bun.sh/install | bash`.
- A [Meta Developer account](https://developers.facebook.com/) with a WhatsApp Business app (free tier OK for testing).
- A WABA phone number connected to that app (Meta provides a free test number for development).
- A way to expose your local webhook publicly: [cloudflared](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/install-and-setup/installation/) (recommended), [ngrok](https://ngrok.com/), or any HTTPS tunnel.

## Quick Setup
> See [ACCESS.md](./ACCESS.md) for DM policies, groups, and multi-user setups.

**1. Create a WhatsApp Business app on Meta.**

Go to [developers.facebook.com](https://developers.facebook.com/), create a new app, choose **Business** type, and add the **WhatsApp** product. Meta gives you a test phone number and a temporary 24h access token immediately.

For production use, you'll want a **System User token** (long-lived) instead of the temporary one. From your **Business Manager → Users → System Users**, create one with `whatsapp_business_messaging` and `whatsapp_business_management` permissions, then generate a token.

You'll need four IDs from the Meta App Dashboard:

- **App Secret** — App Settings → Basic → App Secret (Show)
- **Phone Number ID** — WhatsApp → API Setup (NOT the phone number itself)
- **WhatsApp Business Account ID** — same page
- **Access Token** — System User token from above (or temp 24h one for testing)

Plus pick a **Verify Token** of your own — any random string. You'll paste the same value into Meta when configuring the webhook URL.

```sh
# generate one with:
openssl rand -hex 32
```

**2. Install the plugin.**

These are Claude Code commands — run `claude` to start a session first.

```
/plugin marketplace add github://riasistemas/claude-channel-whatsapp
/plugin install whatsapp@riasistemas
/reload-plugins
```

**3. Save credentials.**

```
/whatsapp:configure WHATSAPP_ACCESS_TOKEN=<token> WHATSAPP_PHONE_NUMBER_ID=<id> WHATSAPP_WABA_ID=<id> WHATSAPP_VERIFY_TOKEN=<random> WHATSAPP_APP_SECRET=<secret>
```

Writes to `~/.claude/channels/whatsapp/.env` (mode 600). You can also write that file by hand or set the variables in your shell — shell takes precedence.

**4. Expose the local webhook over HTTPS.**

In a separate terminal:

```sh
cloudflared tunnel --url http://localhost:3789
```

Cloudflared prints a public URL like `https://random-words.trycloudflare.com`. Copy it.

> The free anonymous tunnel URL changes every restart. For a stable setup, run `cloudflared tunnel login` once and create a [named tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/get-started/create-remote-tunnel/), or use `ngrok http 3789` with a reserved domain.

**5. Configure the webhook in Meta.**

In your Meta App Dashboard → **WhatsApp → Configuration → Webhook → Edit**:

- **Callback URL**: `<your-tunnel-url>/webhook`
- **Verify token**: same value you set as `WHATSAPP_VERIFY_TOKEN`

Click **Verify and save** — Meta hits your local server with a verification request, your plugin echoes back the challenge, and the webhook is active.

Then click **Manage** under **Webhook fields** and subscribe to at least `messages`.

**6. Relaunch Claude Code with the channel flag.**

The server only starts when this flag is present. Exit your session and start a new one:

```sh
claude --channels plugin:whatsapp@riasistemas
```

**7. Allow your number.**

The default `dmPolicy` is `allowlist` — strangers are dropped silently. Add the phone(s) that should reach you:

```
/whatsapp:access allow <phone>
```

`<phone>` is digits-only, E.164, no `+` (e.g. `15551234567`). Send a WhatsApp message from that number to the WABA and it reaches your Claude Code session.

If you'd rather have senders self-onboard via a code instead of pre-adding them (e.g. customer support), flip to `pairing`:

```
/whatsapp:access policy pairing
```

In `pairing` mode, an unknown sender gets a 6-character code reply; you approve it with `/whatsapp:access pair <code>`. Each pairing reply costs one outbound message inside the WhatsApp 24-hour customer-service window.

## Access control

See **[ACCESS.md](./ACCESS.md)** for DM policies, groups, mention detection, delivery config, skill commands, and the `access.json` schema.

Quick reference: phones are stored as **digits-only E.164** (no `+`). Default policy is `allowlist`. Set `WHATSAPP_PHONE_REGION=BR` to handle Brazilian DDD9 matching automatically.

## Tools exposed to the assistant

| Tool | Purpose |
| --- | --- |
| `reply` | Send to a chat. Takes `chat_id` + `text`, optionally `files` (absolute paths) for attachments. Files are dispatched by extension: `.jpg`/`.jpeg`/`.png`/`.webp` → inline image; `.ogg`/`.opus` → **voice note** (forced `voice:true` — must be OGG/Opus, MP3 won't render as voice); everything else (`.pdf`, `.docx`, `.mp4`, etc.) → document. Auto-chunks long text at 4000 chars. |
| `react` | Add an emoji reaction to a message by ID. Any single emoji works. |
| `chat_messages` | Read recent message history for a chat (or all chats), straight from the local SQLite. |

### Permission relay

When `WHATSAPP_PERMISSION_TARGET` is set, the channel exposes the `claude/channel/permission` capability. Claude Code's permission prompts (when a tool needs approval to run) are forwarded to the configured phone as a WhatsApp message with three interactive buttons:

- **✅ Allow** — approve once
- **🔁 Always** — approve and auto-approve future requests matching the same pattern for the rest of the session
- **❌ Deny** — reject

Text replies also work (`yes XXXXX` / `always XXXXX` / `no XXXXX` where `XXXXX` is the 5-char request ID shown in the message). Replies are honored **only** from the configured target — other allowlisted contacts cannot answer permission prompts.

## Environment variables

### Required

| Var | What it is |
| --- | --- |
| `WHATSAPP_ACCESS_TOKEN` | Long-lived System User token with `whatsapp_business_messaging` scope |
| `WHATSAPP_PHONE_NUMBER_ID` | The phone number ID from Meta App Dashboard |
| `WHATSAPP_WABA_ID` | WhatsApp Business Account ID |
| `WHATSAPP_VERIFY_TOKEN` | Random string you choose; paste same value into Meta's webhook config |
| `WHATSAPP_APP_SECRET` | Meta App Secret used for HMAC-SHA256 signature verification |

### Optional

| Var | Default | Purpose |
| --- | --- | --- |
| `WHATSAPP_PORT` | `3789` | Local HTTP port for webhook receiver |
| `WHATSAPP_SELF_PHONE` | (empty) | Your WABA phone — used to filter outbound echoes |
| `WHATSAPP_PERMISSION_TARGET` | (empty) | Phone allowed to answer Claude's permission prompts |
| `WHATSAPP_PHONE_REGION` | `intl` | Set to `BR` for Brazilian DDD9 matching |
| `WHATSAPP_TIMEZONE` | `UTC` | IANA timezone for `local_time` in notifications |
| `WHATSAPP_STATE_DIR` | `~/.claude/channels/whatsapp` | Override state directory |
| `WHATSAPP_GRAPH_API_VERSION` | `v24.0` | Graph API version |

## What you don't get

- **Templates** — sending pre-approved message templates outside the 24-hour customer-service window. The Cloud API supports them; this plugin currently doesn't expose a tool for it. Workaround: call the Graph API directly from a script.
- **Buffered inbound when offline** — Meta retries webhook deliveries for up to 24 hours, so short outages recover automatically. Longer outages drop messages. If you need indefinite buffering, put a webhook receiver of your own (CF Worker, Lambda, etc.) in front of the plugin.

## Limitations

- **24-hour customer-service window**: Meta restricts free-form messages to the 24 hours after the user's last inbound. Outside that window you can only send pre-approved templates.
- **Rate limits**: Meta enforces per-second and per-day limits depending on your messaging tier (free test number caps at 250 unique recipients/24h).
- **HTTPS required**: Meta delivers webhooks only to valid HTTPS URLs. Localhost direct doesn't work — you need a tunnel.
- **One plugin instance per machine**: the plugin uses a PID lock at `~/.claude/channels/whatsapp/plugin.pid` to kill stale instances. Running multiple Claude sessions with this plugin on the same state directory will not work as expected.

## Development

Dependencies are pinned via `bun.lock`, which is committed. Plugin start runs `bun install --frozen-lockfile`, so an out-of-date lockfile fails fast instead of silently pulling new versions.

To update a dependency:

```sh
bun update <package>      # bumps to latest within caret range
# or edit package.json by hand, then:
bun install               # regenerates bun.lock
```

Then commit both `package.json` and `bun.lock` together. CI runs `bun install --frozen-lockfile` and rejects PRs with lockfile drift.

## Privacy

All plugin data lives on the user's local machine. The maintainer does not operate any backend that this plugin talks to. See [PRIVACY.md](./PRIVACY.md) for the full policy.

## License

Apache-2.0.

## Maintainer

Maintained by [RIA Systems](https://github.com/riasistemas), a Meta Tech Provider. Issues and PRs welcome at the [standalone repo](https://github.com/riasistemas/claude-channel-whatsapp).
