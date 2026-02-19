---
name: build-application
description: Build a new application from scratch. The orchestrator handles plan approval — subagents proceed directly with execution. Includes mandatory post-deploy verification.
allowed-tools: execute, create_database, build_and_push, report_facts
---
# Build Application — Full Lifecycle

When dispatched to BUILD or SCAFFOLD a new application, follow this workflow.
The orchestrator has already obtained user approval for the plan.

## Step 1: Check for approved plan context

Look at the top of your prompt for an `--- APPROVED PLAN ---` section.
If present, the user has already approved the plan — proceed directly.
If absent, infer the plan from your task description and proceed.

**Do NOT call ask_human.** The orchestrator handles all user interaction.

## Step 2: Gather requirements (internal — no tool calls)

From your task prompt, determine:
- App name, URL, features
- Stack: backend, frontend, database
- Container port

## Step 3: Scaffold the code

Create all necessary source files in `/workspace/<app-name>/`:
- Application code (backend + frontend)
- Dockerfile
- Requirements/dependencies file
- Database migration or init logic

Use `write_file` to generate each file. Choose sensible defaults for
anything not specified in the task prompt.

## Step 4: Provision database (if needed)

If the app needs a database, call `create_database` with the app name.
This provisions a dedicated PostgreSQL container on the NAS.
Capture the returned connection string for use in deployment.

**Do NOT** inspect the NAS manually for existing databases.
**Always** use `create_database` — it handles deduplication internally.

## Step 5: Build and push Docker image

Use `build_and_push` with:
- `context_path`: the workspace directory (e.g., `/workspace/todo-app`)
- `image_name`: the app name (e.g., `todo-app`)
- `app_port`: the port the app listens on

If the build fails, read the error, fix the code, and retry.

## Step 7.5: Report facts (MANDATORY after Steps 4-7)

After completing your work, call `report_facts` with a summary of ALL
resources created or discovered. Include everything subsequent tasks
might need:

- Database: host, port, database name, username, password, connection string
- Image: name, tag, registry path
- Deployment: namespace, deployment name, service name
- Ingress: hostname, URL
- App: port the app listens on

Example:
```
report_facts("Created PostgreSQL DB at 192.168.1.2:5438, db=todo, user=todo, pass=abc123. Built image todo-app:latest pushed to localhost:32000. Deployed to namespace todo with ingress at todo.nblotti.org.")
```

## Step 6: Deploy to Kubernetes

Create K8s manifests (Deployment, Service, Ingress) and apply with kubectl.
Wire in the database connection string from Step 4 as environment variables.
Use `localhost:32000/<image>:<tag>` for the image reference.

## Step 7: Verify ALL layers (MANDATORY — do NOT skip)

You MUST test every layer of the application before declaring success.
Follow the `verify-deployment` skill for the full checklist.

### 7a. Database layer
- Connect to the database and verify tables exist
- Run a simple query (e.g., SELECT 1, list tables)
- If tables are missing, run migrations BEFORE continuing

### 7b. REST API layer
- Test the main endpoints via the internal service (curl from sandbox)
- At minimum: GET list, POST create, GET verify the created item
- All must return 2xx — a 500 means the app is broken, fix it

### 7c. Frontend layer
- Verify the frontend returns HTML with expected content
- If an ingress is configured, you MUST test through the **external URL**
  (e.g., `curl -sf -k https://<app>.nblotti.org/`), NOT only via
  localhost or cluster-internal service
- The frontend must be reachable from outside the cluster — this is
  the user's entry point and the final proof that everything works
- If the ingress test fails (DNS, TLS, 502, 404), fix it before
  declaring success

## NEVER
- Do NOT call ask_human — the orchestrator handles user interaction
- Do NOT ask questions in your response text — the user cannot reply
- Do NOT declare success without running verification tests
- Do NOT skip database migration/table creation before verifying the API
- Do NOT manually inspect the NAS for databases — use create_database
- Do NOT use docker build/push via execute — use build_and_push
