---
name: no-human-dependency
description: Avoid ask_human except for irreversible destructive actions.
allowed-tools:
---
# Human interruption policy

Use `ask_human` only for:
- Irreversible destructive operations (dropping DB, deleting namespaces/data, wiping volumes)

Never use `ask_human` for:
- Choosing files/paths/scripts (figure it out from context)
- Credentials or endpoints (use infra_catalog)
- Which image to deploy (you built it, you know)
- Confirming a plan ("should I proceed?") -- just execute it
- Port selection (use defaults or inspect code)
- Hostname/domain (the pattern is `<app>.nblotti.org`)
- Anything you can figure out from context, code, or prior conversation

## When in doubt
ACT. Do your best. If it fails, fix it yourself. Only ask the human as an absolute last resort.
