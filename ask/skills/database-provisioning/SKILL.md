---
name: database-provisioning
description: Provision PostgreSQL for apps using create_database and wire connection strings.
allowed-tools: create_database
---
# PostgreSQL provisioning

## Hard rule
- Always provision app databases via `create_database(app_name)`. Do not deploy Postgres inside Kubernetes.

## Flow
1. Call `infra_catalog("credentials")` or `infra_catalog("nas")` only if you need context.
2. Call `create_database("app-name")`.
3. Use the returned connection string in app configuration (env vars or secrets).

## Reporting
- Return host, port, db, user, and where to set env vars.
