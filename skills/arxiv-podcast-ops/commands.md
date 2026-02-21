# Commands: Arxiv Podcast Pipeline

## Checking Status

### List recent workflows
```bash
kubectl get wf -n podcast --sort-by=.metadata.creationTimestamp | tail -20
```

### List recent pods
```bash
kubectl get pods -n podcast --sort-by=.metadata.creationTimestamp | tail -30
```

### Describe a workflow (see DAG status, step durations)
```bash
kubectl describe wf <workflow-name> -n podcast
```

### Get logs for a specific step
```bash
kubectl logs -n podcast <pod-name> -c main
```

Pod naming: `<workflow>-<template>-<hash>`. To find pods for a workflow:
```bash
kubectl get pods -n podcast --no-headers -o custom-columns=NAME:.metadata.name | grep "^<workflow-name>"
```

### Check Argo Events (sensors, eventsources)
```bash
kubectl get eventsources,sensors,eventbus -n podcast
kubectl get events -n podcast --sort-by='.lastTimestamp' | tail -20
```

## Checking Candidates

### List available Arxiv candidates
```bash
kubectl run -n podcast --rm -it --restart=Never --image=python:3.13-slim check-candidates \
  -- python3 -c "
import json, urllib.request
url = 'http://blog-api.blog/articles/podcast-candidates?blog_id=15&limit=10'
req = urllib.request.Request(url)
with urllib.request.urlopen(req, timeout=30) as resp:
    data = json.load(resp)
print(f'Found {len(data)} candidates:')
for a in data:
    print(f'  [{a[\"id\"]}] {a[\"title\"][:90]}')
"
```

### List available non-Arxiv candidates
```bash
kubectl run -n podcast --rm -it --restart=Never --image=python:3.13-slim check-candidates-blog \
  -- python3 -c "
import json, urllib.request
url = 'http://blog-api.blog/articles/podcast-candidates?exclude_blog_id=15&limit=10'
req = urllib.request.Request(url)
with urllib.request.urlopen(req, timeout=30) as resp:
    data = json.load(resp)
print(f'Found {len(data)} candidates:')
for a in data:
    print(f'  [{a[\"id\"]}] {a[\"title\"][:90]}')
"
```

## Triggering Podcast Generation

### Trigger N Arxiv podcasts manually

Replace `limit=4` with the desired count:

```bash
kubectl apply -n podcast -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: trigger-arxiv-manual
  namespace: podcast
spec:
  restartPolicy: Never
  containers:
  - name: trigger
    image: python:3.13-slim
    command: ["python3", "-c"]
    args:
    - |
      import json, os, sys, urllib.request, urllib.error

      BLOG_API_URL = os.environ["BLOG_API_URL"].rstrip("/")
      ZEUS_URL = os.environ["ZEUS_URL"].rstrip("/")
      ZEUS_TECHNICAL_SECRET = os.environ["ZEUS_TECHNICAL_SECRET"]
      ARXIV_BLOG_ID = "15"
      LIMIT = 4

      def get_zeus_token():
          data = json.dumps({"secret": ZEUS_TECHNICAL_SECRET, "service_name": "manual-podcast"}).encode()
          req = urllib.request.Request(f"{ZEUS_URL}/token/technical", data=data, headers={"Content-Type": "application/json"}, method="POST")
          with urllib.request.urlopen(req, timeout=20) as resp:
              return json.load(resp)["access_token"]

      def trigger_podcast(article_id, token):
          data = json.dumps({"articleId": article_id}).encode()
          req = urllib.request.Request(
              f"{BLOG_API_URL}/podcasts", data=data,
              headers={"Content-Type": "application/json", "Authorization": f"Bearer {token}"},
              method="POST",
          )
          try:
              with urllib.request.urlopen(req, timeout=30) as resp:
                  result = json.load(resp)
                  print(f"  -> id={result.get('id')} status={result.get('status')}")
                  return result
          except urllib.error.HTTPError as e:
              print(f"  -> ERROR {e.code}: {e.read().decode()}", file=sys.stderr)
              return None

      url = f"{BLOG_API_URL}/articles/podcast-candidates?blog_id={ARXIV_BLOG_ID}&limit={LIMIT}"
      print(f"GET {url}")
      with urllib.request.urlopen(urllib.request.Request(url), timeout=30) as resp:
          candidates = json.load(resp)
      print(f"Found {len(candidates)} candidates")
      for a in candidates:
          print(f"  - [{a['id']}] {a['title'][:80]}")

      token = get_zeus_token()
      ok = 0
      for article in candidates:
          print(f"Triggering {article['id']}: {article['title'][:60]}...")
          if trigger_podcast(article["id"], token):
              ok += 1
      print(f"Done: {ok}/{len(candidates)} created.")
    env:
    - name: BLOG_API_URL
      value: "http://blog-api.blog"
    - name: ZEUS_URL
      value: "http://zeus.default:8000"
    - name: ZEUS_TECHNICAL_SECRET
      valueFrom:
        secretKeyRef:
          name: arxiv-pipeline-secret
          key: ZEUS_TECHNICAL_TOKEN_SECRET
EOF
```

Check logs and clean up:
```bash
kubectl logs -n podcast trigger-arxiv-manual --follow
kubectl delete pod trigger-arxiv-manual -n podcast
```

### Re-run the daily CronWorkflow

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

## Checking Open Notebook Health

### Test Open Notebook API
```bash
kubectl run -n podcast --rm -it --restart=Never --image=python:3.13-slim test-notebook \
  --overrides='{"spec":{"containers":[{"name":"test","image":"python:3.13-slim","command":["python3","-c","import urllib.request,json,os\npassword=os.environ[\"OPEN_NOTEBOOK_PASSWORD\"]\nfor endpoint in [\"/api/models\",\"/api/speaker-profiles\",\"/api/episode-profiles\"]:\n    req=urllib.request.Request(f\"https://notebook.nblotti.org{endpoint}\",headers={\"Authorization\":f\"Bearer {password}\"})\n    try:\n        with urllib.request.urlopen(req,timeout=10) as r:\n            data=json.load(r)\n            print(f\"{endpoint}: {len(data)} items\")\n    except Exception as e:\n        print(f\"{endpoint}: ERROR {e}\")\n"],"env":[{"name":"OPEN_NOTEBOOK_PASSWORD","valueFrom":{"secretKeyRef":{"name":"arxiv-pipeline-secret","key":"OPEN_NOTEBOOK_PASSWORD"}}}]}]}}' \
  -- echo
```

## Checking RSS Feeds

### View current RSS feed
```bash
aws s3 cp s3://arxiv-podcasts-nblotti/podcasts/daily/feed_daily_en.xml - | head -50
```

### List recent MP3 uploads
```bash
aws s3 ls s3://arxiv-podcasts-nblotti/podcasts/daily/ --recursive | sort | tail -20
```

### Invalidate CloudFront cache for a feed
```bash
aws cloudfront create-invalidation \
  --distribution-id E2GIX1RJ77NOQ0 \
  --paths "/podcasts/daily/feed_daily_en.xml" "/podcasts/daily/feed_daily_fr.xml"
```

## Monitoring Workflow Progress

### Watch all workflows in real-time
```bash
kubectl get wf -n podcast -w
```

### Check generate-podcast logs for a specific workflow
```bash
for pod in $(kubectl get pods -n podcast --no-headers -o custom-columns=NAME:.metadata.name | grep "^<workflow>-generate"); do
  echo "=== $pod ==="
  kubectl logs -n podcast "$pod" -c main 2>&1 | tail -5
  echo
done
```

### Check publish results for a batch of workflows
```bash
for pod in $(kubectl get pods -n podcast --no-headers -o custom-columns=NAME:.metadata.name | grep "^blog-podcast-.*-publish"); do
  echo "=== $pod ==="
  kubectl logs -n podcast "$pod" -c main 2>&1 | grep -E '(published_episodes|episode_name|cloudfront_url|file_size)' | head -10
  echo
done
```
