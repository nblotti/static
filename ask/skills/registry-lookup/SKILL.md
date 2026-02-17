---
name: registry-lookup
description: List images/tags and registry usage guidance using registry_lookup tool.
allowed-tools: registry_lookup
---
# Container registry operations

## Required flow
1. Call `infra_catalog("registry")` for authoritative registry details.
2. Use `registry_lookup()` to list repositories or `registry_lookup("my-image")` for tags.

## Notes
- Prefer registry_lookup over guessing image names/tags.
- If an image pull fails, re-check registry host/port rules from infra_catalog and retry accordingly.
