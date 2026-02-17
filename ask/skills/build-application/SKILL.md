---
name: build-application
description: Build a new application from scratch. Requires presenting a detailed plan with modules and technologies to the user for confirmation before writing any code. Includes mandatory post-deploy verification.
allowed-tools: task, ask_human
---
# Build Application — Mandatory Planning Phase

When the user asks to CREATE, BUILD, or SCAFFOLD a new application from scratch,
you MUST follow this strict workflow. Do NOT skip any step.

## Step 1: Gather requirements (internal — no tool calls)

From the user request, determine:
- What the application does (purpose, features)
- Any explicit technology preferences mentioned

## Step 2: Present a detailed plan via ask_human (MANDATORY)

Before writing ANY code, you MUST call `ask_human` with a structured plan containing:

1. **Application name** and hostname (e.g. `myapp.nblotti.org`)
2. **Architecture overview** — list every module/component:
   - Backend: framework, language, key libraries
   - Frontend: framework, bundler, UI approach
   - Database: type, provisioning method
   - Docker: number of images, base images
3. **Technology stack summary** — e.g.:
   - Backend: Python 3.12 + FastAPI + SQLAlchemy + Alembic
   - Frontend: React 18 + Vite + TypeScript
   - Database: PostgreSQL 16 (provisioned on NAS via create_database)
   - Containerization: 2 Docker images (backend, frontend)
4. **Deployment plan** — K8s namespace, services, ingress
5. **Estimated steps** — numbered list of what will be done

## Step 3: Wait for user confirmation

- If the user confirms → proceed to Step 4
- If the user requests changes → revise the plan and ask again
- Do NOT start building until the user explicitly confirms

## Step 4: Execute the plan

Spawn subagents via `task` to implement each part of the confirmed plan.
Follow the sequential dependency rules:
- Provision DB can run in parallel with code scaffolding
- Build images AFTER code is written (sequential)
- Deploy AFTER images are built (sequential)
- Database migration / table creation MUST happen BEFORE verification

## Step 5: Verify EVERYTHING works (MANDATORY — do NOT skip)

After deployment completes, you MUST spawn a verification task that tests:

1. **Pod health**: all pods Running, 0 restarts
2. **Database**: tables exist, connectivity works
3. **API smoke tests**: call the main endpoints (GET list, POST create, GET verify)
4. **Frontend**: returns HTML (if applicable)
5. **Ingress**: external URL responds with 200

Only report success if ALL checks pass. If any check fails, fix it and re-verify.

The subagent should follow the `verify-deployment` skill instructions for the
full checklist and reporting format.

## NEVER
- Do NOT start writing code before user confirmation
- Do NOT skip the ask_human planning step
- Do NOT present a vague plan — be specific about every module and technology
- Do NOT declare success without running verification tests
- Do NOT skip database migration/table creation before verifying the API
