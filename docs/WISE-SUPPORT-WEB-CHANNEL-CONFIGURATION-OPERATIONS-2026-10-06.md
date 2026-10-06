# WISE Support — Website Inbox Configuration & Web-Channel Notes — 2026-10-06

**Status:** Pilot-prep operational checkpoint  
**Environment:** OCI-hosted Chatwoot / WISE Health Website Support  
**Public portal:** https://support.wisehealth.in  
**WISE entry point:** https://wisehealth.in/support

## 1. Website Token — authoritative configuration rule

The WISE Health Website Inbox exposes several token/credential values in the Chatwoot UI. They are **not interchangeable**.

### Website Widget token

The definitive Website Widget token is the value embedded in the generated **Script** shown under the Website Inbox **Settings** page.

The generated script contains:

    window.chatwootSDK.run({
      websiteToken: '<website widget token>',
      baseUrl: 'https://support.wisehealth.in'
    })

This websiteToken is the value persisted as:

    Channel::WebWidget.website_token

and is the value required by the browser widget request:

    /widget?website_token=<website widget token>

### Identity Validation Secret Key

The **Configuration** tab contains a separate **Identity Validation → Secret Key**.

This is an identity-validation/HMAC credential. It is **not** the Website Widget token and must not be copied into:

    VITE_WISE_SUPPORT_CHAT_WEBSITE_TOKEN

Do not rotate or replace either credential merely to make them match.

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

The following is proven:

    wisehealth.in/support
          ↓
    Chatwoot Website Widget
          ↓
    Start Conversation
          ↓
    Patient message
          ↓
    WISE Health™ Support Inbox

The current fresh-browser rehearsal showed the new web conversations arriving in **Unassigned**.

Automatic assignment remains an operational verification item and must be tested separately with a genuinely new conversation while an eligible agent has active Chatwoot online presence.

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

## 9. Follow-up items

1. Complete web E2E: patient → Inbox → assignment/claim → Ops reply → browser.
2. Verify conversation persistence/re-entry.
3. Reconcile the final auto-assignment behaviour under active agent presence.
4. Decide whether the channel greeting timing is acceptable for pilot or needs a small Chatwoot-side configuration/implementation change.
5. Keep the welcome wording flexible; if code-level environment configuration is later required, implement it within the Support implementation boundary rather than the WISE entry-point page.
