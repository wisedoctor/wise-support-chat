# WISE Support — Telegram Welcome / /start UX Checkpoint — 2026-10-03

**Status:** Investigation complete; configuration/code change deferred pending one controlled UX test
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

Therefore, the next step should be a configuration/client-behaviour test, not a code refactor.

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

**Implementation:** DEFERRED — one controlled client + Chatwoot behaviour test is required.

**Next input:** record the results of Test A/B/C, then decide whether the pilot needs only CTA copy/configuration or a narrowly scoped implementation change.