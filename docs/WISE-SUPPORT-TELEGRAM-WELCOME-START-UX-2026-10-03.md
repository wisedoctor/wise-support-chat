# WISE Support — Telegram Welcome / /start UX Checkpoint — 2026-10-03

**Status:** Narrow implementation candidate added; runtime validation pending
**Environment:** OCI hosted dress rehearsal
**POC bot:** @wise_chatwoot_poc_bot
**Chatwoot:** self-hosted v4.14.2 lineage
**Portal:** https://support.wisehealth.in

## 1. Purpose

Determine what the current Telegram + Chatwoot rehearsal can do for first-contact onboarding, and avoid introducing a provider-specific customization before the actual Telegram client behaviour is verified.

The pilot requirement is not to eliminate Telegram's technical mechanism at any cost. It is to make first contact feel like **Start Support**, avoid unnecessary "bot" terminology, and preserve useful entry-point context where possible.

## 2. Current repository finding

The current Telegram inbound service has no special /start handling or WISE-specific welcome-message branch.

The inbound path is:

    webhook
      → Webhooks::TelegramEventsJob
      → Telegram::IncomingMessageService
      → set_contact
      → set_conversation
      → create inbound message

The service accepts the incoming Telegram message as normal message content. No code change is required merely to support a normal /start message.

The controlled client-behaviour test was followed by a deliberately narrow implementation candidate on `feature/telegram-start-welcome`; no provider-wide onboarding refactor was introduced.

## 3. Chatwoot capability finding

Current Chatwoot documentation describes a general inbox-level **Greeting** option: a greeting can be enabled and configured to auto-send when a new conversation begins. Chatwoot also documents Telegram as a native inbox/channel. However, the current Telegram-specific setup documentation does not explicitly document a Telegram-specific greeting configuration workflow.

Therefore this rehearsal must **verify the actual v4.14.2 Telegram inbox UI/behaviour** before treating a Chatwoot greeting as available for this channel.

The existing WISE Website Inbox greeting capability is separately proven/configured and should not be assumed to imply identical Telegram behaviour.

## 4. Telegram capability finding

Telegram officially supports bot deep links in the form:

    https://t.me/<bot_username>?start=<parameter>

When such a bot deep link includes a start parameter, Telegram can present a **Start** button and invoke the bot with that start parameter. The documented start parameter is limited to 64 base64url characters.

This gives WISE a potentially cleaner patient-facing entry mechanism than asking the user to type /start manually.

Example for the rehearsal only:

    https://t.me/wise_chatwoot_poc_bot?start=support

Do not replace any production CTA with this URL yet.

## 5. Controlled UX test

Use a test Telegram account/contact that has not previously started the POC bot if possible.

### Test A — normal bot entry

1. Open the POC bot.
2. Observe the first-contact UI.
3. Record whether Telegram presents a **Start** button.
4. Press Start.
5. Observe the first message(s) shown by the bot/Chatwoot.
6. Record whether /start is visible as a technical command.
7. Record whether a Chatwoot-generated greeting is sent.
8. Confirm the conversation reaches the expected POC Telegram Inbox and Mine/assignment path.

### Test B — deep-link entry

Use:

    https://t.me/wise_chatwoot_poc_bot?start=support

1. Open the link.
2. Record the first-contact UI.
3. Confirm whether Telegram presents **Start** rather than requiring the user to type /start.
4. Press Start.
5. Record the actual message received in Chatwoot.
6. Check whether the start parameter is preserved or visible anywhere in the Chatwoot conversation/contact attributes.
7. Confirm the conversation is created/continued normally.
8. Confirm Support Rep assignment and reply still work.

### Test C — repeat-entry behaviour

After the first test, open the same deep link again.

Record whether:

- Telegram again shows Start;
- a second start event is generated;
- Chatwoot creates a second conversation;
- the existing conversation is reused;
- the start parameter changes anything.

Do not alter the Telegram webhook, bot token, or application code during these tests.

## 6. Decision gate

### Current implementation candidate

For the WISE deep-link entry (`/start support`), the candidate implementation now:

- resolves the contact and existing/new conversation normally;
- does not persist the technical `/start support` command as an incoming Chatwoot message;
- creates one outgoing WISE welcome message in the conversation;
- uses a deterministic `source_id` derived from the Telegram message id so a webhook retry does not create another welcome message;
- leaves other Telegram messages and unknown `/start <parameter>` values on the existing path.

Runtime validation is still required before this is treated as pilot-ready.

### If Chatwoot greeting works for the Telegram inbox

Use a concise WISE greeting to establish expectations immediately after the first support conversation begins.

Candidate direction:

> **Welcome to WISE Support 👋**
> Tell us what you need help with. If this is about an existing WISE request, please share the request/reference number if you have it.

Keep the exact copy as a UX decision after the behaviour test.

### If Chatwoot greeting does not work for Telegram

Do not build a custom greeting mechanism yet.

Use the Telegram deep-link Start experience as the first-contact UX and consider a WISE-controlled entry page/CTA to explain the first action.

A provider-neutral future Support onboarding contract can later express:

    entry context → channel onboarding → minimum useful context → support conversation

without making Chatwoot's greeting implementation the WISE architecture.

## 7. Context handoff boundary

The deep-link parameter is potentially useful as an entry-point signal, but it should not be treated as a complete patient context contract.

Future context should remain provider-neutral and follow the established WISE model:

    source_surface
    entry_point
    intent
    location
    identity
    existing_request
    context_notes

Only minimum necessary, safe, non-clinical context should cross the Support boundary.

## 8. Guardrails

- Do not change the existing @wescura_support_bot.
- Do not change the POC webhook during this UX test.
- Do not migrate the production Telegram CTA yet.
- Do not add custom /start code until the native Telegram/Chatwoot behaviour is verified.
- Do not treat a deep-link parameter as trusted identity.
- Do not make the future provider-neutral Support contract depend on Chatwoot-specific greeting fields.

## 9. Current disposition

**Investigation:** PASS — native Telegram deep-link capability is documented; current repository has no special /start handling.

**Implementation:** CANDIDATE — narrow `/start support` handling is implemented on `feature/telegram-start-welcome`; local focused test and OCI runtime validation are still required.

**Next input:** record the results of Test A/B/C, then decide whether the pilot needs only CTA copy/configuration or a narrowly scoped implementation change.

## 10. OCI runtime forensic finding — 2026-10-06

The first OCI runtime validation of the /start support candidate proved that Telegram deep-link handling is working correctly:

- Telegram sent the webhook payload with text: "/start support".
- Chatwoot persisted the technical /start support as an inbound message for the conversation.
- The candidate IncomingMessageService created the WISE welcome message with source_id: "telegram_start:<message_id>".
- Chatwoot created and enqueued SendReplyJob for the welcome message.
- The welcome did not reach Telegram.

The root cause is Chatwoot's normal outbound-channel guard. Base::SendOnChannelService treats any outgoing message with a populated source_id as having originated from the channel and therefore does not send it through the channel transport. The candidate implementation had used source_id as the /start support idempotency marker, so the welcome was incorrectly classified as channel-originated.

The Chatwoot portal evidence also confirms the /start support inbound event itself is present in the conversation; the failure was specifically the outbound welcome delivery, not Telegram parameter handling.

### Corrected implementation

The welcome message now:

- remains message_type: outgoing;
- leaves source_id unset so the normal Telegram SendReplyJob path can deliver it;
- stores the originating Telegram message id in additional_attributes.telegram_start_message_id as the idempotency marker;
- ignores a retry of the same Telegram /start support event without creating a second welcome.

This preserves both required behaviours: deliverable outbound message and retry-safe welcome creation.

### Required runtime validation

After the corrected image is published and deployed to OCI WEB + WORKER:

1. send one fresh ?start=support deep-link invocation;
2. confirm the WISE welcome appears in the Telegram client;
3. confirm the welcome appears once in Chatwoot and is outgoing;
4. confirm the technical /start support is not shown to the patient as the welcome content;
5. send a normal patient reply and confirm it remains in the same conversation;
6. confirm no duplicate welcome is produced by a repeated/retried start event;
7. retain the current immutable OCI image as rollback until this validation passes.

## 11. Current disposition

Deep-link parameter handling: PASS — Telegram and webhook preserve /start support.

Candidate implementation: corrected — outbound welcome idempotency no longer uses source_id.

Local focused validation: pending for the corrected code/image.

OCI runtime validation: pending for the corrected image.

Pilot disposition: keep the /start support slice open until Telegram-visible welcome delivery and retry-safe behaviour are proven end-to-end.

## 12. Corrected image publication and OCI pull — 2026-10-06

The corrected implementation passed the focused local RSpec suite (`33 examples, 0 failures`), was built and published successfully, and the exact immutable image was pulled to the OCI validation VM.

Deployment candidate:

`ghcr.io/wisedoctor/wise-support-chat@sha256:fc11023a6f146ca475bd003a2071d6dc54f0dbaa2d3c868c792e8f1acad3df19`

OCI pull and digest verification: **PASS**.

The candidate has **not yet replaced** the running WEB + WORKER. The previously running corrected-but-not-delivering candidate remains available as the immediate rollback baseline.

Next validation is intentionally limited to WEB + WORKER replacement followed by runtime startup, HTTPS, and fresh `/start support` E2E. No infrastructure or Telegram webhook changes are required.



## Latest promotion / runtime checkpoint — 2026-10-06

**Status: PASS — Telegram Welcome runtime proven; WISE Health application main promoted**

The corrected Telegram Welcome implementation is now proven on the OCI rehearsal runtime, and the companion WISE Health application changes have been promoted to `quick-chat-landing/main`.

### Support runtime result

- Corrected immutable image: `ghcr.io/wisedoctor/wise-support-chat@sha256:fc11023a6f146ca475bd003a2071d6dc54f0dbaa2d3c868c792e8f1acad3df19`.
- WEB healthy; HTTPS through `support.wisehealth.in` returned 200.
- WORKER healthy; Sidekiq connected to Redis and processed scheduled jobs.
- Deep-link `https://t.me/wise_chatwoot_poc_bot?start=support` reached Chatwoot as `/start support`.
- WISE Support welcome appeared in Chatwoot and was delivered back to Telegram.
- Three deliberate start events each produced the expected welcome; these were separate events, not duplicate processing of one event.

### Why the corrected implementation matters

The first candidate stored its idempotency marker in `source_id`. Chatwoot's outbound channel guard interprets populated `source_id` as channel-originated, so the welcome was persisted but not delivered through Telegram. The corrected implementation leaves `source_id` unset and stores the Telegram start-message identifier in `additional_attributes.telegram_start_message_id`, preserving idempotency without blocking channel delivery.

### Telegram /start observation

Telegram may display `/start` in its client UI while Chatwoot receives `/start support` from the deep-link webhook payload. This is accepted as an internal technical distinction. The patient-facing outcome is the WISE Support welcome; no further work is required to make the technical command itself display as `/start support`.

### WISE Health application promotion

The companion `quick-chat-landing` changes are now on remote `main`:

- native `/support` route/page;
- Support navigation/footer entry points;
- `SupportChatWidget` application integration;
- Care Partner Telegram rehearsal CTA using the POC deep link;
- existing `@wescura_support_bot` path remains untouched.

Remote `main` was verified at:

```
48813329dcc8b3d2881ec3ee8639462d4543b8c5
```

This establishes the intended repository boundary: `quick-chat-landing` owns the WISE Health application/entry-point experience, while `wise-support-chat` owns the Chatwoot/provider-specific Support implementation and operational runtime.

### Next milestone

With the Welcome / /start gate closed, return to the remaining pilot-prep sequence:

1. context handoff;
2. production entry-point audit;
3. then RBAC + initial operational hand-off.

The final patient-facing Telegram bot identity and production hardening remain separate decisions.
