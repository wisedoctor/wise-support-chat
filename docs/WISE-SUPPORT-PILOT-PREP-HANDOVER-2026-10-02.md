# WISE Support Pilot Prep — Session Handover / Seed Context

**Prepared:** 2026-10-02  
**Latest checkpoint:** WISE Health Website Support Inbox configured; native `/support` integration is next  
**Repository:** wisedoctor/wise-support-chat  
**Branch:** develop  
**Purpose:** Continue WISE Support pilot preparation in a fresh ChatGPT session without redoing the OCI/Chatwoot dress rehearsal investigation.

---

## 1. Current objective

We are moving from a successful **Chatwoot + Telegram technical dress rehearsal** toward a controlled **WISE Support pilot**.

The immediate goal is not to build the full future WISE context-aware router yet. The current sequence is:

1. close the remaining pilot-prep / smoke-test gaps;
2. move to **RBAC + initial operational hand-off**;
3. then continue production entry-point and context-aware routing work deliberately.

The larger product vision is:

    discover / explore
            ↓
    self-serve where possible
            ↓
    clear intent → relevant WISE capability
            ↓
    unclear / blocked / operational issue
            ↓
    context-aware WISE Support
            ↓
    human resolution

Support should therefore be a **human-resolution path**, not a generic destination for every CTA.

---

## 2. Architecture direction already established

WISE Support is intended to be channel/provider agnostic.

Current provider:
- self-hosted Chatwoot.

Current rehearsal channel:
- Telegram.

Future channels can include:
- WISE web chat;
- WhatsApp;
- email;
- other channels.

The intended conceptual separation remains:

- apps own authoritative workflows;
- support handles conversations, identity/context, routing and human servicing;
- bridges handle integrations;
- support must not become a duplicate CRM or duplicate authoritative domain database.

The previously documented fallback architecture also remains intentionally deferred:
- Chatwoot-backed path;
- direct Telegram fallback path;
- explicit webhook/polling ownership;
- RBAC and operational controls before implementation.

---

## 3. Two Telegram bots — keep strictly separate

### Dress-rehearsal bot
@wise_chatwoot_poc_bot

Used for:
- OCI Chatwoot dress rehearsal;
- Telegram ↔ Chatwoot E2E;
- current pilot-prep testing.

### Existing WISE Support bot
@wescura_support_bot

Used by the existing WISE backend support path.

Credential:
- TELEGRAM_SUPPORT_BOT_TOKEN

**Never mix these two bots or their webhook ownership.**

The POC token has been exposed during diagnostics and should be rotated before any production use.

Final patient-facing bot identity remains deferred.

---

## 4. OCI Chatwoot environment — current known-good state

OCI VM:

- public IP: 140.245.237.47
- Oracle Linux Server 9.8 x86_64
- 1 OCPU / 16 GB RAM
- Docker installed
- private Docker network: wise-support-net

Containers:

- wise-support-chat-web
- wise-support-chat-worker
- wise-support-postgres
- wise-support-redis

Chatwoot image:

    ghcr.io/wisedoctor/wise-support-chat:sha-a6b2176

Verified image:
- linux/amd64
- Ruby 3.4.4
- Rails 7.2.3.1
- Chatwoot v4.14.2 lineage

Postgres:
- pgvector image;
- migrations complete;
- 107 public tables observed;
- all migrations up.

Redis:
- persistent AOF;
- reachable from Chatwoot image.

Chatwoot web:
- Rails/Puma listening on port 3000 internally.

Public validation:
- Nginx terminates HTTPS on port 443;
- temporary self-signed certificate for IP 140.245.237.47;
- OCI security list TCP/443 currently opened for validation;
- firewalld HTTPS enabled;
- SELinux httpd_can_network_connect enabled.

This is a validation/dress-rehearsal environment, not yet production hardened.

---

## 5. Telegram webhook currently used for POC

The POC bot was moved from the old public WISE path to the OCI Chatwoot endpoint:

    https://140.245.237.47/webhooks/telegram/<bot-token>

Telegram getWebhookInfo confirmed:

- webhook active;
- custom certificate enabled;
- IP = 140.245.237.47;
- pending updates cleared.

Do not copy or expose the actual bot token.

Webhook ownership remains an important operational guardrail because Telegram allows only one active webhook per bot.

---

## 6. Proven two-way text E2E

The following path is proven:

    Patient Telegram
        ↓
    Telegram webhook
        ↓
    OCI Nginx / HTTPS
        ↓
    Chatwoot Telegram channel
        ↓
    Chatwoot / Sidekiq
        ↓
    Conversation
        ↓
    Support-rep reply
        ↓
    Chatwoot Telegram adapter
        ↓
    Patient Telegram

Proven behaviours:

- inbound Telegram text reaches Chatwoot;
- contact is created/identified;
- conversation is created;
- Sidekiq processes Telegram events;
- support rep can reply from Chatwoot;
- reply reaches patient Telegram.

This is now the baseline regression path.

---

## 7. Important assignment investigation — what we learned

The assignment investigation was deeper than simply checking the green availability dot.

Chatwoot 4.14.2 source showed:

    inbox.auto_assignment_v2_enabled?
    inbox.enable_auto_assignment?
    inbox.available_agents

The assignment service then selects an available agent and assigns the conversation atomically.

Most important finding:

    def available_agents
      online_agent_ids = fetch_online_agent_ids
      return inbox_members.none if online_agent_ids.empty?

      inbox_members
        .joins(:user)
        .where(users: { id: online_agent_ids })        .includes(:user)
    end

and:

    OnlineStatusTracker.get_available_users(account_id)
      .select { |_key, value| value.eql?('online') }

Online presence is therefore materially relevant to auto-assignment.

The browser's Chatwoot ActionCable connection showed:

- RoomChannel subscribed;
- regular update_presence calls;
- presence updates being broadcast;
- browser heartbeat/ping traffic.

The connector source showed:

- PRESENCE_INTERVAL = 20000 ms;
- update_presence sent through ActionCable;
- RoomChannel calls OnlineStatusTracker.update_presence.

OCI Redis inspection subsequently showed:

    PRESENCE_KEY=ONLINE_PRESENCE::1::USERS
    PRESENCE=["1"]
    STATUS="online"

and later:

    USERS={"1" => "online"}
    SCORE=<recent timestamp>
    AGE=15

So the browser session was genuinely establishing current online presence.

### Crucial operational observation

At one diagnostic point:

    AVAILABLE_USERS={}
    AVAILABLE_AGENT_IDS=[]

even though the user-level status was online.

This explains why assignment could initially remain Unassigned: the availability path is not simply the static UI green dot.

Later, after the browser presence was active, a conversation was visibly assigned:

> Assigned to Sreedhar Byreeka by Default Policy

and appeared under My Inbox / Assigned to you.

This was **not manually pushed via a script**.

The assignment was performed by Chatwoot's own automatic-assignment path, using the configured Default Policy.

The worker-side AssignmentJob and AssignmentService were also inspected.

### Important testing nuance

A follow-up message on an already-open conversation does not necessarily create a new conversation. Therefore, testing assignment using subsequent messages on the same open conversation is not equivalent to testing assignment of a genuinely new conversation.

For a clean assignment test:
1. use a new patient/contact or resolve the previous conversation;
2. create a genuinely new conversation;
3. ensure the rep's ActionCable presence is active;
4. observe AssignmentJob / My Inbox / Unassigned.

---

## 8. Current new-patient rehearsal

A second Android device was used to test the new CTA with a new patient context.

Patient Telegram:

    /start
    Need medicines for my son

Support replied:

    do you have Rx?

Chatwoot showed:

- new contact: Durga;
- conversation #2;
- correct Telegram POC inbox;
- conversation currently visible under Unassigned in the latest screenshot;
- support rep can respond from Chatwoot.

This is a particularly useful pilot scenario because it exercises:

**new user → medicine intent → human support**

rather than merely continuing the original test conversation.

Do not treat the current Unassigned state as evidence that auto-assignment is completely broken; the preceding investigation demonstrated that assignment depends on the online/presence path and timing. Continue diagnosing systematically.

---

## 9. Attachment/media gap — pilot blocker

Text is proven, but attachments are not.

Observed failures:

### Chatwoot → Telegram .txt
A simple outbound text-file attachment:
- did not reach the patient;
- Chatwoot displayed the outbound message as failed/red.

### Chatwoot → Telegram Rx image
A prescription-sheet image/photo:
- did not reach the patient;
- also showed failure/red.

Treat these as a potentially broader **attachment/media handling** issue.

Investigate before marking the channel pilot-ready:

- Chatwoot attachment upload;
- Active Storage/object storage configuration;
- generated attachment URL;
- public accessibility of attachment;
- OCI/Nginx HTTPS;
- Telegram document/photo API handling;
- Chatwoot Telegram channel adapter;
- Sidekiq worker logs;
- Telegram API response.

Potential validation matrix:

| Direction | Text | Image/Rx | Document |
|---|---|---|---|
| Patient → Chatwoot | proven | verify | verify |
| Chatwoot → Patient | proven | failing | failing |

---

## 10. Welcome message / first-contact UX

The current Chatwoot Telegram channel demonstrates that a configurable welcome/onboarding message is available.

This is now considered an architectural capability, not merely UI copy.

Desired future behaviour:

- explain what WISE Support can help with;
- make the first interaction friendly;
- capture the minimum useful context;
- avoid asking the patient to repeat context already known from the entry point;
- establish expectations for human support.

Potential context:

- source / entry point;
- current WISE ecosystem surface;
- location/pincode where relevant;
- medicine/request context;
- existing request/booking reference;
- what the user wants to accomplish.

The eventual WISE Support abstraction should expose an equivalent welcome/onboarding capability even if Chatwoot is replaced later.

---

## 11. Telegram /start UX

The current Telegram first-contact flow exposes /start.

Open questions:

1. Can the explicit user-facing /start action be avoided?
2. If not, what can be customized around it?
3. Can holding-page/CTA text explain it naturally?
4. Can the welcome message make the interaction feel like "Start Support" rather than a technical bot command?
5. Can unnecessary "bot" terminology be removed from the user-facing experience?

Do not assume Telegram limitations until verified against the actual implementation.

---

## 12. Context-aware CTA / smart routing vision

The earlier WISE/Wescura design work envisioned a smarter routing layer rather than hard-coded destinations.

Relevant concepts include:

- prospects / exploratory curiosity;
- actual intent;
- medicine-request intent;
- broader WISE ecosystem discovery;
- direct registration/offer/workflow entry where appropriate;
- location/serviceability;
- current WISE capabilities;
- existing journey/source context;
- Support as the human-resolution fallback.

The current /smart route in quick-chat-landing is an important historical placeholder for this idea.

The future direction is:

    Who + What + Where
           +
    Why / When where available
           ↓
    WISE context / intent router
           ↓
    self-serve / relevant offer / registration / workflow
           OR
    context-aware Support

The immediate pilot should **not** implement the full router prematurely.

First audit the production entry points and current hard-coded destinations.

---

## 13. Production entry-point audit — current planned work

Inventory:

- current holding page;
- /smart;
- QR destinations;
- Wescura Medicines entry points;
- future WISE Health web chat widget;
- WISE Doctor entry points;
- existing-request/status links;
- campaign/deep-link entry points.

For each determine:

- current destination;
- intended audience/intent;
- context already available;
- self-serve option;
- Support fallback;
- future smart-routing opportunity;
- whether current behaviour should remain unchanged during pilot.

**Do not migrate production code merely because the smarter architecture exists.**

Audit first → design transition → implement deliberately.

---

## 15. Native WISE Health web-support checkpoint

A dedicated Chatwoot Website Inbox has now been created for the WISE Health portal.

Configured state:

- Inbox: `WISE Health™ Support`
- Website domain: `https://wisehealth.in`
- Welcome heading: `WISE Health™ Support`
- Welcome message: configured
- Friendly sender identity: WISE Health
- Collaborators: configured
- Widget preview: verified
- Website token: generated; secret is not recorded in Git or this handover

This is configuration readiness only. The following browser E2E path is still to be proven:

```
wisehealth.in/support
      ↓
native WISE Health Support page
      ↓
Start Support
      ↓
Chatwoot Website Inbox
      ↓
WISE Health™ Support
      ↓
Support Rep
      ↓
browser reply
```

### Repository boundary

Keep the existing architecture decision explicit:

- `wisedoctor/quick-chat-landing` owns the WISE Health application, native `/support` page, presentation/navigation and entry-point experience.
- `wisedoctor/wise-support-chat` owns Chatwoot/provider-specific support implementation and operational configuration.
- Do not copy the earlier provider-specific `ChatwootWidget.tsx` experiment wholesale into the core repo.
- Do not commit the Website Inbox token.
- Telegram POC and existing `@wescura_support_bot` remain separate and must not be changed as part of this web-support slice.

### Next implementation sequence

1. In `quick-chat-landing`, establish the native `/support` route/page from the current application branch/state.
2. Keep the page's support-launch interface provider-neutral at the application boundary.
3. Connect that interface to the configured Chatwoot Website Inbox using secret/runtime configuration rather than source-controlled credentials.
4. Deploy to the appropriate WISE Health validation environment.
5. Prove browser → Website Inbox → Support Rep → browser reply.
6. Verify assignment, persistence/re-entry and refresh behaviour.
7. Run the existing Telegram two-way text regression without changing its configuration.
8. Record the E2E result before moving to the next pilot-prep item.

### Explicit non-goals for this slice

- no full context-aware router;
- no production Telegram bot migration;
- no replacement of the existing Telegram POC path;
- no broad support-provider abstraction implementation beyond what is required to keep the application boundary clean;
- no production entry-point migration before the audit.

---

## 14. RBAC + initial hand-off milestone

After the immediate five pilot-prep items are sufficiently closed, move into the first operational milestone.

### RBAC

Define:

- Support Rep;
- Support/Ops supervisory roles;
- administrative/platform roles;
- channel/inbox administration permissions;
- view/claim/assign/reply/resolve/private-note capabilities;
- boundary between Chatwoot permissions and WISE domain permissions;
- audit requirements.

### Initial hand-off

Prove the first controlled:

    Support conversation
          ↓
    known context / intent
          ↓
    appropriate WISE operational module
          ↓
    normal authoritative workflow
          ↓
    audited outcome


---

## 20. Latest verified stable-hostname checkpoint

The stable HTTPS migration is now complete for the Telegram POC rehearsal.

Verified:

- `support.wisehealth.in` resolves to the OCI validation IP `140.245.237.47`;
- trusted Let's Encrypt TLS is active;
- the Chatwoot support-rep portal is reachable at `https://support.wisehealth.in`;
- the POC bot `@wise_chatwoot_poc_bot` webhook is now:
  `https://support.wisehealth.in/webhooks/telegram/<bot-token>`;
- Telegram reports `has_custom_certificate: false`;
- Telegram reports `pending_update_count: 0`;
- two-way Telegram ↔ Chatwoot text E2E was rerun after the hostname migration and passed.

Operationally, this removes the cryptic raw-IP URL from the support-rep portal and establishes the stable hostname as the rehearsal's public support ingress.

The raw IP remains an underlying validation-environment address only. It must not be treated as a patient-facing URL or permanent production infrastructure identity.

Related environment log update:

- `docs/WISE-SUPPORT-DRESS-REHEARSAL-ENVIRONMENT-CHANGELOG-2026-10-02.md`
- commit: `f2ef66c5f7bcf30658ec9f355480e3b4aca2d0e4`



## 22. Latest attachment/storage checkpoint — 2026-10-02

Before making any Active Storage change, the rehearsal state was captured.

- WEB container Mounts: []
- WORKER container Mounts: []
- WEB /app/storage contains only three 8-byte objects from recent patient_guidance_test_doc.txt uploads.
- WORKER contains no corresponding storage files.
- The three current WEB objects are identical test payloads.
- PostgreSQL contains additional blob metadata not fully represented in the current WEB filesystem.
- Rails has already produced ActiveStorage::FileNotFoundError for a DB-referenced blob.

Therefore attachment/media is now understood as both a storage-sharing problem and an existing DB/filesystem consistency problem.

Do not recreate containers or mount an empty shared volume over /app/storage yet.

Next safe sequence:
preserve/correlate existing files → establish persistent shared storage → mount into WEB + WORKER → verify → rerun media E2E.

The dedicated smoke-test record is:
docs/WISE-SUPPORT-E2E-SMOKE-TEST-2026-10-02.md


## 23. Active Storage OCI remediation checkpoint — 2026-10-02

The storage investigation has now reached a clean technical checkpoint.

### Proven independently

1. Existing OCI bucket is empty and available as the selected persistent-storage candidate.
2. S3-compatible access from the actual Chatwoot WEB container passed LIST/PUT/HEAD/GET/DELETE.
3. Rails Active Storage itself successfully used `ActiveStorage::Service::S3Service` against the OCI bucket.
4. AWS SDK S3 version in the deployed image is `1.208.0`.
5. Temporary probe objects were deleted and the bucket was left empty.

### Current live runtime remains unchanged

The persistent WEB and WORKER containers have **not yet been switched** from local DiskService to S3-compatible storage.

Therefore:

- current text E2E remains the baseline;
- Telegram webhook ownership remains unchanged;
- PostgreSQL/Redis remain unchanged;
- Nginx/TLS remains unchanged;
- no production-facing bot has been touched.

### Next exact step

Before the persistent storage change, capture the exact current container creation configuration for WEB and WORKER, including:

- image;
- environment variables (redacting secrets);
- network;
- port mapping;
- command;
- restart policy;
- dependencies/links;
- existing PostgreSQL/Redis hostnames;
- existing SECRET_KEY_BASE handling.

Then recreate only WEB and WORKER with the same configuration plus:

```
ACTIVE_STORAGE_SERVICE=s3_compatible
STORAGE_ACCESS_KEY_ID=<OCI access key>
STORAGE_SECRET_ACCESS_KEY=<OCI secret key>
STORAGE_REGION=ap-hyderabad-1
STORAGE_BUCKET_NAME=oracle-oci-bucket-chatwoot-wisehealth
STORAGE_ENDPOINT=https://ax3kknx0qqfm.compat.objectstorage.ap-hyderabad-1.oraclecloud.com
STORAGE_FORCE_PATH_STYLE=true
```

The secret values must never enter Git, documentation or chat.

After recreation, verify both WEB and WORKER report `ActiveStorage::Service::S3Service` before testing attachments.

### Required regression order

1. Container health / Chatwoot portal.
2. Rails Active Storage service class in WEB + WORKER.
3. Fresh Chatwoot attachment upload/read.
4. Telegram two-way text regression.
5. Document attachment E2E.
6. Rx-image attachment E2E.
7. Patient → Chatwoot attachment validation.

Do not change Telegram webhook configuration as part of this storage remediation.


## 24. Persistent Active Storage remediation checkpoint — 2026-10-02

The controlled storage remediation has now been completed at the runtime-configuration level.

### Evidence preserved

Before the WEB replacement, the three recoverable local diagnostic objects were copied to the VM backup location:

`~/wise-support-storage-backup-20261002/`

They remain forensic evidence only and were not copied into the OCI bucket.

The prior WEB and WORKER containers were preserved as:

- `wise-support-chat-web-disk-20261002`
- `wise-support-chat-worker-disk-20261002`

### Live WEB verification — PASS

The replacement WEB container uses:

```
ACTIVE_STORAGE_SERVICE=s3_compatible
STORAGE_REGION=ap-hyderabad-1
STORAGE_BUCKET_NAME=oracle-oci-bucket-chatwoot-wisehealth
STORAGE_ENDPOINT=https://ax3kknx0qqfm.compat.objectstorage.ap-hyderabad-1.oraclecloud.com
STORAGE_FORCE_PATH_STYLE=true
```

Secret values are not recorded.

Rails reports:

```
service=s3_compatible
class=ActiveStorage::Service::S3Service
```

A live Rails probe from WEB successfully uploaded, read/downloaded and deleted an object; deletion was verified.

### Live WORKER verification — PASS

The replacement WORKER uses the same Active Storage service configuration and reports:

```
service=s3_compatible
class=ActiveStorage::Service::S3Service
```

Sidekiq started normally and resumed scheduled/background processing.

### Important current distinction

The storage remediation is **technically configured and verified**, but the attachment capability is **not yet marked proven**. The next message/test cycle will establish whether the original Chatwoot → Telegram document/Rx-image failures are resolved end-to-end.

### Required next regression order

1. Chatwoot portal/runtime health.
2. Telegram two-way text regression.
3. Chatwoot → Telegram document attachment.
4. Chatwoot → Telegram Rx-image attachment.
5. Patient → Chatwoot attachment validation.
6. Inspect WEB/WORKER logs only where a media test fails.

Do not alter Telegram webhook ownership during this validation.

### Security follow-up

The OCI secret key used during the remediation was exposed during diagnostics. Rotate it after functional validation, then update WEB/WORKER configuration and re-verify storage access. The secret is not recorded in this handover.


## 25. Latest media E2E checkpoint — 2026-10-02

The first attachment tests after persistent OCI Active Storage remediation have materially narrowed the problem.

### Proven now

- Chatwoot → Telegram document attachment delivery: **PASS**.
- Chatwoot → Telegram image attachment delivery: **PASS**.

Therefore the shared OCI Active Storage backend is demonstrably usable for outbound attachment delivery.

### Still failing

**Telegram → Chatwoot document:** message arrives, but the attachment cannot be opened/downloaded.

A direct object request produced OCI `NoSuchKey` for the referenced object key. This proves the requested object was not present at that key when accessed, but does not yet establish where the inbound pipeline lost or mis-keyed the object.

**Telegram → Chatwoot image:** message arrives, but the attachment cannot be downloaded/rendered.

### New duplicate-message finding

The same inbound attachment message is being displayed multiple times. One document test showed three copies of the same inbound message. Refreshing the **Mine** inbox appeared to increase the visible count further; the same pattern was seen for the image test.

Do not clean up records yet. First establish whether the duplication is caused by repeated Telegram update processing, repeated Sidekiq execution, duplicate DB message persistence, API response/pagination behaviour, frontend refresh/render behaviour, or a combination.

### Next exact investigation

Use one fresh inbound document and one fresh inbound image and capture, before/after a single inbox refresh:

1. Telegram `update_id` / webhook evidence.
2. Rails webhook + Sidekiq logs for that update.
3. Chatwoot message IDs and timestamps for the resulting copies.
4. Active Storage blob IDs/keys for the attachment.
5. OCI object listing/head for the exact blob key.
6. Browser network response for the conversation/inbox refresh.

The goal is to distinguish **object persistence/keying failure** from **duplicate processing/rendering** before changing code or data.

### Current gate

The attachment gate remains **OPEN**, but its scope has changed:

- outbound media delivery: now proven for document + image;
- inbound media delivery/retrieval: failing;
- inbound duplicate-message behaviour: investigation required.

Telegram two-way text remains the regression baseline and must continue to pass during this investigation.
