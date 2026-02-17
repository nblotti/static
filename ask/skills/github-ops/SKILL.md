---
name: github-ops
description: GitHub access conventions and safe API usage with tokens.
allowed-tools: infra_catalog
---
# GitHub operations

## Flow
1. infra_catalog("github") for auth method and expected env vars.
2. Prefer GitHub API for listings (repos, PRs) when available.
3. The sandbox has GITHUB_TOKEN and KUBECONFIG pre-set. Use them directly.

## If user asks vaguely for repos
- Default to listing the authenticated user repos via GitHub API.
- If that fails, report the exact error and fall back to asking for a repo name only if necessary.
