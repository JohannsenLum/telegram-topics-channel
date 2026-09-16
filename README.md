# telegram-topics-channel

A Telegram channel plugin for Claude Code — messaging bridge with built-in access control, pairing, allowlists, group support, **and native Telegram forum-topic scoping with automatic per-topic reply routing.**

This is a fork of Anthropic's Apache-2.0 [`telegram`](https://github.com/anthropics/claude-plugins-official) channel plugin (`telegram@0.0.7`). See [What's different from the official plugin](#whats-different-from-the-official-plugin) below.

## What it does

The MCP server logs into Telegram as a bot and gives Claude tools to reply, react, and edit messages. When someone messages the bot, the server forwards the message into your Claude Code session. Everything from the upstream plugin — pairing, allowlists, DM policy, groups, mention-triggering, attachments, permission-relay buttons — works exactly as documented in [ACCESS.md](./ACCESS.md).

On top of that, this fork understands **Telegram forum topics**: a supergroup with the Topics feature turned on, where each topic is its own thread. You can either pin the bot to one topic, or let it operate across a whole multi-topic group and have replies land back in the right thread automatically.

## What's different from the official plugin

**Change statement (Apache License 2.0 §4(b)):** `server.ts` is modified from Anthropic's original. All additions are tagged `TOPIC-CHANNEL:` in the source, and the file carries a matching header comment. Four areas changed:

1. **`GroupPolicy.topicId`** — an optional field on a group's access-control entry. When set, only messages from that one forum topic thread are delivered; everything else in the group (including the General topic) is dropped at the gate.
2. **Inbound meta surfaces `message_thread_id`** — the `<channel>` block Claude sees now includes which topic a message came from, when the group has topics enabled.
3. **The `reply` tool accepts `message_thread_id`** — pass it explicitly to reply into a specific topic thread (works for text and for file/photo sends).
4. **Auto-reply routing (the new part, not in upstream, not in the original patch this was built from either):** if you don't pass `message_thread_id` and the group has no fixed `topicId` configured, `reply` now defaults to the topic the *triggering* inbound message came from. The server tracks the last delivered message's thread per chat in memory and uses it as the final fallback. Net effect: run one bot across an entire multi-topic group, and every reply lands back in the topic that prompted it — zero per-message effort, no risk of Claude accidentally replying into the wrong topic or General. Resolution order is always: explicit `message_thread_id` arg > configured `topicId` > topic of the triggering message.

Everything else — pairing flow, allowlists, `dmPolicy`, attachments, permission-relay, chunking — is unmodified upstream behavior.

## Install

**1. Add the marketplace and install.**

```
/plugin marketplace add JohannsenLum/telegram-topics-channel
/plugin install telegram-topics-channel@telegram-topics-channel
/reload-plugins
```

**2. Create a bot with [@BotFather](https://t.me/BotFather)** (if you don't already have one) and grab the token — `123456789:AAHfiqksKZ8...`.

**3. Give the server the token.**

```
/telegram-topics-channel:configure 123456789:AAHfiqksKZ8...
```

**4. Relaunch with the channel flag** — the server won't connect without this:

```sh
claude --channels plugin:telegram-topics-channel@telegram-topics-channel
```

**5. Pair.** DM your bot on Telegram — it replies with a 6-character code:

```
/telegram-topics-channel:access pair <code>
```

**6. Lock it down.** Once everyone who should reach you is paired, switch off open pairing:

```
/telegram-topics-channel:access policy allowlist
```

Full access-control reference (DM policies, groups, mention detection, delivery config, the `access.json` schema): [ACCESS.md](./ACCESS.md).

## Configuring topic scoping

Groups live in `access.json` under `groups.<groupId>`. Two modes:

**Pin to one topic** — add `topicId`:

```jsonc
"groups": {
  "-1001654782309": {
    "requireMention": true,
    "allowFrom": [],
    "topicId": "42"
  }
}
```

Only messages from forum thread `42` are delivered. Everything else in the group, including General, is dropped.

**Whole-group mode (default improvement)** — leave `topicId` unset. Every topic in the group is delivered (still subject to `requireMention`/`allowFrom` as normal), and `reply` automatically routes each response back into the topic its triggering message came from. This is the mode most people want for a single bot covering a multi-topic group.

There's no `/telegram-topics-channel:access` CLI flag for `topicId` yet — edit `access.json` directly. It's re-read on every inbound message, so no restart is needed.

### Finding a topic's id

Open the topic in Telegram Desktop or the web client — the URL is `https://t.me/c/<chat>/<topicId>`. Or leave the group in whole-group mode, send a message in the topic you care about, and check the `message_thread_id` Claude reports seeing in the inbound `<channel>` meta.

## Tools exposed to the assistant

| Tool | Purpose |
| --- | --- |
| `reply` | Send to a chat. Takes `chat_id` + `text`, optionally `reply_to` (message ID) for native threading, `message_thread_id` (see above) for topic routing, and `files` (absolute paths) for attachments. |
| `react` | Add an emoji reaction to a message by ID (Telegram's fixed whitelist only). |
| `edit_message` | Edit a message the bot previously sent. |
| `download_attachment` | Download a Telegram file attachment to the local inbox. |

## Prerequisites

- [Bun](https://bun.sh) — the MCP server runs on Bun. Install with `curl -fsSL https://bun.sh/install | bash`.

## License and attribution

Licensed under the [Apache License, Version 2.0](./LICENSE) (unmodified, carries the original "Copyright 2026 Anthropic, PBC" notice). See [NOTICE](./NOTICE) for the required attribution statement.

This plugin is a derivative work of Anthropic's `telegram` channel plugin (https://github.com/anthropics/claude-plugins-official), used and modified under the terms of the Apache License 2.0. It is an independent, unofficial fork — not affiliated with or endorsed by Anthropic, PBC.
