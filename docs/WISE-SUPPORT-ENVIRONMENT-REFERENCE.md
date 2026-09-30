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

### OCI S3-compatible credential mapping

The OCI Console terminology is easy to confuse with generic S3 terminology. For the current validation environment, map the values as follows:

| Render environment variable | OCI Console source |
|---|---|
| `STORAGE_ACCESS_KEY_ID` | **Profile → User settings → Customer secret keys → copy Access key** |
| `STORAGE_SECRET_ACCESS_KEY` | **Profile → User settings → Customer secret keys → Generate secret key → copy the one-time visible Secret key value** |
| `STORAGE_BUCKET_NAME` | OCI Object Storage bucket name |
| `STORAGE_REGION` | OCI region, currently `ap-hyderabad-1` |
| `STORAGE_ENDPOINT` | OCI S3-compatible endpoint, not a Pre-Authenticated Request (PAR) URL |

Important: the OCI User OCID is **not** the S3-compatible Access Key ID. The Customer Secret Key's Access key is the value used for `STORAGE_ACCESS_KEY_ID`, while the one-time visible Secret key is used for `STORAGE_SECRET_ACCESS_KEY`.

Do not use an OCI Pre-Authenticated Request URL as the Chatwoot storage endpoint. PAR URLs contain a temporary authorization token and are a different access mechanism.

Attachment upload/download should be tested end-to-end before considering the object-storage integration validated.

## Queue / cache

### Web concurrency versus background workers

These are two different kinds of workers/processes and must not be conflated.

- **Puma/Web workers** serve HTTP requests for the Chatwoot UI, APIs, webhooks and other synchronous web traffic. The repository's `config/puma.rb` defaults `WEB_CONCURRENCY` to `0`, which means one Puma process rather than zero web service capability. Concurrency within that process is controlled by `RAILS_MIN_THREADS` / `RAILS_MAX_THREADS`; the current validation configuration uses 5 threads.
- **Sidekiq workers** are separate background worker processes. They consume Redis/Valkey-backed jobs such as Chatwoot's asynchronous webhook processing and other background work. For the WISE Support Telegram/Chatwoot flow, the Render Background Worker is therefore required for the queued work to be processed after the Web service accepts/enqueues it.
- `WEB_CONCURRENCY=0` does **not** disable Sidekiq and does **not** mean that Chatwoot cannot serve multiple end users. It only avoids Puma's clustered multi-process mode, which is appropriate for the small validation instance.
- Multiple patients/end users and multiple support agents are supported at the application level. The validation topology should use one Web service plus a separate Sidekiq Worker; scaling capacity later can be addressed independently by increasing web capacity and/or Sidekiq concurrency/workers as workload requires.

For the initial pilot, 1–2 support agents and multiple concurrent patient conversations are a valid workload for the architecture. The Free validation instance is a capacity-test environment, not a production capacity target.

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

## Current OCI Validation Update — 2026-09-30

The active hosted validation path is now OCI Compute; Render is paused/archived as a validation path.

### OCI validation environment
- Runtime: OCI Compute, Hyderabad (`ap-hyderabad-1`), AD-1
- Instance: `wise-support-chat-oci-validation`
- Shape: Intel `VM.Standard3.Flex`, 1 OCPU / 16 GB RAM (Linux exposes 2 logical CPUs)
- OS: Oracle Linux Server 9.8 x86_64
- Container runtime: Docker Engine 29.8.1; Docker Compose plugin 5.5.1
- VCN: `wise-support-chat-vcn`
- Subnet: `wise-support-chat-public-subnet`
- Current public IP: `140.245.237.47` (validation infrastructure; may change)
- Image architecture required: `linux/amd64` / x86_64
- GHCR repository: `ghcr.io/wisedoctor/wise-support-chat`
- OCI Object Storage remains the S3-compatible attachment-storage option: bucket `oracle-oci-bucket-chatwoot-wisehealth`, region `ap-hyderabad-1`.

This VM is the current validation/E2E environment, not a final production infrastructure decision. The Always Free A1 pool can continue to be retried separately if capacity becomes available.


### OCI image and runtime validation — 2026-09-30

The OCI VM has now passed image-level and container-runtime smoke validation for the exact Chatwoot image selected for the x86_64 OCI host.

#### GHCR authentication
- GHCR package: private
- GitHub repository: public
- OCI Docker client authenticated to GHCR using a GitHub Personal Access Token (classic) with package-read access.
- The credential is not stored in Git and must not be recorded in this document.

#### Exact image validated
- Image: `ghcr.io/wisedoctor/wise-support-chat:sha-a6b2176`
- GitHub Actions build: verified `linux/amd64`
- OCI host: `linux/amd64`
- Pulled successfully on OCI
- Image ID: `sha256:94260b80fcf4f72fab0f1fc91b7ec7ce1a767f3e44af10077b19d9845828544a`
- RepoDigest: `ghcr.io/wisedoctor/wise-support-chat@sha256:94260b80fcf4f72fab0f1fc91b7ec7ce1a767f3e44af10077b19d9845828544a`
- The digest matches the previously verified GitHub Actions AMD64 build artifact.

#### Container runtime smoke test
The exact image was executed on the OCI VM with a non-persistent smoke-test container. The container successfully reported:
- `IMAGE_SMOKE_OK`
- `x86_64`
- Ruby `3.4.4`
- Rails `7.2.3.1`

This establishes that the exact image can be pulled and executed successfully on the OCI Intel/x86_64 VM. It does not yet validate PostgreSQL, Redis/Valkey, Chatwoot migrations, persistent storage, web/worker startup, HTTPS ingress, or Telegram E2E.


#### Dependency image pull checkpoint — 2026-09-30
The clean OCI VM had no existing PostgreSQL or Redis/Valkey containers and no listeners on ports 5432/6379. The dependency images were then pulled successfully:
- PostgreSQL: `postgres:16-alpine`
  - Digest: `sha256:721873c34ceb9f8d8fc265984940dc982404c105f19ad51be9fdc5970a6080ea`
- Redis: `redis:7-alpine`
  - Digest: `sha256:858f009f9709ce576febc734aa78b8f6d624b82571f9ddb6bda4377c833b3499`

The images are downloaded but not yet started. No Chatwoot application state has been changed at this stage.









#### Chatwoot image → PostgreSQL network connectivity verified — 2026-09-30
The exact image `ghcr.io/wisedoctor/wise-support-chat:sha-a6b2176` successfully reached PostgreSQL over the private `wise-support-net` Docker network using the container DNS name `wise-support-postgres`.

Result:
`wise-support-postgres:5432 - accepting connections`

This validates the application-container-to-database network path before any Chatwoot Rails initialization or schema migration is attempted.

#### Private dependency network attached — 2026-09-30
Both dependency containers are now attached to `wise-support-net`:
- `wise-support-postgres`: `bridge` + `wise-support-net`
- `wise-support-redis`: `bridge` + `wise-support-net`

Chatwoot can therefore use the container DNS names `wise-support-postgres` and `wise-support-redis` over the private Docker network. The existing localhost host bindings remain unchanged.

#### Private Docker network created — 2026-09-30
Created the Docker network `wise-support-net` on the OCI validation VM.
Network ID:
`d5b39d3d75b17f95456ef724e515ffe23e35b7c9cd2af7fe48e14aca95a32269`.

The network will be used for private container-to-container communication between Chatwoot, PostgreSQL, and Redis without publicly exposing the dependency ports.

#### Redis readiness verified — 2026-09-30
The running `wise-support-redis` container passed Redis connectivity validation:
`PONG` from `redis-cli ping`.

PostgreSQL and Redis are therefore both independently running and responding on the OCI validation VM. Chatwoot has not yet been connected to these dependencies.

#### Redis container started — 2026-09-30
Redis 7 Alpine is now running on the OCI validation VM:
- Container: `wise-support-redis`
- Image: `redis:7-alpine`
- Persistent volume: `wise-support-redis-data`
- Host binding: `127.0.0.1:6379 -> container 6379`
- Redis persistence enabled with AOF (`--appendonly yes`)
- The Redis port is therefore not publicly exposed by the VM.

Current dependency containers shown by `docker ps`:
- `wise-support-postgres` — PostgreSQL 16, localhost:5432
- `wise-support-redis` — Redis 7, localhost:6379

#### PostgreSQL readiness verified — 2026-09-30
The running `wise-support-postgres` container passed PostgreSQL readiness validation:
`/var/run/postgresql:5432 - accepting connections`.

This confirms the PostgreSQL server is accepting connections inside the container. Chatwoot schema/migrations have not yet been initialized.

#### PostgreSQL container started — 2026-09-30
PostgreSQL 16 Alpine is now running on the OCI validation VM:
- Container: `wise-support-postgres`
- Image: `postgres:16-alpine`
- Database: `chatwoot_production`
- Database user: `chatwoot`
- Persistent volume: `wise-support-postgres-data`
- Host binding: `127.0.0.1:5432 -> container 5432`
- The database port is therefore not publicly exposed by the VM.

Initial `docker ps` verification showed the container `Up` successfully. Database readiness/application connectivity has not yet been separately validated.

#### Persistent dependency volumes — 2026-09-30
Created the following Docker named volumes on the OCI validation VM:
- `wise-support-postgres-data`
- `wise-support-redis-data`

These volumes are intended to keep PostgreSQL and Redis data independent of container lifecycle. The dependency containers have not yet been started.

#### Validation checkpoint
Current chain proven:

OCI x86_64 VM → Docker 29.8.1 → exact GHCR image → linux/amd64 → Ruby 3.4.4 → Rails 7.2.3.1 → successful container execution.

Next validation stage: provision the runtime dependencies (PostgreSQL and Redis/Valkey) and initialize Chatwoot's database before starting the persistent Web/Sidekiq services.

### Render validation status — paused/archived

Render is no longer the active hosted validation runtime. The Render path is retained as historical/reference infrastructure and should not be treated as the current deployment target.

Reasons for archival:
1. The original Render source-build Web service exhausted the available build memory while compiling the large Chatwoot dependency tree.
2. The replacement image-backed validation path was then explored, but fresh-database initialization and process-start/Docker-command behavior did not produce a clean, repeatable Web deployment.
3. OCI now provides a directly controllable VM runtime for the hosted validation and E2E work.

Existing Render resources/configuration should be retained only until explicitly decommissioned.


#### PostgreSQL extension compatibility checkpoint — 2026-09-30
The first Chatwoot database-initialization attempt exposed an important migration prerequisite: the generic postgres:16-alpine image does not include the PostgreSQL vector/pgvector extension required by this Chatwoot build.

Observed failure during the exact image's bundle exec rails db:chatwoot_prepare:
- Rails production boot succeeded after supplying a stable SECRET_KEY_BASE.
- Chatwoot's database preparation then failed because PostgreSQL could not create/use extension vector.
- PostgreSQL reported that /usr/local/share/postgresql/extension/vector.control was missing.
- The current PostgreSQL 16 Alpine container has pg_stat_statements, pg_trgm, pgcrypto, and plpgsql, but not vector.
- The earlier installation_configs does not exist message occurred while the database was still uninitialized and should not be treated as a separate schema defect at this stage.

This is an environment-migration lesson: database engine/version compatibility is not sufficient; required PostgreSQL extensions must also be present in the actual database runtime image/service before application schema initialization.

##### Preventive migration checklist — PostgreSQL extension dependencies
For any future migration of Chatwoot or another PostgreSQL-backed WISE service:
1. Inspect the application schema (db/schema.rb / structure.sql) and migrations for CREATE EXTENSION / extension dependencies before selecting the PostgreSQL image.
2. Enumerate the required extensions explicitly and compare them with the target PostgreSQL service/image's installed extension set.
3. Verify extension availability with SELECT name, default_version FROM pg_available_extensions or equivalent before running application migrations.
4. Prefer an official/known PostgreSQL image that already includes the required extension (for example a pgvector-enabled PostgreSQL image) where appropriate, rather than discovering the dependency during first boot.
5. If using a managed PostgreSQL service, verify that the required extension is supported and enabled in that service before migration.
6. Keep database engine version and extension versions as explicit environment prerequisites alongside the application image architecture/runtime requirements.
7. Perform a disposable schema/bootstrap test against the exact target database image/service before treating the environment as migration-ready.
8. Preserve the persistent volume while replacing a dependency container only when the database compatibility procedure has been verified; never delete a potentially useful database volume merely to change the container image.

Current remediation status: pending. The existing PostgreSQL data volume is being preserved; no destructive database reset or volume deletion has been performed.

#### Chatwoot database bootstrap prerequisite — 2026-09-30
The exact Chatwoot image exposes db:chatwoot_prepare, documented by Rails task output as: Runs setup if database does not exist, or runs migrations if it does.

The image's docker/entrypoints/rails.sh does not automatically run database migrations; it waits for PostgreSQL and then executes the supplied process command. Therefore future environment setup procedures must explicitly provision the required PostgreSQL extensions before invoking db:chatwoot_prepare.

The production Rails environment also requires a persistent SECRET_KEY_BASE. The validation setup generated one locally on the OCI VM; secrets must not be recorded in this document or committed to Git.

#### PostgreSQL pgvector remediation checkpoint — 2026-09-30
The initial PostgreSQL container (`postgres:16-alpine`) was stopped without deleting its container or persistent volume, preserving the existing database state for safe replacement.

A compatible PostgreSQL image was pulled and verified:
- Image: `pgvector/pgvector:pg16`
- Digest: `sha256:ccc6e83d6e35e931dc7c5def2022729d5a6c370318d099181995567ff1fb4d6b`
- PostgreSQL: 16.15
- Verified pgvector extension control file: `/usr/share/postgresql/16/extension/vector.control`

The extension was verified by inspecting the disposable image directly rather than starting a temporary PostgreSQL server. A previous nested-server test was interrupted after multiple SIGTERM/SIGINT signals; it was disposable and did not use the persistent PostgreSQL volume.

Next remediation step: recreate `wise-support-postgres` from `pgvector/pgvector:pg16` using the existing `wise-support-postgres-data` volume, then verify PostgreSQL readiness and installed extensions before rerunning Chatwoot `db:chatwoot_prepare`.

#### pgvector PostgreSQL container recreated — 2026-09-30
The PostgreSQL container was recreated successfully from `pgvector/pgvector:pg16` using the existing `wise-support-postgres-data` persistent volume and attached to `wise-support-net`. PostgreSQL readiness was then verified successfully:
`/var/run/postgresql:5432 - accepting connections`.

No database reset or volume deletion was performed. The next checkpoint is verification of the `vector` extension in the running database before retrying Chatwoot initialization.


#### PostgreSQL extension availability verified — 2026-09-30
The running `wise-support-postgres` database was queried for the required PostgreSQL extensions before retrying Chatwoot initialization. The target database reports all expected extensions as available:
- `pg_stat_statements` — 1.10
- `pg_trgm` — 1.6
- `pgcrypto` — 1.3
- `plpgsql` — 1.0
- `vector` — 0.8.6

The previous missing-`vector` migration blocker is therefore resolved at the database runtime level. The next step is to rerun Chatwoot `db:chatwoot_prepare` against this pgvector-backed database.


#### Chatwoot pgvector bootstrap retry — 2026-09-30
The first lines of the post-remediation `db:chatwoot_prepare` retry show the same early production-boot warning/error pattern involving `installation_configs` while the database is still being initialized. The output then continues with `Loading Installation config`. This checkpoint is intentionally recorded as **in progress / not yet classified as a migration failure** until the command reaches its final exit status and output.


#### Chatwoot database preparation verified — 2026-09-30
After the pgvector-backed retry of `db:chatwoot_prepare` returned to the shell without a terminal error, the database was inspected directly. The `public.installation_configs` table exists and the database contains 107 public tables:

`installation_configs | 107`

This confirms that the earlier `installation_configs does not exist` log line was an initialization-time lookup during successful database preparation, not the final migration blocker. The previous missing-`vector` blocker is resolved and the Chatwoot schema has been created. Next validation should move to application runtime startup (Web/Sidekiq) rather than rerunning database preparation.


#### Chatwoot public-table inventory captured — 2026-09-30
The initialized `chatwoot_production` database was queried directly for all tables in the `public` schema. The result contains 107 tables, including core Chatwoot domains such as `accounts`, `users`, `contacts`, `contact_inboxes`, `inboxes`, `conversations`, `messages`, `attachments`, `teams`, `labels`, `webhooks`, channel-specific tables, Active Storage tables, reporting/automation/SLA tables, Captain/AI tables, and `installation_configs`.

This inventory is retained as the database-side schema checkpoint. Exact one-to-one reconciliation against the migrations/schema embedded in the pinned `sha-a6b2176` application image remains the next verification step; no external generic table-count claim is being treated as authoritative.
