# past.dev for Devin

Your Devin sessions become memory, each session opens with the project's brief, and every prompt
reaches the model with what past.dev recalls for it. The full guide is the repository's
[README](https://github.com/pastdotdev/devin-plugin#readme).

## Configuration

`~/.past/config.json`, shared with every past.dev plugin on the machine:

| Key | Default | What it does |
|---|---|---|
| `apiKey` | none | the project API key, `past_sk_…`; `/past:connect` writes it |
| `identity` | none | who every memory is attributed to, and who recall reads as; recall is off without it |
| `apiUrl` | `https://api.past.dev` | a self-hosted deployment's address |
| `audience` | the whole project | the audience slug what is sent is kept to |
| `recall` | `true` | `false` turns off every recall: the brief, the prompt's and `past_recall` |
| `promptRecall` | `true` | `false` turns off the recall beside every prompt alone |
| `ingest` | `true` | `false` keeps sessions on this machine |
| `sittingMinutes` | `30` | a pause this long starts a new sitting; 5 at least |
| `sittingKilobytes` | `50` | this much conversation starts a new sitting; 10 at least |
| `idleMinutes` | `720` | how long a turn that never finished holds its session |
| `deny` | `[]` | paths whose sessions are never read, matched as substrings of the folder |

`PAST_API_KEY`, `PAST_IDENTITY` and `PAST_API_URL` override the file.

## Files

- `~/.past/devin/<session>.jsonl`: one journal per session, prompts and replies, timed.
- `~/.past/devin/past.mjs`: the launcher the skills run.
- `~/.past/state.json`: what was sent, under `hosts.devin`.
- `~/.past/hosts/devin.json`: when the hooks last ran.

Every file is readable only by you.
