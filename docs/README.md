# WISE Support Documentation

This directory is the documentation home for the **WISE Support** capability in the WISE Universe.

`wise-support-chat` is intended to host the channel-independent Support/Conversation capability and its integration contracts. It is **not** defined as a Chatwoot-specific application, nor as a standalone CRM.

## Current architectural position

WISE Support sits between user-facing touchpoints and the wider WISE operating model:

```text
Telegram / WISE Web / WhatsApp / Email / other touchpoints
                         |
                         v
                  WISE Support
                         |
             +-----------+-----------+
             |                       |
       conversation             context hooks
             |                       |
             v                       v
       Support Rep /          Identity / CRM /
       support workflow        Prospects / WISE context
                                     |
                                     v
                              routing / navigation
                                     |
                 +-------------------+-------------------+
                 |                   |                   |
              Support          WISE workflow       other capability
```

The current implementation proves Telegram transport through Chatwoot. Chatwoot is therefore a **provider/servicing implementation for today**, not the definition of WISE Support.

## Source-of-truth documentation

The wider WISE/Wescura architecture remains documented in `wisedoctor/quick-chat-landing`. Relevant source material includes:

- [WISE Health / Wescura project instructions](https://github.com/wisedoctor/quick-chat-landing/blob/main/AGENTS.md)
- [Wescura fulfilment architecture](https://github.com/wisedoctor/quick-chat-landing/blob/feature/wescura-fulfilment-live-architecture/docs/wescura-fulfilment-architecture.md)
- [Wescura Patient Support SOP](https://github.com/wisedoctor/quick-chat-landing/blob/feature/wescura-fulfilment-live-architecture/docs/product/wescura-patient-support-sop.md)
- [Wescura Ops documentation](https://github.com/wisedoctor/quick-chat-landing/tree/feature/wescura-fulfilment-live-architecture/docs/ops)
- [Ops Portal documentation](https://github.com/wisedoctor/quick-chat-landing/tree/main/docs/ops-portal)
- [WISE Rx signal architecture](https://github.com/wisedoctor/quick-chat-landing/blob/main/docs/wise-rx-signal-architecture.md)

## Documents

| Document | Purpose |
|---|---|
| [WISE Support Module](./WISE-SUPPORT-MODULE.md) | Module charter, boundaries, principles and relationship to the WISE Universe |
| [WISE Support Engine](./WISE-SUPPORT-ENGINE.md) | Core conceptual model, identity/context, conversation, routing and integration contracts |
| [WISE Support Channels and Providers](./WISE-SUPPORT-CHANNELS-AND-PROVIDERS.md) | Channel abstraction and provider abstraction; Telegram/Chatwoot is the first proven implementation |
| [WISE Support Context, Routing and Engagement](./WISE-SUPPORT-CONTEXT-ROUTING-ENGAGEMENT.md) | User/context awareness, need classification, routing and contextual discovery of relevant WISE capabilities |
| [WISE Support Operations](./WISE-SUPPORT-OPERATIONS.md) | Support Rep/Ops servicing model, patient-safe guardrails and operational readiness |
| [WISE Support Environment Reference](./WISE-SUPPORT-ENVIRONMENT-REFERENCE.md) | Infrastructure choices and alternatives retained for validation and future production decisions |
| [WISE Support Environment Matrix](./WISE-SUPPORT-ENVIRONMENT-MATRIX.md) | Development/validation/production environment separation and production decision gates |
| [WISE Support Production & Maintenance](./WISE-SUPPORT-PRODUCTION-MAINTENANCE.md) | Concrete deployment, maintenance, debugging and E2E operational learnings |

## Documentation rule

These documents describe the **stable architecture and contracts**. They should not become a second copy of every product's implementation detail. Domain applications remain authoritative for their own workflows and data.

Environment and maintenance documents record implementation learnings and operational procedures without making those implementation details part of the conceptual Support contract.

When a new channel or provider is introduced, extend the appropriate Support document rather than redefining the Support architecture inside that channel's implementation.

## Security and data handling

Do not commit Telegram bot tokens, Chatwoot PATs, webhook secrets, patient data, raw prescriptions, production contact data or environment values to this repository. Use references to secret names/configuration contracts only.
