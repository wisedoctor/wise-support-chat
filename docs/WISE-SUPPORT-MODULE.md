# WISE Support Module

## 1. Purpose

WISE Support is the WISE Universe capability for **user engagement, conversation servicing, contextual support and governed navigation into the wider WISE ecosystem**.

It provides a common foundation for users who reach WISE through different touchpoints and should prevent each product from independently rebuilding support, identity/context handling, routing and cross-service engagement.

The guiding principle is:

> **A support conversation is an entry point into the WISE ecosystem, not an isolated destination.**

## 2. What WISE Support is

WISE Support provides the common seams for:

- channel-independent conversations;
- inbound and outbound message handling;
- support-agent servicing;
- user/contact identity resolution hooks;
- context attachment and retrieval hooks;
- need/intent classification hooks;
- routing to the appropriate team or WISE capability;
- contextual navigation to relevant WISE services where appropriate;
- interaction and outcome capture;
- provider abstraction so the servicing provider can change without redesigning every WISE touchpoint.

## 3. What WISE Support is not

WISE Support is not:

- the authoritative owner of every WISE domain workflow;
- a replacement for the WISE identity/Prospects/CRM capability;
- a duplicate medicine-request, fulfilment, catalogue or partner workflow;
- a clinical decision-making system;
- synonymous with Chatwoot;
- a generic marketing/cross-sell engine.

Commercial outcomes such as service adoption, retention or LTV may be measured from contextual engagement, but the architectural purpose is broader: **help the user reach the right WISE capability at the right point in their journey**.

## 4. Relationship to the WISE Universe

The wider WISE architecture separates responsibilities across applications, bridges, context, knowledge, agents and orchestration. WISE Support follows that model.

```text
                    WISE UNIVERSE
                         |
              user-facing touchpoints
                         |
                         v
                  WISE SUPPORT
                         |
        +----------------+----------------+
        |                |                |
   Conversation       Context          Routing /
   / servicing      integration       navigation
        |                |                |
        +----------------+----------------+
                         |
                         v
                  WISE domain modules
```

Apps remain authoritative for their workflows. Bridges/providers remain replaceable. Context is resolved through explicit contracts rather than copied into every channel implementation.

## 5. Core actors

WISE Support must be able to represent or reference:

- Patient
- Caregiver/family member
- Doctor
- Clinic or healthcare organisation
- Pharmacy/fulfilment partner
- Pharma organisation representative
- WISE support/Ops representative
- Other WISE users introduced later

The Support layer should not assume that every inbound contact is already a registered WISE user.

## 6. Existing versus new user

A core capability is to establish, where permitted and possible:

1. who is contacting WISE;
2. what role or relationship they have;
3. whether an existing WISE identity/context can be matched;
4. what context is already available;
5. whether the current interaction creates a new prospect/contact/context record;
6. what next action is appropriate.

Identity matching must follow the authoritative WISE identity model when it becomes available. Support should use integration contracts rather than creating a competing identity store.

## 7. Context-aware engagement

A user may enter through one product while benefiting from another WISE capability. WISE Support should provide the common hook for contextual navigation instead of requiring each touchpoint to invent its own engagement logic.

Canonical example:

```text
Wescura Medicines request
        |
        v
mobile-number-based interaction
        |
        v
WISE identity/context resolution
        |
        +--> existing WISE Doctor identity?
        +--> repeat requester?
        +--> family request?
        +--> relevant care context?
        |
        v
contextual care-journey pathway
        |
        v
WISE Doctor / wider WISE journey
```

The user should be offered a relevant pathway only when it is useful, appropriate and governed by the applicable consent/privacy rules. The system must be able to conclude that no additional pathway is appropriate.

## 8. Channel neutrality

Telegram is the first proven external channel, but Telegram must not become the Support architecture.

The target model is:

```text
Telegram  ---+
WISE Web  ---+
WhatsApp  ---+--> WISE Support --> Support workflow
Email     ---+
future    ---+
```

## 9. Provider neutrality

The current servicing provider is Chatwoot. It is an implementation detail behind a provider boundary.

```text
WISE Support contract
        |
        +--> Chatwoot today
        +--> WISE-native servicing later, if required
        +--> another provider if required
```

Replacing a provider should not require rebuilding channel integrations or product-specific support flows.

## 10. Healthcare and patient-safety boundary

WISE Support is operational/support infrastructure. It must not silently become a clinical advice engine.

Patient-facing support must preserve the established Wescura guardrails: curated status, minimum necessary information, no partner/internal commercial exposure, no unreviewed clinical guidance, and appropriate escalation where clinical or emergency concerns arise.

## 11. Scope freeze direction

For the current WISE Support workstream, the architectural scope is frozen around:

- channel abstraction;
- provider abstraction;
- conversation lifecycle;
- identity/context hooks;
- routing hooks;
- support-agent servicing;
- interaction/outcome capture;
- contextual WISE-navigation hooks;
- secure production configuration and observability.

A full autonomous recommendation engine, universal CRM replacement or AI support agent is **not** required to freeze these boundaries.
