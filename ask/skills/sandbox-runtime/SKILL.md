---
name: sandbox-runtime
description: Rules for the sandbox environment, installs, and runtime quirks.
allowed-tools:
---
# Sandbox runtime rules

## Environment
- You are running in a privileged Ubuntu container as root.
- If ANY tool is missing, install it yourself. Do NOT ask the human.

## Install rule (critical)
- Chain install and use in the same execution step whenever possible.
- Tool availability may not persist across different execution contexts or subagents.

## Daemon services
- If a service needs a daemon (e.g. Docker needs dockerd), start it yourself.
- You have full root privileges.

## Sandbox cleanup
- After completing a task, decide whether to recycle sandbox pods (reboot_pod).
- Recycle if the task created large temp files, installed packages, or modified system state.
- Keep if the task was read-only or the user might continue with current state.
