---
name: build-image
description: Build and push Docker container images from the sandbox. Use the build_and_push tool — it handles daemon startup, build, smoke test, and push atomically. NEVER use Kaniko, buildah, or any other build tool.
---
# Build Docker Image

## Use the `build_and_push` tool (PREFERRED)

Call the **`build_and_push`** MCP tool instead of running raw docker commands.
It atomically handles: daemon start → build → local smoke test → push.
If the container crashes during the smoke test, the push is **blocked** and
you get the crash logs back so you can fix the issue.

```
build_and_push(
  context_path="/workspace/my-app",   # directory with Dockerfile
  image_name="my-app",                # image name (no registry prefix)
  app_port=8000,                      # port the app listens on
  session_id=<your_session_id>,
  tag="latest",                       # optional, default "latest"
)
```

The tool returns:
- `ok`: true if build + smoke test + push all succeeded
- `image`: the K8s-ready image reference (e.g. `localhost:32000/my-app:latest`)
- `output`: full build/test/push log

If `ok` is false, read the `output` for error details, fix the code, and retry.

## NEVER
- Do NOT use Kaniko, buildah, podman, or any other build tool.
- Do NOT run `apt-get install docker.io` — Docker is already installed.
- Do NOT use `localhost:32000` for `docker push` from the sandbox — use `192.168.1.7:32000`.

## Manual fallback (only if build_and_push is unavailable)

```bash
# All in ONE execute call:

# 1. Start Docker daemon
dockerd --host=unix:///var/run/docker.sock &>/tmp/dockerd.log &
sleep 3
docker info  # verify daemon is running

# 2. Build the image
docker build -t 192.168.1.7:32000/<image-name>:<tag> .

# 3. LOCAL SMOKE TEST (mandatory)
docker run -d --name smoke-test -p 9999:<APP_PORT> 192.168.1.7:32000/<image-name>:<tag>
sleep 5
docker ps --filter name=smoke-test --format '{{.Status}}'
docker logs smoke-test
# If crashed → fix and rebuild BEFORE pushing
docker rm -f smoke-test

# 4. Push to registry (only after smoke test passes)
docker push 192.168.1.7:32000/<image-name>:<tag>
```

## Registry details
- Sandbox to registry: `192.168.1.7:32000` (plain HTTP, insecure, pre-configured in `/etc/docker/daemon.json`)
- K8s pods to registry: `localhost:32000` (node containerd trusts it)
- In K8s manifests, reference images as `localhost:32000/<name>:<tag>`
- List images: `curl -s http://192.168.1.7:32000/v2/_catalog | jq`
- List tags: `curl -s http://192.168.1.7:32000/v2/<image>/tags/list | jq`
