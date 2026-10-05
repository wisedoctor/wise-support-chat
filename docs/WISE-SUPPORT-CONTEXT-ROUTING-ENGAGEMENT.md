# WISE Support Context, Routing and Engagement

## 1. Purpose

WISE Support should be context-aware without becoming a duplicate CRM or product database.

Its role is to **facilitate the user + context** and help route the interaction to the right WISE capability.

The core questions are:

```text
Who is this?
Is this an existing WISE user/contact?
What role do they have?
What context is already known?
What are they trying to accomplish?
Which team/workflow/capability should handle it?
Is there another relevant WISE capability that should be surfaced?
```

## 2. Identity and context

Support should resolve or reference the authoritative WISE identity/context layer rather than creating a parallel identity universe.

Context may include:

- patient/doctor/partner/organisation role;
- existing versus new contact state;
- channel identities;
- QR/campaign source;
- current or previous WISE request;
- care area/category;
- family/caregiver relationship where available;
- relevant product/service usage;
- open support issue;
- prior interaction/outcome.

Only the minimum context necessary for the current action should be exposed to a Support Rep or downstream module.

## 3. Routing model

Support routing is a governed navigation function, not an uncontrolled transfer mechanism.

```text
Conversation
    |
    v
Identity/context
    |
    v
Need / intent
    |
    v
Routing decision
    |
    +-------------------+-------------------+
    |                   |                   |
    v                   v                   v
Support team       WISE workflow       other capability
```

Examples:

- medicine-request status -> medicines/fulfilment workflow;
- Rx clarification -> appropriate Ops/Rx workflow;
- partner operational issue -> partner/network workflow;
- technical issue -> technical support;
- doctor onboarding interest -> doctor workflow;
- patient care-journey request -> WISE Doctor/patient journey capability.

The authoritative domain module owns the resulting workflow/state mutation.

## 4. Support-created actions

A Support Rep may act on behalf of a user. Where the action corresponds to an existing WISE domain workflow, it should enter that same workflow rather than creating a parallel Support-only record.

```text
Support conversation
      |
      v
Support Rep identifies need
      |
      v
WISE domain action
      |
      v
normal authoritative lifecycle
      |
      v
Support can continue servicing using context
```

This preserves the integrated-module advantage: Support sees the context while the domain system remains authoritative.

## 5. Contextual engagement

The WISE Universe contains multiple products/services and user touchpoints. A user who is already interacting with one WISE capability may be a strong candidate for another capability because their current journey reveals a relevant unmet need.

The Support layer should provide a reusable hook for this rather than requiring every product to reinvent:

- CTA wording;
- eligibility checks;
- identity linking;
- context retrieval;
- attribution;
- interaction logging;
- conversion/outcome tracking.

The architecture should describe this as **contextual navigation / care-journey engagement** rather than hard-coding a generic cross-sell engine into Support.

## 6. Canonical medicines -> WISE Doctor example

A patient may use Wescura Medicines with only a mobile number required for request tracking/payment-related coordination. On a later request, the existing interaction can provide context for a relevant care-journey pathway.

```text
Wescura Medicines
       |
       v
repeat medicine request
       |
       v
identity/context resolution
       |
       +--> existing WISE Doctor identity?
       +--> repeat user?
       +--> family request?
       +--> relevant continuity context?
       |
       v
contextual pathway
       |
       v
"Unlock your full care journey"
       |
       v
WISE Doctor registration/linking
       |
       v
longitudinal care context
```

The integration must remain non-blocking: a user who declines or cannot complete the wider journey must still be able to continue the immediate medicines workflow.

## 7. Demand intelligence

Support interactions can generate useful signals for the wider WISE ecosystem, for example:

- repeat medicine requests;
- family/caregiver demand;
- repeated doctor-seeking questions;
- requests for laboratory/diagnostic services;
- homecare/service enquiries;
- technical/support friction;
- gaps in serviceability;
- emerging category demand.

These signals should be structured and governed. They should not be exposed as raw internal labels to patients or unauthorised users.

## 8. Measurement

Where a contextual pathway is surfaced, measurement should distinguish at least:

- opportunity identified;
- pathway shown;
- user accepted/declined;
- registration/link initiated;
- registration/link completed;
- downstream capability used;
- outcome/failure reason.

This allows WISE to learn from actual demand while avoiding a design where every interaction becomes a sales prompt.

## 9. Commercial principle

Cross-sell, upsell, retention and LTV are legitimate business outcomes, but the Support architecture should optimise for **relevance and user value** rather than indiscriminate conversion.

A valid outcome is:

```text
no additional pathway appropriate
```

The capability should be useful even when it recommends nothing.

## 10. Privacy and healthcare boundary

Contextual engagement must not reveal sensitive health information merely because a user entered through another channel. Identity, consent, purpose and access controls remain applicable.

Support should surface only the context necessary for the next action and should preserve provenance of context-derived decisions/signals.
