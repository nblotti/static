---
name: inspect-app
description: Inspect, find, or check the status of an application. When looking for an app, checking if it exists, or reviewing its infrastructure, ALWAYS check both Kubernetes AND the NAS for databases.
allowed-tools: execute
---
# Inspect Application — Full Infrastructure Check

When the user asks to INSPECT, FIND, CHECK, LIST, or REVIEW an application
(e.g. "do you see X?", "is X running?", "what apps are deployed?", "show me X"),
you MUST check ALL infrastructure layers — not just Kubernetes.

## Mandatory checks (ALWAYS do both)

### 1. Kubernetes resources
```bash
kubectl get all -A | grep -i <app-name>
kubectl get ingress -A | grep -i <app-name>
```
Report: namespace, deployment status, pod status, service type/port, ingress hostname.

### 2. NAS databases (Synology 192.168.1.2)
```bash
ssh nblotti@192.168.1.2 "docker ps -a --filter name=<app-name>"
ssh nblotti@192.168.1.2 "ls /volume1/ | grep -i <app-name>"
```
Report: any database containers (running or stopped), data directories.

## Output format

Present a unified view of the application:
- **Kubernetes**: deployment, pods, services, ingress, image
- **Database (NAS)**: container name, status, port, data directory
- **Missing**: clearly state if any component was NOT found

## NEVER
- Do NOT check only Kubernetes and skip the NAS
- Do NOT assume an app has no database without checking
