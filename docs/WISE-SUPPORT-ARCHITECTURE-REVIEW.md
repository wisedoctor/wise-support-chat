# WISE Support Architecture Review

## 1. Purpose

This review reconciles the WISE Support documentation with the existing WISE/Wescura architecture and Ops documentation in `wisedoctor/quick-chat-landing`.

The intent is to freeze the Support architectural boundary without duplicating domain workflows or prematurely implementing future capabilities.

## 2. Source material reviewed

The review is grounded in the currently documented WISE/Wescura material, including:

- Wescura Fulfilment Architecture
- Wescura Patient Support SOP
- Ops Portal Iteration 4 Forensic Architecture / Refactoring Contract
- the wider WISE/Wescura project and Ops documentation referenced by those documents
- the current `wise-support-chat` Support module, engine, channel/provider, context/routing/engagement and operations documents

The WISE Support repository remains the implementation/contract home for Support. `quick-chat-landing` remains the wider WISE/Wescura domain and operational source of truth.

## 3. Architectural alignment

### 3.1 Support is a modular Operations Platform capability

The Ops architecture explicitly defines Support as a top-level Operations Platform module alongside Work, Network, Catalogue, Technical and Admin. It also states that not all target modules need to be implemented immediately.

WISE Support therefore fits as a bounded module rather than as another medicine-request cockpit feature.

### 3.2 Medicine Request remains authoritative

The medicine-request workflow remains the primary Ops workspace. Support may create or assist a medicine request, but the resulting request must enter the normal authoritative request lifecycle.

Support must never create a parallel request state merely because the request originated in conversation.

### 3.3 Signals are first-class integration seams

The Ops architecture defines:

`Signal -> contextual action -> authoritative module -> audited mutation -> signal resolved`

WISE Support should use the same pattern for support-derived opportunities and operational signals.

Examples include:

- repeat medicine request;
- family/caregiver demand;
- medicine not in catalogue;
- serviceability gap;
- recurring support issue;
- contextual care-journey opportunity.

A Support signal is not itself the authoritative workflow.

### 3.4 Context travels with actions

The Ops architecture requires contextual actions to open the relevant workspace with useful context already supplied, rather than forcing Ops users to copy/paste identifiers and search again.

The Support contract should therefore carry references such as:

- WISE identity/contact reference;
- conversation reference;
- source/channel;
- related request/service reference;
- need/intent;
- relevant care area/category;
- signal/opportunity reference;
- provenance/correlation information.

Raw domain records should not be copied into Support merely to make this possible.

### 3.5 Support and provider are separate boundaries

Chatwoot is the current servicing provider. Telegram is the current proven external channel. Neither defines WISE Support.

This permits:

- Telegram -> Chatwoot today;
- WISE Web -> the same Support capability;
- WhatsApp/email later;
- Chatwoot replacement later;
- WISE-native servicing later if justified.

### 3.6 Patient-safe servicing remains a domain guardrail

The Wescura Patient Support SOP remains authoritative for medicines-pilot patient servicing. Support-module documentation must therefore avoid replacing detailed patient-safe wording, non-response procedures, cancellation procedures or other domain SOPs.

The Support layer provides the common operating boundary; Wescura Medicines provides the detailed patient workflow and safety rules.

### 3.7 AI/agent readiness is an architectural seam, not current scope

The Ops architecture explicitly intends the Operations Platform to become an engine room that can later accept AI/CDSS and worker-agent capabilities through explicit service/action boundaries.

WISE Support should therefore expose stable action/contract boundaries suitable for future workers, while keeping current routing and consequential actions governed and human-reviewable where required.

## 4. Architectural seams to freeze now

The following seams should be treated as part of the Support architecture even if implementation remains staged.

### A. Channel contract

Normalise inbound/outbound interaction independently of Telegram, Web, WhatsApp, email or future channels.

### B. Provider contract

Keep provider conversation IDs, webhooks, delivery semantics and secrets behind provider adapters.

### C. Identity/context contract

Resolve or reference the authoritative WISE identity/context layer. Do not create a competing identity store in Support.

### D. Conversation contract

Maintain a provider-neutral conversation reference, participant/contact reference, source/channel, assignment and lifecycle state.

### E. Domain-action contract

Allow a Support Rep or future governed worker to initiate an action in the authoritative WISE domain module without recreating the workflow inside Support.

### F. Signal/opportunity contract

Allow Support interactions to emit structured signals that can be consumed by Ops/Work, engagement or domain capabilities.

### G. Contextual-engagement contract

Allow a product/touchpoint to ask the Support/context capability whether a relevant next WISE pathway exists, rather than independently implementing identity linking, CTA logic, attribution and outcome tracking.

A valid response is **no pathway appropriate**.

### H. Provenance/correlation contract

Preserve channel, provider, identity, request/service, source/campaign, Support Rep/team, action and outcome references across integration boundaries.

### I. Security/consent contract

Keep minimum-necessary disclosure, identity verification, consent/purpose rules, role-aware access and auditability explicit at integration boundaries.

## 5. Canonical reference journey

The Wescura Medicines -> WISE Doctor journey is the reference use case for contextual engagement:

```text
Wescura Medicines
      |
      v
repeat / family medicine interaction
      |
      v
identity + context resolution
      |
      +--> existing WISE identity?
      +--> existing WISE Doctor identity?
      +--> repeat requester?
      +--> family relationship?
      +--> relevant continuity context?
      |
      v
contextual care-journey pathway
      |
      v
WISE Doctor registration/linking
      |
      v
longitudinal WISE care context
```

This must remain non-blocking. Declining or failing to complete the wider pathway must not prevent the immediate medicines workflow from continuing.

## 6. Scope boundary

### In scope for Support architecture

- channel abstraction;
- provider abstraction;
- conversation lifecycle;
- identity/context hooks;
- support-agent servicing;
- routing/navigation hooks;
- domain-action integration hooks;
- signal/opportunity hooks;
- contextual WISE-navigation hooks;
- provenance/correlation;
- security/consent boundaries;
- production configuration/observability requirements.

### Not required to freeze the boundary

- full universal CRM replacement;
- autonomous recommendation engine;
- autonomous AI Support Rep;
- full cross-sell/upsell automation;
- replacement of Chatwoot;
- replacement of Telegram;
- redesign of the medicine-request state machine;
- wholesale patient-login redesign;
- automated clinical decision-making.

## 7. Implementation consequences

The next implementation work should proceed in this order:

1. Freeze stable Support contracts and environment configuration.
2. Complete production-grade channel/provider transport for the proven Telegram path.
3. Remove temporary debugging proxy/ngrok dependencies from the production design.
4. Preserve the two Telegram identities and credentials as separate configuration domains.
5. Define the WISE Web channel against the same Support conversation model.
6. Add identity/context resolution as an explicit integration boundary rather than embedding it in channel code.
7. Add Support-created domain actions using authoritative WISE workflows.
8. Add structured Support signals and contextual-engagement measurement.
9. Expand channels/providers only after the common contracts remain stable.

## 8. Documentation ownership

`wisedoctor/wise-support-chat` owns:

- Support contracts;
- channel/provider boundaries;
- Support engine model;
- Support operational boundary;
- Support integration requirements.

`wisedoctor/quick-chat-landing` owns:

- WISE/Wescura product architecture;
- medicine/fulfilment domain workflows;
- Ops Portal architecture;
- detailed domain SOPs;
- product-specific user guides and implementation documentation.

The repositories should link to one another rather than duplicating the same detailed workflow.

## 9. Review outcome

The current Support documentation is architecturally consistent with the existing WISE/Wescura and Ops direction.

The main seam that needed to be made explicit is the combination of **domain actions + structured signals + contextual engagement**. These are now treated as first-class Support integration boundaries without making Support authoritative for the underlying domain data or workflows.

The architecture is therefore ready to be treated as the baseline for the next implementation phase, subject to normal review when concrete API/schema contracts are introduced.
