---
name: delete-app
allowed-tools: execute
description: Delete a Kubernetes application entirely — namespace, deployments, services, ingress, secrets, and all associated resources. Use this when the user asks to remove, delete, or tear down an application from the cluster.
---
# Delete Application from Kubernetes

## ALWAYS
- Use `kubectl` to delete resources.
- Delete the **entire namespace** when the app has its own namespace — this removes all resources at once.
- Verify deletion with `kubectl get ns <namespace>` (should return NotFound).
- If the app also has a database on the NAS, remind the user (or orchestrator) that the database still exists and ask whether to delete it too.

## NEVER
- Do NOT delete namespaces `default`, `kube-system`, `kube-public`, `ingress-nginx`, `cert-manager`, `a2a`, or `ask` — these are system namespaces.
- Do NOT ask the user for confirmation — the orchestrator already confirmed the action before dispatching you.

## Steps

### 1. Identify the application resources
```bash
kubectl get all,ingress,secret,configmap,pvc -n <namespace>
```

### 2. Delete the namespace (preferred — removes everything)
```bash
kubectl delete namespace <namespace>
```

### 3. Verify deletion
```bash
kubectl get ns <namespace>
```

### 4. Check for associated NAS resources
If the application used a database provisioned via create_database, report back that the database container on the NAS (192.168.1.2) still exists and ask whether to delete it too.

## If the app is in a shared namespace
Delete only its specific resources:
```bash
kubectl delete deployment <name> -n <namespace>
kubectl delete service <name> -n <namespace>
kubectl delete ingress <name> -n <namespace>
kubectl delete secret <name>-tls -n <namespace>
kubectl delete configmap <name>-config -n <namespace>
kubectl delete pvc -l app=<name> -n <namespace>
```
