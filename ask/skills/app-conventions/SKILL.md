---
name: app-conventions
description: Organization-specific conventions like hostnames and default ports.
allowed-tools:
---
# App conventions

## Hostnames
- Default hostname convention: `<app>.nblotti.org`
- DNS is externally managed.

## Default ports
- FastAPI/Uvicorn: 8000
- Flask: 5000
- Node/Next: 3000
- nginx: 80

## TLS
- The cluster has cert-manager with a `letsencrypt-prod` ClusterIssuer.
- ALL web applications MUST use HTTPS via Ingress with TLS.
- Always add annotation `cert-manager.io/cluster-issuer: letsencrypt-prod` and a `tls` section.

## Rule
- Don't ask the user which port/host unless the code clearly contradicts these defaults.
- If code differs, prefer code truth over defaults and report what you found.
