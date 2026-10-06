# WISE Support — Assignment / Inbox / Conversation Operating Model — 2026-10-03

**Status:** Pilot operating-model checkpoint  
**Environment:** OCI hosted dress rehearsal  
**Public portal:** https://support.wisehealth.in  
**POC channel:** @wise_chatwoot_poc_bot

## Purpose

Capture the operational behaviour now proven during the Telegram + Chatwoot rehearsal so that assignment/inbox behaviour can be treated as an SOP concern rather than continuing as an open technical mystery.

This document deliberately distinguishes **observed/proven behaviour** from lifecycle cases that still require formal SOP rehearsal.

## 1. Core concepts

### Inbox

The Chatwoot Inbox is the channel/support queue context.

For the current rehearsal, the relevant Telegram inbox is:

`wise_chatwoot_poc_bot`

The WISE Health Website Inbox is a separate channel/inbox:

`WISE Health™ Support`

These are separate channel contexts and must not be conflated.

### Conversation

A Conversation is the individual patient/support thread inside an Inbox.

The operational unit for a support representative is therefore:

`Inbox → Conversation → assigned support representative`

### Mine

**Mine** represents conversations assigned to the currently logged-in support representative.

For the latest inbound document + image validation, both messages landed in **Mine**.

### Unassigned

**Unassigned** represents conversations received by the Inbox but not currently assigned to the support representative.

The earlier rehearsal had observed both Unassigned and subsequently assigned states, so assignment timing/configuration remains a documented operational concern rather than an application defect by default.

## 2. Current proven Telegram operating path

```
Patient Telegram
      ↓
@wise_chatwoot_poc_bot
      ↓
stable HTTPS webhook
      ↓
Chatwoot Telegram Inbox
      ↓
Conversation
      ↓
Support Rep assignment
      ↓
Mine
      ↓
Support Rep reply
      ↓
Patient Telegram
```

The latest controlled OCI validation demonstrated this path with:

- inbound document;
- inbound image;
- no duplicate inbound messages;
- successful attachment retrieval/opening;
- messages visible in Mine;
- successful Support Rep replies back to Telegram.

## 3. Assignment evidence

### Telegram rehearsal

Earlier Telegram rehearsal evidence recorded a system-generated assignment:

`Assigned to Sreedhar Byreeka by Default Policy`

No manual assignment script was used for that successful assignment. This remains valid evidence that Chatwoot's normal automatic-assignment path can work when an eligible agent is available through the runtime presence path.

### Website Inbox rehearsal

The latest fresh-browser WISE Health™ Support tests created genuinely new Website Inbox conversations. Both remained in **Unassigned** after waiting and after logging out/in again.

The Website Inbox configuration currently shows:

- collaborators: Kranthi and Sreedhar Byreeka;
- automatic conversation assignment: enabled;
- Default assignment rules;
- earliest-created conversations first;
- round-robin distribution.

The current evidence therefore does **not** justify marking Website Inbox auto-assignment PASS. The next controlled test must verify runtime agent availability/presence while creating a genuinely new web conversation.

### Current conclusion

**Telegram assignment path: previously proven PASS for the tested scenario.**

**Website Inbox auto-assignment: OPEN / not yet proven.**

Do not work around this by manually assigning every web conversation. First verify the normal Chatwoot availability and assignment path.

## 4. Required lifecycle SOP

Before formally closing the assignment/inbox workstream, rehearse and document:

1. **New conversation**
   - patient starts a new support conversation;
   - conversation reaches the intended Inbox;
   - assignment occurs as configured.

2. **Assigned conversation**
   - conversation appears in Mine for the assigned rep;
   - rep can reply;
   - reply reaches the patient.

3. **Unassigned conversation**
   - verify why/when a conversation can remain Unassigned;
   - verify the intended operational action to claim/assign it;
   - distinguish normal timing/configuration from an actual failure.

4. **Continuation**
   - subsequent patient messages remain in the expected Conversation;
   - they do not create an unintended second conversation.

5. **Resolution**
   - rep resolves the conversation;
   - verify the resulting Inbox/Mine state.

6. **Patient returns**
   - verify the behaviour when the patient sends a new message after resolution;
   - establish whether the existing conversation reopens/continues or a new conversation is created;
   - record the behaviour as the Chatwoot operating rule for the pilot.

7. **Refresh / re-entry**
   - refresh the portal;
   - leave and re-enter the Inbox;
   - confirm message/conversation counts do not change unexpectedly.

## 5. Operational guardrails

- Do not manually assign every conversation as a workaround.
- Do not infer assignment failure solely from a transient Unassigned state without checking worker/configuration timing.
- Do not modify Telegram webhook ownership during assignment testing.
- Do not mix the POC Telegram bot with the existing `@wescura_support_bot`.
- Do not change the WISE domain/request assignment model as part of Chatwoot assignment tuning.
- Keep Chatwoot/provider-specific assignment behaviour within the Support implementation boundary; WISE domain authorization remains a separate concern.

## 6. Architecture boundary for future RBAC

The Support channel/inbox assignment model is not the WISE RBAC model.

The broader Ops architecture remains:

`User → Role → Permissions → Modules → Queues → Actions`

Chatwoot assignment determines the current support work ownership within the Support capability. Future WISE RBAC must determine which users can enter the Support capability, see relevant queues/modules and perform permitted business actions.

Do not create a second independent RBAC system inside `wise-support-chat`.

## 7. Pilot disposition

### Proven

- Telegram Inbox receives patient conversations.
- Fresh inbound document and image messages appear once.
- Attachments open successfully.
- Tested conversations appear in Mine.
- Support Rep replies return to Telegram.

### Remaining formal SOP coverage

- Unassigned handling.
- Resolution and post-resolution re-entry.
- Continuation vs new conversation semantics.
- Refresh/re-entry consistency.
- Explicit support-rep training/operating procedure.

These are operational verification items, not a reason to reopen the proven attachment/idempotency implementation.

## 8. Next pilot-prep workstream

After recording this checkpoint, proceed to:

1. Welcome message / Telegram `/start` UX.
2. Context handoff.
3. Production entry-point audit.
4. RBAC + initial operational hand-off.

The assignment/inbox lifecycle cases above can be completed as a focused SOP rehearsal without extending the underlying Telegram implementation.
