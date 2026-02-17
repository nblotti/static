---
name: verify-deployment
description: After deploying or updating any application, run mandatory verification tests (health, API, database, frontend) before reporting success. NEVER skip verification.
allowed-tools: execute
---
# Verify Deployment — Mandatory Post-Deploy Testing

After deploying, updating, or fixing ANY application, you MUST run verification
tests BEFORE reporting success to the user. Do NOT declare "done" until all
applicable checks pass.

## When this applies

- After initial deployment of a new app
- After fixing a bug (404, 500, crash-loop, etc.)
- After updating code, images, or configuration
- After database migrations or schema changes

## Mandatory checks (run ALL that apply)

### 1. Pod health
```bash
kubectl get pods -n <namespace> -l app=<app> -o wide
# Verify: STATUS=Running, RESTARTS=0, READY=1/1
```

### 2. Health endpoint
```bash
kubectl exec -n <namespace> deploy/<app> -- curl -sf http://localhost:<port>/health
# OR from the sandbox:
curl -sf http://<service-clusterip>:<port>/health
```

### 3. Database connectivity and schema
```bash
kubectl exec -n <namespace> deploy/<backend> -- python -c "
from sqlalchemy import create_engine, inspect
import os
engine = create_engine(os.environ['DATABASE_URL'])
tables = inspect(engine).get_table_names()
print('Tables:', tables)
assert len(tables) > 0, 'No tables found!'
"
```
If the app uses a different DB framework, adapt accordingly.
If tables are missing, run the app's migration or create_all BEFORE declaring success.

### 4. API smoke tests
Test the MAIN endpoints the app exposes (not just health):
```bash
# List endpoint (should return 200, not 500)
curl -sf http://<service>:<port>/api/<resource> | head -c 500

# Create endpoint (POST with minimal valid payload)
curl -sf -X POST http://<service>:<port>/api/<resource> \
  -H "Content-Type: application/json" \
  -d '{"title":"test item"}' | head -c 500

# Verify the created item appears in the list
curl -sf http://<service>:<port>/api/<resource> | head -c 500
```

### 5. Frontend (if applicable)
```bash
curl -sf http://<frontend-service>:<port>/ | head -c 200
# Verify: returns HTML with expected title/framework markers
```

### 6. External access (ingress)
```bash
curl -sf -k https://<app>.nblotti.org/ | head -c 200
curl -sf -k https://<app>.nblotti.org/api/<resource> | head -c 200
```

## Reporting

After running ALL checks, report in this format:

```
Pods: 1/1 Running, 0 restarts
Health: /health returns 200
Database: connected, tables=[todos]
API: GET /api/todos -> 200 (empty list)
API: POST /api/todos -> 201 (created)
Frontend: returns HTML
Ingress: https://app.nblotti.org -> 200
```

If ANY check fails, FIX the issue and re-run ALL checks.
Do NOT report success until everything passes.

## NEVER
- Do NOT say "deployment successful" based only on kubectl rollout status
- Do NOT skip API tests — pod running does NOT mean app working
- Do NOT skip database verification when the app uses a database
- Do NOT report partial success — ALL checks must pass
