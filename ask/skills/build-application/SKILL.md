---
name: build-application
description: Build a new application from scratch. The orchestrator handles plan approval — subagents proceed directly with execution. Includes mandatory post-deploy verification.
allowed-tools: execute, create_database, build_and_push
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

## Step 6: Deploy to Kubernetes

Create K8s manifests (Deployment, Service, Ingress) and apply with kubectl.
Wire in the database connection string from Step 4 as environment variables.
Use `localhost:32000/<image>:<tag>` for the image reference.

## Step 7: Verify EVERYTHING works (MANDATORY — do NOT skip)

After deployment, run verification tests:

1. **Pod health**: all pods Running, 0 restarts
2. **Database**: tables exist, connectivity works
3. **API smoke tests**: call main endpoints (GET list, POST create, GET verify)
4. **Frontend**: returns HTML (if applicable)
5. **Ingress**: external URL responds with 200

Only report success if ALL checks pass. If any fails, fix and re-verify.
Follow the `verify-deployment` skill for the full checklist.

## NEVER
- Do NOT call ask_human — the orchestrator handles user interaction
- Do NOT ask questions in your response text — the user cannot reply
- Do NOT declare success without running verification tests
- Do NOT skip database migration/table creation before verifying the API
- Do NOT manually inspect the NAS for databases — use create_database
- Do NOT use docker build/push via execute — use build_and_push
