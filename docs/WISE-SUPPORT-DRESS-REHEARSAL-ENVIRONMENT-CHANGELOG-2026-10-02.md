# WISE Support — Dress Rehearsal Environment Change Log

**Purpose:** Historical, implementation-specific record of the environment changes made to establish and rehearse the hosted WISE Support + Chatwoot + Telegram integration.

**Scope:** OCI hosted dress rehearsal, supporting GHCR image selection, abandoned Render validation, Chatwoot runtime bootstrap, public HTTPS validation, Telegram webhook/channel configuration, and operational test configuration.

**Status:** Dress rehearsal / pilot-prep environment. Not a production hardening record.

**Updated:** 2026-10-02  
**Latest checkpoint:** WISE Health Website Support Inbox configured; native `/support` integration next

---

## 1. Why this document exists

The main environment reference records the current infrastructure state and long-term environment principles.

This companion document records the **sequence of changes actually made during the dress rehearsal**, including temporary validation changes and diagnostic configuration. It is intended to answer:

- What did we change?
- Why did we change it?
- What was temporary vs persistent?
- What was proven?
- What should be retained, hardened, or removed before production?

It deliberately sits separately from the environment setup reference and from the pilot/E2E backlog.

---

# 2. Starting point

The WISE Support implementation began with:

- Chatwoot v4.14.2 running locally;
- Telegram → Chatwoot → Support Rep proven locally;
- Chatwoot → Telegram reverse reply proven locally;
- a development-only ngrok/proxy ingress;
- WISE backend/frontend running separately;
- a requirement to prove the same architecture on a remotely hosted environment.

The local environment was not treated as the production topology.

The objective of the hosted dress rehearsal was to prove:

```
Patient Telegram
      ↓
public HTTPS
      ↓
Chatwoot
      ↓
Sidekiq
      ↓
Support Inbox
      ↓
Support Rep
      ↓
Telegram
      ↓
Patient
```

before proceeding into pilot operations, RBAC and production entry-point work.

---

# 3. Render validation attempt — subsequently archived

## 3.1 Initial Render direction

The first hosted validation direction used Render:

- Render Web Service;
- Render Background Worker;
- Render PostgreSQL 16;
- Render Key Value / Redis-compatible service;
- GHCR image-backed deployment;
- OCI Object Storage as S3-compatible attachment storage.

A Render source-build attempt was made first.

### Result

The large Chatwoot source build exhausted the available build memory.

The source-build path was therefore abandoned in favour of a prebuilt GHCR image.

---

## 3.2 Image-backed Render attempt

A GHCR image-backed Render service was then attempted.

A fresh Chatwoot database exposed initialization requirements that were not cleanly handled by the available Render process model:

- fresh database did not yet have Chatwoot initialization state;
- `installation_configs` was absent during first boot;
- Render command/process configuration did not provide a sufficiently clean, repeatable migration/release separation for this validation;
- the resulting deployment path was not considered a reliable hosted rehearsal.

Render was therefore paused/archived as the active hosted validation runtime.

### Decision

Use a directly controllable OCI Compute VM for the dress rehearsal.

Render resources/configuration remain historical/reference infrastructure until explicitly decommissioned.

---

# 4. GHCR image architecture correction

The OCI VM ultimately selected was x86_64/AMD64.

An earlier GitHub Actions build had produced an ARM64 image for the develop branch after a multi-architecture build attempt proved impractical for the rehearsal.

The rehearsal therefore deliberately selected the verified AMD64 build:

```
ghcr.io/wisedoctor/wise-support-chat:sha-a6b2176
```

Verified:

- GitHub Actions runner: x64;
- Build platform: linux/amd64;
- OCI host: linux/amd64;
- image digest:
  `sha256:94260b80fcf4f72fab0f1fc91b7ec7ce1a767f3e44af10077b19d9845828544a`.

The exact image was pulled successfully on OCI and smoke-tested:

```
IMAGE_SMOKE_OK
x86_64
Ruby 3.4.4
Rails 7.2.3.1
```

### Important operational rule

Do not use the current `develop` GHCR tag blindly for an x86_64 OCI deployment if it points to the later ARM64 build.

The rehearsal uses the immutable AMD64 SHA tag above.

---

# 5. OCI Compute environment created

A successful OCI Intel instance was provisioned after several unsuccessful Always Free A1 capacity attempts.

Instance:

- Name: `wise-support-chat-oci-validation`
- Shape: `VM.Standard3.Flex`
- 1 OCPU / 16 GB RAM
- Linux exposes 2 logical CPUs
- Oracle Linux Server 9.8
- x86_64
- public IP: `140.245.237.47`
- user: `opc`

The A1/ARM capacity attempts are historical and were not used for the rehearsal.

---

# 6. OCI networking

The following networking resources were created and retained:

### VCN

`wise-support-chat-vcn`

CIDR:

`10.0.0.0/16`

### Public subnet

`wise-support-chat-public-subnet`

CIDR:

`10.0.0.0/24`

Regional public subnet.

### Internet gateway

`wise-support-chat-igw`

### Route

```
0.0.0.0/0 → Internet Gateway
```

IPv6 was not enabled.

### Security list

TCP 443 from `0.0.0.0/0` was opened for the hosted HTTPS dress rehearsal.

This is a **validation configuration** and must be reviewed/hardened before production.

The OCI networking resources were not recreated during later steps; subsequent work used the existing network.

---

# 7. Docker runtime installed on OCI

Oracle Linux did not provide the required Docker Engine package directly from the initial repositories.

The official Docker CentOS repository was added:

```
https://download.docker.com/linux/centos/docker-ce.repo
```

Docker was installed and enabled.

Verified runtime:

- Docker Engine 29.8.1
- containerd 2.3.6
- Buildx 0.37.1
- Compose plugin 5.5.1

The `opc` user was added to the Docker group and the SSH session was re-established so Docker could be used without sudo.

---

# 8. GHCR authentication

The GHCR package was private even though the GitHub repository was public.

A GitHub Personal Access Token with package-read permission was created and used to authenticate the OCI Docker client.

The credential was deliberately not stored in Git or this document.

The exact AMD64 Chatwoot image was then pulled successfully.

---

# 9. Initial PostgreSQL + Redis dependency setup

The initial dependency images were:

- `postgres:16-alpine`
- `redis:7-alpine`

Named Docker volumes were created:

- `wise-support-postgres-data`
- `wise-support-redis-data`

The dependency containers were initially bound only to localhost:

- PostgreSQL: `127.0.0.1:5432`
- Redis: `127.0.0.1:6379`

This prevented direct public exposure of the database/cache ports.

---

# 10. Private Docker network created

Created:

```
wise-support-net
```

Network ID:

```
d5b39d3d75b17f95456ef724e515ffe23e35b7c9cd2af7fe48e14aca95a32269
```

PostgreSQL and Redis were attached to this network.

Chatwoot was subsequently connected to the same private network.

The intended topology became:

```
             wise-support-net
       ┌──────────┬───────────┐
       │          │           │
   Chatwoot    PostgreSQL    Redis
       │
   Sidekiq
```

Dependency ports remained host-local rather than public.

---

# 11. PostgreSQL readiness and first migration failure
PostgreSQL initially became healthy and accepted connections.

The exact Chatwoot image successfully reached PostgreSQL through the private Docker network.

Chatwoot database preparation was then attempted.

The first meaningful environment failure was PostgreSQL extension compatibility.

The generic PostgreSQL 16 Alpine image did not contain:

```
vector.control
```

The Chatwoot migration path required the pgvector extension.

This was an **environment dependency discovery**, not a Chatwoot application-code defect.

---

# 12. PostgreSQL replaced with pgvector image

The existing PostgreSQL data volume was preserved.

The PostgreSQL container was replaced with:

```
pgvector/pgvector:pg16
```

Verified image digest:

```
sha256:ccc6e83d6e35e931dc7c5def2022729d5a6c370318d099181995567ff1fb4d6b
```

The same persistent PostgreSQL volume was reused.

Extensions were verified, including:

- pg_stat_statements
- pg_trgm
- pgcrypto
- plpgsql
- vector

The exact Chatwoot image was then able to complete:

```
bundle exec rails db:chatwoot_prepare
```

---

# 13. Chatwoot database initialization

A stable `SECRET_KEY_BASE` was generated for the hosted validation environment.

It is not recorded here.

Chatwoot database preparation was rerun successfully.

The resulting database showed:

- `installation_configs` present;
- 107 public tables observed;
- complete Chatwoot migration status;
- all migrations `up`.

The final observed migration was:

```
20260924000000 Add icon to conversation monitors
```

The exact pinned Chatwoot image therefore has a successfully initialized database on the OCI PostgreSQL/pgvector runtime.

---

# 14. Redis runtime

Redis was configured as a persistent AOF-backed container.

Container:

```
wise-support-redis
```

Image:

```
redis:7-alpine
```

Volume:

```
wise-support-redis-data
```

Host binding:

```
127.0.0.1:6379
```

Connectivity was verified independently with:

```
PONG
```

The exact Chatwoot image also successfully reached Redis over the private Docker network.

---

# 15. Persistent Chatwoot Web container

Created:

```
wise-support-chat-web
```

Using:

```
ghcr.io/wisedoctor/wise-support-chat:sha-a6b2176
```

Configured to use:

- PostgreSQL via private Docker DNS;
- Redis via private Docker DNS;
- production Rails environment;
- stable validation `SECRET_KEY_BASE`.

Puma started successfully.

Verified:

- Ruby 3.4.4;
- Rails 7.2.3.1;
- Puma 7.2.1;
- x86_64;
- listening on `0.0.0.0:3000`.

Host-local access initially returned:

```
302 Found
→ /installation/onboarding
```

The Chatwoot onboarding UI was then accessed through an SSH tunnel from the development machine because local port 3000 was already occupied.

SSH tunnel used:

```
-L 3001:127.0.0.1:3000
```

Browser:

```
http://localhost:3001
```

---

# 16. Chatwoot account and initial user

The Chatwoot installation was completed through the browser.

The resulting application account was:

- Account 1

A support user was created:

- Sreedhar Byreeka
- Account 1
- SuperAdmin during validation
- account_user ID 1
- user ID 1

The initial dashboard showed:

- My Inbox;
- no channel/inbox before Telegram onboarding.

This was the starting point for the support-rep rehearsal.

---

# 17. Persistent Sidekiq worker created

Created:

```
wise-support-chat-worker
```

Using the same pinned Chatwoot image.

Command:

```
bundle exec sidekiq -C config/sidekiq.yml
```

No public host port was exposed.

The worker successfully processed scheduled/background jobs.

This was essential because Telegram inbound processing is asynchronous.

The final hosted logical topology became:

```
Internet
   ↓
Nginx :443
   ↓
Chatwoot Web :3000
   ↓
Redis / PostgreSQL
   ↑
Sidekiq Worker
```

---

# 18. Current hosted container set

The final rehearsal container set was:

| Container | Role |
|---|---|
| wise-support-chat-web | Chatwoot Rails/Puma |
| wise-support-chat-worker | Sidekiq |
| wise-support-postgres | PostgreSQL + pgvector |
| wise-support-redis | Redis |

Private Docker network IPs observed:

- PostgreSQL: .2
- Redis: .3
- Web: .4
- Worker: .5

All four were attached to `wise-support-net`.

---

# 19. Public HTTPS validation added

The hosted Chatwoot application initially had no public HTTPS entry point.

Nginx 1.20.1 was installed on the OCI VM.

A temporary self-signed TLS certificate was generated for:

```
140.245.237.47
```

Certificate/key locations:

```
/etc/nginx/ssl/chatwoot-ip.crt
/etc/nginx/ssl/chatwoot-ip.key
```

The private key must never be copied into Git or documentation.

The certificate was configured with the IP address as SAN/CN for validation.

---

# 20. Nginx reverse proxy added

Nginx was configured to:

```
listen 443 ssl
```

and proxy:

```
https://140.245.237.47/
        ↓
http://127.0.0.1:3000
```

Forwarded headers include:

- Host- X-Real-IP
- X-Forwarded-For
- X-Forwarded-Proto

WebSocket upgrade headers were also configured for Chatwoot ActionCable.

Nginx configuration test passed.

Nginx was enabled to start with the VM.

---

# 21. SELinux adjustment for Nginx

The first Nginx HTTPS request returned 502 even though Chatwoot itself was healthy.

The cause was SELinux blocking Nginx from making the upstream connection.

The boolean:

```
httpd_can_network_connect
```

was enabled persistently:

```
sudo setsebool -P httpd_can_network_connect 1
```

After this change:

```
curl -k -I https://140.245.237.47/
```

returned HTTP 200.

This was an environment-level fix and should be retained in the hosted VM configuration if this Nginx/SELinux topology remains.

---

# 22. HTTPS ingress firewall changes

For Telegram/Chatwoot dress rehearsal:

- OCI Security List TCP 443 was opened to `0.0.0.0/0`;
- VM firewalld HTTPS service was enabled.

Ports 5432 and 6379 remained non-public/localhost-bound.

The HTTPS opening is currently a validation configuration and must be reviewed for production.

---

# 23. Public webhook endpoint validation

The public Chatwoot webhook endpoint was tested directly:

```
POST https://140.245.237.47/webhooks/telegram/test
```

The endpoint returned HTTP 200.

This established the public route:

```
Internet
   ↓
OCI TCP 443
   ↓
Nginx
   ↓
Chatwoot :3000
```

before changing the Telegram webhook.

---

# 24. Telegram POC webhook ownership changed

The POC bot:

```
@wise_chatwoot_poc_bot
```

was previously pointed at the older WISE public path.

Telegram `getWebhookInfo` showed:

- old webhook URL;
- pending updates;
- HTTP 404 responses from the old path.

The webhook was then changed to the OCI Chatwoot path:

```
https://140.245.237.47/webhooks/telegram/<bot-token>
```

using the temporary self-signed certificate.

Telegram then reported:

- `has_custom_certificate: true`
- `pending_update_count: 0`
- `ip_address: 140.245.237.47`

This proved Telegram accepted the IP-based HTTPS webhook with the supplied certificate.

### Important

Telegram permits only one active webhook per bot.

Therefore the webhook ownership must always be checked before testing:

```
getWebhookInfo
→ test/change
→ getWebhookInfo
```

Do not point the POC bot at the existing WISE Support bot's webhook.

---

# 25. Chatwoot Telegram Inbox created

The Telegram POC channel was configured in Chatwoot.

Inbox:

```
wise_chatwoot_poc_bot
```

Associated Telegram identity:

```
@wise_chatwoot_poc_bot
```

The Chatwoot UI flow used:

```
Choose Channel
→ Create Inbox
→ Add Agents
→ Ready
```

The support user was added as an inbox collaborator.

---

# 26. Chatwoot inbox settings inspected

The following settings were inspected during the rehearsal:

### Conversation Routing

Configured:

```
Create new conversations
```

### Greeting

Off during the initial test.

### Business availability

Off.

### Collaborators

Support user present.

Automatic conversation assignment was enabled.

Default assignment policy was observed:

- earliest-created conversations first;
- round-robin distribution.

The custom-policy feature was shown as a Business-plan capability and was not used.

No unnecessary configuration changes were made during the initial inspection.

---

# 27. Telegram → OCI → Chatwoot E2E proof

A Telegram test message from the test patient account successfully traversed:

```
Telegram
   ↓
OCI public HTTPS
   ↓
Nginx
   ↓
Chatwoot Telegram webhook
   ↓
Sidekiq
   ↓
Contact
   ↓
Conversation
   ↓
Message
```

Worker logs showed:

- TelegramEventsJob performed;
- contact creation/identification;
- conversation creation;
- message creation;
- assignment job;
- SendReplyJob activity.

This established that the complete hosted inbound path was working.

---

# 28. Chatwoot → Telegram reverse E2E proof

A support representative replied from the Chatwoot Inbox.

The reply was successfully delivered to the patient's Telegram chat.

This established the hosted two-way text baseline:

```
Patient
  ↕
Telegram
  ↕
Chatwoot
  ↕
Support Rep
```

This is the core communication path now considered proven.

---

# 29. Auto-assignment investigation changes / diagnostics

Assignment initially appeared inconsistent.

A conversation could arrive under:

```
Unassigned
```

despite:

- support user being an inbox member;
- automatic assignment enabled;
- green/online-looking UI state.

The investigation inspected:

- `users`;
- `account_users`;
- `inbox_members`;
- `inboxes`;
- assignment policy;
- `AutoAssignment::AssignmentService`;
- `AutoAssignment::AssignmentJob`;
- `AutoAssignment::RoundRobinSelector`;
- `AutoAssignment::InboxRoundRobinService`;
- `AutoAssignment::RateLimiter`;
- `InboxAgentAvailability`;
- `OnlineStatusTracker`;
- browser ActionCable presence.

No manual assignment script was used to create the successful assignment.

The successful assignment observed later:

> Assigned to Sreedhar Byreeka by Default Policy

was generated by Chatwoot's own automatic assignment path.

The browser's ActionCable connection was confirmed to be sending presence updates every 20 seconds.

Redis showed current user presence and online status.

This investigation should remain recorded as pilot/E2E evidence rather than environment setup.

---
# 30. New Android patient-context rehearsal

A separate Android device was used to test the new CTA as a genuinely new patient context.

Telegram conversation:

```
/start
Need medicines for my son
```

Support replied:

```
do you have Rx?
```

Chatwoot showed:

- new contact: Durga;
- new conversation #2;
- Telegram POC inbox;
- initially Unassigned.

This is a valuable pilot scenario because it validates:

```
new user
  ↓
medicine intent
  ↓
Support
```

The assignment behaviour remains an open pilot-prep item until a clean new-conversation assignment test is completed.

---

# 31. Attachment/media validation attempted

The text-only path is proven.

Attachment testing exposed failures:

### Outbound .txt

A simple .txt attachment sent from Chatwoot:

- was shown as failed/red;
- did not reach the Telegram patient.

### Outbound Rx image

A prescription-sheet image/photo:

- was shown as failed/red;
- did not reach the Telegram patient.

These are **not yet solved**.

They have therefore been classified as pilot blockers in:

```
docs/WISE-SUPPORT-PILOT-PREP-BACKLOG.md
```

The current environment change log records the fact that attachment validation was attempted; diagnosis belongs in the pilot/E2E workstream.

---

# 32. Chatwoot welcome-message capability inspected

The Telegram Inbox exposes a configurable welcome/onboarding capability.

This was identified as important beyond the current rehearsal:

- welcome text can establish expectations;
- it can guide first-contact context;
- it may help reduce repeated questions;
- it provides a current Chatwoot implementation of a future provider-neutral Support capability.

No broad architecture dependency on Chatwoot welcome configuration was introduced.

The eventual WISE Support abstraction should preserve the capability even if Chatwoot is replaced.

---

# 33. Telegram /start observation

The new Android rehearsal confirmed that the Telegram first-contact experience currently exposes:

```
/start
```

This is now an explicit pilot UX investigation item.

Open questions:

- can the user-facing action be avoided?
- can it be made less technical?
- can the holding-page CTA explain it?
- can the welcome message provide friendlier guidance?
- can the word "bot" be de-emphasized?

No production UX change has yet been made as a result.

---

# 34. What has been proven by the dress rehearsal

### Proven

- OCI x86_64 host can run the exact pinned Chatwoot image.
- PostgreSQL/pgvector runtime can initialize Chatwoot.
- Redis works as the Chatwoot queue/cache dependency.
- Chatwoot Web starts successfully.
- Chatwoot Sidekiq worker starts successfully.
- Nginx can expose Chatwoot through HTTPS.
- A stable DNS hostname now resolves to the OCI Chatwoot ingress.
- Let's Encrypt trusted TLS is active for `support.wisehealth.in`.
- Chatwoot portal is reachable through `https://support.wisehealth.in` for support-rep access.
- Telegram POC webhook has been migrated from the temporary IP endpoint to `https://support.wisehealth.in/webhooks/telegram/<bot-token>`.
- Telegram reports `has_custom_certificate: false` on the hostname webhook, as expected with publicly trusted TLS.
- Telegram reports `pending_update_count: 0` after migration.
- Telegram inbound text reaches Chatwoot through the stable hostname.
- Chatwoot creates/continues conversations.
- Support Rep can service the conversation through the hostname-based portal.
- Chatwoot outbound text reaches Telegram through the stable hostname path.
- The two-way text E2E was rerun successfully after the hostname migration.
- New patient context can enter the support conversation.
- Chatwoot welcome-message capability exists.
- Auto-assignment path has been exercised and observed assigning a conversation through Default Policy.

### Not yet proven / not production-ready

- reliable attachment/media delivery;
- final attachment storage configuration;
- production Telegram bot identity;
- production webhook;
- production secrets management;
- production backup/restore;
- production monitoring/log retention/alerting;
- final OCI firewall hardening;
- boot/restart/recovery rehearsal;
- object-storage E2E;
- final RBAC;
- formal Support → WISE operational hand-off;
- context-aware production entry-point routing.

---

# 35. Temporary dress-rehearsal changes that must not silently become production assumptions

The following are deliberately temporary or validation-specific:

| Change | Status |
|---|---|
| Public OCI IP as infrastructure address | Validation-environment value; not a patient-facing URL |
| Self-signed IP TLS certificate | Historical validation-only; trusted hostname TLS is now active |
| TCP 443 open broadly | Validation-only; harden for production |
| Telegram POC bot | Dress rehearsal only |
| POC bot webhook on OCI IP | Replaced by stable hostname webhook |
| GHCR SHA image selected for x86_64 | Retain immutable-image principle; production image decision later |
| SuperAdmin validation user | Rework under RBAC |
| Chatwoot local/default assignment policy | Pilot configuration; final policy to be established |
| Welcome message configuration | Rehearsal capability; final copy/process TBD |
| Current public OCI IP | May change |
| Direct IP browser access with certificate warning | Historical validation-only |

---

# 37. WISE Health Website Support Inbox created

A second Chatwoot channel has now been configured for the next native WISE Health web-support slice.

Configured Website Inbox:

- **Inbox name:** `WISE Health™ Support`
- **Website domain:** `https://wisehealth.in`
- **Welcome heading:** `WISE Health™ Support`
- **Welcome message:** configured for WISE Health care/support context
- **Sender mode:** Friendly
- **Sender identity:** WISE Health
- **Collaborators:** configured
- **Widget preview:** verified in Chatwoot
- **Launcher:** enabled/configured
- **Conversation/greeting settings:** configured for the pilot

The Chatwoot Website Inbox has generated its own website token and installation script.

### Credential handling

The website token is a secret and is **not recorded in this document, Git, screenshots, or chat context**.

The generated Chatwoot script is treated as provider-specific implementation detail. The token should be supplied only through the appropriate runtime/environment configuration when the web integration is implemented.

### Architecture boundary

The native WISE Health `/support` page belongs to the core `quick-chat-landing` application.

The Chatwoot-specific Website Inbox configuration and provider implementation remain owned by:

`wisedoctor/wise-support-chat`

The core application should not absorb a broad Chatwoot integration or duplicate Chatwoot configuration.

### Current status

The Website Inbox configuration is complete enough to begin the **native WISE Health `/support` integration**.

What is **not yet proven**:

- `wisehealth.in/support` native page;
- browser → Chatwoot Website Inbox;
- web conversation → configured Support Rep;
- Support Rep → browser reply;
- conversation persistence/re-entry;
- assignment behaviour for Website conversations.

These become the next web-support validation slice.

---

# 36. Environment state at the end of dress rehearsal

The hosted rehearsal currently represents:

```
Patient Telegram
      ↓
@wise_chatwoot_poc_bot
      ↓
https://support.wisehealth.in
      ↓
OCI Security List / firewalld
      ↓
Nginx trusted TLS termination
      ↓
Chatwoot Web :3000
      ↓
Chatwoot Telegram Channel
      ↓
Sidekiq
      ↓
PostgreSQL + pgvector / Redis
      ↓
Support Inbox
      ↓
Support Rep via https://support.wisehealth.in
      ↓
Telegram patient
```

The stable hostname is now the operational public ingress for the rehearsal.

The Chatwoot support-rep portal is also available at:

```
https://support.wisehealth.in
```

The Telegram POC webhook is:

```
https://support.wisehealth.in/webhooks/telegram/<bot-token>
```

Telegram verification after migration:

- `has_custom_certificate: false`;
- `pending_update_count: 0`;
- IP address: `140.245.237.47`.

The two-way text E2E was rerun after this migration and passed.

This is still **not the final production topology**. The hostname is stable for the validation environment, while the underlying OCI IP may change.

---

# 37. Relationship to other documents

Use this document when asking:

> What changes did we actually make to establish the hosted dress rehearsal?

Use:

- `WISE-SUPPORT-ENVIRONMENT-REFERENCE.md` for the current environment/infrastructure reference and architectural environment decisions.
- `WISE-SUPPORT-STABLE-HTTPS-INGRESS-AND-TLS-RUNBOOK-2026-10-02.md` for the stable hostname, certificate issuance/renewal and Nginx TLS procedure.
- `WISE-SUPPORT-PILOT-PREP-BACKLOG.md` for open pilot work and sequencing.
- `WISE-SUPPORT-PILOT-PREP-HANDOVER-2026-10-02.md` for new-session context and continuation instructions.
- `WISE_SUPPORT_CURRENT_ENVIRONMENT_AND_MAINTENANCE_RUNBOOK.md` for local/development diagnostics and maintenance.
- `WISE-SUPPORT-CHATWOOT-TELEGRAM-FALLBACK-ARCHITECTURE.md` for the deferred fallback/restore architecture.

---

# 38. Final transition rule

The dress rehearsal has served its purpose:

**prove the support communication technology and expose the operational gaps.**

The next work should therefore move toward:

```
Pilot-prep closure
       ↓
RBAC
       ↓
Initial Support → WISE hand-off
       ↓
Production entry-point audit/design
       ↓
Controlled production migration
```

Do not continue changing the rehearsal environment merely for experimentation once a pilot-prep item has enough evidence to proceed to the next milestone.
 Temporary dress-rehearsal changes that must not silently become production assumptions

The following are deliberately temporary or validation-specific:

| Change | Status |
|---|---|
| Public OCI IP as Chatwoot endpoint | Validation-only |
| Self-signed IP TLS certificate | Validation-only |
| TCP 443 open broadly | Validation-only; harden for production |
| Telegram POC bot | Dress rehearsal only |
| POC bot webhook on OCI IP | Dress rehearsal only |
| GHCR SHA image selected for x86_64 | Retain immutable-image principle; production image decision later |
| SuperAdmin validation user | Rework under RBAC |
| Chatwoot local/default assignment policy | Pilot configuration; final policy to be established |
| Welcome message configuration | Rehearsal capability; final copy/process TBD |
| Current public OCI IP | May change |
| Direct IP browser access with certificate warning | Validation-only |

---

# 36. Environment state at the end of dress rehearsal

The hosted rehearsal currently represents:

```
Patient Telegram
      ↓
@wise_chatwoot_poc_bot
      ↓
https://140.245.237.47
      ↓
OCI Security List / firewalld
      ↓
Nginx TLS termination
      ↓
Chatwoot Web :3000
      ↓
Chatwoot Telegram Channel
      ↓
Sidekiq
      ↓
PostgreSQL + pgvector / Redis
      ↓
Support Inbox
      ↓
Support Rep
      ↓
Telegram patient
```

This is the **current proven hosted dress-rehearsal path**.

It is not the final production topology.

---

# 37. Relationship to other documents

Use this document when asking:

> What changes did we actually make to establish the hosted dress rehearsal?

Use:

- `WISE-SUPPORT-ENVIRONMENT-REFERENCE.md` for the current environment/infrastructure reference and architectural environment decisions.
- `WISE-SUPPORT-PILOT-PREP-BACKLOG.md` for open pilot work and sequencing.
- `WISE-SUPPORT-PILOT-PREP-HANDOVER-2026-10-02.md` for new-session context and continuation instructions.
- `WISE_SUPPORT_CURRENT_ENVIRONMENT_AND_MAINTENANCE_RUNBOOK.md` for local/development diagnostics and maintenance.
- `WISE-SUPPORT-CHATWOOT-TELEGRAM-FALLBACK-ARCHITECTURE.md` for the deferred fallback/restore architecture.

---

# 38. Final transition rule

The dress rehearsal has served its purpose:

**prove the support communication technology and expose the operational gaps.**

The next work should therefore move toward:

```
Pilot-prep closure
       ↓
RBAC
       ↓
Initial Support → WISE hand-off
       ↓
Production entry-point audit/design
       ↓
Controlled production migration
```

Do not continue changing the rehearsal environment merely for experimentation once a pilot-prep item has enough evidence to proceed to the next milestone.
