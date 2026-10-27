# Privacy Policy

Last updated: 2026-10-26

This privacy policy describes how the **`whatsapp-local` plugin for Claude Code** (the "plugin") handles data. It applies to anyone who installs and uses the plugin.

The plugin is open-source software distributed under Apache-2.0 and maintained by RIA Systems. The maintainer does **not** operate any backend that the plugin connects to. All data flow is between the user's own machine, the user's own WhatsApp Business Account, and Meta Platforms, Inc.

## TL;DR

- The plugin runs entirely on the user's machine.
- The maintainer (RIA Systems) does **not** receive, store, or have any access to messages, contacts, tokens, or any other data the plugin handles.
- The plugin talks to **Meta's WhatsApp Cloud API** using the user's own credentials. Meta's privacy practices apply to that traffic — see [Meta's WhatsApp Business Privacy Policy](https://www.whatsapp.com/legal/business-policy).
- Optional: if the user supplies a third-party API key (e.g. for a transcription provider), the plugin sends data to that provider on the user's behalf. The user's agreement with that provider applies.

## Data the plugin handles

### Stored locally

The plugin reads and writes the following on the user's machine:

| Data | Location | Purpose |
| --- | --- | --- |
| WhatsApp message history (inbound + outbound) | `~/.claude/channels/whatsapp/messages.db` (SQLite) | Lets the `chat_messages` tool surface recent context to the assistant |
| Allowlist, group policy, pairing pending entries | `~/.claude/channels/whatsapp/access.json` | Access control state |
| Cloud API credentials | The plugin's options in Claude Code (token and secrets in the system keychain), or `~/.claude/channels/whatsapp/.env` (mode 600) for installs set up before v0.2.0 | Required to talk to the Meta Cloud API |
| Inbound media (images, audio, documents, video) | `~/.claude/channels/whatsapp/media/` | So the assistant can read attached files via local file paths |
| PID lock and operational logs | `~/.claude/channels/whatsapp/plugin.pid`, `plugin.log` | Lifecycle management |

The state directory can be moved with the `WHATSAPP_STATE_DIR` environment variable. None of this data ever leaves the user's machine through any path controlled by the plugin maintainer.

### Sent to third parties

The plugin makes outbound network calls only to:

1. **Meta Graph API** (`https://graph.facebook.com/v24.0/...`). Outbound: messages, reactions, media uploads, webhook configuration. Inbound: webhook deliveries received by the plugin's local HTTP server. Authenticated with the `WHATSAPP_ACCESS_TOKEN` provided by the user. Meta's privacy practices govern this traffic.

2. **The user's chosen tunnel provider** (cloudflared, ngrok, or any other HTTPS tunnel the user configures). The tunnel forwards Meta's webhook POSTs to the plugin's local server. The tunnel provider's privacy practices govern this traffic.

The plugin does **not** make any network calls to RIA Systems infrastructure or to any other third party not explicitly configured by the user.

## What RIA Systems can see

Nothing.

There is no telemetry, no error reporting, no usage analytics, and no remote logging in this plugin. RIA Systems is the maintainer of the open-source code; it is not an operator of any service that the plugin uses.

If a user opens an issue or pull request on the [GitHub repository](https://github.com/riasistemas/claude-channel-whatsapp), the information they choose to share in that issue/PR is processed by GitHub under [GitHub's privacy policy](https://docs.github.com/en/site-policy/privacy-policies/github-general-privacy-statement).

## Data subject rights

Because all plugin data lives on the user's local machine, the user is in full control:

- **Access**: open `~/.claude/channels/whatsapp/messages.db` with any SQLite client.
- **Export**: copy the state directory.
- **Delete**: remove the state directory (`rm -rf ~/.claude/channels/whatsapp/`). The plugin recreates an empty state on next start.
- **Rectification**: edit `access.json` directly or use the `/whatsapp-local:access` skill.

For data held by Meta (the messages themselves, contact metadata, etc.), refer to [Meta's WhatsApp Business Privacy Policy](https://www.whatsapp.com/legal/business-policy) and the rights it grants under applicable law.

## Children

The plugin is a developer tool intended for adult use. It is not directed to anyone under the age of 13. Do not configure the plugin to receive messages from minors without consulting the applicable legal framework in your jurisdiction.

## Changes to this policy

Changes to this policy are tracked in the repository's git history. The "Last updated" date at the top of this file reflects the most recent change.

## Contact

For questions about this policy or about RIA Systems' role as maintainer:

- Email: ti@riasistemas.com.br
- GitHub Issues: https://github.com/riasistemas/claude-channel-whatsapp/issues
