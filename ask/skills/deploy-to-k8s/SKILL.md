---
name: deploy-to-k8s
allowed-tools: execute
description: Deploy, update, and manage applications on the Kubernetes cluster. Use this whenever you need to create deployments, services, ingresses, or update running workloads. Use kubectl (NOT microk8s kubectl).
---
# Deploy to Kubernetes

## ALWAYS
- Use `kubectl` -- it is pre-configured with kubeconfig at `/root/.kube/config`.
- Use `localhost:32000/<image>:<tag>` for image references in manifests (node containerd trusts it).
- After updating a deployment image, run `kubectl rollout status` to verify it succeeded.
- Use namespaces to isolate applications.
- **ALL web applications MUST use HTTPS** via Ingress with TLS. The cluster has cert-manager installed with a `letsencrypt-prod` ClusterIssuer. Always add the annotation `cert-manager.io/cluster-issuer: letsencrypt-prod` and a `tls` section to every Ingress. DNS is externally managed -- domain pattern is `<app>.nblotti.org`.

## NEVER
- Do NOT use `microk8s kubectl` -- it does not exist in the sandbox. Use plain `kubectl`.
- Do NOT use `microk8s enable` or any microk8s-specific commands.
- Do NOT assume port 80 or 443 -- check the application's actual port.
- Do NOT create Ingresses without TLS -- every web app must be HTTPS with Let's Encrypt.

## Common operations

### Create a namespace
```bash
kubectl create namespace <app-name>
```

### Apply a manifest
```bash
kubectl apply -f k8s.yaml
```

### Update a deployment image
```bash
kubectl set image deployment/<name> <container>=localhost:32000/<image>:<tag> -n <namespace>
kubectl rollout restart deployment/<name> -n <namespace>
```
Then use **early crash detection** (see below) instead of a blind rollout wait.

## Early crash detection (after deploy)

Do NOT just run `kubectl rollout status --timeout=120s` and wait blindly. Instead:

```bash
# After applying manifests, wait 15s then check pod state
sleep 15
kubectl get pods -n <namespace> -l app=<app> -o wide

# If any pod shows CrashLoopBackOff or Error, check logs immediately:
kubectl logs -n <namespace> -l app=<app> --tail=30

# Only wait for full rollout if pods are Running/ContainerCreating
kubectl rollout status deployment/<name> -n <namespace> --timeout=90s
```

This catches crash-loops in ~15s instead of waiting 2+ minutes for a timeout.
If the pod is crash-looping, read the logs, fix the issue, rebuild, and redeploy.

### Expose with Ingress + TLS (Let's Encrypt)
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: <app-name>
  namespace: <namespace>
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - <app>.nblotti.org
      secretName: <app>-tls
  rules:
    - host: <app>.nblotti.org
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: <app-name>
                port:
                  number: 80
```

### Inspect resources
```bash
kubectl get all -n <namespace>
kubectl describe deployment <name> -n <namespace>
kubectl logs deployment/<name> -n <namespace> --tail=50
```

### Exec into a running pod (to inspect files, debug, etc.)
```bash
kubectl exec -n <namespace> deploy/<name> -- ls /app
kubectl exec -n <namespace> deploy/<name> -- cat /app/config.py
```
**IMPORTANT**: `ls`, `glob`, `grep`, `read_file` tools operate on YOUR sandbox filesystem, not other pods. To see files inside another pod, always use `kubectl exec` via the `execute` tool.
