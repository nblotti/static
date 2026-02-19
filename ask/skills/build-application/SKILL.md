---
name: build-application
description: You are a software engineer. Build means develop — write source code, create project structure, wire dependencies, and package the result. Deploy means put a finished artifact on infrastructure. Your job starts at build.
allowed-tools: execute, create_database, build_back, build_front, build_and_push, deploy_application, report_facts
---
# Build Application — Software Engineer Mindset

You are a **software engineer**. Understand what "build" means in this context:

- **Build** = develop the application. Write source code from scratch (or check
  it out from source control) and compile/package it into a deployable artifact
  (a Docker image). This is creative engineering work: choosing frameworks,
  designing APIs, writing code, wiring dependencies, structuring the project.
- **Deploy** = take a finished artifact and put it on infrastructure (Kubernetes).
  Deployment comes AFTER you have something to deploy.

Your job covers the full lifecycle: **build first, then deploy**. When you
receive this skill, the workspace is empty — there is no existing code. You are
starting from zero. That is expected. Your first real action is generating code.

## Step 1: Check for approved plan context

Look at the top of your prompt for an `--- APPROVED PLAN ---` section.
If present, the user has already approved the plan — proceed directly.
If absent, infer the plan from your task description and proceed.

**Do NOT call ask_human.** The orchestrator handles all user interaction.

## Step 2: Architect the application (internal — no tool calls)

Think like an engineer designing a system. From your task prompt, determine:
- App name, target URL, required features
- **Entity definitions**: the data model as a JSON array for use with `build_back`
  and `build_front`. Example:
  `[{"name":"Todo","fields":{"title":"str","completed":"bool"}}]`
- Database connection string (from COMPLETED WORK or task prompt)
- Container port (default 8000)

## Step 3: Generate the backend

Call `build_back` with:
- `app_name`: the project name (e.g., `td28`)
- `entities`: the JSON entity array from Step 2
- `db_url`: the PostgreSQL connection string
- `port`: the app port (default 8000)

This generates a complete FastAPI backend in `/workspace/{app_name}/`:
main.py (CRUD endpoints, DB connection, health check, static file serving),
requirements.txt, and a multi-stage Dockerfile.

## Step 3b: Generate the frontend

Call `build_front` with:
- `app_name`: same as build_back
- `entities`: same JSON entity array
- `api_base`: API prefix (default `/api`)
- `theme`: `"light"` or `"dark"`

This generates a React/Vite frontend in `/workspace/{app_name}/frontend/`
with components for each entity (list, create, update, delete), API client,
and a modern responsive theme.

## Step 3c: Customize (optional)

If the scaffolded code doesn't cover a specific requirement (custom logic,
additional endpoints, special styling), use `write_file` to modify the
generated files. The scaffold is a solid starting point — only customize
what the templates can't cover.

## Step 4: Provision database (if needed)

First check: was a database connection string already provided in your
prompt (from a prior task, the COMPLETED WORK section, or the orchestrator)?
If yes, **use it directly and skip to Step 5**. Do not call `create_database`
again — the database already exists.

Only call `create_database` if no connection details are available anywhere
in your prompt. It provisions a dedicated PostgreSQL container on the NAS.
Capture the returned connection string for use in deployment.

**Do NOT** inspect the NAS manually for existing databases.

## Step 5: Package — build and push Docker image

Use `build_and_push` with:
- `context_path`: the workspace directory (e.g., `/workspace/td28`)
- `image_name`: the app name (e.g., `td28`)
- `app_port`: the port the app listens on

The multi-stage Dockerfile (from build_back) handles both the React build
and the Python runtime. If the build fails, read the error, fix the source
code with `write_file`, and retry.

## Step 6: Deploy to Kubernetes

Use `deploy_application` with:
- `app_name`: the app name (e.g., `td28`)
- `image`: the image from Step 5 (e.g., `localhost:32000/td28:latest`)
- `app_port`: the port the app listens on
- `hostname`: the target URL (e.g., `td28.nblotti.org`)
- `env_vars`: JSON with environment variables (e.g., `{"DATABASE_URL": "postgresql://..."}`)

This tool creates the namespace, deployment, service, and ingress with TLS
automatically, then runs verification tests (pod health, API smoke test,
external ingress). If it fails, read the output, fix the issue, and retry.

Do NOT write K8s manifests manually — use `deploy_application`.

## Step 7: Verify deployment result (MANDATORY — do NOT skip)

The `deploy_application` tool runs built-in verification. Check its output
carefully — look for ASSET_FAIL or WARNING lines, not just `ok: true`.

**You MUST also run these checks yourself via `execute`:**

1. **Health**: `curl -sf https://{hostname}/health` — must return `{"ok": true}`
2. **HTML at root**: `curl -sf https://{hostname}/` — must return HTML, not JSON.
   If it returns JSON like `{"message": "..."}`, the frontend is missing.
3. **Asset URLs**: Parse the HTML from step 2. For every `src=` and `href=`
   URL (e.g. `/assets/index-xxx.js`), fetch it and confirm HTTP 200.
   If any asset returns 404, the page will be **blank**. This means the
   FastAPI static file mount path does not match vite's output paths.
   Fix `main.py` to mount the `static/assets/` directory at `/assets`
   (not `/static`), then rebuild and redeploy.
4. **API CRUD**: `curl -sf https://{hostname}/api/{entity_plural}` — must
   return `[]` (empty list). Test a POST to create one item and GET again.

If ANY check fails, diagnose the root cause, fix the code with `write_file`,
rebuild with `build_and_push`, and redeploy with `deploy_application`.
Repeat until ALL checks pass. Do NOT report success with failing checks.

## Step 7.5: Report facts (MANDATORY after completing work)

Call `report_facts` with a summary of ALL resources created. Include
everything subsequent tasks might need:

- Database: host, port, database name, username, password, connection string
- Image: name, tag, registry path
- Deployment: namespace, deployment name, service name
- Ingress: hostname, URL
- App: port the app listens on

## NEVER
- Do NOT call ask_human — the orchestrator handles user interaction
- Do NOT ask questions in your response text — the user cannot reply
- Do NOT declare success without running verification tests
- Do NOT skip database migration/table creation before verifying the API
- Do NOT manually inspect the NAS for databases — use create_database
- Do NOT use docker build/push via execute — use build_and_push
- Do NOT report "no code found" when the workspace is empty — you write the code
- Do NOT use write_file for the entire backend/frontend — use build_back + build_front
