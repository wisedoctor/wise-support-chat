# WISE Support — Production and Maintenance Runbook

## 1. Purpose

This runbook records the concrete maintenance knowledge learned while proving the WISE Support Telegram/Chatwoot integration locally and preparing the hosted validation environment.

It complements the architectural documents. It does not redefine the Support contracts.

## 2. Environment layers

### Local development

Current proven local topology:

```text
Telegram
   |
   v
ngrok
   |
   v
local proxy :8080
   |----------------------|
   v                      v
Chatwoot :3000          WISE backend :8000
   |
   v
Rails + Sidekiq + PostgreSQL + Redis
```

The local proxy multiplexes:

- `/webhooks/telegram/*` → Chatwoot
- `/api/webhooks/chatwoot` → WISE backend

This arrangement is development/debugging infrastructure only.

### Hosted validation

Target topology:

```text
GitHub
  |
  v
GitHub Actions
  |
  v
GHCR
  |
  +------------------+
  v                  v
Render Web       Render Worker
  |                  |
  +--------+---------+
           |
     DB / Key Value
           |
      OCI Object Storage
```

The Web and Worker processes must run the same application image/version.

## 3. Current deployment method

The source-build attempt on Render exceeded the available build memory.

The validated replacement approach is:

1. GitHub Actions builds the Chatwoot-derived Docker image.
2. GitHub Actions publishes the image to GHCR.
3. Render consumes the prebuilt image.

The first successful image build was produced from `develop`.

This separates image-build capacity from runtime capacity and avoids requiring Render to compile the large Chatwoot source tree during deployment.

## 4. Local process maintenance

Typical local processes used during E2E proving:

- Chatwoot Docker containers
- WISE backend: Uvicorn on port 8000
- frontend development server where required
- local proxy on port 8080
- ngrok forwarding to port 8080

Check local listeners before starting another copy:

```bash
ss -ltnp | grep -E ':3000|:8000|:8080'
```

Check Docker containers:

```bash
docker ps
```

The actual Chatwoot container names observed in the proven setup are:

- `deployment-rails-1`
- `deployment-sidekiq-1`
- `deployment-postgres-1`
- `deployment-redis-1`

The repository's Docker Compose service names do not necessarily match these container names. When `docker compose exec rails` or `docker compose logs sidekiq` reports that the service is not running, inspect `docker ps` and use the actual container name rather than guessing a Compose service name.

Useful direct commands:

```bash
docker logs deployment-rails-1 --tail 200
docker logs deployment-sidekiq-1 --tail 200
docker restart deployment-sidekiq-1
docker exec -it deployment-rails-1 bash
```

## 5. Rails console diagnostics

Use the Rails container for application-level diagnostics rather than querying the database with assumptions about provider schema.

Example:

```bash
docker exec -it deployment-rails-1 bundle exec rails console
```

The proven Telegram channel schema contains:

```text
id
bot_name
account_id
bot_token
created_at
updated_at
```

There is no `inbox_id` column on `Channel::Telegram`. Inbox linkage is through the application association.

## 6. Telegram channel identity learning

Two Telegram bots are deliberately separate:

### WISE Support bot

- Bot: `@wescura_support_bot`
- Credential/config name: `TELEGRAM_SUPPORT_BOT_TOKEN`
- Existing WISE backend webhook: `/api/webhooks/telegram-support`

### Chatwoot POC bot

- Bot: `@wise_chatwoot_poc_bot`
- Separate Chatwoot Telegram channel credential
- Chatwoot Account 1 / Channel 2 / Inbox 2 in the proven local setup

Never substitute one credential for the other.

## 7. Telegram webhook token learning

The Telegram webhook job receives the full bot token.

Chatwoot's stored token lookup therefore cannot rely on a direct equality between the incoming value and a value with an additional suffix.

The proven resolution is:

1. Extract the bot ID/prefix before the first colon.
2. Resolve the stored Telegram channel using the bot-ID prefix.
3. Process the event through that channel's inbox.

This fix was implemented as:

- `caeebe204b` — resolve channel by bot-token prefix
- `159242f184` — extract bot ID from webhook token

The latter was required because the incoming webhook parameter contains the complete token.

Do not undo either fix without reproducing the channel-resolution E2E test.

## 8. Proven local E2E

The following path has been proven:

```text
Patient Telegram account
        |
        v
Telegram webhook
        |
        v
ngrok / local proxy
        |
        v
Chatwoot Telegram channel
        |
        v
Sidekiq
        |
        v
Chatwoot conversation
        |
        v
Support Rep reply
        |
        v
Chatwoot Telegram outbound
        |
        v
Patient Telegram
```

The patient inbound message was also observed by the existing WISE Support notification path. That is a separate downstream consumer/context path and must not be removed merely because Chatwoot also displays the conversation.

The two paths must not be conflated into an assumption that Telegram itself forwarded the message between the two bots.

## 9. Sidekiq restart learning

After changing `Webhooks::TelegramEventsJob`, the running Sidekiq process initially retained the previously loaded class.

A Sidekiq restart was required before the new channel-resolution behavior was exercised.

Operational rule:

> After changing Ruby job/application code, ensure the worker process has loaded the new code before interpreting an E2E failure as a code regression.

## 10. Webhook maintenance

Development currently uses a temporary ngrok URL and local proxy.

Production must not use either as the permanent ingress.

Before production:

- establish stable WISE-owned webhook endpoints;
- separate inbound channel/provider delivery from provider-to-WISE support events where required;
- configure provider webhook secrets/signature verification where supported;
- record correlation IDs;
- test retries and duplicate deliveries;
- verify idempotent handling.

Exact production endpoint paths remain a design decision and should be agreed before implementation.

## 11. Object storage maintenance

The current validation object store is OCI Object Storage using its S3-compatible API.

Keep the bucket private.

Use dedicated S3-compatible access credentials, not general OCI account credentials.

Store credentials only in the deployment secret/environment configuration.

Validation requirement:

- upload an attachment;
- retrieve/download it;
- verify the persisted object;
- verify the Support/Chatwoot UI can still access it;
- test the failure path with invalid/expired credentials before production.

## 12. Database maintenance

Current validation database:

- Render PostgreSQL 16
- `wise-support-chat-db`
- Singapore
- Free validation tier

This is not yet the final production database decision.

Retained alternatives:

- existing WISE Supabase PostgreSQL;
- persistent Render PostgreSQL;
- OCI-hosted PostgreSQL.

Do not move to the WISE Supabase database merely to eliminate an infrastructure resource. First establish schema ownership, isolation, migrations, connectivity, pooling, backup/restore and operational ownership.

## 13. Queue/cache maintenance

Current validation dependency:

- Render Key Value
- Redis-compatible

Production requires an explicitly selected managed/persistent Redis-compatible service with an appropriate availability and recovery model.

## 14. Image maintenance

The preferred deployment artifact is an immutable GHCR image.

For production releases:

1. Build from the controlled source ref.
2. Publish a SHA-tagged image.
3. Validate the image.
4. Deploy that immutable image to Web and Worker.
5. Record the source commit/image SHA used by the environment.

Avoid relying on a mutable `develop` tag as the final production deployment identifier.

## 15. Logs and failure triage

For a webhook failure, inspect in this order:

1. External provider webhook status.
2. Stable ingress/proxy status.
3. Rails/web application logs.
4. Sidekiq worker logs.
5. Channel lookup.
6. Conversation/contact creation.
7. Outbound provider delivery.
8. WISE-side integration event/logs.

Do not change multiple layers simultaneously during diagnosis.

## 16. Security rules

Never commit:

- Telegram bot tokens
- Chatwoot access tokens
- webhook secrets
- database passwords
- OCI access secrets
- patient data
- raw prescriptions
- production contact data
- production environment files

Use secret names and configuration contracts in documentation.

## 17. Production readiness checklist

Before production sign-off:

- [ ] Stable WISE-owned ingress
- [ ] Production Telegram bot/configuration separated from POC
- [ ] Web and Worker on same immutable image
- [ ] Persistent production database selected
- [ ] Persistent Redis/Valkey selected
- [ ] OCI/S3-compatible object storage E2E validated
- [ ] Inbound Telegram E2E validated
- [ ] Outbound Support Rep reply E2E validated
- [ ] Duplicate/retry/idempotency behavior validated
- [ ] Webhook security validated
- [ ] Secrets externalized
- [ ] Monitoring and alerting defined
- [ ] Backup/restore tested
- [ ] Support/Ops ownership and escalation defined
- [ ] Patient-safe servicing SOP available
- [ ] Production release/source/image recorded

## 18. Maintenance principle

Chatwoot, Telegram, Render, Supabase, OCI and GHCR are current implementation tools.

They are not the WISE Support architecture.

Maintenance changes must preserve the higher-level boundaries:

- channel abstraction;
- provider abstraction;
- authoritative WISE domain workflows;
- explicit context and identity integration;
- structured signals;
- provenance/correlation;
- security/consent;
- auditable consequential actions.


## Current Hosted Validation Update — 2026-09-30

The active hosted validation runtime is now OCI Compute.

OCI validation VM:
- Instance: wise-support-chat-oci-validation
- Region: ap-hyderabad-1, AD-1
- Shape: Intel VM.Standard3.Flex, 1 OCPU / 16 GB RAM
- OS: Oracle Linux Server 9.8 x86_64
- Docker Engine: 29.8.1
- Docker Compose plugin: 5.5.1
- VCN: wise-support-chat-vcn
- Subnet: wise-support-chat-public-subnet
- GHCR image must be the x86_64/linux/amd64 build; the ARM64-only A1 image is not suitable for this VM.
- OCI Object Storage bucket: oracle-oci-bucket-chatwoot-wisehealth, region ap-hyderabad-1.

### Render validation — paused/archived

Render is no longer the active validation runtime. It is retained as historical/reference infrastructure.

Reasons:
1. The original source-build Web service exceeded available build memory during Chatwoot dependency compilation.
2. The image-backed replacement was explored, but fresh-database initialization and process-start/Docker-command behavior did not produce a clean, repeatable deployment.
3. OCI now provides direct VM-level control for the hosted validation/E2E phase.

Do not treat Render as the current validation or production target unless deliberately reactivated.
