# WISE Support — E2E Smoke Test Record — 2026-10-02

Repository: wisedoctor/wise-support-chat
Branch: develop
Environment: OCI hosted dress rehearsal
Public ingress: https://support.wisehealth.in
POC channel: @wise_chatwoot_poc_bot

## Purpose

Record the current smoke-test evidence for the hosted WISE Support rehearsal and preserve the regression baseline while attachment/storage and web-support work remain open.

## Stable-hostname smoke test

- DNS to OCI validation ingress: PASS
- Trusted Let's Encrypt TLS: PASS
- Chatwoot portal via stable hostname: PASS
- POC Telegram webhook on stable hostname: PASS
- Telegram custom certificate after migration: false, as expected with trusted hostname TLS
- Telegram pending updates after migration: 0
- Raw IP as patient-facing endpoint: NO; validation infrastructure only

The POC webhook is now the stable-hostname path:
support.wisehealth.in/webhooks/telegram/<bot-token>

The actual bot token is intentionally not recorded.

## Telegram two-way text E2E

Inbound path:
Patient Telegram → POC bot → support.wisehealth.in → Nginx → Chatwoot Telegram channel → Sidekiq → conversation/message

Result: PASS.

Outbound path:
Support Rep → Chatwoot → Telegram adapter → POC bot → Patient Telegram

Result: PASS.

This two-way text path was rerun after the stable-hostname migration and remains the regression baseline.

## New-patient scenario

A separate Android patient context was tested with:

/start
Need medicines for my son

Support replied:

do you have Rx?

Chatwoot showed a new contact and conversation in the POC Telegram inbox.

Result: PASS for conversation creation and servicing.

Assignment behaviour remains separately tracked because a genuinely new-conversation assignment test must distinguish agent presence/availability from static UI status.

## Assignment smoke evidence

Chatwoot automatic assignment was investigated through its availability/presence path.

Observed successful system-generated assignment:

Assigned to Sreedhar Byreeka by Default Policy

No manual assignment script was used for that successful assignment.

Current status: assignment path exercised; clean repeatable new-conversation test remains open.

## Attachment/media smoke test

| Direction | Text | Image/Rx | Document |
|---|---|---|---|
| Patient → Chatwoot | PASS | VERIFY | VERIFY |
| Chatwoot → Patient | PASS | FAIL | FAIL |

Outbound .txt and Rx image attempts were displayed as failed/red and did not reach Telegram.

## Active Storage diagnosis

Current container inspection showed:

wise-support-chat-web    Mounts: []
wise-support-chat-worker Mounts: []

WEB currently has only three 8-byte Active Storage files under /app/storage. All three are identical test payloads from recent patient_guidance_test_doc.txt uploads.

WORKER has no corresponding storage files.

PostgreSQL contains additional blob metadata not fully represented in the current WEB filesystem, and Rails has produced ActiveStorage::FileNotFoundError for a DB-referenced blob.

Status: OPEN / infrastructure defect.

Required safe sequence:
preserve/correlate existing files
→ establish persistent shared Active Storage backend
→ mount identically into WEB and WORKER
→ verify existing and new attachment reads
→ rerun Telegram media E2E

Do not replace the current /app/storage with an empty shared volume before existing files are preserved/correlated.

## WISE Health Website Inbox

Configured Chatwoot Website Inbox:

- Inbox: WISE Health™ Support
- Domain: https://wisehealth.in
- Friendly sender: WISE Health
- Collaborators configured
- Widget preview verified
- Website token generated and kept out of source control

Status: configuration PASS; browser E2E OPEN.

Target browser smoke path:
wisehealth.in/support → native WISE Health Support page → Start Support → Chatwoot Website Inbox → Support Rep → browser reply

## Regression guardrail

Any attachment/storage change must rerun the proven Telegram text path:

1. Patient → Chatwoot text.
2. Support Rep → Patient Telegram text.
3. Confirm conversation/inbox continuity.
4. Then test attachment/media.

Do not use attachment debugging as a reason to alter either Telegram bot identity or the existing WISE Support bot path.

## Current disposition

Text communication: PROVEN
Stable HTTPS ingress: PROVEN
Support-rep portal: PROVEN
New-patient conversation: PROVEN
Automatic assignment: EXERCISED; clean repeatability still open
Attachments/media: OPEN / FAILING
Shared Active Storage: OPEN
Website browser E2E: OPEN
RBAC: NEXT OPERATIONAL MILESTONE after pilot-prep closure

This is a point-in-time smoke-test record. Current infrastructure details should be cross-checked against the environment reference and dress-rehearsal changelog.


## Active Storage remediation checkpoint — 2026-10-02

The storage infrastructure remediation has now progressed from diagnosis to persistent runtime configuration.

### PASS

- Recoverable WEB local storage files preserved before replacement.
- Previous WEB and WORKER containers preserved for rollback.
- Replacement WEB running successfully with `ActiveStorage::Service::S3Service`.
- Live WEB Rails Active Storage PUT/EXIST/DOWNLOAD/DELETE probe passed.
- Replacement WORKER running successfully with `ActiveStorage::Service::S3Service`.
- WORKER Sidekiq scheduled/background processing resumed normally.
- PostgreSQL, Redis, Nginx, Telegram webhook ownership and bot identities were not changed by the storage remediation.

### Still open

The infrastructure change is **not itself an attachment E2E result**. The following remain to be tested:

| Direction | Text | Image/Rx | Document |
|---|---|---|---|
| Patient → Chatwoot | regression | verify | verify |
| Chatwoot → Patient | regression | pending | pending |

The user will provide the actual media-test results separately. Until then, attachment/media remains OPEN.

### Rollback retention

Keep these preserved containers until media regression is complete:

- `wise-support-chat-web-disk-20261002`
- `wise-support-chat-worker-disk-20261002`

### Security follow-up

The OCI secret key used during diagnostics must be rotated before production use; no secret is recorded in this document.


## 2026-10-02 media E2E results — post-OCI storage remediation

The first real attachment E2E was performed after switching WEB + WORKER to the shared OCI S3-compatible Active Storage backend.

### Chatwoot → Telegram — document attachment

**Result: PASS for delivery.**

A support-rep reply containing a `.txt` document was delivered successfully to the Telegram patient. The Telegram client displayed the document attachment.

### Chatwoot → Telegram — image attachment

**Result: PASS for delivery.**

A support-rep reply containing an image was delivered successfully to the Telegram patient and was visible in the patient Telegram conversation.

These are material improvements over the pre-remediation state, where outbound document and image attachments failed/red and did not reach Telegram.

### Telegram → Chatwoot — document attachment

**Result: PARTIAL / FAIL for attachment retrieval.**

A Telegram patient sent a document attachment and the message reached the Chatwoot portal. However, opening/downloading the attachment from Chatwoot failed.

A direct inspection of the generated OCI object URL returned an OCI `NoSuchKey` response stating that the requested object key was not found in bucket `oracle-oci-bucket-chatwoot-wisehealth`.

This proves the referenced object was not available at that requested key at retrieval time. The exact creation/upload path still needs to be traced.

### Telegram → Chatwoot — image attachment

**Result: PARTIAL / FAIL for attachment retrieval.**

A Telegram patient sent an image attachment and the message reached the Chatwoot portal, but the attachment could not be downloaded/rendered successfully in Chatwoot.

### Duplicate/repeated inbound attachment messages

The same inbound attachment message appeared multiple times in the Chatwoot conversation. In one observed case, a single Telegram document message resulted in three visible copies. Subsequent refreshes of the **Mine** inbox appeared to increase the number of visible attachment messages further; the same pattern was observed for the image attachment.

This is not yet classified as a frontend-only rendering issue or a duplicate Telegram update issue. It requires correlation of Telegram update IDs, Chatwoot webhook/Sidekiq processing, persisted message IDs/timestamps, conversation API responses before and after refresh, and browser/network requests during inbox refresh.

Do not alter or clean up records until the source of duplication is established.

### Updated attachment matrix

| Direction | Text | Image/Rx | Document |
|---|---|---|---|
| Patient → Chatwoot | PASS baseline | received but attachment retrieval FAIL | received but attachment retrieval FAIL |
| Chatwoot → Patient | PASS baseline | PASS delivery | PASS delivery |

### Current conclusion

The OCI shared Active Storage change has resolved the original outbound attachment delivery failure sufficiently to prove Chatwoot → Telegram document and image delivery.

It did not yet establish bidirectional attachment readiness. The remaining blocker is now concentrated on the Telegram → Chatwoot inbound attachment persistence/retrieval path, together with the newly observed duplicate-message behaviour.

The two-way text path remains the regression baseline.



## Telegram inbound retry / duplicate-message resolution checkpoint — 2026-10-02

The duplicate inbound attachment investigation now has a concrete causal explanation and application fix.

Root cause chain:
- one Telegram update was received and one Sidekiq job was enqueued;
- the first attempt could persist the message and then fail in the post-create callback while generating attachment URL data;
- the immediate trigger was the missing worker FRONTEND_URL;
- Sidekiq retried the same event;
- Telegram::IncomingMessageService had no retry/idempotency lookup;
- messages.source_id was indexed but not unique;
- the retry therefore created a second message with the same Telegram message_id.

The worker did not restart during the observed event.

Definite fix:
- branch: fix/telegram-retry-idempotency;
- draft PR: #1 — fix(telegram): make inbound retries idempotent;
- existing Telegram message is reused within the relevant contact/inbox scope;
- intact attachment is treated as already processed;
- missing attachment storage is repaired on the existing message;
- regression specs cover text dedupe, intact attachment dedupe and missing-object repair.

Current status:
- cause established: YES;
- fix implemented: YES;
- merged to develop: NO;
- CI/test suite completed: NOT YET;
- OCI rehearsal deployment: NOT YET;
- inbound media E2E after fix: NOT YET.

Attachment/media remains open until controlled validation is complete. Two-way Telegram text remains the regression baseline.

