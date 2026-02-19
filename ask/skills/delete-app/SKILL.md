---
name: delete-app
allowed-tools: execute
description: Delete an application completely — Kubernetes resources AND any associated NAS database. Always check both K8s and NAS when deleting an app.
---
# Delete Application (Full Cleanup)

The orchestrator handles user confirmation via submit_plan before dispatching
this task. Proceed directly with deletion.

When deleting an application, ALWAYS check and clean up BOTH locations:
1. **Kubernetes** — namespace, deployments, services, ingress, secrets
2. **NAS (192.168.1.2)** — associated PostgreSQL database container + data directory

## NEVER
- Do NOT delete namespaces: default, kube-system, kube-public, ingress-nginx, cert-manager, a2a, ask.
- Do NOT call ask_human — it is deprecated. The user already confirmed.

## Steps

### Step 1: Identify and delete Kubernetes resources

```bash
# List everything in the namespace
kubectl get all,ingress,secret,configmap,pvc -n <namespace>

# Delete the entire namespace (removes all resources at once)
kubectl delete namespace <namespace>

# Verify deletion
kubectl get ns <namespace>
```

If the app is in a shared namespace, delete only its specific resources:
```bash
kubectl delete deployment,service,ingress,secret,configmap -l app=<name> -n <namespace>
```

### Step 2: Check NAS for associated database

ALWAYS do this — even if not explicitly asked. Applications often have a PostgreSQL database on the NAS.

```bash
# List all postgres containers, look for ones matching the app name
ssh nblotti@192.168.1.2 'docker ps -a --filter ancestor=postgres --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"'
```

Common naming patterns: `<app>-postgres`, `<app>-db`, `<app>-pg`

### Step 3: Delete NAS database (if found)

```bash
# Get the data directory from the container
ssh nblotti@192.168.1.2 'docker inspect <container-name> --format "{{range .Mounts}}{{.Source}} -> {{.Destination}}{{println}}{{end}}"'

# Remove the container
ssh nblotti@192.168.1.2 'docker rm -f <container-name>'

# Delete the data directory (use Docker for privileged file ops)
ssh nblotti@192.168.1.2 'docker run --rm -v /volume1:/volume1 alpine rm -rf /volume1/<data-dir>'

# Verify
ssh nblotti@192.168.1.2 'docker ps -a --filter name=<container-name>'
ssh nblotti@192.168.1.2 'ls /volume1/<data-dir> 2>&1 || echo "Directory deleted"'
```

### Step 4: Report summary

Report what was deleted:
- K8s namespace/resources removed: yes/no
- NAS database container removed: yes/no (name, port)
- NAS data directory removed: yes/no (path)
