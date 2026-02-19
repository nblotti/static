---
name: delete-database
allowed-tools: delete_application execute
description: Delete a database provisioned on the NAS — stop and remove the Docker container, then delete the data directory. Use this when the user asks to remove, delete, or clean up a database.
---
# Delete Database from NAS

## Preferred: use delete_application

If the database belongs to an application, use `delete_application` instead —
it handles both K8s and NAS cleanup atomically with built-in user confirmation:

```
delete_application(app_name="<name>")
```

## Manual fallback (database-only, no K8s app)

Only use these steps if the database is standalone (no associated K8s app) and
`delete_application` is not appropriate.

### ALWAYS
- Connect via SSH: `ssh nblotti@192.168.1.2 '<command>'`
- SSH keys are pre-mounted at `/root/.ssh/`. No password needed.
- Stop and remove the Docker container first, then delete the data directory.
- Use a Docker Alpine container for privileged file deletion (the SSH user has no sudo).
- Verify both container removal and directory deletion.

### NEVER
- Do NOT delete containers that are NOT related to the target application.
- Do NOT use `sudo` — it requires a password on the NAS.
- Do NOT use `root@` — always use `nblotti@192.168.1.2`.

### Steps

1. Find: `ssh nblotti@192.168.1.2 'docker ps -a --filter "name=<app-name>"'`
2. Inspect: `ssh nblotti@192.168.1.2 'docker inspect <container> --format "{{range .Mounts}}{{.Source}}{{println}}{{end}}"'`
3. Remove: `ssh nblotti@192.168.1.2 'docker rm -f <container>'`
4. Delete data: `ssh nblotti@192.168.1.2 'docker run --rm -v /volume1:/volume1 alpine rm -rf /volume1/<data-dir>'`
5. Verify both are gone.
