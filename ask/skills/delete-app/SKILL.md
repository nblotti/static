---
name: delete-app
allowed-tools: delete_application report_facts
description: Delete an application completely — Kubernetes resources AND any associated NAS database. Use the delete_application tool which handles everything atomically with built-in user confirmation.
---
# Delete Application (Full Cleanup)

## ALWAYS

Use the `delete_application` tool. It handles EVERYTHING:
- Discovers K8s namespace + NAS database container automatically
- Shows the user exactly what was found and asks for confirmation
- Deletes K8s namespace (all resources inside it)
- Removes NAS postgres container + data directory via SSH
- Verifies cleanup

## How to use

```
delete_application(app_name="<name>")
```

- `app_name`: the application name (e.g. `td32`). Used to find the K8s
  namespace and the NAS container (`<name>-postgres`).
- `namespace`: optional, defaults to `app_name`.
- `skip_nas`: set to `True` if the app has no database on the NAS.

The tool will discover what exists, present findings to the user for
confirmation, then delete everything after approval.

## After deletion

Call `report_facts` with a summary of what was deleted (namespace, container,
data directory) so subsequent tasks are aware.

## NEVER

- Do NOT run `kubectl delete` manually — use `delete_application`.
- Do NOT run `ssh` commands manually — the tool handles NAS access.
- Do NOT delete protected namespaces: default, kube-system, kube-public,
  kube-node-lease, ingress-nginx, cert-manager, a2a, ask.
