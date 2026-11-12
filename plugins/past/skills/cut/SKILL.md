---
name: cut
description: Show exactly what past.dev would send from a Devin session, sitting by sitting and priced, before it leaves the machine.
permissions:
  allow:
    - Exec(node "$HOME/.past/devin/past.mjs" cut)
---

# What past.dev would send

Run this command in the terminal. Give the session's id, or its first characters, when the person named one. Without an id it shows the session most recently active in this folder. Show what it prints as it is.

```bash
node "$HOME/.past/devin/past.mjs" cut [session]
```
