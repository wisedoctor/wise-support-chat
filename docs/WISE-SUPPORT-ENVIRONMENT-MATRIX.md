# WISE Support — Environment Matrix

## Purpose

This matrix defines the working environment separation for WISE Support as the implementation moves from local proof to hosted validation and then production.

It is an environment reference, not a commitment to a single infrastructure vendor.

## Environment matrix

| Area | Development / local | Validation / demo | Production target |
|---|---|---|---|
| Purpose | Engineering, debugging, E2E proving | Hosted integration validation | Live WISE Support |
| Source branch | Local working branch | `develop` during validation | Controlled release branch/tag |
| Image build | Local Docker or source runtime | GitHub Actions → GHCR | GitHub Actions → GHCR |
| Image registry | Local build / GHCR as needed | GHCR | GHCR or approved OCI registry |
| Runtime | Local Docker / processes | OCI Compute (current); Render archived | Managed production runtime |
| Rails web | Local Chatwoot container | OCI `wise-support-chat-web` | Dedicated production web service |
| Sidekiq | Local Chatwoot container | OCI `wise-support-chat-worker` | Dedicated production worker |
| Database | Local PostgreSQL/Chatwoot DB | OCI `wise-support-postgres` / pgvector:pg16 | Render PostgreSQL, WISE Supabase PostgreSQL, or OCI PostgreSQL after decision |
| Queue/cache | Local Redis-compatible service | OCI `wise-support-redis` | Approved persistent/managed Redis-compatible service |
| Object storage | Local/test storage as required | OCI Object Storage | Approved persistent object storage |
| Conversation provider | Chatwoot | Chatwoot | Chatwoot initially; provider remains replaceable |
| External channel | Telegram | Telegram | Telegram initially; additional channels later |
| Telegram support bot | `@wescura_support_bot` / WISE-owned support path | Same only where required | Separate production configuration |
| Chatwoot Telegram bot | `@wise_chatwoot_poc_bot` | POC/validation bot | Dedicated production bot/configuration; never reuse POC credentials |
| Webhook ingress | ngrok + local proxy | `support.wisehealth.in` → OCI Nginx | Stable WISE-owned ingress |
| Local proxy | Used for multiplexing local Chatwoot + WISE backend | Not required | Prohibited as production architecture |
| Public tunnel | ngrok | Not required once stable ingress is available | Not used |
| Secrets | Local `.env` / secret store | OCI/runtime secrets; never committed | Production secret manager/environment |
| Patient data | Synthetic/test data only | Synthetic or explicitly approved test data | Real data under production controls |
| Database migrations | Local Rails migration workflow | Controlled deployment migration | Controlled release migration |
| Attachments | Test uploads | OCI S3-compatible storage test | Persistent production object storage |
| Monitoring | Console/container logs | OCI/container logs + application logs | Centralized logs, alerts, audit/operational monitoring |
| Webhook security | Local testing | Validate provider/auth/signature controls | Mandatory verification where supported |
| Backup/restore | Local/test | Validation only | Required and tested |
| Availability expectation | Developer workstation | Pilot/validation | Production SLA/operational target |
| Data retention | Disposable | Validation lifecycle | Defined production policy |

## Current validation resources

The earlier Render validation resources were assembled in Singapore and are now historical/archived:

- Render PostgreSQL: `wise-support-chat-db`
- Render Key Value: `wise-support-chat-redis`
- Render Web Service: `wise-support-chat-prod` (transitioning to prebuilt GHCR image)
- Render Background Worker: `wise-support-chat-worker-prod` (planned)
- GHCR image: `ghcr.io/wisedoctor/wise-support-chat:develop` plus SHA-tagged image
- OCI Object Storage: user-created bucket using S3-compatible access
- GitHub repository: `wisedoctor/wise-support-chat`

## Configuration separation rules

### 1. Never share bot credentials across environments

The existing WISE Support bot and the Chatwoot POC bot are different channel identities.

Do not use `TELEGRAM_SUPPORT_BOT_TOKEN` for the Chatwoot Telegram channel.

Do not reuse the POC Chatwoot bot credential for production.

### 2. Never share production secrets with local/validation environments

Use separate:

- Telegram bot credentials
- Chatwoot credentials
- webhook secrets
- database credentials
- object-storage credentials
- application encryption/secrets

### 3. Do not promote temporary ingress

The local ngrok/proxy arrangement proves the integration but is not a production topology.

Production should terminate channel/provider webhooks at a stable WISE-owned endpoint and use explicit provider/channel adapters.

### 4. Do not let infrastructure choices become Support contracts

Render, Supabase, OCI, Chatwoot, Telegram, Redis/Valkey and GHCR are implementation/environment choices.

Support contracts should remain channel-neutral and provider-neutral.

## Production decision gates

Before calling the environment production-ready, confirm:

1. Production database host selected and persistent.
2. Redis/Valkey host selected and persistence/availability requirements satisfied.
3. Object storage configured and attachment upload/download tested.
4. Rails web and Sidekiq worker deployed from the same immutable image.
5. Stable WISE-owned webhook ingress configured.
6. Inbound Telegram E2E tested.
7. Outbound Support Rep reply E2E tested.
8. Failure/retry/idempotency behavior tested.
9. Two Telegram bot identities remain isolated.
10. Production secrets are stored outside Git.
11. Monitoring/logging and ownership are defined.
12. Backup/restore has been tested.
13. Patient-safe Support SOP and escalation path are available to operators.
14. Production release is made from an explicitly controlled branch/tag.

## Database decision note

Render PostgreSQL is the current validation database only.

The following remain valid candidates for the eventual production database:

- Existing WISE Supabase PostgreSQL
- Render PostgreSQL on an appropriate paid/persistent plan
- OCI-hosted PostgreSQL

The final choice must be based on ownership, isolation, connectivity, migrations, backup/restore, operational burden and cost—not merely proximity to the runtime.



## Current Environment Status — 2026-09-30

The active hosted validation environment is now OCI Compute. Render is paused/archived as a validation path.

### Active OCI validation
- OCI region: Hyderabad (ap-hyderabad-1), AD-1
- Instance: wise-support-chat-oci-validation
- Shape: Intel VM.Standard3.Flex, 1 OCPU / 16 GB RAM
- OS: Oracle Linux Server 9.8 x86_64
- Docker Engine: 29.8.1; Compose plugin: 5.5.1
- VCN/subnet: wise-support-chat-vcn / wise-support-chat-public-subnet
- GHCR: ghcr.io/wisedoctor/wise-support-chat
- Image architecture: linux/amd64
- OCI Object Storage: oracle-oci-bucket-chatwoot-wisehealth in ap-hyderabad-1

### Render status
Render is paused/archived for validation. The source-build service exceeded available build memory; the subsequent image-backed attempt did not achieve a clean repeatable database/process startup path. Render remains historical/reference infrastructure until explicitly decommissioned.
