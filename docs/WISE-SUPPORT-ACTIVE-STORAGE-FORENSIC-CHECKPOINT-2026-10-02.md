# WISE Support — Active Storage Forensic Checkpoint
## 2026-10-02

**Status:** Forensic diagnosis complete; shared persistent storage remediation ready to begin  
**Environment:** OCI dress rehearsal / pilot-prep runtime  
**Repository:** `wisedoctor/wise-support-chat`  
**Branch:** `develop`

## 1. Purpose

Record the storage investigation performed after Telegram/Chatwoot attachment delivery failures exposed Active Storage file-availability problems.

This checkpoint deliberately separates:

- evidence preservation and diagnosis;
- storage remediation design;
- actual storage configuration changes.

No Chatwoot storage backend change was made during the forensic investigation.

## 2. Confirmed current Active Storage configuration

The deployed Chatwoot image is using Rails Active Storage with:

```yaml
local:
  service: Disk
  root: <%= Rails.root.join("storage") %>
```

The runtime therefore uses:

```
ActiveStorage::Service::DiskService
        ↓
/app/storage
```

No S3/S3-compatible environment override was present in the inspected runtime configuration.

## 3. Container filesystem evidence

Neither Chatwoot application container has a Docker mount for `/app/storage`:

```
wise-support-chat-web     Mounts: []
wise-support-chat-worker  Mounts: []
```

The WEB container currently contains exactly three physical Active Storage files.

All three are 8-byte diagnostic uploads with the same SHA-256:

```
1d5f671fbc083af9a0ac801f24b93569fc6f9702af3fceee0ea7ca1a0018f001
```

Their keys correspond to recent `patient_guidance_test_doc.txt` uploads:

- `598kn56ozb7skhrsu5dm7r7p9m71`
- `bd2m557vhad2hqf22bqf14vspp4o`
- `roo8jmr0z2bk9q47yqx30bgjbghs`

The WORKER container has no corresponding files.

A Rails-level `ActiveStorage::Blob#service.exist?` mapping confirmed:

- WEB: blobs 8, 9 and 14 exist;
- WEB: blobs 1–7 and 10–13 are absent;
- WORKER: blobs 1–14 are absent.

Both containers report `ActiveStorage::Service::DiskService`.

## 4. Missing-file evidence

Rails logs recorded `ActiveStorage::FileNotFoundError` while attempting to process/read an attachment whose database blob metadata exists but whose physical DiskService file is unavailable.

This is consistent with the container-local storage topology and explains the observed attachment failures.

## 5. Anonymous Docker volume investigation

Docker reported one anonymous volume:

```
18d798c3fd8f6a756ca4c46ba811ae24b0887bd9e3b29cc520565d98b57520e0
```

Its host mountpoint was inspected read-only:

```
/var/lib/docker/volumes/18d798c3fd8f6a756ca4c46ba811ae24b0887bd9e3b29cc520565d98b57520e0/_data
```

Result:

```
0
```

The volume is empty and is not attached to WEB or WORKER.

**Disposition:** not a recoverable copy of Chatwoot Active Storage; do not repurpose or delete as part of the storage remediation without separate housekeeping/provenance review.

## 6. OCI Object Storage bucket investigation

An existing OCI Object Storage bucket was identified:

- Name: `oracle-oci-bucket-chatwoot-wisehealth`
- OCID: `ocid1.bucket.oc1.ap-hyderabad-1.aaaaaaaacdgkq6ucmmrxsaxsuebe2xwrpopfgoycds6q7tac7lwajgmeiphq`
- Region: `ap-hyderabad-1`
- Lifecycle state: `active`
- Created: `2026-09-29 05:41:31 UTC`

The OCI Console Objects view was inspected and showed:

```
No items to display
0 bytes combined Object Storage / Archive Storage usage
```

No historical Chatwoot/Active Storage objects were found in the bucket.

The bucket may have originated from the earlier Render validation attempt, because that historical architecture included OCI Object Storage as the intended S3-compatible attachment backend. Its original purpose is not asserted as proven by this checkpoint.

## 7. Storage remediation decision

The empty OCI bucket is now the identified candidate for a **persistent shared Active Storage backend**.

The target topology is:

```
                    OCI Object Storage
              oracle-oci-bucket-chatwoot-wisehealth
                           |
                  Active Storage S3
                           |
              +------------+------------+
              |                         |
        Chatwoot WEB              Chatwoot WORKER
        Rails/Puma                 Sidekiq
              |                         |
              +------ shared objects --+
```

This removes the current failure mode in which WEB and WORKER have separate container-local `/app/storage` filesystems.

The bucket should be reused only after access/configuration is verified.

## 8. Safe remediation sequence

The agreed sequence is:

1. Preserve/correlate the currently recoverable WEB files.
2. Verify OCI Object Storage S3-compatible endpoint, namespace and region.
3. Establish appropriate OCI Customer Secret Key credentials and required IAM permission.
4. Test S3-compatible access to the empty bucket without changing Chatwoot.
5. Configure Rails Active Storage for the S3-compatible backend.
6. Provide identical storage configuration to WEB and WORKER.
7. Recreate/restart containers deliberately.
8. Verify new object upload, WEB read and WORKER read/process.
9. Rerun Telegram text regression.
10. Rerun attachment/media E2E:
    - document;
    - image/Rx;
    - patient → Chatwoot;
    - Chatwoot → patient.
11. Only after successful E2E, update the pilot-prep attachment gate.

## 9. Guardrails

- Do not delete the existing PostgreSQL Active Storage metadata.
- Do not delete the three recoverable WEB files before preservation/correlation.
- Do not replace `/app/storage` with an empty Docker volume.
- Do not point either Telegram bot at a new webhook during storage work.
- Keep the existing `@wescura_support_bot` path untouched.
- Preserve the proven Telegram two-way text path as a regression test.
- Do not place OCI secret keys in Git, documentation or chat.
- Do not copy diagnostic 8-byte files into the new production-like object store as if they were valid patient attachments.

## 10. Next immediate step

The next step is **S3-compatible access validation against the existing empty OCI bucket**, before changing Chatwoot.

Required OCI inputs:

- Object Storage namespace;
- region: `ap-hyderabad-1`;
- bucket name: `oracle-oci-bucket-chatwoot-wisehealth`;
- Customer Secret Key access key + secret key;
- IAM permission sufficient for bucket/object access.

Credentials must remain outside Git and should be supplied to the VM only through a temporary secure environment/test mechanism until the final container configuration is agreed.



## 11. OCI S3-compatible storage probe — PASS

A standalone S3-compatible probe was executed from the live Chatwoot WEB container against the existing OCI bucket.

Inputs used:

- namespace: `ax3kknx0qqfm`
- region: `ap-hyderabad-1`
- bucket: `oracle-oci-bucket-chatwoot-wisehealth`
- endpoint:
  `https://ax3kknx0qqfm.compat.objectstorage.ap-hyderabad-1.oraclecloud.com`
- path-style access: enabled
- OCI Customer Secret Key credentials: supplied transiently to the test process; not stored in Git or documentation.

The probe successfully performed:

```
LIST bucket       PASS
PUT object        PASS
HEAD object       PASS
GET object        PASS
DELETE object     PASS
verify deletion   PASS
```

The temporary probe object was:

```
_wise_support_storage_probe_20261002.txt
```

It was deleted successfully and the subsequent HEAD check confirmed that it no longer exists.

### SDK version check note

The preliminary expression:

```
Aws::S3::VERSION
```

raised `NameError`. This was only a version-constant check; it did not indicate that the S3 SDK was unavailable. The subsequent AWS SDK S3 probe loaded and executed successfully, proving the required SDK functionality is available in the deployed Chatwoot runtime.

### Result

The OCI Object Storage layer is independently proven usable from the actual Chatwoot WEB container.

This removes OCI connectivity, credentials, endpoint reachability, and basic S3 object operations as unresolved blockers.

The next validation layer is Rails Active Storage itself, followed by deliberate persistent container configuration.



## 12. Rails Active Storage → OCI S3 probe — PASS

A second, Rails-level probe was executed from the live Chatwoot WEB container with the OCI S3-compatible configuration supplied transiently through environment variables.

Observed configuration:

- `ACTIVE_STORAGE_SERVICE=s3_compatible`
- service class: `ActiveStorage::Service::S3Service`
- `aws-sdk-s3`: `1.208.0`
- region: `ap-hyderabad-1`
- bucket: `oracle-oci-bucket-chatwoot-wisehealth`
- path-style access: enabled

The probe successfully performed through Rails Active Storage itself:

```
PUT       PASS
EXIST     PASS
DOWNLOAD  PASS
DELETE    PASS
VERIFY DELETE PASS
```

Temporary object:

```
_wise_support_active_storage_probe_20261002
```

The object was deleted and deletion was verified.

### Result

The complete storage chain is now independently proven from the actual Chatwoot runtime:

```
Rails Active Storage
        ↓
S3Service
        ↓
AWS SDK S3 1.208.0
        ↓
OCI S3 Compatibility API
        ↓
oracle-oci-bucket-chatwoot-wisehealth
```

Therefore OCI storage connectivity, credentials, endpoint configuration and Rails Active Storage S3 integration are no longer unknowns.

### Important distinction

This probe did **not** change the running Chatwoot containers' persistent configuration. It injected the S3-compatible settings only into the probe process.

The live Web/Worker containers therefore remain on their original DiskService configuration until the deliberate remediation step.

## 13. Next remediation step

The next step is now the controlled persistent configuration change:

1. preserve the currently recoverable local files;
2. capture the current WEB and WORKER container environment/configuration needed to recreate them faithfully;
3. configure `ACTIVE_STORAGE_SERVICE=s3_compatible` and the OCI storage variables identically for WEB and WORKER;
4. recreate/restart the containers without changing PostgreSQL, Redis, Telegram webhook ownership or Nginx;
5. verify Rails reports `S3Service` in both containers;
6. create a fresh attachment through Chatwoot and verify WEB + WORKER access;
7. rerun the Telegram text regression;
8. rerun document/image attachment E2E.

No storage configuration change should be made until the current container creation parameters are captured so that the existing rehearsal topology can be reproduced safely.
