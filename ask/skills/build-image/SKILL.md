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

# 3. LOCAL SMOKE TEST (mandatory — see below)
docker run -d --name smoke-test -p 9999:<APP_PORT> 192.168.1.7:32000/<image-name>:<tag>
sleep 5
docker ps --filter name=smoke-test --format '{{.Status}}'
docker logs smoke-test
# If crashed → fix and rebuild BEFORE pushing
docker rm -f smoke-test

# 4. Push to registry (only after smoke test passes)
docker push 192.168.1.7:32000/<image-name>:<tag>
```

## Validate before push (MANDATORY)

After building, ALWAYS test the image locally before pushing:

```bash
# Run the container locally
docker run -d --name smoke-test -p 9999:<APP_PORT> <image>
sleep 5

# Check if still running (exit code 0 = running, 1 = crashed)
docker ps --filter name=smoke-test --format '{{.Status}}'

# Check logs for errors
docker logs smoke-test

# If it is running, optionally verify with curl:
curl -s http://localhost:9999/ || true

# Clean up
docker rm -f smoke-test
```

If the container crashes or logs show import errors / exceptions:
- Fix the issue (e.g. add missing dependency to requirements.txt)
- Rebuild the image
- Re-run the smoke test
- Only push after the local test passes

**This catches missing dependencies, wrong ports, and config errors in seconds
instead of waiting 2-3 minutes for a K8s rollout timeout.**

## Registry details
- Sandbox to registry: `192.168.1.7:32000` (plain HTTP, insecure, pre-configured in `/etc/docker/daemon.json`)
- K8s pods to registry: `localhost:32000` (node containerd trusts it)
- List images: `curl -s http://192.168.1.7:32000/v2/_catalog | jq`
- List tags: `curl -s http://192.168.1.7:32000/v2/<image>/tags/list | jq`
