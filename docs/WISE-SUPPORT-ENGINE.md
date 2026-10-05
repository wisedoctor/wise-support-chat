# WISE Support Engine

## 1. Engine model

The Support engine is the common orchestration-facing layer between a user's communication channel and the WISE capability that should handle the user's need.

```text
Channel
  |
  v
Conversation
  |
  v
Identity + Context
  |
  v
Need / intent
  |
  v
Routing / navigation
  |
  +-------------------+
  |                   |
  v                   v
Human support       WISE capability
  |                   |
  +---------+---------+
            |
            v
        Outcome / signal
```

## 2. Conceptual entities

The first implementation does not require every entity below to become a new database table. They are conceptual boundaries for contracts.

### Support Identity

Reference to the person/contact across channels and WISE systems.

Potential attributes include:

- internal/canonical identity reference;
- role/person type;
- channel identities;
- verified contact references;
- existing/new state;
- identity-resolution confidence/state;
- consent/communication eligibility references.

### Support Conversation

A user-facing or support-agent conversation independent of its transport provider.

Potential attributes:

- conversation identifier;
- identity/contact reference;
- channel;
- provider conversation reference;
- source/campaign;
- status;
- assigned team/agent;
- timestamps;
- related WISE request/service references.

### Support Message

A channel-normalized inbound/outbound message with provider-specific identifiers retained as correlation data.

### Support Context

References to relevant WISE state rather than copying authoritative domain records into Support.

Examples:

- current medicine request;
- prior request;
- prescription-review context;
- partner/doctor relationship;
- campaign/QR source;
- relevant product/service usage;
- current support state.

### Support Route

A governed decision describing which support team, workflow or WISE capability should handle the interaction.

### Support Signal

A structured observation or opportunity produced by an interaction, such as a repeat-request signal, support escalation, unmet need or contextual care-journey opportunity.

## 3. Contract boundaries

The Support engine should evolve around explicit contracts such as:

```text
resolve_identity(channel_identity, supplied_context)
        -> identity reference + resolution state

resolve_context(identity, conversation)
        -> relevant context references

classify_need(conversation, context)
        -> need / intent + confidence / state

resolve_route(need, context, available_capabilities)
        -> route / team / capability

record_interaction(conversation, outcome)
        -> interaction reference

record_signal(context, signal)
        -> signal reference
```

These are conceptual contracts, not a commitment to these exact function names or a particular API style.

## 4. Domain ownership

Support should reference authoritative domain state rather than becoming authoritative for it.

Examples:

- Medicine request -> Wescura/medicines workflow.
- Catalogue -> catalogue capability.
- Partner/network issue -> network/partner capability.
- Technical problem -> technical support workflow.
- Doctor workflow -> WISE Doctor capability.
- Patient longitudinal record -> WISE Doctor/identity/context capability.

A Support Rep may initiate an action on behalf of a user, but the action should enter the same governed workflow used by the corresponding product surface.

## 5. Conversation lifecycle

The provider-neutral lifecycle should support at least:

```text
new
  -> identified / unidentified
  -> triaged
  -> assigned
  -> in_progress
  -> waiting_on_user
  -> waiting_on_internal_workflow
  -> resolved
  -> closed
```

Exact status codes belong to implementation contracts and should not be duplicated across providers without a mapping.

## 6. Correlation and provenance

Every integration should preserve enough provenance to answer:

- where did the interaction originate?
- which channel carried it?
- which provider handled it?
- which WISE identity/context was associated?
- which request/service was related?
- which Support Rep/team acted?
- what outcome resulted?

Correlation IDs should be propagated across integration boundaries wherever the participating systems support them.

## 7. Security model

Support must follow the WISE security direction:

- least-privilege access;
- role-aware support views;
- minimum necessary data exposure;
- explicit consent where required;
- auditability for privileged/support actions;
- provider secrets held in environment/secret management rather than source control;
- no raw patient data in test fixtures or public documentation.

The Support engine should not expose raw operational tables directly to public clients.

## 8. Failure and fallback behaviour

Transport failure, identity-resolution failure and downstream workflow failure must remain distinguishable.

Examples:

```text
message received
  -> provider accepted
  -> WISE processing failed
```

must not be represented as if the message was never received.

Similarly:

```text
identity unresolved
```

should not block basic support unless the requested action genuinely requires verified identity/context.

## 9. Current proven implementation

The current proof of transport uses:

```text
Patient Telegram
   -> WISE backend
   -> Chatwoot Telegram Inbox
   -> Support Rep

Support Rep in Chatwoot
   -> Chatwoot webhook
   -> WISE backend
   -> Telegram Bot API
   -> patient Telegram
```

The current Chatwoot deployment is therefore evidence of the transport/provider implementation, not the final Support engine boundary.

## 10. Evolution rule

Future channels and providers should plug into the same Support contracts. Product teams should not create channel-specific copies of identity, conversation, routing or engagement logic when the capability belongs in Support or the wider WISE context layer.
