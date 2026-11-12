---
name: ingest
description: Send one Devin session to past.dev now, without waiting for it to end. Run it only when the person asks for it.
triggers:
  - user
permissions:
  allow:
    - Exec(node "$HOME/.past/devin/past.mjs" ingest)
---

# Send a session to past.dev now

Run this command in the terminal. Give the session's id, or its first characters, when the person named one. Without an id it sends the session most recently active in this folder. Show what it prints, word for word.

```bash
node "$HOME/.past/devin/past.mjs" ingest [session]
```

If the command cannot reach the network from where it runs, ask the person to allow it, then run it again.
