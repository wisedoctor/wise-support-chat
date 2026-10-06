# WISE Support — Website Inbox Configuration & Web-Channel Notes — 2026-10-06

**Status:** Pilot-prep operational checkpoint — **Web Support E2E CLOSED / PASS**  
**Environment:** OCI-hosted Chatwoot / WISE Health Website Support  
**Public portal:** https://support.wisehealth.in  
**WISE entry point:** https://wisehealth.in/support

## 1. Website Widget token — authoritative configuration rule

The Website Inbox has more than one token/credential visible in the Chatwoot UI. They must **not be conflated**.

### Website Widget token

The authoritative Website Widget token for the running WISE Health™ Support channel is the value embedded in the generated **Script** shown under the Website Inbox **Settings** page.

The generated script contains:

    window.chatwootSDK.run({
      websiteToken: '<website widget token>',
      baseUrl: 'https://support.wisehealth.in'
    })

This is the value persisted as:

    Channel::WebWidget.website_token

and is the value required by the browser widget request:

    /widget?website_token=<website widget token>

### Identity Validation credential

The **Configuration → Identity Validation → Secret Key** is a separate credential in Chatwoot's documented identity-validation feature. It is used for identity verification/HMAC flows and is **not interchangeable with the Website Widget token**.

Operationally, for this deployment, the key rule is simple:

- do not invent or substitute a value into the generated widget Script;
- use the Website Widget token that Chatwoot generates in the Script;
- keep the Identity Validation credential separate unless identity validation is deliberately enabled and implemented.

### Vercel synchronization rule

The WISE Health frontend variable:

    VITE_WISE_SUPPORT_CHAT_WEBSITE_TOKEN

must match the Website Widget token shown in the generated **Script** tab.

The current working configuration was restored by synchronizing Vercel with the Script-tab website token.

**Operational rule:** when troubleshooting a web-widget token, inspect the generated Script first. Do not use the Identity Validation Secret Key shown on the Configuration tab.

## 2. Website URL and Allowed Domains

Current Website Inbox:

- Name: WISE Health™ Support
- Website URL: https://wisehealth.in
- Allowed Domains: blank during pilot validation

Blank Allowed Domains is currently intentional for validation. It is not the cause of the earlier widget failure.

The widget endpoint was independently validated with the database-backed Website Widget token and returned HTTP 200 with rendered Chatwoot widget HTML.

## 3. Widget avatar

The Website Inbox **Channel Avatar** is a widget presentation setting.

Chatwoot exposes the inbox avatar as avatar_url, and the rendered widget configuration supplies it as:

    window.chatwootWebChannel.avatarUrl

Therefore the avatar configured for the WISE Health™ Support inbox is consumed by the web widget.

The exact visual placement can vary by widget state/version (welcome panel, conversation/header or related channel presentation). Treat avatar selection as presentation/configuration, not as a routing or identity control.

Pilot guidance:

- use the WISE Health / WISE Support brand asset;
- keep the avatar legible at small size;
- do not use a patient/person-specific image;
- changing the avatar does not require changing the website token or Vercel configuration.

## 4. Widget welcome vs conversation greeting

Chatwoot currently exposes two distinct layers:

### Widget welcome

Configured under Widget features:

- Welcome Heading
- Welcome Tagline

Current WISE Health™ Support wording is configured as:

    Welcome to WISE Health™ Support. We’re here to help with your care journey, medicine requests, and anything that needs human help.

This is the pre-conversation widget presentation.

### Channel greeting

Under Channel Preferences, **Enable channel greeting** is enabled.

Chatwoot describes this as:

    Auto-send greeting messages when customers start a conversation and send their first message.

The configured greeting currently observed in the web conversation is:

    Let us know, what care help you looking for.
    No unnecessary formalities... we will guide you with the best care path whatever be your current situation.

This explains why the conversation-level greeting currently appears **after the patient's first message**, rather than immediately when the patient clicks Start Conversation.

For the pilot, this is accepted as current Chatwoot behaviour. A future UX enhancement may investigate whether the greeting can be delivered at conversation creation/start rather than after the first inbound message.

Do not implement a frontend-generated fake/hidden patient message merely to force the greeting.

## 5. Web Support E2E checkpoint

The controlled fresh-browser rehearsal has now proven:

    wisehealth.in/support
          ↓
    Chatwoot Website Widget
          ↓
    Start Conversation
          ↓
    Patient message
          ↓
    WISE Health™ Support Inbox
          ↓
    Default Policy assignment
          ↓
    Sreedhar Byreeka / Mine
          ↓
    Support Rep reply
          ↓
    Browser receives reply

The successful test recorded **Assigned to Sreedhar Byreeka by Default Policy** and demonstrated two-way messaging in the same conversation.

The earlier Unassigned observations are retained as a diagnostic/operational observation. The controlled test confirms that automatic assignment works when the normal Chatwoot agent availability/presence path is satisfied.

## 6. Assignment configuration observed

The WISE Health™ Support Inbox currently shows:

- collaborators: Kranthi; Sreedhar Byreeka
- automatic conversation assignment: enabled
- Default assignment rules
- earliest-created conversations first
- round-robin distribution

Chatwoot's assignment implementation additionally depends on runtime availability/online presence. Inbox membership and a static logged-in/superadmin state should not be treated as sufficient proof of eligibility.

For the current web pilot checkpoint, do not manually rewrite assignment logic or create a custom workaround until the normal Chatwoot assignment path has been re-tested with active presence.

## 7. Anonymous web visitor naming

Fresh anonymous web conversations may receive generated names such as:

- Muddy-Snowflake-479
- Rough-Rain-963

This is expected Chatwoot anonymous-contact behaviour observed during the rehearsal. It is not currently a pilot defect and should not be customized as part of this workstream.

## 8. Operational guardrails

- Keep the Website Widget token and Identity Validation Secret Key conceptually separate.
- Never commit either secret to source control.
- Vercel frontend configuration must use the Website Widget token from the generated Script.
- Do not rotate the token merely because the Identity Validation Secret Key differs.
- Keep provider-specific widget configuration inside wise-support-chat; quick-chat-landing owns the WISE web entry point.
- Do not introduce frontend workarounds to simulate Chatwoot conversation events.
- Record configuration changes in the Support operational documentation.

## 9. Closure evidence

The native Web Support lifecycle is now **PASS / CLOSED for pilot preparation**.

The final persistence/re-entry test confirmed that the Website Inbox conversation and history remained available after:

- closing and reopening the widget;
- refreshing the WISE Health `/support` page;
- logging out of Chatwoot and logging back in;
- reopening the Inbox/conversation view.

Together with the earlier controlled fresh-browser test, the proven lifecycle is:

    wisehealth.in/support
          ↓
    Chatwoot Website Widget
          ↓
    Start Conversation
          ↓
    Patient message
          ↓
    WISE Health™ Support Inbox
          ↓
    Default Policy assignment
          ↓
    Support Rep / Mine
          ↓
    Support Rep reply
          ↓
    Browser receives reply
          ↓
    Conversation/history persists across re-entry

## 10. Remaining non-blocking UX consideration

The channel greeting currently appears after the patient's first message because that is the configured Chatwoot Channel Greeting behaviour. This is accepted for pilot readiness and is not a closure blocker.

If the positioning later requires an immediate post-Start welcome, treat that as a separate small UX enhancement. Do not reopen the closed E2E workstream merely to change greeting timing.

The welcome wording should remain flexible; if code-level environment configuration is later required, implement it within the Support implementation boundary rather than the WISE entry-point page.

## 11. Formal handover

Include this closure evidence in the formal Ops Support SOP/handover before moving to the RBAC workstream.
