---
name: build-application
description: Build a new application from scratch. Requires presenting a detailed plan with modules and technologies to the user for confirmation before writing any code.
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

Example ask_human message:
```
Here is my plan for the todo2 application:

**Architecture:**
- Backend: Python 3.12 + FastAPI + psycopg + Alembic (port 8000)
- Frontend: React 18 + Vite + TypeScript (port 8080, served by nginx)
- Database: PostgreSQL 16 on NAS (via create_database tool)
- Docker: 2 images — todo2-backend, todo2-frontend

**Modules:**
1. backend/app/main.py — FastAPI app with CORS, health check
2. backend/app/models.py — SQLAlchemy models (Todo table)
3. backend/app/routes.py — CRUD API endpoints
4. backend/alembic/ — DB migrations
5. frontend/src/App.tsx — Main React component
6. frontend/src/api.ts — API client
7. k8s.yaml — Namespace, Deployments, Services, Ingress (todo2.nblotti.org)

**Steps:**
1. Provision PostgreSQL database
2. Scaffold backend code + Dockerfile
3. Scaffold frontend code + Dockerfile
4. Build & push both images
5. Deploy to Kubernetes with TLS ingress

Shall I proceed?
```

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

## NEVER
- Do NOT start writing code before user confirmation
- Do NOT skip the ask_human planning step
- Do NOT present a vague plan — be specific about every module and technology
