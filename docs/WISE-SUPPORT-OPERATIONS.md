# WISE Support Operations

## 1. Purpose

This document translates the WISE Support architecture into an operational model for Support Representatives and WISE Ops.

The current Wescura patient-support SOP remains the detailed patient-safety and medicines-pilot operational reference. This document defines the broader Support-module boundary and should not replace the domain-specific SOP.

## 2. Support Rep operating model

A Support Rep should be able to see enough context to answer:

```text
Who is contacting us?
What channel did they use?
What are they asking about?
What existing request/service is relevant?
What has already happened?
What action is needed now?
Which WISE team/module owns the next action?
```

The Rep should not need to manually copy/paste the user's request into another WISE module when an explicit integration can create or route the appropriate action.

## 3. Patient-safe servicing

For Wescura Medicines and patient-facing support, preserve the established guardrails:

- curated patient-visible status;
- identity verification before disclosing request details;
- minimum necessary personal information;
- no exposure of partner internal details, bids, margins, internal scoring or audit trails;
- no clinical advice or substitution recommendation;
- approved safe wording for medicine coordination;
- escalation where the request exceeds the Support role.

Refer to the Wescura Patient Support SOP for detailed scenarios and pilot procedures.

## 4. Support triage

A basic triage sequence is:

1. Receive and acknowledge the interaction.
2. Resolve or establish identity/context as required.
3. Determine the user's immediate need.
4. Locate related WISE request/service context.
5. Route or initiate the authoritative WISE workflow.
6. Keep the user informed using curated status.
7. Record the interaction and outcome.
8. Capture a structured signal when the interaction reveals a meaningful recurring demand or opportunity.

## 5. Channel continuity

If the same user moves between channels, Support should use identity/context resolution to maintain continuity where the identity match is established and permitted.

The conversation history should remain understandable even when provider-specific identifiers differ.

## 6. Escalation

Escalate when:

- identity cannot be safely verified for a sensitive action;
- a domain workflow is blocked;
- the issue involves a partner dispute or compliance concern;
- a technical defect prevents the requested service;
- the user requests clinical advice beyond the Support role;
- a privacy/security concern is reported;
- a new capability or exception requires Product/Ops decision.

## 7. Operational metrics

The Support capability should eventually support measurement such as:

- conversations received by channel;
- first response time;
- resolution time;
- unresolved/aged conversations;
- routing outcomes;
- escalation volume;
- repeat-contact rate;
- requests created by Support;
- channel/source attribution;
- contextual pathway opportunities;
- pathway acceptance/completion;
- user outcome/feedback.

Metrics should be used to improve service quality and ecosystem design, not to expose sensitive operational detail to users.

## 8. Production readiness

Before production, each channel/provider combination requires:

- stable WISE-owned webhook ingress;
- environment-specific configuration;
- secret management;
- webhook authentication/signature verification where supported;
- idempotency/retry handling;
- delivery/error observability;
- correlation IDs;
- audit logging for privileged actions;
- defined ownership and escalation path;
- test cases for inbound, outbound and failure paths.

Temporary ngrok/proxy infrastructure belongs to development/debugging and must not be treated as the production architecture.

## 9. Documentation relationship

Use this repository for Support-module contracts and provider/channel implementation boundaries.

Use `quick-chat-landing` for wider WISE/Wescura domain architecture, fulfilment architecture, Ops Portal architecture, medicines workflow documentation and detailed domain SOPs.

The two documentation sets should link to one another rather than silently diverging.
