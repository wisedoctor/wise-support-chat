# WISE Support — Current Environment & Maintenance Runbook

## 1. Purpose and scope

This document records the **currently proven development/local environment** for the WISE Support workstream, including infrastructure, processes, webhooks, diagnostics, operational workflows, and maintenance commands.

It is deliberately implementation-specific and sits **under** the WISE Support abstract architecture. Chatwoot and Telegram are treated here as the current implementation examples/providers used to prove the support-channel model; they are not the definition of WISE Support.

This runbook exists so that Product, Tech, and Ops can:

- recreate the current development environment;
- diagnose inbound and outbound support-message failures without rediscovering the local topology;
- distinguish channel, ingress, application, worker, and servicing-UI failures;
- reproduce an issue locally before fixing and redeploying it to production; and
- map the proven local components to their eventual production equivalents.

> **Current validation status:** Only the local/development environment described below has been proven end-to-end. Production deployment has **not** yet been validated by this runbook.

---

## 2. Architectural position

At the WISE Support level, the intended abstraction is:

```text
User / Partner
      |
      v
Channel Adapter / Ingress
      |
      v
WISE Support Engine
      |
      +--> Identity + User Context
      +--> Conversation / Interaction Context
      +--> Routing / Team Selection
      +--> Support Servicing
      +--> Contextual WISE referrals / next actions
      |
      v
Response through originating channel
```

The current implementation uses:

- **Telegram** as the currently proven external channel;
- **Chatwoot** as the current conversation/support-servicing implementation;
- **ngrok** as development-only public ingress;
- a small local **`wescura_proxy.py`** process as development routing glue;
- WISE backend/frontend processes alongside the Chatwoot stack.

These are replaceable implementation components. Future channels such as WISE Web Chat, WhatsApp, email, or other providers should plug into the same support abstraction rather than redefine it.

---

## 3. Proven local/development topology

The Windows 11 development machine uses WSL/Conda terminals for application processes and Docker containers for the Chatwoot infrastructure.

```text
                         INTERNET
                            |
                            v
              ngrok public HTTPS endpoint
                            |
                            v
                     localhost:8080
                            |
                  wescura_proxy.py
                    /               \
                   /                 \
                  v                   v
       Chatwoot Rails :3000       WISE backend :8000
              |                         |
              |                         |
        +-----+------+                  |
        |            |                  |
     Redis        PostgreSQL            |
        |                               |
        +--------- Sidekiq <-------------+

WISE frontend
    |
    +--> local Vite dev server
```

### Proven local processes observed

| Component | Current local endpoint/process | Role | Status proven |
|---|---|---|---|
| WISE backend | Uvicorn on `0.0.0.0:8000` | WISE application/backend | Local process observed |
| WISE frontend | Vite dev server | WISE web UI | Local process observed |
| Support proxy | `python /tmp/wescura_proxy.py` on `0.0.0.0:8080` | Routes public development webhooks | Proven |
| ngrok | `ngrok http 8080` | Public HTTPS ingress for development | Proven |
| Chatwoot Rails | Docker container `deployment-rails-1` | Chatwoot web/application layer | Proven |
| Chatwoot Sidekiq | Docker container `deployment-sidekiq-1` | Background jobs | Proven |
| Redis | Docker container `deployment-redis-1` | Queue/cache dependency | Proven |
| PostgreSQL | Docker container `deployment-postgres-1` | Chatwoot database | Proven |

Container image versions observed during the E2E validation:

- Chatwoot: `chatwoot/chatwoot:v4.14.2`
- Redis: `redis:alpine`
- PostgreSQL: `pgvector/pgvector:pg16`

The exact production versions and topology are **not yet frozen** and must be documented after production setup is validated.

---

## 4. Development-only ingress: ngrok + proxy

The local Chatwoot instance is not directly reachable from the public Internet. Telegram and external webhook callers therefore need a public HTTPS endpoint.

The current development chain is:

```text
Telegram / external webhook caller
        |
        v
https://<current-ngrok-host>
        |
        v
ngrok -> localhost:8080
        |
        v
/tmp/wescura_proxy.py
        |
        +----> Chatwoot :3000
        |
        +----> WISE backend :8000
```

### Start ngrok

Run in a dedicated terminal:

```bash
ngrok http 8080
```

The public hostname is dynamic in the current development setup. The exact hostname must be copied from the running ngrok process when registering/testing webhooks.

### Start the proxy

Run in a separate terminal:

```bash
python /tmp/wescura_proxy.py
```

The proxy listens on:

```text
0.0.0.0:8080
```

### Current proxy routing

The currently proven proxy routes POST requests as follows:

```python
if self.path.startswith("/webhooks/telegram/"):
    self.proxy("http://127.0.0.1:3000")
elif self.path.startswith("/api/webhooks/chatwoot"):
    self.proxy("http://127.0.0.1:8000")
```

Therefore the two important development webhook destinations are:

#### A. Telegram → Chatwoot

```text
https://<ngrok-host>/webhooks/telegram/<BOT_TOKEN>
```

This is routed by the proxy to:

```text
http://127.0.0.1:3000/webhooks/telegram/<BOT_TOKEN>
```

#### B. Chatwoot → WISE backend

```text
https://<ngrok-host>/api/webhooks/chatwoot
```

This is routed by the proxy to:

```text
http://127.0.0.1:8000/api/webhooks/chatwoot
```

> These are **development webhook destinations**. The ngrok hostname must not be treated as a production endpoint.

### Important troubleshooting note

An HTTP `404` at an ngrok path is not necessarily an ngrok problem. It can mean that the request reached the proxy but the requested path does not match one of the proxy's routing rules, or that the downstream application does not expose that route.

Always inspect:

1. ngrok request log;
2. proxy log;
3. Rails/backend log;
4. Sidekiq log where applicable.

---

## 5. Current Telegram configuration

The development E2E test uses a Telegram bot connected to Chatwoot. The bot token is stored in Chatwoot's Telegram channel configuration and must **never be committed to GitHub or placed in this document**.

The current Chatwoot Telegram channel was verified through Rails console as a `Channel::Telegram` record with an associated bot token and account.

### Webhook registration

Telegram's webhook is registered against the current public ngrok endpoint:

```text
https://<ngrok-host>/webhooks/telegram/<BOT_TOKEN>
```

When the ngrok hostname changes, the Telegram webhook must be updated accordingly for local testing.

Do not paste live bot tokens into tickets, documentation, commits, or chat transcripts.

---

## 6. Chatwoot current implementation

Chatwoot is currently the **support conversation/servicing implementation** used by WISE Support development.

The current local stack consists of:

```text
Rails
Sidekiq
Redis
PostgreSQL
```

The support representative uses the Chatwoot web UI to open the relevant Inbox and service the conversation.

The current proven Telegram flow is:

```text
PATIENT / USER
    |
    | Telegram message
    v
Telegram bot
    |
    | webhook
    v
ngrok -> proxy
    |
    v
Chatwoot Rails
    |
    v
Sidekiq / TelegramEventsJob
    |
    v
Chatwoot Contact / Conversation / Message
    |
    v
Chatwoot Inbox UI
```

For outbound support replies:

```text
Support representative
    |
    v
Chatwoot Inbox UI
    |
    v
Chatwoot Telegram channel
    |
    v
Telegram API
    |
    v
Patient / user Telegram chat
```

The E2E validation confirmed both directions.

---

## 7. Important distinction: notification vs canonical conversation

During the local E2E test, an inbound Telegram message appeared both:

1. in the Chatwoot Inbox/conversation; and
2. in the Wescura Support Telegram chat as a support notification containing Chatwoot conversation/message references.

The support reply from Chatwoot appeared in:

1. the Chatwoot conversation; and
2. the patient's Telegram chat;

but it did **not** appear in the Wescura Support Telegram notification chat.

This indicates that the Wescura Support Telegram chat is currently an **additional notification/integration surface**, not the canonical two-way support conversation.

This distinction must be preserved architecturally:

- the originating channel carries the user-facing conversation;
- the support servicing system holds/services the conversation;
- operational notification channels may receive selected events;
- a notification channel must not accidentally become the canonical support record.

---

## 8. Proven E2E validation

### Inbound test

A patient/user sent a message from Telegram using a test account.

The message was observed in:

- Telegram;
- Chatwoot Inbox;
- WISE/Wescura Support notification chat.

Sidekiq initially showed a channel-resolution failure because the webhook supplied the complete bot token while the lookup incorrectly treated it as a prefix.

The fix was:

```ruby
bot_token_prefix = params[:bot_token].to_s.split(':', 2).first

channel = Channel::Telegram
  .where("bot_token LIKE ?", "#{bot_token_prefix}:%")
  .first
```

The fix was committed as:

```text
caeebe204b fix(telegram): resolve channel by bot token prefix
159242f184 fix(telegram): extract bot id from webhook token
```

### Outbound test

A support representative sent:

```text
REVERSE E2E TEST 001 - SUPPORT REPLY (sent from chatwoot UI)
```

from the Chatwoot UI.

The message was recorded in Chatwoot as an outgoing message and was successfully delivered to the patient's Telegram chat.

Therefore the local environment has proven:

```text
User -> Telegram -> Chatwoot -> Support servicing
Support servicing -> Chatwoot -> Telegram -> User
```

### Current validation boundary

This proves the **local/development** path only.

It does not yet prove:

- production ingress;
- production Chatwoot hosting;
- production webhook URLs;
- production WISE backend/frontend integration;
- production secrets/configuration;
- production TLS/domain configuration;
- production worker/queue reliability;
- production persistence/backup/recovery.

Those must be validated separately during production setup.

---

## 9. Process and health checks

### Check Docker services

```bash
docker compose ps
```

If the Compose project uses generated container names, use the actual names shown by `docker ps`/`docker compose ps`.

Current observed names:

```text
deployment-rails-1
deployment-sidekiq-1
deployment-redis-1
deployment-postgres-1
```

### Check all containers

```bash
docker ps
```

### Check local application processes

```bash
ps aux | grep -E 'uvicorn|npm|ngrok|wescura_proxy'
```

### Check Rails container

```bash
docker ps --filter name=deployment-rails-1
```

### Check Sidekiq container

```bash
docker ps --filter name=deployment-sidekiq-1
```

### Check proxy

```bash
ps aux | grep wescura_proxy
```

### Check ngrok

```bash
ps aux | grep ngrok
```

---

## 10. Start / stop / restart reference

### Chatwoot stack

Use the repository's Docker Compose configuration from the Chatwoot working directory.

Start:

```bash
docker compose up -d
```

Stop:

```bash
docker compose down
```

Restart the stack:

```bash
docker compose restart
```

Check status:

```bash
docker compose ps
```

> If `docker compose` reports a service such as `rails` as not running, first inspect `docker compose ps` and `docker ps`. The actual container name may be `deployment-rails-1`, while the Compose service/project naming can differ depending on how the stack was started.

### Rails container restart

If the actual container is `deployment-rails-1`:

```bash
docker restart deployment-rails-1
```

### Sidekiq restart

```bash
docker restart deployment-sidekiq-1
```

### Development proxy

Stop the running process with `Ctrl-C` in its terminal, then restart:

```bash
python /tmp/wescura_proxy.py
```

### ngrok

Stop with `Ctrl-C`, then restart:

```bash
ngrok http 8080
```

> Restarting ngrok normally changes the public hostname. Re-check and, if necessary, re-register the Telegram webhook after restarting it.

---

## 11. Logs and diagnostics

### Rails

```bash
docker logs -f --tail=200 deployment-rails-1
```

### Sidekiq

```bash
docker logs -f --tail=200 deployment-sidekiq-1
```

This is the primary log for background processing such as `Webhooks::TelegramEventsJob`.

### Redis

```bash
docker logs -f --tail=200 deployment-redis-1
```

### PostgreSQL

```bash
docker logs -f --tail=200 deployment-postgres-1
```

### Proxy

The proxy writes request lines to its terminal, for example:

```text
[proxy] 127.0.0.1 "POST /webhooks/telegram/... HTTP/1.1" 200 -
```

### ngrok

The ngrok terminal shows the public request and HTTP result, for example:

```text
POST /webhooks/telegram/... 200 OK
```

or a route failure such as `404 Not Found`.

### Targeted Sidekiq search

```bash
docker logs deployment-sidekiq-1 --since 5m | grep -Ei 'telegram|conversation|message|error|discarded'
```

### Targeted Rails search

```bash
docker logs deployment-rails-1 --since 5m | grep -Ei 'telegram|webhook|conversation|message|error'
```

---

## 12. Rails console and shell

The local host does not necessarily have the Ruby/Bundler toolchain installed. The supported way for this Dockerized environment is to execute Rails inside the running Rails container.

### Rails console

```bash
docker exec -it deployment-rails-1 bundle exec rails console
```

Use this for database/model inspection and controlled diagnostics.

Examples used during the Telegram channel investigation:

```ruby
Channel::Telegram.column_names
Channel::Telegram.all
```

To inspect the stored bot token safely, avoid printing the complete token into logs or transcripts. Use only derived/non-secret values when debugging, for example a numeric bot ID/prefix and token length.

### Rails shell / application shell

Use the running Rails container for shell-level diagnostics:

```bash
docker exec -it deployment-rails-1 sh
```

From inside the container, application commands can be executed using the repository's bundled Ruby environment.

---

## 13. Standard troubleshooting workflow

When a support message does not appear, diagnose from the outside in rather than changing application code immediately.

### Inbound

```text
1. Did the originating channel receive/send the message?
        |
        v
2. Did ngrok/public ingress receive the webhook?
        |
        v
3. Did the local proxy receive and route it?
        |
        v
4. Did Chatwoot Rails receive it?
        |
        v
5. Did Sidekiq enqueue/process the job?
        |
        v
6. Was the channel resolved?
        |
        v
7. Was the contact/conversation created/found?
        |
        v
8. Did the message appear in the servicing UI?
```

### Outbound

```text
1. Did the support representative send from the servicing UI?
        |
        v
2. Was the outgoing message created successfully?
        |
        v
3. Did the channel/provider send it?
        |
        v
4. Did the originating channel receive it?
        |
        v
5. If not, inspect provider/API and worker logs.
```

This sequence should be followed before changing code or infrastructure.

---

## 14. Environment-specific configuration to capture

The following values are environment-specific and must be maintained separately for DEV and PROD. Secrets must live in the relevant secret/configuration store and **must not be committed to Git**.

### Development

Record/configure:

- WISE frontend local URL/port;
- WISE backend local URL/port;
- Chatwoot Rails URL/port;
- Redis connection/configuration;
- PostgreSQL connection/configuration;
- current ngrok hostname;
- proxy listening port;
- Telegram bot/channel configuration;
- Telegram webhook URL;
- Chatwoot → WISE webhook URL;
- any WISE backend environment variables required by the Chatwoot integration;
- any frontend environment variables required by the support UI.

The current known webhook paths are:

```text
Telegram -> Chatwoot:
/webhooks/telegram/<BOT_TOKEN>

Chatwoot -> WISE backend:
/api/webhooks/chatwoot
```

### Production mapping

Production must map each development dependency to a stable production equivalent:

| Development | Production mapping | Status |
|---|---|---|
| ngrok public hostname | Stable WISE-owned HTTPS ingress/domain | Not yet validated |
| `wescura_proxy.py :8080` | Production ingress/routing layer | Not yet frozen |
| Chatwoot Rails `:3000` | Production Chatwoot service | Not yet validated |
| Sidekiq container | Production worker service | Not yet validated |
| Redis container | Production Redis/service | Not yet validated |
| PostgreSQL container | Production PostgreSQL service | Not yet validated |
| WISE backend `:8000` | Production WISE backend | Not yet validated |
| Vite dev server | Production WISE frontend | Not yet validated |
| Telegram webhook | Stable production webhook URL | Not yet validated |
| Chatwoot → WISE webhook | Stable production integration URL | Not yet validated |

The production setup should preserve the **same logical webhook contracts and troubleshooting stages**, even if the infrastructure differs.

---

## 15. DEV → PROD issue reproduction workflow

The intended engineering workflow is:

```text
PROD issue
   |
   v
Capture:
- channel
- user/context
- timestamp
- conversation/message ID
- provider/service
- relevant logs
   |
   v
Reproduce in DEV using equivalent test identities/context
   |
   v
Diagnose at:
channel -> ingress -> application -> worker -> conversation -> outbound
   |
   v
Implement fix
   |
   v
Run local E2E regression
   |
   v
Commit + push
   |
   v
Deploy to PROD
   |
   v
Run controlled production verification
```

Environment-specific values should therefore be documented so that a production incident can be reproduced locally without rediscovering the entire topology.

---

## 16. Security and operational hygiene

- Never commit Telegram bot tokens, API keys, passwords, database credentials, Redis credentials, or production secrets.
- Do not paste full secrets into issue reports or support tickets.
- Use token length, bot ID/prefix, masked values, record IDs, timestamps, and environment labels when diagnosing.
- Treat ngrok as development infrastructure only.
- Do not point production Telegram webhooks at a developer laptop.
- Do not expose local Rails, Redis, or PostgreSQL ports directly to the Internet.
- Keep operational notification channels separate from the canonical support conversation.

---

## 17. Current known-good baseline

At the time this runbook was created, the local environment had successfully demonstrated:

- Telegram inbound webhook reception through ngrok;
- local proxy routing to Chatwoot;
- Chatwoot Telegram channel resolution;
- Sidekiq background processing;
- Chatwoot contact creation;
- Chatwoot conversation/message creation;
- visibility in Chatwoot Inbox;
- WISE/Wescura support-side inbound notification;
- support-agent reply from Chatwoot;
- outbound delivery to the patient/user's Telegram chat.

The relevant WISE repository is:

```text
wisedoctor/wise-support-chat
```

The current development branch is:

```text
develop
```

This runbook should be updated when the production topology is frozen and E2E validated.
