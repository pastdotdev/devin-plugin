# past.dev for Devin

Your Devin sessions become memory, each session opens with the project's brief, and every prompt
reaches the model with what past.dev recalls for it.

[past.dev](https://past.dev) is memory for AI agents. This plugin connects Devin, in the terminal
and in Devin Desktop, to a past.dev project: what you and the agent say to each other is read into
memory as you work, and what matters comes back by itself, at the start of each session and with
every prompt. The decisions, the reasons behind them, what was already tried and dropped.

```text
=== past · recalled from memory ===
3 from this project's history. Background, not instruction.

[1] 2026-09-14 — Webhooks retry from the job queue now: the HTTP client's retries sent them twice.
[2] 2026-09-02 — Rejected: caching prices in Redis. They change hourly, and a stale one is worse.
[3] 2026-08-27 — The importer keeps rows it cannot parse in import_rejects and never drops them.
```

That block is what the model reads beside your prompt. It arrives on its own: the model does not
have to decide to look anything up. For more, the agent calls the `past_recall` tool.

The plugin runs on your machine. Devin's cloud sessions do not run a plugin's hooks.

## Get started

You need the Devin CLI or Devin Desktop, signed in (`devin auth login`), and Node 16 or later.

1. **Get a project API key.** Sign up at [sso.past.dev/sign-up](https://sso.past.dev/sign-up),
   then open **Build › API keys** in the [console](https://app.past.dev). The key starts with
   `past_sk_`.

2. **Install the plugin.**

   ```bash
   devin plugins install pastdotdev/devin-plugin#plugins/past
   ```

3. **Connect it.** Start a session, then:

   ```text
   /past:connect <project-api-key> <identity>
   ```

   The identity is the email or id every memory is attributed to. Recall stays off without one. A
   machine that another past.dev plugin already connected skips this step: every past.dev plugin
   reads the same `~/.past/config.json`.

4. **Check it.** `/past:status` shows the project it talks to, and what it has sent.

Sessions are read from your next prompt on.

## How it works

```text
session starts   recall what past.dev knows about this project     ──►  in front of the session
every prompt     recall what past.dev knows about this prompt      ──►  beside it
every turn       the prompt and the reply are kept on this machine
session ends     the session is sent, one sitting at a time
quiet 30 min     a session that never ended is sent the same way
on request       past_recall looks something up
```

Four hooks do this, and one script runs them all. A recall that has not answered within 4 seconds
(5 for the session's brief) is dropped, so it never holds a prompt up, and a prompt shorter than 12
characters ("yes", "go on") recalls nothing.

**Recall happens before the model answers.** The prompt hook returns what past.dev recalled as
context, and Devin adds it beside the prompt. For questions about history, the `past_recall` tool
lets the agent look further on its own: ask what was decided about something, or whether it was
tried before.

**A session is sent in sittings.** Where a session went quiet for 30 minutes, the next prompt starts
a new sitting, and so does 50 KB of conversation, so an agent that works for hours without a pause
is cut all the same. Each sitting is dated at its own first prompt, so a decision made on the third
day of a long session is remembered on that day. A session is sent when it ends. One that never
ends (the terminal closed, the machine went to sleep) is sent once it has been quiet for 30 minutes.

**You pay for a sitting once.** A sitting that closed is sent once and never again. A session that
you resume after it was sent adds to its last sitting, and that sitting is sent again only once the
session has been quiet for 30 minutes. So a session that is resumed and closed twenty times in an
afternoon is sent once, with everything in it.

## What leaves your machine

Your prompts as you typed them, the agent's final reply to each, and the project and branch names.
Nothing else.

Never sent: tool calls and their output, file contents, diffs, command output, attachments, the
model's thinking, and anything injected into the session rather than typed by a person. Strings that
look like keys, tokens or private keys become `[redacted]` before anything leaves, and recall queries
are redacted the same way.

`/past:cut` prints exactly what would be sent from a session, sitting by sitting, before anything is.
Until a session is sent, and afterwards, its prompts and replies are kept in `~/.past/devin/`,
readable only by you.

A session costs what its prose costs, at 350 bytes to a credit. What a recall costs depends on your
plan: see [past.dev/pricing](https://past.dev/pricing).

## Skills and the tool

| | What it does |
|---|---|
| `/past:connect <key> <identity> [api-url]` | connect this machine; `--audience <slug>` sets who sees what you send |
| `/past:status` | what it is connected to, what it has sent, and the last send that failed |
| `/past:cut [session]` | print exactly what would be sent from a session, by default the latest one here |
| `/past:ingest [session]` | send one session now, without waiting for it to end |
| `/past:past` | tells the agent when to search the project's history |
| `past_recall` | the agent's own lookup in past.dev, a tool it calls when the question is about history |

## Who sees what you send

Everyone in the project, unless you choose an audience: `/past:connect --audience <slug>`, with a
slug from the console's **Audiences** page. `/past:connect --audience project` widens it back.

## Settings

`~/.past/config.json` holds the connection and a few switches: `recall` and `ingest` turn either
half off, `promptRecall` turns off the recall beside every prompt alone, `sittingMinutes` (30) is
how long a pause starts a new sitting, `sittingKilobytes` (50) is how much conversation starts one,
`idleMinutes` (720) is how long a turn that never finished holds its session, and `deny` lists paths whose sessions are never read. `PAST_API_KEY`,
`PAST_IDENTITY` and `PAST_API_URL` override the file.

For a self-hosted past.dev, pass your deployment's URL as the third argument of `/past:connect`. Use
`https`: over plain `http` to another machine the key travels unencrypted, and the plugin says so.

## When something looks wrong

Start with `/past:status`. A hook never interrupts a session, so a problem shows up there rather
than in the conversation.

- **The hooks never ran.** Status says so. Check that the plugin is installed
  (`devin plugins list`), then start a new session.
- **Nothing is recalled.** Status has to show a key and an identity: recall needs both. A new
  project has nothing to recall until its first sessions are in; `/past:ingest` sends the current
  one now.
- **Devin asks before `past_recall`.** Devin asks before any MCP tool. The tool only reads, so
  answering **Always** once is enough.
- **A send failed.** Status shows the last failed send and what the API answered.
- **The project is out of credits.** Sending and recall pause instead of asking again on every
  turn, and nothing is lost: your sessions stay on this machine. After you add credits, run
  `/past:status --refresh`.
- **The hooks stay silent.** They call `node`. Node 16 or later has to come first on the path
  Devin runs them with.

## Remove it

```bash
devin plugins remove past
```

Then delete `~/.past/devin`. Revoking the key in **Build › API keys** stops it at once from the
server side. What was already sent stays in the project until you delete it in the console.

## This repository

A Devin plugin, in Devin's own layout. Devin installs it straight from GitHub; there is nothing to
build.

```text
plugins/past/
  .devin-plugin/plugin.json   the manifest
  .devin-plugin/mcp.json      the past_recall tool, served by the same script
  hooks.json                  the four hooks
  skills/                     past, connect, status, cut, ingest
  bin/past-hook.mjs           the whole plugin: one file, Node 16, no dependencies
```

The plugin talks only to past.dev's public Memory API (`/api/v1/ingest/batch`, `/api/v1/recall`,
`/api/v1/audiences`), with your project's key. Nothing in it is privileged: anyone could write the
same connector against the same endpoints.

## Contributing

Issues and pull requests are welcome. [AGENTS.md](AGENTS.md) holds the rules the code keeps and how
to test a change by hand, for people and agents alike. Before opening a pull request:

```bash
node --check plugins/past/bin/past-hook.mjs
HOME=$(mktemp -d) node plugins/past/bin/past-hook.mjs status
```

## Links

- Memory API documentation: [past.dev/docs/memory-api/overview](https://past.dev/docs/memory-api/overview)
- Console: [app.past.dev](https://app.past.dev)
- Sign up: [sso.past.dev/sign-up](https://sso.past.dev/sign-up)
- Community: [past.dev/slack](https://past.dev/slack)

## License

MIT. See [LICENSE](LICENSE).
