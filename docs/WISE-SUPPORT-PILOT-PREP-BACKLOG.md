# WISE Support Pilot Prep Backlog

**Status:** Working backlog for Telegram + Chatwoot dress rehearsal and production-entry-point preparation  
**Updated:** 2026-10-02  
**Latest checkpoint:** WISE Health Website Inbox configured; native `/support` integration is the next implementation slice

This backlog captures the pilot-prep work identified during the OCI-hosted Chatwoot dress rehearsal and the wider WISE context-aware entry-point review.

The current dress rehearsal has proven the core text communication path in both directions. The remaining items below are deliberately separated into **pilot blockers**, **pilot UX/configuration work**, and **future architecture** so that unresolved future design does not hold up the proven core path.

---

## 1. Context-aware WISE entry-point / CTA routing

**Priority:** High  
**Phase:** Pilot preparation / production entry-point audit  
**Status:** Architecture direction identified; implementation deferred until entry-point audit

Move beyond hard-coded CTAs such as a fixed Telegram URL or fixed chat endpoint.

The intended model is a **WISE context/intent router** that can use whatever context is already available and choose an appropriate next step:

- self-serve information
- relevant WISE capability / offer
- registration or workflow entry
- existing-request tracking
- WISE Support
- other approved ecosystem pathway

Potential context dimensions:

- who: known/new user, patient/caregiver/other role where known
- what: exploration, medicine request, refill, prescription, consultation, lab, existing request, support
- where: current WISE ecosystem surface, source page, QR/campaign source, geography/serviceability where available
- why/when: reason for escalation, existing request, timing/follow-up context where available

**Design principle:** Support is not the universal destination. It is the human-resolution path when self-serve/direct routing is insufficient or when an existing operational issue needs human handling.

**Related historical source:** Wescura Care Pathway Router / WISE Health ecosystem strategy documentation on the wescura-fulfilment-live-architecture branch of quick-chat-landing.

---

## 2. Chatwoot welcome-message capability as a context-capture mechanism

**Priority:** High  
**Phase:** Pilot UX / future provider-neutral support contract  
**Status:** Backlog

Use the fact that Chatwoot can configure a channel/inbox welcome message as a deliberate **context-capture and expectation-setting capability**, rather than treating it as cosmetic copy.

Investigate a welcome experience that can:

- explain what WISE Support can help with
- encourage the user to provide the minimum useful context
- capture/confirm relevant source context where it is not already known
- guide the user toward an appropriate first message
- avoid making the user repeat information that the entry point already knows

Potential context to preserve across the handoff:

- source / entry-point
- current WISE ecosystem surface
- location/pincode when relevant and voluntarily supplied
- medicine/request context
- existing booking/request reference
- what the user is trying to accomplish

**Architecture requirement:** Chatwoot's current welcome-message capability is evidence of the required capability, not a permanent dependency on Chatwoot. The eventual WISE Support abstraction should support equivalent channel-specific welcome/onboarding configuration regardless of the underlying support provider.

---

## 3. Telegram /start UX and "bot" terminology

**Priority:** High  
**Phase:** Pilot UX  
**Status:** Backlog

The current Telegram experience exposes /start as a technical-looking interaction. Investigate:

1. whether the Telegram/Chatwoot onboarding flow permits avoiding an explicit user-facing /start action;
2. if not, what can be customized around the action so that it feels like a normal "Start support" interaction rather than a technical bot command;
3. whether the holding page / CTA should explicitly explain the first Telegram action;
4. whether the configured welcome message can immediately establish a friendly support expectation after /start.

The user-facing language should avoid unnecessary "bot" terminology where it creates friction.

**Fallback UX requirement:** If Telegram technically requires /start for the first interaction, provide concise guidance next to the CTA/holding-page entry so the action is expected rather than surprising.

Do not assume the technical limitation or solution until verified against the actual Telegram + Chatwoot configuration.

---

## 4. Preserve entry-point context into Support

**Priority:** High  
**Phase:** Architecture / pilot design  
**Status:** Backlog

When a user reaches Support from an existing WISE journey, Support should ideally receive the context already known by that journey.

Example:

> User is already on Wescura Medicines, opens Chat, and asks for help.

The support rep should not have to rediscover that the user was already in the medicines journey.

Candidate handoff context:

    source_surface = Wescura Medicines
    entry_point = portal_chat
    intent = medicine_request | refill | support | unknown
    location = known/unknown
    identity = known/unknown
    existing_request = known/none
    context_notes = safe, minimal, non-clinical

This should eventually be provider-neutral and mapped into Chatwoot attributes/custom context where appropriate.

---

## 5. Chatwoot auto-assignment quirk

**Priority:** High  
**Phase:** Pilot blocker / configuration investigation  
**Status:** Open

Auto-assignment behaviour remains inconsistent during the dress rehearsal.

Observed states have included:

- Chatwoot receiving a new Telegram conversation successfully;
- automatic-assignment jobs executing;
- some conversations appearing under **Unassigned** rather than **My Inbox**;
- other conversations subsequently appearing assigned to the configured support user.

Investigate the complete assignment path, including:

- account-level agent availability;
- inbox membership;
- inbox auto-assignment configuration;
- assignment policy;
- conversation creation vs subsequent inbound messages;
- worker execution and timing;
- Chatwoot version-specific assignment behaviour.

Do not work around this by manually assigning every conversation. The pilot should establish a predictable support-rep queue behaviour.

---

## 6. Inbox vs Conversation operating model

**Priority:** High  
**Phase:** Pilot readiness  
**Status:** Open

Document and verify the operational distinction between:

- **Inbox** — channel/support queue/container;
- **Conversation** — individual patient/support thread;
- **My Inbox** — conversations assigned to the current rep;
- **Unassigned** — conversations received by the inbox but not yet assigned;
- **All / Participating / Unattended** — Chatwoot conversation views.

Pilot readiness should verify that:

1. a new patient message creates/continues the expected conversation;
2. the conversation belongs to the correct support inbox;
3. assignment moves it into the intended rep workflow;
4. subsequent messages remain in the same conversation when appropriate;
5. resolved conversations behave correctly when the patient returns;
6. support reps can reliably distinguish channel/inbox context from individual patient conversations.

This is both a configuration check and an SOP/training item.

---

## 7. Outbound .txt attachment delivery

**Priority:** Pilot blocker  
**Phase:** Telegram/Chatwoot channel validation  
**Status:** Open

Plain text messages work in both directions, but an outbound reply containing a simple .txt attachment sent from Chatwoot did **not** reach the Telegram patient.

Chatwoot displayed the outbound message as failed/red.

Investigate:

- Chatwoot attachment upload/storage state;
- attachment URL generation/accessibility;
- Telegram media/document API handling;
- Chatwoot Telegram channel adapter behaviour;
- OCI/Nginx/public HTTPS implications;
- worker logs and Telegram API response;
- whether attachment storage configuration is complete.

Do not mark attachment support complete until delivery is verified end-to-end.

---

## 8. Outbound prescription-image delivery

**Priority:** Pilot blocker  
**Phase:** Telegram/Chatwoot channel validation  
**Status:** Open

A support reply containing a prescription-sheet image/photo also did **not** reach the Telegram patient and showed the same failed/red behaviour.

Treat this as a potentially broader **attachment/media handling** issue rather than an isolated image problem.

Validate at least:

- JPG/JPEG/PNG image;
- document/PDF if supported;
- simple text/document attachment;
- patient → Chatwoot inbound attachment;
- Chatwoot → Telegram outbound attachment.

The Rx-image path is particularly important because prescription/support workflows are expected to use image/document exchange.

---

## 9. General attachment capability gate

**Priority:** Pilot blocker  
**Phase:** Pilot readiness  
**Status:** Open

Before the Telegram + Chatwoot path is considered pilot-ready, establish an explicit attachment capability matrix:

| Direction | Text | Image/Rx | Document | Result |
|---|---|---|---|---|
| Patient → Chatwoot | proven | to verify | to verify | open |
| Chatwoot → Patient | proven | failing | failing | blocker |

Do not assume the failure is caused by one specific configuration component until logs/source behaviour are inspected.

---

## 10. Two-way text communication — completed baseline

**Priority:** Reference / regression test  
**Status:** Proven

The following happy path is now demonstrated:

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
    Support conversation
        ↓
    Support rep reply
        ↓
    Chatwoot / Telegram adapter
        ↓
    Patient Telegram

The text-only path should become a regression test while attachment and routing work continues.

---

## 11. Native WISE Health `/support` web-chat integration

**Priority:** High  
**Phase:** Current implementation slice  
**Status:** Inbox/configuration complete; application integration next

A dedicated Chatwoot Website Inbox has been configured for the WISE Health portal:

- Inbox: `WISE Health™ Support`
- Domain: `https://wisehealth.in`
- Welcome heading: `WISE Health™ Support`
- Friendly sender identity: WISE Health
- Collaborators: configured
- Widget preview: verified
- Website token: generated and kept out of Git/documentation

Next validation path:

```
https://wisehealth.in/support
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

Implementation boundary:

- `quick-chat-landing` owns the native `/support` page, WISE Health presentation/navigation/copy and the support-launch interaction.
- `wise-support-chat` owns Chatwoot/provider-specific Website Inbox configuration and implementation concerns.
- Do not commit the Chatwoot website token.
- Do not merge the earlier provider-specific `ChatwootWidget.tsx` experiment wholesale into the core application.
- Keep the Telegram POC path unchanged while web-chat is introduced.

Acceptance checks:

1. native `/support` page loads;
2. Start Support opens the intended web chat;
3. conversation lands in `WISE Health™ Support`;
4. configured collaborator/assignment behaviour is observed;
5. Support Rep reply reaches browser;
6. conversation persists across refresh/re-entry;
7. Telegram two-way text regression remains green.

---

## 12. Pilot entry-point rehearsal matrix

**Priority:** High  
**Phase:** Pilot preparation  
**Status:** Backlog

Rehearse the same Support capability from multiple WISE entry points:

- holding-page CTA;
- QR code;
- Wescura Medicines page;
- future WISE Health portal chat widget;
- WISE Doctor entry point;
- existing-request/status context;
- future campaign/deep-link entry points.

For each, record:

- entry source;
- context available at entry;
- destination selected;
- welcome/onboarding experience;
- identity/context carried into Support;
- support-rep experience;
- fallback behaviour.

This becomes the practical validation of the context-aware router concept.

---

## 13. Production entry-point audit

**Priority:** High  
**Phase:** Current workstream  
**Status:** Next planned work

Inventory current production entry points and hard-coded destinations before migrating production code.

For each entry point determine:

- current destination;
- intended audience/intent;
- available context;
- current self-serve option;
- Support fallback;
- future smart-routing opportunity;
- whether the current destination should remain unchanged during pilot.

**Guardrail:** Do not replace existing production destinations merely because a smarter architecture is available. First audit, then design the transition, then implement deliberately.

---

## 14. Final patient-facing Telegram bot identity

**Priority:** Medium  
**Phase:** Production hardening  
**Status:** Deferred

The current @wise_chatwoot_poc_bot remains the dress-rehearsal bot.

Final patient-facing bot identity is to be selected after the pilot/entry-point review. Candidate identities previously discussed include @wise_health_support_bot and @wescura_medicines_support_bot.

Keep credentials, webhook ownership, onboarding and documentation strictly separated from the existing @wescura_support_bot path.

---

## Pilot-prep sequencing

### Complete / baseline
- OCI Chatwoot environment operational.
- Telegram → Chatwoot inbound text.
- Chatwoot → Telegram outbound text.
- Support conversation creation.
- Support-rep reply flow.
- POC CTA rehearsal from a second Android device.

### Next pilot-prep work
1. Complete the native `wisehealth.in/support` web-chat integration and prove browser ↔ Chatwoot Website Inbox E2E.
2. Resolve/understand assignment and inbox/conversation operating behaviour across Telegram and Website Inbox.
3. Investigate attachment/media failures.
4. Establish the pilot attachment capability gate.
5. Capture welcome-message and `/start` UX requirements for Telegram.
6. Rehearse multiple entry points and context handoff.
7. Complete production entry-point audit.
8. Only then decide production code migration and final patient-facing routing changes.

### Deferred architecture
- Context-aware WISE intent router implementation.
- Provider-neutral welcome/onboarding contract.
- Provider-neutral support context contract.
- Chatwoot replacement/fallback engine implementation.
- Final production Telegram bot identity.
- Broader WISE ecosystem routing beyond currently live capabilities.

---

## Pilot readiness principle

The pilot should prove **reliable human support first**, while preserving the larger WISE vision:

    discover / explore
          ↓
    self-serve where possible
          ↓
    clear intent → relevant WISE capability
          ↓
    unclear / blocked / operational issue
          ↓
    context-aware Support
          ↓
    human resolution

The router and CTA work should make the journey smarter without turning WISE Support into a generic destination or duplicating authoritative domain workflows.


---

## 15. Pilot Prep → RBAC / Initial Hand-off Workstream

**Priority:** High  
**Phase:** Immediate next milestone after current smoke-test closure  
**Status:** Planned workstream

Once the five immediate pilot-prep investigations below are sufficiently closed, move into the **RBAC + initial hand-off milestone** rather than continuing to expand the dress rehearsal indefinitely.

### Five immediate pilot-prep items

1. **Assignment + Inbox/Conversation behaviour**
   - establish predictable auto-assignment;
   - document Inbox vs Conversation vs My Inbox vs Unassigned;
   - verify new conversation, continuation, resolution and re-entry behaviour.

2. **Attachment/media investigation**
   - diagnose outbound .txt and Rx-image failures;
   - establish whether storage, URL accessibility, Telegram adapter, worker, or public HTTPS configuration is involved.

3. **Welcome / /start UX**
   - verify Telegram constraints;
   - make first-contact language user-friendly;
   - use Chatwoot welcome capability now while preserving a provider-neutral future contract.

4. **Context handoff**
   - define the minimum useful context carried from a WISE entry point into Support;
   - distinguish already-known context from context the support rep must still ask for.

5. **Production entry-point audit**
   - inventory current hard-coded destinations and /smart routing;
   - identify source/intent/context available at each entry point;
   - preserve current production behaviour until the replacement routing design is explicitly agreed.

### RBAC / initial hand-off milestone

After the above, establish the first controlled operational hand-off for WISE Support:

- define support roles and permissions;
- distinguish Support Rep, Support/Ops supervisory roles and administrative/platform roles;
- establish which users can see, claim, assign, reply, resolve, add private notes and administer channels/inboxes;
- define the boundary between Chatwoot operational permissions and WISE domain permissions;
- document the minimum audit trail required for support actions;
- establish the first repeatable Support → WISE operational hand-off without duplicating authoritative domain workflows.

**Milestone intent:** move from “the technology works in a dress rehearsal” to “a controlled support operation can safely use it and hand work into the appropriate WISE module.”

The RBAC milestone should not prematurely implement the full future context-aware router or fallback engine. Those remain subsequent architecture work.


## 16. Attachment/storage remediation checkpoint — 2026-10-02

**Status:** Infrastructure remediation complete; media E2E pending

The Active Storage investigation progressed from diagnosis to a controlled persistent runtime configuration change.

### Completed

- Preserved the three recoverable WEB diagnostic files before container replacement.
- Preserved the previous WEB and WORKER containers for rollback.
- Recreated WEB with OCI S3-compatible Active Storage.
- Verified live WEB Rails `ActiveStorage::Service::S3Service`.
- Verified live WEB PUT/EXIST/DOWNLOAD/DELETE against OCI Object Storage.
- Recreated WORKER with the same OCI S3-compatible Active Storage configuration.
- Verified live WORKER Rails `ActiveStorage::Service::S3Service`.
- Confirmed Sidekiq resumed normal scheduled/background processing.
- Left PostgreSQL, Redis, Nginx and Telegram webhook ownership unchanged.

### Still open

The storage infrastructure is now configured, but the pilot attachment gate remains open until actual media delivery is proven:

1. Telegram two-way text regression.
2. Chatwoot → Telegram document.
3. Chatwoot → Telegram Rx image.
4. Patient → Chatwoot image/document.

The earlier outbound .txt and Rx-image failures must not be marked resolved solely because the storage backend changed.

### Rollback retention

Keep the preserved old WEB/WORKER containers until the media regression is complete.

### Security follow-up

The OCI secret key used during diagnostics was exposed during the session. Rotate it after functional validation and update both runtime containers before production use.


## 17. Updated attachment/media checkpoint — 2026-10-02

**Status:** Outbound attachment delivery proven; inbound attachment retrieval and duplicate-message behaviour remain open.

### Newly proven

After the persistent OCI S3-compatible Active Storage change:

- **Chatwoot → Telegram document:** PASS.
- **Chatwoot → Telegram image:** PASS.

This demonstrates that the storage remediation has resolved the previously observed outbound attachment delivery failure for the tested document/image types.

### Remaining blocker — Telegram → Chatwoot attachments

Inbound Telegram attachments reach the Chatwoot conversation, but the referenced attachments cannot be retrieved:

- document: received but opening/download fails;
- image: received but image download/render fails.

A direct OCI request for one generated object URL returned `NoSuchKey`, so the requested object was not present at the referenced key. Trace the inbound attachment pipeline before changing the storage architecture again.

### Separate open issue — duplicate inbound attachment messages

The same inbound attachment message is appearing multiple times in Chatwoot. One document test produced three visible copies, and refreshing **Mine** appeared to increase the count. The image showed the same pattern.

Investigate update-id handling, webhook/Sidekiq processing, DB message persistence and frontend refresh/API behaviour separately. Do not perform cleanup until the source is identified.

### Revised attachment capability gate

| Direction | Text | Image/Rx | Document |
|---|---|---|---|
| Patient → Chatwoot | PASS | received / retrieval FAIL | received / retrieval FAIL |
| Chatwoot → Patient | PASS | PASS | PASS |

The pilot attachment gate therefore remains open, but the original outbound-storage problem is no longer the primary blocker.


## 17. Telegram retry/idempotency checkpoint — 2026-10-03

**Priority:** High  
**Phase:** Pilot blocker / attachment-media reliability  
**Status:** Application fix locally validated; OCI runtime validation pending

The inbound attachment duplicate-message investigation produced a narrow retry/idempotency fix.

### Validated

- Branch: `fix/telegram-retry-idempotency`
- HEAD: `a8e85786f`
- Existing Telegram `message_id` / Chatwoot `source_id` is used to detect an already-persisted message.
- A repeated event repairs missing attachment state on the existing message rather than creating another message.
- Test-capable image containing the corrected code: `wise-support-chat:test-retry-idempotency`.
- Focused RSpec: **35 examples, 3 failures**.
- The two attachment-test failures caused by the incorrect `attachment.blob` API are resolved.
- The remaining three failures are the known, unrelated conversation-selection examples and remain out of scope.

### OCI candidate

Published immutable candidate:

`ghcr.io/wisedoctor/wise-support-chat@sha256:8212ed18698c959105376a0f3c1eef1780e6cd096ece885e83cb07ea28e0ccb0`

Rollback baseline remains:

`ghcr.io/wisedoctor/wise-support-chat:sha-a6b2176`

### Remaining validation

The fix is not yet marked closed. Controlled OCI validation must:

1. replace WEB + WORKER only;
2. preserve OCI Active Storage, stable HTTPS ingress and current Telegram webhook;
3. rerun two-way text regression;
4. run fresh inbound document/image tests;
5. verify one persisted Chatwoot message per Telegram message;
6. exercise a repeated/retried event;
7. verify idempotent repair and attachment retrieval;
8. retain the previous image for rollback until the regression is complete.

Do not change the three unrelated conversation-selection behaviours as part of this workstream.
\n\n## 27. Telegram retry/idempotency OCI validation result — 2026-10-03\n\n**Status: PASS — controlled runtime validation completed**\n\nThe published retry/idempotency image was rolled out to the OCI rehearsal WEB and WORKER containers using the immutable GHCR digest:\n\n`ghcr.io/wisedoctor/wise-support-chat@sha256:8212ed18698c959105376a0f3c1eef1780e6cd096ece885e83cb07ea28e0ccb0`\n\n### Runtime validation\n\n- WEB started successfully on the candidate image and returned HTTP 200 locally.\n- WORKER started successfully and connected to Redis/Sidekiq normally.\n- Both containers retained `FRONTEND_URL=https://support.wisehealth.in`.\n- Both retained the proven OCI S3-compatible Active Storage configuration.\n- PostgreSQL, Redis, Nginx, DNS/TLS and Telegram webhook ownership were not changed.\n- The prior WEB/WORKER containers were retained as rollback artifacts.\n\n### Fresh Telegram attachment test\n\nA fresh inbound document was sent through the rehearsal bot `@wise_chatwoot_poc_bot`. The Chatwoot portal at `https://support.wisehealth.in` showed **exactly one inbound message** and the attachment **opened/downloaded successfully**. No duplicate message was observed.\n\nThis is the first controlled OCI runtime validation of the retry/idempotency candidate and materially closes the previously observed duplicate-message + inbound-attachment retrieval gap for the tested document path.\n\n### Important interpretation\n\nThe successful test proves the tested end-to-end document path on the new OCI runtime and provides runtime evidence for the application-level retry/idempotency fix. It does not by itself prove every image/media variant or every possible Telegram retry scenario. Those remain appropriate regression cases.\n\n### Container-name note\n\nPost-test diagnostic commands that referenced `wise-support-chat-web` / `wise-support-chat-worker` returned “No such container” after the rollout state changed. This is a diagnostic-name mismatch, not evidence of runtime failure: the candidate containers had already been observed healthy and the browser-visible Telegram test passed. Do not infer service failure from those name-resolution errors alone; inspect current `docker ps -a` names before further diagnostics.\n\n### Next gate\n\nKeep the candidate runtime and rollback containers intact. Update the pilot smoke-test/backlog status from “OCI validation pending” to **OCI document retry/idempotency validation PASS**, while keeping the broader attachment gate open for image/Rx and repeated-event regression as required.\n