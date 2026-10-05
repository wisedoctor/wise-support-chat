# WISE Support — GHCR Image Publication Checkpoint — 2026-10-03

**Repository:** `wisedoctor/wise-support-chat`  
**Branch at build/publication:** `fix/telegram-retry-idempotency`  
**Commit:** `a8e85786f92bc85f67d38ddf6dbbec1689c671e5`  
**Purpose:** Record the immutable GHCR publication of the Telegram retry/idempotency candidate image for controlled OCI validation.

---

## 1. Build state

The local Docker image was confirmed as:

- OS: `linux`
- Architecture: `amd64`
- source image: `wise-support-chat:telegram-retry-idempotency`

The image was not rebuilt during the GHCR publication retries.

---

## 2. GHCR publication

Published tag:

```
ghcr.io/wisedoctor/wise-support-chat:sha-a8e85786
```

The first push attempt timed out during layer upload. A subsequent retry encountered a transient Docker Desktop DNS/network failure resolving `ghcr.io`. After network recovery, the third push completed successfully, with existing layers reused by GHCR.

No image rebuild was required.

---

## 3. Registry verification

GHCR returned:

```
Digest:
sha256:8212ed18698c959105376a0f3c1eef1780e6cd096ece885e83cb07ea28e0ccb0

MediaType:
application/vnd.oci.image.index.v1+json
```

The registry manifest inspection showed:

```
Platform: linux/amd64
Manifest digest:
sha256:69f4ad624253f68d73fb591cb08cf260373d28a4e35167341056334f20a37942
```

An additional `unknown/unknown` OCI attestation manifest was present and referenced the AMD64 image manifest. This is expected registry attestation metadata and is not a second runtime platform.

### Immutable OCI deployment reference

For controlled deployment, use the image-index digest:

```
ghcr.io/wisedoctor/wise-support-chat@sha256:8212ed18698c959105376a0f3c1eef1780e6cd096ece885e83cb07ea28e0ccb0
```

---

## 4. Relationship to the current rehearsal image

The previously deployed rehearsal image remains the rollback reference:

```
ghcr.io/wisedoctor/wise-support-chat:sha-a6b2176
```

The new `sha-a8e85786` image has **not yet been deployed to the OCI WEB/WORKER runtime** at the time of this checkpoint.

Do not remove or overwrite the existing rehearsal image/container state until the new candidate has passed controlled validation.

---

## 5. Required next operational sequence

### Step 1 — validate code/tests

Complete the outstanding focused Telegram retry/idempotency test validation locally.

In particular, ensure the test uses the correct Chatwoot attachment API:

```ruby
attachment.file.blob.service
```

rather than the incorrect:

```ruby
attachment.blob.service
```

Then distinguish any retry/idempotency failures from the three previously observed conversation-selection baseline failures.

### Step 2 — capture current OCI runtime

Before replacing WEB/WORKER, capture the current:

- image;
- environment variables with secrets redacted;
- network;
- port mapping;
- command;
- restart policy;
- PostgreSQL/Redis configuration;
- Active Storage configuration;
- `FRONTEND_URL=https://support.wisehealth.in`;
- Telegram configuration.

### Step 3 — controlled OCI rollout

Replace WEB and WORKER together with the immutable GHCR digest:

```
ghcr.io/wisedoctor/wise-support-chat@sha256:8212ed18698c959105376a0f3c1eef1780e6cd096ece885e83cb07ea28e0ccb0
```

Do not change Nginx, PostgreSQL, Redis, Telegram webhook ownership, or bot identity as part of this image rollout.

### Step 4 — runtime verification

Verify:

- WEB starts and serves Chatwoot;
- WORKER starts and processes jobs;
- both use the new image digest;
- both retain OCI S3-compatible Active Storage;
- `FRONTEND_URL` remains correct;
- stable HTTPS ingress remains healthy;
- Telegram webhook remains on `support.wisehealth.in`.

### Step 5 — regression/E2E

Run the established Telegram text regression first.

Then exercise the retry/idempotency scenario that originally produced duplicate inbound attachment messages.

Preserve the proven two-way text path as the regression baseline.

---

## 6. Security / operational note

The GHCR PAT used for publication was updated to include package write permission. The credential is not recorded here.

The OCI storage secret used during earlier diagnostics was exposed during the rehearsal and remains a separate security follow-up. Rotate it before treating the environment as production-safe.

---

## 7. Status

**GHCR publication: COMPLETE**

**Registry verification: COMPLETE**

**OCI deployment: PENDING**

**Focused test validation: PENDING**

**Controlled pilot rollout: PENDING**

This checkpoint intentionally separates image publication from runtime deployment so that the OCI rehearsal remains rollback-safe.


## Local validation completed — 2026-10-03

The published candidate was traced back to branch `fix/telegram-retry-idempotency`, HEAD `a8e85786f`.

Focused RSpec was executed using the test-capable image containing the corrected source:

`wise-support-chat:test-retry-idempotency`

Result:

```
35 examples, 3 failures
```

The two failures caused by the earlier incorrect `attachment.blob` API are resolved. The remaining three failures are the pre-existing conversation-selection cases and are outside the retry/idempotency change.

The immutable candidate remains the controlled OCI rollout target:

`ghcr.io/wisedoctor/wise-support-chat@sha256:8212ed18698c959105376a0f3c1eef1780e6cd096ece885e83cb07ea28e0ccb0`

The image is therefore **locally validated for the intended retry/idempotency slice; OCI runtime validation remains pending**.

## /start support delivery fix — 2026-10-06

The previously published /start support candidate (image-index digest sha256:0b1c591f56a1e7ad180f67a0646ca6fc04b957abcd01fbb96f2e183eef467d5b) was deployed and runtime-validated, but the WISE welcome was not delivered to Telegram.

Forensic evidence showed Telegram was correctly sending /start support and Chatwoot was creating the outgoing welcome, but the welcome used source_id as its idempotency marker. Chatwoot's outbound channel guard interprets a populated source_id as channel-originated and therefore skipped Telegram delivery.

The fix on feature/telegram-start-welcome removes source_id from the welcome message, while retaining retry idempotency in additional_attributes.telegram_start_message_id.

### Corrected source commits

- 583455f9445d41467c83d9af1c27e42dc9499ab9 — fix(telegram): keep start welcome eligible for channel delivery
- 4a985aae696aa8cb94247f215976fc4c0a750456 — test(telegram): keep start welcome idempotent and deliverable
- preceding /start support candidate commits remain part of the branch history.

### Required publication sequence

1. Run the focused IncomingMessageService RSpec suite locally.
2. Confirm the new test covers both `source_id` remaining nil and duplicate-start suppression.
3. Build a new linux/amd64 image from the corrected branch.
4. Publish a new immutable GHCR tag/digest.
5. Verify the registry digest before deployment.
6. On OCI, preserve the current 0b1c591f56a1... candidate as rollback and replace WEB + WORKER only.
7. Verify HTTPS 200, WEB/WORKER health, OCI Active Storage and FRONTEND_URL.
8. Run the Telegram /start support welcome E2E.
9. Run normal two-way text regression and one duplicate/retry start check.

Do not alter Nginx, DNS/TLS, PostgreSQL, Redis, Telegram webhook ownership, or either Telegram bot as part of this fix.

**Corrected image publication:** PENDING

**Corrected OCI deployment:** PENDING

**Telegram-visible welcome E2E:** PENDING

## Corrected `/start support` image — OCI pull verified 2026-10-06

The corrected application image has completed the publication gate and has been pulled to the OCI validation VM by immutable digest.

- branch: `feature/telegram-start-welcome`
- latest code/docs commit at time of publication: `a355512ab162e60a1d3329f16256e0cd95105e75`
- image tag: `ghcr.io/wisedoctor/wise-support-chat:sha-a355512a`
- registry image-index digest: `sha256:fc11023a6f146ca475bd003a2071d6dc54f0dbaa2d3c868c792e8f1acad3df19`
- linux/amd64 manifest: `sha256:3e3f937932d6f604870eb50f9afeef533f22eb6021eeba1893a6e049439f1a91`
- OCI pull: PASS
- OCI `RepoDigests`: confirmed exact image-index digest
- OCI image ID: `sha256:fc11023a6f146ca475bd003a2071d6dc54f0dbaa2d3c868c792e8f1acad3df19`

The currently running OCI WEB + WORKER candidate (`sha256:0b1c591f56a1...`) remains the rollback baseline and has not been replaced yet.

### Next gate

Perform an image-only WEB + WORKER replacement using the exact immutable digest above. Preserve the captured environment files and current rollback containers. Do not alter Nginx, DNS/TLS, PostgreSQL, Redis, Active Storage configuration, Telegram webhook ownership, or either bot identity.

After replacement, validate container startup and HTTPS first, then perform the fresh `/start support` Telegram E2E. The final gate remains Telegram-visible WISE welcome delivery, same-conversation patient reply, and retry-safe single welcome.

