---
name: kubernetes-ops
description: Kubernetes introspection and safety rules; prefer kubectl for live state.
allowed-tools: infra_catalog
---
# Kubernetes operations

## Hard rules
- Use kubectl for Kubernetes state (get, describe, logs).
- Do NOT search the filesystem for manifests as a substitute for kubectl.
- The glob tool must NEVER be called with root path / as it will hang for minutes.

## Safety
- Avoid broad filesystem scans.
- If a sandbox pod/runtime seems unhealthy, use reboot_pod. NEVER delete pods in the a2a namespace.

## Flow
1. infra_catalog("kubernetes") for access instructions.
2. Use narrow kubectl commands: kubectl get, kubectl describe, kubectl logs.
