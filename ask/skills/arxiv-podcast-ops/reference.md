# Reference: Arxiv Podcast Pipeline

## Secret: `arxiv-pipeline-secret` (namespace `podcast`)

| Key | Used by | Description |
|-----|---------|-------------|
| `OPEN_NOTEBOOK_PASSWORD` | sync-notebook, generate-podcast | Bearer token for Open Notebook API |
| `OPENAI_API_KEY` | sync-notebook, generate-podcast, select-articles | OpenAI API key |
| `ZEUS_TECHNICAL_TOKEN_SECRET` | pick-and-generate, callback-status, select-articles | Secret for Zeus technical JWT |
| `BLOG_ID` | pick-and-generate (as `ARXIV_BLOG_ID`), select-articles | Arxiv blog ID in Blog API (value: `15`) |
| `AWS_ACCESS_KEY_ID` | generate-podcast, publish-podcasts | AWS credentials for S3 |
| `AWS_SECRET_ACCESS_KEY` | generate-podcast, publish-podcasts | AWS credentials for S3 |

## Environment Variables by Step

### sync-notebook

| Env | Value |
|-----|-------|
| `OPEN_NOTEBOOK_BASE_URL` | `https://notebook.nblotti.org` |
| `OPEN_NOTEBOOK_PASSWORD` | from secret |
| `OPENAI_API_KEY` | from secret |

### pick-and-generate

| Env | Value |
|-----|-------|
| `BLOG_API_URL` | `http://blog-api.blog` |
| `ZEUS_URL` | `http://zeus.default:8000` |
| `ZEUS_TECHNICAL_SECRET` | from secret key `ZEUS_TECHNICAL_TOKEN_SECRET` |
| `ARXIV_BLOG_ID` | from secret key `BLOG_ID` (value: `15`) |
| `CANDIDATES_PER_CATEGORY` | `2` (hardcoded in YAML) |

### select-articles

| Env | Value |
|-----|-------|
| `OPENAI_API_KEY` | from secret |
| `S2_MIN_SECONDS_BETWEEN_CALLS` | `1` |
| `BLOG_API_URL` | `http://blog-api.blog` |
| `BLOG_ID` | from secret |
| `ZEUS_URL` | `http://zeus.default:8000` |
| `ZEUS_TECHNICAL_SECRET` | from secret key `ZEUS_TECHNICAL_TOKEN_SECRET` |

### generate-podcast

| Env | Value |
|-----|-------|
| `OPEN_NOTEBOOK_BASE_URL` | `https://notebook.nblotti.org` |
| `OPEN_NOTEBOOK_PASSWORD` | from secret |
| `OPENAI_API_KEY` | from secret |
| `AWS_ACCESS_KEY_ID` | from secret |
| `AWS_SECRET_ACCESS_KEY` | from secret |
| `AWS_REGION` | `us-east-1` |
| `PODCAST_S3_BUCKET` | `arxiv-podcasts-nblotti` |
| `PODCAST_S3_PREFIX` | `podcasts/` |
| `CLOUDFRONT_DOMAIN` | `d192ozvnkhed8.cloudfront.net` |

### publish-podcasts

| Env | Value |
|-----|-------|
| `AWS_ACCESS_KEY_ID` | from secret |
| `AWS_SECRET_ACCESS_KEY` | from secret |
| `AWS_REGION` | `us-east-1` |
| `PODCAST_S3_BUCKET` | `arxiv-podcasts-nblotti` |
| `PODCAST_S3_PREFIX` | `podcasts/` |
| `CLOUDFRONT_DOMAIN` | `d192ozvnkhed8.cloudfront.net` |
| `CLOUDFRONT_DISTRIBUTION_ID` | `E2GIX1RJ77NOQ0` |

### callback-status (exit handler)

| Env | Value |
|-----|-------|
| `ZEUS_TOKEN_URL` | `https://zeus.nblotti.org/token/technical` (external URL for callbacks) |
| `ZEUS_TECHNICAL_TOKEN_SECRET` | from secret |

## API Endpoints

### Blog API (`http://blog-api.blog` in-cluster)

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/articles/podcast-candidates?blog_id=15&limit=N` | Get top N Arxiv candidates |
| GET | `/articles/podcast-candidates?exclude_blog_id=15&limit=N` | Get top N non-Arxiv candidates |
| POST | `/podcasts` | Trigger podcast generation (`{"articleId": <id>}`, needs `Authorization: Bearer <zeus-jwt>`) |
| PATCH | `<callback-url>` | Update podcast status (`{"status": "processing\|generated\|error"}`) |

### Zeus (`http://zeus.default:8000` in-cluster, `https://zeus.nblotti.org` external)

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/token/technical` | Get technical JWT (`{"secret": "<ZEUS_TECHNICAL_TOKEN_SECRET>", "service_name": "<name>"}`) |

Response: `{"access_token": "<jwt>", "expires_in": <seconds>}`

### Open Notebook (`https://notebook.nblotti.org`)

Auth: `Authorization: Bearer <OPEN_NOTEBOOK_PASSWORD>`

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/models` | List configured models |
| POST | `/api/models` | Create model |
| DELETE | `/api/models/{id}` | Delete model |
| GET | `/api/speaker-profiles` | List speaker profiles |
| POST | `/api/speaker-profiles` | Create speaker profile |
| DELETE | `/api/speaker-profiles/{id}` | Delete speaker profile |
| GET | `/api/episode-profiles` | List episode profiles |
| POST | `/api/episode-profiles` | Create episode profile |
| DELETE | `/api/episode-profiles/{id}` | Delete episode profile |
| POST | `/api/sources` | Create source (article text for generation) |
| DELETE | `/api/sources/{id}` | Delete source |
| GET | `/api/episodes/{id}` | Poll episode status (check if generation complete) |

## S3 / CloudFront / RSS

| Setting | Value |
|---------|-------|
| S3 bucket | `arxiv-podcasts-nblotti` |
| S3 prefix | `podcasts/` |
| CloudFront domain | `d192ozvnkhed8.cloudfront.net` |
| CloudFront distribution ID | `E2GIX1RJ77NOQ0` |
| Cover image | `https://d192ozvnkhed8.cloudfront.net/podcasts/cover3.jpg` |

### RSS Feed Paths

| Feed | S3 Key | Language |
|------|--------|----------|
| Daily EN | `podcasts/daily/feed_daily_en.xml` | English |
| Daily FR | `podcasts/daily/feed_daily_fr.xml` | French |
| Articles EN | `podcasts/articles/feed_articles_en.xml` | English |
| Articles FR | `podcasts/articles/feed_articles_fr.xml` | French |

MP3 files are stored at:
- Daily: `podcasts/daily/<slug>-<lang>.mp3`
- Articles: `podcasts/articles/<slug>-<lang>.mp3`

### Feed Metadata

| Field | Value |
|-------|-------|
| Title (EN) | `Arxiv LLM Daily (en)` |
| Title (FR) | `Arxiv LLM Daily (fr)` |
| Author | `Nicholas Blotti` |
| Email | `nblotti@gmail.com` |
| Website | `https://nicholasblotti.wpcomstaging.com/category/ai/podcast/` |

## Podcast Profiles

Profiles are LLM prompt templates that define the podcast dialogue format. They live in the same `nblotti/static` repo under `podcast-profiles/` and are fetched by the `sync-notebook` step via GitHub raw URL.

### Active Profile

**File**: `podcast-profiles/shipit_profile.txt`
**Raw URL**: `https://raw.githubusercontent.com/nblotti/static/master/podcast-profiles/shipit_profile.txt`
**Podcast name**: "Let's Ship It with AI"
**Speakers**: Jamie (journalist/moderator, woman) + Marc (expert, man)
**Structure**:
1. Intro + source reference (Jamie names the paper, asks Marc the practical use)
2. Executive summary (Marc gives 30-60s overview)
3. Topic-by-topic deep dive (4-8 topics extracted from the document, NOT section-by-section)
4. Practical implications (what should teams do now)
5. Conclusion + closing script (RSS feed + Spotify reference)

**Closing script** (verbatim, never change): "To finish, a quick reminder: the English version of the podcast is available via the RSS feed on nblotti dot org slash i a daily slash en, and also on Spotify — search for Ship It with AI EN. See you tomorrow for more AI news. Have a great day."

### Legacy Profile

**File**: `podcast-profiles/notebook_profiles.txt`
**Podcast name**: "Arxiv LLM Daily EN"
**Speakers**: Alex (journalist/moderator) + Marc (field expert) + Jamie (AI/tech expert)
**Structure**: Context setting, proposal analysis, use cases, under the hood/limitations, results, conclusion
**Closing script**: References "Arxiv LLM Daily EN" on Spotify

### How Profiles Are Used

1. `sync-notebook` downloads the profile URL and parses speaker/episode profile definitions
2. It creates or updates speaker profiles and episode profiles in Open Notebook via `/api/speaker-profiles` and `/api/episode-profiles`
3. `generate-podcast` then uses these profiles when requesting episode generation from Open Notebook

### Changing the Active Profile

1. Edit `podcast-profiles/shipit_profile.txt` (or create a new file) in the `nblotti/static` repo
2. Push to `master`
3. If using a new file, update the `profile-url` parameter in `cron-daily-podcasts.yaml` and `workflow-template-ondemand.yaml`
4. Run sync manually or wait for the next daily run to pick up the changes

## Workflow Parameters

### daily-arxiv-podcast (CronWorkflow)

| Parameter | Default | Description |
|-----------|---------|-------------|
| `image-registry` | `localhost:32000` | Container registry |
| `evaluation-cap` | `3` | Max articles to evaluate |
| `arxiv-days` | `5` | How many days back to search arXiv |
| `no-s2` | `false` | Skip Semantic Scholar lookups |

### podcast-pipeline-ondemand (WorkflowTemplate)

| Parameter | Default | Description |
|-----------|---------|-------------|
| `image-registry` | `localhost:32000` | Container registry |
| `selected-articles` | `{}` | JSON with article data |
| `adapter` | `mattermost` | Publisher adapter (`mattermost`, `arxiv`) |
| `languages` | `en,fr` | Comma-separated language codes |
| `num-podcasts` | `1` | Podcasts per article per language |
| `callback-url` | `""` | Blog API callback URL for status updates |
| `profile-url` | GitHub raw URL | Open Notebook profile configuration |

## Docker Images

All built and pushed to `localhost:32000` (MicroK8s in-cluster registry). Build host uses `192.168.1.7:32000`.

| Image | Source |
|-------|--------|
| `podcast-generator:latest` | `~/python/podcast-generator/` |
| `podcast-publisher-mattermost:latest` | `~/python/podcast-publisher-arxiv/` (adapter=mattermost) |
| `podcast-publisher-arxiv:latest` | `~/python/podcast-publisher-arxiv/` (adapter=arxiv) |
| `podcast-selector-arxiv:latest` | `~/python/podcast-selector-arxiv/` |
