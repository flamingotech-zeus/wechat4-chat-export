# WeChat 4.x Chat Export

[English](README.md) | [简体中文](README.zh-CN.md)

> Export WeChat 4.x chat history to Markdown with image decryption — pure Python, no third-party reverse-engineering tools.

## What this is

This is not a chat-log viewer that requires a patched WeChat client, and it is not a wrapper around an existing reverse-engineering tool. It is a standalone implementation that opens the local WeChat 4.x database directly and writes out plain Markdown.

The reason it exists as a separate project: the well-known tools in this space have disappeared. PyWxDump was taken down under DMCA, `chatlog` has no public release, and the repositories that remain are frequently unreachable from mainland China. So we rebuilt the pipeline from the ground up using nothing but Python and the `cryptography` library.

## Why we open-sourced it

This is one of the tools we use in our own company on a daily basis. It gets used for archiving project conversations, reviewing what was agreed with a partner, and keeping personal records under our own control rather than in someone else's cloud.

It worked well enough that we stopped thinking about it, and that is usually the sign it is worth sharing. Nothing here is customer data or business logic — it is a general-purpose tool that anyone with a local WeChat install needs. So we cleaned it up and put it out.

## What it does

- **SQLCipher 4 database decryption.** WeChat 4.x keeps its messages in an encrypted SQLite database. This project includes the full decryption path.
- **Key recovery by memory scanning.** The database key is not stored in a config file; it is recovered from the running process memory, and the process is documented step by step.
- **V2 image format decryption.** Chat images use a container format that needs its own decoder before you get a normal JPEG or PNG.
- **WAL ring-buffer merging.** WeChat writes through a write-ahead log. If you read only the main database file you silently lose the most recent messages. This project merges the WAL correctly, which fixes both missing and out-of-order messages.
- **A plain-text exporter.** `export_text.py` strips images, emoji, video and voice, and writes a clean readable transcript.
- **Documented boundaries.** What works on desktop is documented alongside what genuinely does not work on an unrooted Android phone, including the official migration path for that case.

## Requirements

Windows · Python 3.9+ · the `cryptography` package

## Installation

```bash
git clone https://github.com/<your-username>/wechat4-chat-export.git
```

Then place the folder into your agent skills directory:

| OS | Path |
|---|---|
| Windows | `C:\Users\<you>\.workbuddy\skills\wechat4-chat-export\` |
| macOS / Linux | `~/.workbuddy/skills/wechat4-chat-export/` |

There is no build step. Restart the session and it takes effect.

## How to use

Ask in plain language. The trigger conditions live in the `description` field of `SKILL.md`, so the skill loads itself when the request matches.

Typical requests:

- "Export my chat history with <contact> to Markdown"
- "Save my WeChat conversation with <contact> as a file"
- "Pull the chat log and decrypt the images"

The skill walks through: locating the database → recovering the key → decrypting → merging the WAL → writing Markdown and decoding images.

## Files

```
└── SKILL.md
```

## Notes and limits

1. **WeChat must be running.** The key is recovered from process memory, so the client has to be open and the account logged in. A locked or logged-out client gives you nothing.
2. **The database format changes.** This was written against WeChat 4.x. A major version bump can move paths and change the schema. If a future version breaks it, the failure will be loud, not silent.
3. **Desktop only.** On Android without root the database is not readable from outside the app sandbox. That is an OS-level boundary, not a missing feature, and the repository explains the supported alternative.
4. **This is a personal-archive tool.** Exporting someone else's private conversation without their consent is not a use case we support.

## License

MIT
