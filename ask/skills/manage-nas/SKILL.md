---
name: manage-nas
allowed-tools: execute
description: Access and manage the Synology NAS at 192.168.1.2 via SSH. Use this whenever you need to list Docker containers on the NAS, manage files on /volume1, inspect databases, or perform any admin task on the NAS.
---
# Manage NAS (Synology -- 192.168.1.2)

## ALWAYS
- Connect via SSH: `ssh nblotti@192.168.1.2 '<command>'`
- SSH keys are PRE-MOUNTED at `/root/.ssh/` (id_rsa + known_hosts). No password needed.
- For privileged file operations on `/volume1/`, use Docker (the SSH user has no sudo but IS in the docker group).

## NEVER
- Do NOT ask the user for SSH credentials -- they are already available.
- Do NOT try to mount NAS paths locally (they are not NFS-mounted in the sandbox).
- Do NOT use `sudo` -- it requires a password on the NAS.

## SSH commands

### List Docker containers on NAS
```bash
ssh nblotti@192.168.1.2 'docker ps'
ssh nblotti@192.168.1.2 'docker ps -a'  # include stopped
```

### Inspect a container
```bash
ssh nblotti@192.168.1.2 'docker inspect <container-name>'
ssh nblotti@192.168.1.2 'docker logs <container-name> --tail=50'
```

### Browse NAS filesystem
```bash
ssh nblotti@192.168.1.2 'ls /volume1/'
ssh nblotti@192.168.1.2 'ls /volume1/docker/'
```

### Privileged file operations (delete, chmod, chown)
The SSH user `nblotti` cannot directly delete files owned by Docker containers (e.g., postgres data dirs). Use Docker as a privileged wrapper:

```bash
# Delete a directory
ssh nblotti@192.168.1.2 'docker run --rm -v /volume1:/volume1 alpine rm -rf /volume1/<path>'

# Change ownership
ssh nblotti@192.168.1.2 'docker run --rm -v /volume1:/volume1 alpine chown -R 1000:1000 /volume1/<path>'

# Change permissions
ssh nblotti@192.168.1.2 'docker run --rm -v /volume1:/volume1 alpine chmod -R 755 /volume1/<path>'
```

### Manage database containers
```bash
# List database containers
ssh nblotti@192.168.1.2 'docker ps --filter "ancestor=postgres"'

# Check database readiness
ssh nblotti@192.168.1.2 'docker exec <container> pg_isready -U <user>'

# Stop/start a database
ssh nblotti@192.168.1.2 'docker stop <container>'
ssh nblotti@192.168.1.2 'docker start <container>'
```

## Database provisioning
- Port range: 5437-5499
- Data directories: under `/volume1/`
- Use the `create_database` tool to provision new databases (it handles SSH, port allocation, and container creation automatically).
