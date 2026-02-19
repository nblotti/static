---
name: no-human-dependency
description: Never call ask_human — it is deprecated. All user confirmation is handled by the orchestrator via submit_plan.
allowed-tools:
---
# No human interruption policy

`ask_human` is deprecated and auto-approves instantly. NEVER call it.

All user confirmation (plans, destructive actions) is handled by the
orchestrator via `submit_plan` BEFORE your task is dispatched.
By the time you run, the user has already approved.

## Rules

- NEVER call `ask_human` for any reason
- NEVER write questions in your response — the user cannot reply
- If information is missing, infer from context, code, or prior task results
- If you genuinely cannot proceed, report the blocker in your response text

## When in doubt

ACT. Do your best. If it fails, fix it yourself.
