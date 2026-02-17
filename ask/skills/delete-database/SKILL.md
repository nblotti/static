---
name: delete-database
allowed-tools: execute
description: Delete a database provisioned on the NAS — stop and remove the Docker container, then delete the data directory. Use this when the user asks to remove, delete, or clean up a database.
---
# Delete Database from NAS

## ALWAYS
- Connect via SSH: `ssh nblotti@192.168.1.2 '<command>'`
- SSH keys are pre-mounted at `/root/.ssh/`. No password needed.
- Stop and remove the Docker container first, then delete the data directory.
- Use a Docker Alpine container for privileged file deletion (the SSH user has no sudo).
- Verify both container removal and directory deletion.

## NEVER
- Do NOT delete containers that are NOT related to the target application.
- Do NOT ask the user for confirmation — the orchestrator already confirmed the action before dispatching you.
- Do NOT use `sudo` — it requires a password on the NAS.

## Steps

### 1. Find the database container
```bash
# List all postgres containers
ssh nblotti@192.168.1.2 'docker ps -a --filter "ancestor=postgres" --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"'

# Or search by name pattern
ssh nblotti@192.168.1.2 'docker ps -a --filter "name=<app-name>" --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"'
```

### 2. Inspect the container to find the data directory
```bash
ssh nblotti@192.168.1.2 'docker inspect <container-name> --format "{{range .Mounts}}{{.Source}} -> {{.Destination}}{{println}}{{end}}"'
```
The data directory is typically `/volume1/<app-name>-postgres/` or similar.

### 3. Stop and remove the container
```bash
ssh nblotti@192.168.1.2 'docker rm -f <container-name>'
```

### 4. Delete the data directory
Use Docker as a privileged wrapper since the SSH user cannot directly delete files owned by the postgres container:
```bash
ssh nblotti@192.168.1.2 'docker run --rm -v /volume1:/volume1 alpine rm -rf /volume1/<data-dir>'
```

### 5. Verify cleanup
```bash
# Container should be gone
ssh nblotti@192.168.1.2 'docker ps -a --filter "name=<container-name>"'

# Directory should be gone
ssh nblotti@192.168.1.2 'ls /volume1/<data-dir> 2>&1 || echo "Directory deleted"'
```

### 6. Clean up Kubernetes secrets (if any)
If the database credentials were stored as a Kubernetes secret:
```bash
kubectl delete secret <app-name>-db -n <namespace>
```
