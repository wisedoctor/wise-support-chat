# WISE Support — Environment Infrastructure Reference

## Purpose

This document records the infrastructure choices and alternatives identified while moving the WISE Support implementation from local development toward a production-capable environment.

It is a reference record, not a final infrastructure lock-in. The WISE Support architecture remains provider-neutral and should not make Chatwoot, Render, PostgreSQL hosting, Redis/Valkey hosting, or object storage a conceptual dependency.

## Current validation environment

The current production-validation setup uses:

- Runtime platform: Render
- Region: Singapore
- Container image: GitHub Container Registry (GHCR)
- Image source: wisedoctor/wise-support-chat, develop
- Image build: GitHub Actions → GHCR
- Web process: Render Web Service
- Worker process: Render Background Worker
- Relational database: Render PostgreSQL, PostgreSQL 16
- Queue/cache dependency: Render Key Value (Redis-compatible), being provisioned
- Object storage: Oracle Cloud Infrastructure (OCI) Object Storage, using its S3-compatible API
- Application/provider layer: Chatwoot is the current support-conversation provider implementation; it is not the architectural definition of WISE Support.

The current Render PostgreSQL instance is being used for validation. Its Free-plan lifecycle/retention limitations mean it should not automatically be treated as the final long-term production database.

## Database hosting alternatives to retain as reference

### Option A — Render PostgreSQL

Useful for:
- Fast Render-native validation
- Simple private connectivity between Render services
- Minimal initial infrastructure work

Current status:
- Provisioned as wise-support-chat-db
- PostgreSQL 16
- Singapore
- Available
- Free plan for validation

This is the current validation choice, not a permanent architectural commitment.

### Option B — Existing WISE Supabase PostgreSQL

The existing WISE Supabase environment should remain a valid candidate for the eventual support runtime database where operationally appropriate.

Potential advantages:
- Existing WISE database platform and operational familiarity
- Existing WISE infrastructure/access patterns
- Avoids introducing another independent PostgreSQL estate if the tenancy and workload boundaries are appropriate

Before adopting this option, explicitly validate:
- Database ownership and isolation boundaries
- Migration ownership
- Connection/security model from Render
- Connection pooling and connection limits
- Backup/restore expectations
- Whether Chatwoot's schema and migrations should remain isolated from core WISE application schemas
- Whether support-provider data belongs in the existing Supabase project or a dedicated database/project

Important: do not point the support runtime at the core WISE database merely for convenience. Shared infrastructure must still preserve explicit ownership and integration boundaries.

### Option C — OCI-hosted PostgreSQL

OCI remains another infrastructure option if there is a reason to consolidate more of the runtime/database stack within OCI.

This should be evaluated only against the operational burden, availability, backup, security, networking, maintenance, and cost implications of running PostgreSQL there.

## Object storage

OCI Object Storage has been selected for the current validation environment because the user has an OCI bucket with the applicable free-tier allocation.

Chatwoot can consume S3-compatible object storage through its Active Storage S3-compatible configuration.

Reference configuration concept:
- Bucket name
- S3-compatible access key ID
- S3-compatible secret access key
- OCI region
- S3-compatible endpoint
- Optional path-style configuration where required

Credentials must be stored as deployment secrets/environment variables and must never be committed to Git.

The bucket should remain private unless a specific application requirement establishes otherwise.

Attachment upload/download should be tested end-to-end before considering the object-storage integration validated.

## Queue / cache

The current validation path uses Render Key Value as the Redis-compatible dependency.

Alternatives to retain for later evaluation include:
- Existing WISE Redis/Valkey infrastructure, if available and appropriate
- OCI-hosted Redis-compatible infrastructure
- Another managed Redis/Valkey service

As with the database, the infrastructure choice should not leak into the WISE Support conceptual architecture.

## Container distribution

The current image pipeline is:

GitHub source → GitHub Actions → GHCR → Render

The first successful image build was produced from the develop branch.

The preferred deployment model is to build the Chatwoot-derived image in GitHub Actions and have Render run the prebuilt image, rather than requiring Render to perform the large source build.

This separates image build capacity, runtime capacity, and application deployment, and avoids tying the application architecture to Render's source-build resource limits.

## Environment separation

Development, validation/demo, and production must remain distinct.

At minimum:
- Development: local Docker/runtime, local webhooks/tunnels where necessary
- Validation/demo: Render/managed services using non-production credentials and test data
- Production: stable WISE-owned ingress, production credentials, persistent database/storage, proper webhook configuration, monitoring, backup/restore, and operational controls

The local ngrok/proxy arrangement is development-only and must not become the production ingress design.

## Architectural principle

Infrastructure is replaceable implementation detail.

The WISE Support engine should depend on explicit application capabilities/contracts rather than directly encoding assumptions such as:
- Render PostgreSQL
- Supabase
- OCI PostgreSQL
- Redis/Valkey vendor
- Chatwoot
- OCI Object Storage
- GHCR

These are environment/runtime choices. The Support module remains channel-neutral and provider-neutral.

## Decision status

| Component | Current validation choice | Alternatives retained |
|---|---|---|
| Runtime | Render | Other managed/container runtime |
| Image registry | GHCR | Other OCI-compatible registry |
| Database | Render PostgreSQL | Existing WISE Supabase PostgreSQL; OCI PostgreSQL |
| Queue/cache | Render Key Value | Existing WISE Redis/Valkey; OCI/other managed Redis-compatible service |
| Object storage | OCI Object Storage | Other S3-compatible provider |
| Conversation provider | Chatwoot | WISE-native provider or another provider |
| Region | Singapore | To be decided per production infrastructure constraints |

This document should be updated when the validation environment is converted into the final production topology.