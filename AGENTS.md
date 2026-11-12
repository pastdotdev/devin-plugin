# past.dev for Devin: working on the plugin

The past.dev plugin for Devin, in the terminal and in Devin Desktop. It runs on the developer's
machine and talks to past.dev's public Memory API. Nothing here is privileged: anyone can write the
same connector against the same endpoints with the same key. What a user reads is in `README.md`;
this file is the rules the code keeps, for whoever changes it.

## How it fits together

```
Devin (the CLI, Devin Desktop)
  hooks.json                  four hooks
  .devin-plugin/mcp.json      past_recall, served by the same script
  skills/*                    past, and four skills run through ~/.past/devin/past.mjs
            |
  bin/past-hook.mjs <mode>
            |
  session starts    start     writes the launcher; starts the sender; recalls the project's brief
  every prompt      prompt    journals the prompt; recalls; returns the block as additionalContext
  end of a turn     stop      journals the reply (last_assistant_message); starts the sender
  session ends      end       journals the end; starts the sender
  detached          send      the sender: once a minute, sends what is due of every session
  MCP server        mcp       past_recall over stdio
            |
  ~/.past/config.json             the connection, shared with every past.dev plugin
  ~/.past/devin/<session>.jsonl   one journal per session: prompts, replies, ends, timed
  ~/.past/devin/past.mjs          the launcher the skills run
  ~/.past/state.json              what was sent, under hosts.devin
  ~/.past/hosts/devin.json        when the hooks last ran, for status
            |
  POST /api/v1/recall · POST /api/v1/ingest/batch · GET /api/v1/audiences/{slug}
```

```
plugins/past/
  .devin-plugin/plugin.json   the manifest: names the MCP file; Devin finds hooks.json and skills/ itself
  .devin-plugin/mcp.json      past_recall: the script in its mcp mode
  hooks.json                  SessionStart, UserPromptSubmit, Stop, SessionEnd
  skills/                     past, connect, status, cut, ingest
  bin/past-hook.mjs           the whole plugin, one file, no dependencies
  assets/logo.svg             the logo the manifest names
  LICENSE                     the same license, for an install that copies this folder alone
.github/workflows/check.yml   every pull request: the script parses and runs, the manifests hold together
```

The plugin sits in `plugins/past/`, not at the repository's root: Devin loads an `AGENTS.md` at a
plugin's root as a rule that is always on, and this file is for contributors, not for every session.

## The rules this code keeps

1. **A hook never breaks the session.** Every mode exits 0, prints valid JSON or nothing, and
   swallows its own failures. `start` and `prompt` print
   `{"hookSpecificOutput": {"hookEventName": …, "additionalContext": …}}` when there is a recall
   block, and nothing otherwise; `stop` and `end` print nothing. A Stop hook's output can keep the
   agent working (`"decision": "block"`), so `stop` must never print. The status skill is where a
   person learns something is wrong, so a failed send is kept in `state.json` (`lastError`) for it.
2. **Only prose travels.** The journal holds three kinds of entry: the prompt as the person typed
   it (`UserPromptSubmit.prompt`), the reply (`Stop.last_assistant_message`, the turn's final
   message) and the end of the session (`SessionEnd`). Tool calls and their output, file contents,
   the model's reasoning and injected context never reach it, and past.dev's own block is dropped by
   its header. A prompt that only runs one of the plugin's skills (`/past:…`) is not conversation,
   and neither is its reply. A session whose journal records no prompt is never sent. Nothing is
   journaled before the machine is connected (no `apiKey`), with sending turned off
   (`ingest: false`), or in a folder `deny` names: `prompt` writes no entry, and `stop` and `end`
   write none without a key and only add to a journal that already exists. So a session is read from the first prompt after
   `/past:connect`, as connect says, and nothing typed before it is ever sent.
3. **A reply belongs to its prompt by `prompt_id`.** Devin gives every hook of a turn the same
   `prompt_id`. A reply with an id answers the latest prompt only when the ids match, so a second
   Stop of the same turn (a hook that kept the agent working) adds a second reply, and a reply to a
   prompt that was never recorded is left out. Without ids, a reply answers the turn still open.
   Devin's own session store is not read: Devin does not describe its format.
4. **A session is sent when it ends, or once it is quiet, and a sitting past.dev holds is sent
   again only once the session is quiet.** One detached sender sends every session; each hook starts
   it when none runs, and it looks once a minute. `dueOf` decides what goes:
   - a sitting that closed goes at once, and once only: it never changes again;
   - the sitting still open goes when the session ends, if past.dev does not hold it yet;
   - an open sitting that past.dev already holds would be a paid revision of the whole sitting, so
     it waits until the session has been quiet for its sitting length (`idleMinutes` after a turn
     that never finished). A session that is resumed and ended many times in a row (a script that
     runs `devin -p -r <session>` per step) is then sent once, not once per end;
   - a session that never ends (the terminal closed, the machine slept) goes by the same quiet rule.

   A sitting that is held back keeps, in `state.json`, the hash of what past.dev holds of it, and
   the session is marked `pending` so the sender reads its journal again.
   - One sender at a time: it holds `~/.past/devin/sender.lock` (a directory, since mkdir is
     atomic) for its life and names itself in `sender.json`. A lock whose holder is gone is taken
     over; one younger than ten seconds belongs to a sender still starting.
   - `SENDER_VERSION` is bumped whenever the sender changes: a sender that an older version
     started is replaced, because it is a long-lived process running the code it started with.
   - A session whose send failed is left for the next sender run, so a revoked key never keeps a
     process asking.
5. **Sittings.** A sitting starts at the first prompt after the session was quiet for
   `sittingMinutes` (30 by default, 5 at least), and also once the sitting so far holds
   `sittingKilobytes` of prose (50 by default, 10 at least, measured on the turns' own text). An
   agent that works for hours without a pause never goes quiet, and without the size its session
   would be one sitting, sent whole again each time it grew. The size cut waits for the agent to
   have answered, so a prompt is never parted from its first answer. A sitting also starts at a
   prompt typed under another connection (rule 14). `state.json` keeps one hash
   per sitting and pins `sittingMinutes` and `sittingKilobytes` per session at its first send, so a
   changed value never moves the boundaries of a session already in past.dev. A resumed session
   sends its last sitting again, with what was added, as a revision (rule 4 says when). The
   sittings that go together go in one `/api/v1/ingest/batch` call.
6. **The brief and the prompt's recall ride on Devin's documented context.** Devin documents
   `additionalContext` for `SessionStart` and `UserPromptSubmit`. The brief is
   `<project> recent decisions and context`, six results, 5 seconds; the prompt's recall is four
   results, 4 seconds. A prompt shorter than 12 characters, or one that runs a skill, recalls
   nothing. `promptRecall: false` turns the prompt's recall off while the brief and `past_recall`
   keep working.
7. **A recall query is redacted like the content it would send.** It crosses the network exactly
   as ingested content does, and both go through `redact()`.
8. **`state.json` changes only through `updateState`.** Hooks, the sender and commands are
   separate processes. `updateState` takes a lock (`~/.past/state.lock`, a directory, taken over
   after 5 s, skipped after 3 s so a hook never hangs), re-reads the file, applies the change and
   renames the new file into place. A file that does not parse is read again a few times and, if it
   still does not, left as it is.
9. **No dependencies.** Node 16 or later, standard library only, so installing is a clone. `fetch`
   is used where Node has it and the `http` modules where it does not. Nothing in the script is
   newer than Node 14 can parse (no `||=`), so on an older Node the hooks stay silent rather than
   fail.
10. **What a session renders does not change by a byte.** The heading and the speaker names that
    `renderSitting` prints are part of each sitting's hash, and so are `SOURCE`, `AGENT` and
    `SPEAKER` at the top of the script: a changed byte sends every sitting of a session that grows
    afterwards again, as a paid revision. `sittingId` and `sittingsOf` are fixed for a related
    reason: changed, they move the boundaries of every session that grows afterwards.
11. **`~/.past` is the person's alone.** The folder is `700` and every file in it is created `600`,
    the journals included. A plain-http `apiUrl` on another machine sends the key unencrypted:
    `connect` warns, and `status` says so on its first line.
12. **A refusal for credits pauses the calls.** After `402 out-of-credits`, sends wait 30 minutes
    and recalls 5 before past.dev is asked again, per kind of call and per key
    (`creditPauses` under `hosts.devin`). During a pause no call is made: a send carries the whole
    backlog, and a recall goes with every prompt. Nothing is lost, since what was not sent stays in
    the journals. `/past:status --refresh` and `/past:ingest` end the pause at once.
13. **The product is written past.dev.** Every sentence a person reads says past.dev. Identifiers
    keep `past`: the plugin's name, the skills (`/past:status`), the tool (`past_recall`), `~/.past`,
    the key prefix `past_sk_`, the data point ids and the recall block's header,
    `=== past · recalled from memory ===`, which the journal reader drops by that exact text.
14. **A prompt belongs to the connection it was typed under.** `prompt` journals every prompt with
    `connection`, a fingerprint of the key and the address in force (the first 16 hex characters of
    their SHA-256, never the key). A prompt typed under another connection than the sitting's starts
    a new sitting (`sittingsOf`), so a sitting is one connection's, and only the sittings of the
    connection in force are sent (`planSend`). Sittings are numbered over the whole journal, whoever
    they belong to, so an id never depends on which key reads the session. After `/past:connect`
    with another project's key, what was typed for the first project stays on the machine, and is
    sent only if its own key comes back. A prompt 0.1.0 journaled names no connection, and 0.1.0
    journaled before the machine was connected, so this version sends none of its sittings: what
    0.1.0 had sent stays as it is in past.dev, what it had not sent stays on the machine, and a
    session resumed after the update goes on in a new sitting.
15. **The folder's path stays on the machine.** The journal keeps it, for `deny` and to find the
    session of the folder a skill runs in. What is sent names the folder (`project`) and the branch
    (`gitBranch`), and no path.

## Beside other past.dev plugins

`~/.past` can be shared with another past.dev plugin on the same machine. `config.json` is then one
connection for all of them. This plugin keeps its sessions under `hosts.devin` in `state.json`, and
everything else of its own in `~/.past/devin/` and `~/.past/hosts/devin.json`. The rest of
`state.json` is never read here, and every write puts it back as it was (rule 8).

## Devin

- **The plugin is in Devin's own layout.** `.devin-plugin/plugin.json` is the manifest; Devin reads
  hooks from `hooks.json` at the plugin's root, wrapped in a `hooks` key, and refuses the whole file
  when it names an event Devin does not know, so it names Devin's events alone. Each entry carries
  an empty `matcher`, which runs it for every occurrence of an event that has no tool.
- **The plugin's folder is `${PLUGIN_ROOT}`.** Devin sets `PLUGIN_ROOT` in every hook's
  environment, where the shell expands it, and expands `${PLUGIN_ROOT}` itself in `mcp.json`'s
  `args`. It sets `DEVIN_PLUGIN_ROOT` in the environment too, but does not expand that one in
  `mcp.json`: the server then fails to list its tools (measured on 3000.11.234).
- **The project comes from `DEVIN_PROJECT_DIR`.** Devin sets it to the project's root in every
  hook's environment, and runs the hook in that folder. The input's `cwd` and the hook's own folder
  come after it, and a folder inside the plugin's own is never taken for the project.
- **A skill's command learns nothing about the plugin's folder.** So every hook writes
  `~/.past/devin/past.mjs`, a two-line launcher that imports the installed script, and the skills
  run `node "$HOME/.past/devin/past.mjs" <mode>`.
- **A plugin's skill is `/<plugin>:<skill>`**, so the skills are `/past:connect`, `/past:status`,
  `/past:cut`, `/past:ingest` and `/past:past`. The two that take a key or send data say
  `triggers: [user]`, so only a person runs them. Each skill that runs the plugin's command allows
  that one command in its `permissions` (`Exec(node "$HOME/.past/devin/past.mjs" status)`), so it
  runs without a prompt; the `past` skill lists `mcp__past__past_recall` in `allowed-tools`.
- **Devin asks before `past_recall` outside the `past` skill.** In its default mode Devin
  auto-approves its own read-only tools, not an MCP tool, whatever its `readOnlyHint`. The person
  answers **Always** once, or adds `mcp__past__past_recall` to `permissions.allow` in
  `~/.config/devin/config.json`.
- **A session id is a name** (`pickled-cook`), and `devin -r <name>` resumes it. The commands print
  it whole.
- **Plugin hooks are best effort.** Devin runs them in local sessions (the CLI and Devin Desktop),
  and a hook that fails is let through. Cloud sessions run no plugin hook.

## Contracts this depends on

- `POST /api/v1/ingest/batch`: one item per sitting. `id` gives replacement (`devin:<session id>`
  for the first sitting, `devin:<session id>:<n>` for the others), `timestamp` dates every memory
  drawn from the item, `label` names the session (`Devin · <project> · <branch>`), and `metadata`
  carries `sessionId`, `project` (the folder's name), `gitBranch`, `sitting` and `turns`. The
  folder's path is not sent (rule 15).
- `POST /api/v1/recall`: **`identity` is required**. Recall stays off until `/past:connect` sets
  one.
- `audience` on ingest: a slug; absent means the whole project. `connect --audience` checks it with
  `GET /api/v1/audiences/{slug}`.
- Devin gives every hook `hook_event_name`, `session_id` and `prompt_id` (absent before the first
  prompt), and sets `DEVIN_PROJECT_DIR` and `PLUGIN_ROOT`. `SessionStart` gives `source`
  (`startup`, `resume`) and takes `hookSpecificOutput.additionalContext`; `UserPromptSubmit` gives
  `prompt` and takes the same; `Stop` gives `stop_hook_active` and `last_assistant_message`;
  `SessionEnd` gives `reason`. A turn that ends on a refused tool call fires no `Stop`; the prompt is
  in the journal already, and `SessionEnd` still sends it. All of this was read on Devin CLI
  3000.11.234.

If any of these contracts changes, this file changes with it.

## Testing

By hand: there is no test suite. CI parses the script on Node 14, runs it on Node 16 and 22, and
checks that the manifests hold together.

For the hooks and the sender, point a throwaway home at a local stand-in for the API, so nothing
touches your real `~/.past` or a real project: `HOME=/tmp/past-test` with a `config.json` whose
`apiUrl` is `http://127.0.0.1:<port>`, a few lines of Node answering `POST /api/v1/recall` and
`POST /api/v1/ingest/batch` and logging what arrives, and each hook mode fed its stdin JSON by hand:

```bash
echo '{"session_id":"…","source":"startup"}' | DEVIN_PROJECT_DIR=… HOME=/tmp/past-test node plugins/past/bin/past-hook.mjs start
echo '{"session_id":"…","prompt_id":"p1","prompt":"…"}' | DEVIN_PROJECT_DIR=… HOME=/tmp/past-test node plugins/past/bin/past-hook.mjs prompt
echo '{"session_id":"…","prompt_id":"p1","last_assistant_message":"…"}' | HOME=/tmp/past-test node plugins/past/bin/past-hook.mjs stop
echo '{"session_id":"…","reason":"exit"}' | HOME=/tmp/past-test node plugins/past/bin/past-hook.mjs end
```

A journal entry written by hand needs the `connection` the `prompt` hook gives it (rule 14), so
let the hook write the journal and move its times back afterwards. A journal with times in the past
makes a session due at once, so `send` can be watched sending without waiting half an hour. `mcp` reads JSON-RPC lines on its input: `initialize`,
`tools/list` and `tools/call` with `past_recall`.

On a real Devin, install the plugin from the folder. Devin links its cache to the folder, so a
change takes effect at the next session; `devin plugins update past` reloads the manifest. Point the
plugin at the stand-in with `PAST_API_URL`, `PAST_API_KEY` and `PAST_IDENTITY`, which override
`~/.past/config.json` for that process alone, and drive sessions in print mode:

```bash
devin plugins install --local -y ./plugins/past
PAST_API_URL=http://127.0.0.1:<port> PAST_API_KEY=past_sk_test PAST_IDENTITY=tester@example.com \
  devin -p --respect-workspace-trust false -- "List every probe word in your context."
```

A second local plugin whose hooks append their stdin to a file shows what each hook receives.

**On a new install, and after each Devin release**, check the four things the plugin stands on:

1. Each hook's input still carries the fields above: a temporary hook that writes its stdin to a
   file shows them.
2. The reply still reaches the journal: after a turn, the session's journal holds a `reply` entry
   with the text Devin showed. If it is empty, `last_assistant_message` moved (rule 3).
3. The brief and the prompt's recall still reach the model: a local plugin whose `SessionStart`
   returns `additionalContext` "The first probe word is …" and whose `UserPromptSubmit` returns "The
   second probe word is …", then a prompt that asks the model for every probe word in its context.
4. `SessionEnd` still fires: after the session closes, its journal ends with an `end` entry.
