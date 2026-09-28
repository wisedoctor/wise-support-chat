# WISE Support Channels and Providers

## 1. Purpose

WISE Support separates two concepts that must not be conflated:

1. **Channel** — how the user communicates with WISE.
2. **Provider** — which system currently services/stores the conversation for Support staff.

## 2. Channel abstraction

The target channel model is:

```text
Telegram  ---+
WISE Web  ---+
WhatsApp  ---+--> WISE Support --> common conversation model
Email     ---+
future    ---+
```

The user should not have to know which provider is behind the Support experience.

## 3. Provider abstraction

The current provider is Chatwoot.

```text
WISE Support
     |
     +--> Chatwoot (current)
     +--> WISE-native support service (future option)
     +--> another provider (future option)
```

Provider-specific conversation IDs, webhook formats and delivery mechanisms are integration details.

## 4. Current Telegram proof

There are deliberately **two Telegram bots** in the current development setup and they must never be conflated.

### WISE patient-support bot

- Bot: `@wescura_support_bot`
- Secret: `TELEGRAM_SUPPORT_BOT_TOKEN`
- Location: WISE backend environment
- Existing WISE webhook: `/api/webhooks/telegram-support`
- Role: existing WISE patient-support channel

### Chatwoot POC bot

- Bot: `@wise_chatwoot_poc_bot`
- Secret: separate Chatwoot Telegram channel credential
- Chatwoot Account: 1
- Chatwoot Channel ID: 2
- Chatwoot Inbox ID: 2
- Channel type: `Channel::Telegram`
- Role: Telegram channel configured for the Chatwoot provider POC

**The Chatwoot channel must not use `TELEGRAM_SUPPORT_BOT_TOKEN`.**

## 5. Current proven transport

### Inbound

```text
Patient
  |
  | Telegram
  v
WISE backend
  |
  v
Chatwoot Telegram webhook
  |
  v
Chatwoot Telegram Inbox
  |
  v
Support Rep
```

### Reverse

```text
Support Rep
  |
  v
Chatwoot conversation
  |
  v
Chatwoot webhook
  |
  v
WISE backend /api/webhooks/chatwoot
  |
  v
Telegram Bot API
  |
  v
@wise_chatwoot_poc_bot
  |
  v
Patient Telegram chat
```

Both directions have been functionally proven during the current POC.

## 6. Chatwoot-specific implementation note

The self-hosted Chatwoot v4.14.2 Telegram inbound job was adjusted to resolve the Telegram channel using the supplied token prefix because Telegram supplied the token prefix while the stored Chatwoot token contained additional token data.

The existing Chatwoot change is already committed separately and must not be duplicated merely because this document describes it.

## 7. Environment abstraction

Development may use a temporary ngrok endpoint and local proxy. Production must use stable WISE-owned ingress/configuration.

Do not document a temporary ngrok hostname as a production endpoint.

The production design should allow the two directions to be configured independently:

```text
External Telegram / provider webhook
        -> stable WISE ingress
        -> Support channel adapter

Provider webhook for outbound agent replies
        -> stable WISE ingress
        -> Support service
        -> channel adapter
```

Exact production URLs and secrets belong in environment configuration, not this document.

## 8. Provider replacement rule

A provider may be replaced without changing the conceptual Support API.

When adding a provider, document:

- provider conversation identifier;
- inbound event mapping;
- outbound message mapping;
- signature/authentication mechanism;
- delivery/error semantics;
- attachment/media semantics;
- status mapping;
- retry/idempotency behaviour;
- correlation/provenance fields;
- secret/configuration requirements.

## 9. Future WISE Web Chat

WISE Web Chat should use the same Support conversation model rather than becoming a separate support system.

```text
WISE Web UI
   |
   v
WISE Web channel adapter
   |
   v
WISE Support conversation
   |
   v
same Support servicing workflow
```

This is important for the eventual end state where a patient may move between Telegram, web and other channels while remaining the same WISE user/context.
