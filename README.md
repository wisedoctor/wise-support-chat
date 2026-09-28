<img src="./.github/screenshots/header.png#gh-light-mode-only" width="100%" alt="Header light mode"/>
<img src="./.github/screenshots/header-dark.png#gh-dark-mode-only" width="100%" alt="Header dark mode"/>

___

# WISE Support — Chatwoot Implementation

This repository currently hosts the Chatwoot-based implementation used to develop and validate the **WISE Support** channel/conversation servicing foundation.

Chatwoot is an implementation/provider within the wider WISE Support architecture, not the definition of WISE Support itself. The intended model is channel-neutral and provider-neutral so that Telegram, WISE Web Chat, WhatsApp, email, or other future channels can use the same support engine and context model.

## Current status

**Development/local environment: E2E validated.**

The current local setup has successfully demonstrated:

- Telegram inbound message → Chatwoot;
- Chatwoot conversation/contact/message creation;
- visibility in the Chatwoot Inbox;
- WISE/Wescura support-side inbound notification;
- support representative reply from Chatwoot;
- outbound Chatwoot → Telegram → patient delivery.

Production setup and production E2E validation are still pending.

## Documentation

### WISE Support current environment & maintenance runbook

See [`docs/WISE_SUPPORT_CURRENT_ENVIRONMENT_AND_MAINTENANCE_RUNBOOK.md`](docs/WISE_SUPPORT_CURRENT_ENVIRONMENT_AND_MAINTENANCE_RUNBOOK.md) for the current proven development topology, including:

- WSL/Conda + Docker environment details;
- Rails, Sidekiq, Redis and PostgreSQL processes;
- WISE frontend/backend processes;
- development-only ngrok and `wescura_proxy.py` setup;
- the two current webhook destinations and routing;
- Telegram/Chatwoot implementation details;
- start/stop/restart commands;
- process and health checks;
- Rails console/shell access;
- Rails/Sidekiq/proxy/ngrok logs;
- inbound/outbound troubleshooting workflow;
- DEV → PROD configuration mapping;
- security and operational hygiene;
- the current known-good E2E baseline.

The runbook explicitly separates the **WISE Support abstraction** from its current Chatwoot/Telegram implementation so that future channel/provider changes do not change the underlying architecture.

## Upstream project

This repository is based on the open-source Chatwoot project. The upstream project remains the reference for the underlying Chatwoot platform and its general documentation.

## Branching model

The working base branch for this repository is `develop`.

## License

Chatwoot is released under the MIT License. See the upstream project and repository license files for applicable licensing information.
