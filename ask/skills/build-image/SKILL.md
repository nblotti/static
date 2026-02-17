---
name: build-image
description: Build and push Docker container images from the sandbox. Use this whenever you need to build a Dockerfile, push to the local registry, or create container images. ALWAYS use Docker (pre-installed). NEVER use Kaniko, buildah, or any other build tool.
---
# Build Docker Image

## ALWAYS
- Use Docker -- it is PRE-INSTALLED in the sandbox. Do NOT install it.
- Start the Docker daemon first (it is not running by default).
- Chain ALL steps (daemon start + build + push) in ONE execute call.
- Push to the insecure registry at `192.168.1.7:32000`.
- In Kubernetes manifests, reference images as `localhost:32000/<name>:<tag>` (the node containerd trusts it).

## NEVER
- Do NOT use Kaniko, buildah, podman, or any other build tool.
- Do NOT run `apt-get install docker.io` -- Docker is already installed.
- Do NOT split the daemon start and build into separate execute calls (different calls may land on different pods, losing the daemon).
- Do NOT use `localhost:32000` for `docker push` from the sandbox -- use `192.168.1.7:32000`.

## Steps

```bash
# All in ONE execute call:

# 1. Start Docker daemon
dockerd --host=unix:///var/run/docker.sock &>/tmp/dockerd.log &
sleep 3
docker info  # verify daemon is running

# 2. Build the image
docker build -t 192.168.1.7:32000/<image-name>:<tag> .

# 3. Push to registry
docker push 192.168.1.7:32000/<image-name>:<tag>

# 4. Deploy to Kubernetes (if needed)
kubectl set image deployment/<name> <container>=localhost:32000/<image-name>:<tag> -n <namespace>
kubectl rollout restart deployment/<name> -n <namespace>
kubectl rollout status deployment/<name> -n <namespace> --timeout=120s
```

## Registry details
- Sandbox to registry: `192.168.1.7:32000` (plain HTTP, insecure, pre-configured in `/etc/docker/daemon.json`)
- K8s pods to registry: `localhost:32000` (node containerd trusts it)
- List images: `curl -s http://192.168.1.7:32000/v2/_catalog | jq`
- List tags: `curl -s http://192.168.1.7:32000/v2/<image>/tags/list | jq`
