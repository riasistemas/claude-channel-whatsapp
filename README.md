# WhatsApp

A Claude plugin that helps you answer WhatsApp messages. Paste a
conversation or a screenshot and Claude drafts the reply in your voice, reads
where the lead stands and suggests the next step. You send the reply yourself.

Works on claude.ai, the Claude desktop and mobile apps, Cowork and Claude
Code. No WhatsApp login, no Meta app, no tokens.

## Skills

| Skill | What it does |
| --- | --- |
| `/whatsapp:inbox` | Drafts a reply to a WhatsApp conversation you paste (text or screenshot), with a short lead read and a next step. Claude also uses it on its own when you paste a WhatsApp conversation. |
| `/whatsapp:configure` | Explains what the plugin does on each Claude app and the options to connect a number. |

Example:

> **You:** /whatsapp:inbox *(screenshot of a chat: "hi, do you have a slot
> this week? how much is the first visit?")*
>
> **Claude:** a two-sentence reply with a `[price]` placeholder and one
> question about the best day, then: *Lead: warm, wants to book, budget
> unknown. Next step: answer, then offer two time slots.*

## Install

Find "whatsapp" in the [Claude directory](https://claude.ai/directory) and
add it. A plugin added on claude.ai also loads in Claude Code through
account sync.

In Claude Code you can also install it from RIA Systems' catalog:

```
/plugin marketplace add riasistemas/claude-plugins
/plugin install whatsapp@riasistemas
```

## Connected version

This plugin doesn't read or send WhatsApp messages by itself. RIA Systems is
building a connected version where you connect your WhatsApp Business
number and Claude reads and answers your messages. To hear when it opens,
join the [waitlist](https://claude-plugins.riasistemas.com.br/en#waitlist).

## Local channel for developers

Up to v0.1.10 this plugin also ran a local MCP server that received Meta's
webhooks with your own Meta app. That channel is now a separate plugin,
`whatsapp-local`, for Claude Code:

```
/plugin marketplace add riasistemas/claude-plugins
/plugin install whatsapp-local@riasistemas
```

Its code and docs live on the [`local` branch](https://github.com/riasistemas/claude-channel-whatsapp/tree/local).
Credentials saved by earlier versions in `~/.claude/channels/whatsapp/.env`
keep working there.

## Privacy

The plugin has no server, makes no network calls and stores nothing. See
[PRIVACY.md](./PRIVACY.md).

## License

Apache-2.0.

## Maintainer

Maintained by [RIA Systems](https://github.com/riasistemas). Not affiliated
with or endorsed by Meta or Anthropic. WhatsApp is a trademark of Meta
Platforms, Inc. Issues and pull requests are welcome in the
[repository](https://github.com/riasistemas/claude-channel-whatsapp).
