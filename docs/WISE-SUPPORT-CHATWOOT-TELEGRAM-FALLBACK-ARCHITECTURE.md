# WISE Support Module — Chatwoot / Telegram Fallback Architecture

**Status:** Architecture reference / future implementation guide
**Current implementation:** Chatwoot dress rehearsal in progress; formal fallback/restore mechanics deferred until after current integration work and RBAC.

## 1. Core Design Principle

The WISE Support module is a **replaceable engine**, not a fixed solution. Chatwoot is the current implementation of the support-conversation engine. Direct Telegram support is the fallback implementation. The architecture must also allow Chatwoot to be replaced by another support provider, or temporarily operate without a support portal.

Chatwoot is therefore an implementation layer behind WISE Support capabilities and contracts; it is not the definition of the WISE Support module.

## 2. The Two Support Paths

### Path A — Chatwoot-backed support

```text
Patient
   ↕
Telegram Bot
   ↕
(( WISE / Chatwoot webhook boundary ))
   ↕
Chatwoot
   ↕
Support Rep
```

Current dress rehearsal:

```text
Patient ↔ @wise_chatwoot_poc_bot ↔ Telegram ↔ Chatwoot Telegram Channel/Inbox ↔ Chatwoot conversation ↔ Support Rep
```

Where required, Chatwoot webhook events cross back into the WISE Support/core application so support remains integrated with wider WISE workflows.

### Path B — Direct Telegram fallback

```text
Patient
   ↕
Telegram Bot
   ↕
(( WISE Support Telegram handling ))
   ↕
Support Rep
```

This path has **no Chatwoot dependency**. The support representative handles the conversation directly through Telegram. An Android device is the accepted operational mitigation for known Telegram/iPhone-client quirks.

Once formally implemented, fallback should use Telegram polling (`getUpdates`) rather than simultaneously consuming the same bot through webhook and polling.

### Important transport distinction

The two paths share the support-channel concept, but the Telegram transport mechanics differ once fallback is formally implemented:

- `CHATWOOT_ACTIVE`: Telegram webhook delivery into the Chatwoot path.
- `FALLBACK_ACTIVE`: Telegram webhook removed and a controlled polling process consumes updates.

## 3. Why the Two-Bot Dress Rehearsal Matters

The current use of two Telegram bots is also a practical rehearsal of keeping the two support paths separate:

```text
                         ┌── Chatwoot ─────→ Support Rep
Patient → Telegram Bot ──┤
                         └── WISE direct ──→ Support Rep
```

The bot identity is **not** the architectural abstraction. The abstraction is:

```text
WISE Support
     │
     ├── Chatwoot support-engine adapter
     ├── Direct Telegram support adapter
     └── Future support-engine adapter(s)
```

The current POC bot and the existing WISE Support bot therefore remain deliberately separate during the dress rehearsal.

## 4. Two Operational Modes

| Mode | Telegram handling | Support Rep interface | Patient experience |
|---|---|---|---|
| `CHATWOOT_ACTIVE` | Telegram webhook → Chatwoot | Chatwoot web dashboard | Telegram chat; replies via Chatwoot |
| `FALLBACK_ACTIVE` | Telegram polling → WISE direct support handling | Telegram app, Android preferred | Telegram chat; replies via Telegram |

**Critical constraint:** a Telegram bot must operate through either webhook delivery or polling. The same bot must not be actively consumed through both mechanisms at once. Switching must be controlled and atomic.

## 5. Transition Tasks

### Alert
Notify the authorised operator/Product Head of health failure, mode transition, fallback activation, recovery, restore completion, and repeated transition conditions.

### Close the current source tap
- Fallback: call `deleteWebhook` with `drop_pending_updates: false`, then verify with `getWebhookInfo`.
- Restore: stop the fallback polling process and verify that polling has stopped.

### Open the target tap
- Fallback: start the controlled polling process using `getUpdates` and verify it is active.
- Restore: call `setWebhook` for the Chatwoot endpoint with the configured secret where applicable, then verify with `getWebhookInfo`.

## 6. Trigger Paths

### Automatic fallback
A future health-check process may monitor Chatwoot Web, PostgreSQL, Redis, Sidekiq, and relevant public ingress/webhook health. On confirmed failure: alert → acquire lock → `TRANSITIONING` → close webhook → verify → start polling → verify → `FALLBACK_ACTIVE` → log transition → release lock.

### Planned fallback
The authorised Product Head/Super Admin can choose **Force Fallback**. The same controlled transition sequence is used without requiring Chatwoot to have failed.

### Planned restore
The authorised Product Head/Super Admin can choose **Force Restore**: acquire lock → `TRANSITIONING` → stop polling → verify → `setWebhook` → verify → `CHATWOOT_ACTIVE` → log and alert → release lock.

### Automatic restore
Automatic restore should require sustained health rather than a single successful check. Future safeguards should include N consecutive successful checks, a cooldown period, and flip-flop protection.

## 7. State Machine and Locking

```text
telegram_mode = CHATWOOT_ACTIVE | FALLBACK_ACTIVE | TRANSITIONING
```

Rules:
- All automatic and manual transitions acquire a lock.
- `TRANSITIONING` prevents competing fallback/restore operations.
- `deleteWebhook` and `setWebhook` should be safe to retry.
- Multiple polling processes must be prevented; Telegram can return `409 Conflict`.
- After every mode change, `getWebhookInfo` must confirm the actual Telegram state.
- Persist the verified resulting state, not merely the requested state.
- Record timestamp, trigger, operator identity where applicable, and verification/error details.

## 8. Super Admin / RBAC

The eventual Super Admin control surface should display current mode, last health check, last transition, trigger, alert history, and active support-engine/channel information.

Manual controls:
- **Force Fallback**
- **Force Restore**

These controls must be protected by RBAC. Super Admin may change support mode; Ops Head, Ops Rep and Support Rep are operational support roles and should not receive unrestricted mode-switch authority.

Every manual state change must be auditable. No mode change should depend on ad-hoc scripts or manual Telegram API calls outside the controlled support administration path.

## 9. Separation of Concerns

The support engine must not become the authoritative owner of WISE business workflows.

```text
Channel / Provider
       ↓
WISE Support
       ↓
Identity / Context / Need / Route / Interaction / Signal
       ↓
Authoritative WISE domain module
       ↓
Audited workflow mutation
```

Chatwoot may provide the conversation UI, agent workspace, contact/conversation state and channel integration. Direct Telegram may provide patient/support-rep communication during fallback. Neither should replace WISE's authoritative domain systems.

## 10. Known Operational Limitation

**Telegram iPhone client quirks:** ensure an Android device is available for support representatives in direct Telegram fallback mode. This is an operational mitigation, not an architectural dependency.

## 11. Future Data Model

Potential future records include:

### `support_mode_state`
Current mode, last transition, trigger, lock, health status and cooldown information.

### `support_alert_log`
Alerts sent to the authorised operator.

### `support_transition_log`
Timestamp, previous/requested/verified mode, trigger source, operator identity, verification result and error details.

Exact schema should be defined when fallback/restore implementation begins.

## 12. Deferred Implementation

The complete fallback state machine is intentionally **not** part of the current Chatwoot dress rehearsal.

Future work includes:
- health-check worker
- alert dispatcher
- support-mode state manager and database locking
- Telegram webhook/polling controller
- controlled fallback polling process
- automatic fallback and restore
- Super Admin endpoints such as `/force-fallback`, `/force-restore`, `/status`
- Super Admin dashboard and RBAC enforcement
- monitoring, backup/recovery, production ingress and secret management.

## 13. Current Dress-Rehearsal Boundary

The immediate objective is to prove:

1. Chatwoot can operate as the current support engine.
2. A separate Telegram bot can be onboarded into Chatwoot.
3. Telegram → Chatwoot → Support Rep works.
4. Chatwoot → Telegram works.
5. Chatwoot events can cross the WISE Support webhook boundary correctly.
6. The existing WISE direct Telegram support path remains separate.
7. The two bots and two support paths do not become accidentally coupled.
8. RBAC can subsequently become the formal control point for support-engine mode changes.

Once the current integration and RBAC work is complete, fallback/restore mechanics can be implemented against this document rather than rediscovered during an outage.

## 14. Summary

```text
                 ┌── Chatwoot support engine ──→ Support Rep
Patient ↔ Telegram
                 └── Direct Telegram fallback ─→ Support Rep
```

WISE Support should survive the presence, absence, replacement, planned retirement, or unexpected failure of Chatwoot.

The long-term design is therefore:

> **WISE Support = channel/provider-neutral support capability, with replaceable support engines and controlled operational modes.**

Chatwoot is the current engine. Direct Telegram is the fallback engine. A future provider can replace Chatwoot without changing the core support architecture.

## 15. Source Reference

This document is based on the original **Wise Support Module — Chatwoot / Telegram Fallback Architecture** reference and incorporates the clarified two-path representation agreed during the Chatwoot dress rehearsal.

The source establishes the replaceable-engine principle, Chatwoot-active/fallback modes, Telegram webhook/polling transition, state machine, Super Admin controls, audit requirements, and Android mitigation for Telegram iPhone client quirks.