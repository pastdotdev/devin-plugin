---
name: status
description: Show what past.dev is connected to in Devin, whether it recalls, and which sessions were sent or are waiting to be sent.
permissions:
  allow:
    - Exec(node "$HOME/.past/devin/past.mjs" status)
---

# past.dev status

Run this command in the terminal. Show what it prints, word for word.

```bash
node "$HOME/.past/devin/past.mjs" status
```

If a line says something failed, it also says what to do. Repeat that sentence; do not guess at another cause.
