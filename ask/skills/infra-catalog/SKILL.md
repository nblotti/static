---
name: infra-catalog
description: Discover infrastructure endpoints, credentials, and connection instructions via infra_catalog tool.
allowed-tools: infra_catalog
---
# Infrastructure discovery (infra_catalog)

Use `infra_catalog()` before touching any external system (NAS, registry, GitHub, Docker, Kubernetes, databases).

## When to use
- Any task that mentions: NAS, SSH, registry, images, GitHub, Kubernetes, secrets, credentials, DB hosts.

## How to use
- `infra_catalog()` returns everything.
- `infra_catalog("nas")`, `infra_catalog("registry")`, `infra_catalog("github")`, `infra_catalog("docker")`, `infra_catalog("kubernetes")`, `infra_catalog("credentials")` for targeted results.

## Hard rules
- Never ask the user for credentials or endpoints if infra_catalog can provide them.
- If access fails, re-check infra_catalog and then proceed with a different method or report the exact permission failure.
