---
name: arxiv-podcast-ops
description: Investigate, debug, and operate the Arxiv podcast generation pipeline running on Kubernetes with Argo Workflows. Use when the user asks about podcasts not being generated, workflow failures, Open Notebook issues, publishing errors, or wants to trigger manual podcast runs.
---

# Arxiv Podcast Ops

## Architecture

The podcast pipeline runs in Kubernetes namespace `podcast` using Argo Workflows and Argo Events.

```
Daily flow (Mon-Fri, Europe/Zurich):

04:00  CronWF daily-arxiv-podcast
         └─ select-articles (podcast-selector-arxiv) ──► Blog API

05:30  CronWF daily-podcast-generation
         ├─ Step 1: sync-notebook (sync Open Notebook once)
         └─ Step 2: pick-and-generate
              └─ GET /articles/podcast-candidates (2 arxiv + 2 blog)
              └─ POST /podcasts for each ──► Blog API webhook

On-demand (webhook triggers):
  Blog API POST /blog-api ──► Sensor ──► Workflow (podcast-pipeline-ondemand)
    DAG: notify-processing → split-languages → generate (per lang) → publish
    onExit: callback status to Blog API
```

### Kubernetes Resources

| Resource | Name | Purpose |
|----------|------|---------|
| CronWorkflow | `daily-arxiv-podcast` | Select articles from arXiv, push to Blog API (04:00) |
| CronWorkflow | `daily-podcast-generation` | Sync Open Notebook, then pick candidates + trigger generation (05:30) |
| WorkflowTemplate | `podcast-pipeline` | Article selection DAG |
| WorkflowTemplate | `podcast-pipeline-ondemand` | Generate + publish DAG (used by all `blog-podcast-*` workflows) |
| EventSource | `webhook` | Webhook server on port 12000 (`/blog-api`, `/mattermost`) |
| Sensor | `blog-api-podcast` | Listens `/blog-api`, submits `podcast-pipeline-ondemand` workflow |
| Sensor | `mattermost-podcast` | Listens `/mattermost`, submits same template |
| EventBus | `default` | JetStream-based event bus |
| Secret | `arxiv-pipeline-secret` | All credentials (see [reference.md](reference.md)) |

### External Services

| Service | In-cluster URL | External URL | Auth |
|---------|---------------|--------------|------|
| Blog API | `http://blog-api.blog` | — | Zeus JWT (`Bearer <token>`) |
| Zeus | `http://zeus.default:8000` | `https://zeus.nblotti.org` | `POST /token/technical` with secret |
| Open Notebook | — | `https://notebook.nblotti.org` | `Bearer <OPEN_NOTEBOOK_PASSWORD>` |
| S3 | — | bucket `arxiv-podcasts-nblotti` | AWS keys from secret |
| CloudFront | — | `d192ozvnkhed8.cloudfront.net` | Distribution `E2GIX1RJ77NOQ0` |

### Source Code

| Path | Purpose | Key file |
|------|---------|----------|
| `~/python/podcast-workflows/` | Argo YAML definitions | `cron-daily-podcasts.yaml`, `workflow-template-ondemand.yaml` |
| `~/python/podcast-generator/` | Sync + generate podcasts | `main.py`, `Dockerfile` |
| `~/python/podcast-publisher-arxiv/` | Publish to S3/RSS | `main.py` |
| `~/python/podcast-selector-arxiv/` | Select arXiv articles | `main.py` |

Image registry: `localhost:32000` (in-cluster MicroK8s registry).

## Debugging Workflow

When investigating "podcasts aren't generating":

### Step 1: Check recent workflows

```bash
kubectl get wf -n podcast --sort-by=.metadata.creationTimestamp | tail -20
```

Look for non-`Succeeded` status. Workflows named `blog-podcast-*` are individual podcast runs. `daily-podcast-generation-*` is the daily cron.

### Step 2: Find the failing step

```bash
kubectl get pods -n podcast --sort-by=.metadata.creationTimestamp | tail -30
```

Pod names follow the pattern `<workflow>-<template>-<hash>`. Key templates:
- `sync-notebook` — syncs Open Notebook models/profiles
- `pick-and-generate` — selects candidates and triggers Blog API
- `generate-podcast` — calls Open Notebook to generate audio (one per language)
- `publish-podcasts` — uploads MP3 to S3, updates RSS feed

### Step 3: Read logs

```bash
kubectl logs -n podcast <pod-name> -c main
```

### Step 4: Match log patterns

| Log pattern | Meaning | Action |
|-------------|---------|--------|
| `did not complete in 10 cycles` | Open Notebook generation timed out (10 min) | Re-trigger; check Open Notebook health |
| `500` on `/api/sources` or `/api/episode-profiles` | Open Notebook overloaded or race condition | Check if multiple syncs ran in parallel |
| `failed transaction` | SurrealDB conflict under concurrent load | Has 3-retry backoff; if persistent, reduce concurrency |
| `401` / `403` | Token/auth issue | Verify `arxiv-pipeline-secret` keys, check Zeus |
| `429` | Rate limited | Will auto-retry; check if too many concurrent workflows |
| `ImagePullBackOff` | Registry issue | Check `localhost:32000` registry, rebuild image |

## Known Failure Patterns

**Sync race condition** — Multiple `blog-podcast-*` workflows used to run `sync-notebook` in parallel, causing destructive interference in Open Notebook (deletes + recreates profiles). Fixed: sync now runs once in `daily-podcast-generation` CronWorkflow before triggering individual workflows. The `podcast-pipeline-ondemand` DAG does NOT include sync.

**Open Notebook timeouts** — Episode generation polls every 60s for 10 cycles. Transient overload causes timeouts. Re-triggering usually works. If persistent, check Open Notebook pod health.

**Transaction conflicts** — Open Notebook's SurrealDB returns `500: failed transaction` under concurrent load. The generator retries 3 times with exponential backoff. Usually self-healing.

## Manual Operations

### Check available candidates

```bash
kubectl run -n podcast --rm -it --restart=Never --image=python:3.13-slim check-candidates \
  -- python3 -c "
import json, urllib.request
url = 'http://blog-api.blog/articles/podcast-candidates?blog_id=15&limit=10'
req = urllib.request.Request(url)
with urllib.request.urlopen(req, timeout=30) as resp:
    data = json.load(resp)
for a in data:
    print(f'[{a[\"id\"]}] {a[\"title\"][:90]}')
"
```

`blog_id=15` is the Arxiv blog. Use `exclude_blog_id=15` for non-Arxiv blogs.

### Trigger N podcast generations

Create a pod with access to the `arxiv-pipeline-secret`. Full YAML template in [commands.md](commands.md).

### Run sync manually

```bash
kubectl run -n podcast --rm -it --restart=Never \
  --image=localhost:32000/podcast-generator:latest \
  --overrides='{"spec":{"containers":[{"name":"sync","image":"localhost:32000/podcast-generator:latest","command":["python","main.py","sync","--profile-url","https://raw.githubusercontent.com/nblotti/static/master/podcast-profiles/shipit_profile.txt"],"env":[{"name":"OPEN_NOTEBOOK_BASE_URL","value":"https://notebook.nblotti.org"},{"name":"OPEN_NOTEBOOK_PASSWORD","valueFrom":{"secretKeyRef":{"name":"arxiv-pipeline-secret","key":"OPEN_NOTEBOOK_PASSWORD"}}},{"name":"OPENAI_API_KEY","valueFrom":{"secretKeyRef":{"name":"arxiv-pipeline-secret","key":"OPENAI_API_KEY"}}}]}]}}' \
  manual-sync -- echo
```

### Re-run the daily CronWorkflow manually

Extract the workflowSpec from the CronWorkflow and submit:

```bash
kubectl get cronworkflow daily-podcast-generation -n podcast \
  -o jsonpath='{.spec.workflowSpec}' | \
  python3 -c "
import json, sys
spec = json.load(sys.stdin)
wf = {
  'apiVersion': 'argoproj.io/v1alpha1',
  'kind': 'Workflow',
  'metadata': {'generateName': 'daily-podcast-generation-manual-', 'namespace': 'podcast'},
  'spec': spec
}
print(json.dumps(wf))
" | kubectl create -f -
```

## Additional Resources

- Secret keys, env vars, API endpoints: [reference.md](reference.md)
- Copy-paste debugging commands: [commands.md](commands.md)
