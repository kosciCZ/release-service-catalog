# Direct Container Signing POC Design

## Overview

Replace the UMB/robosignatory-based container signing flow with direct `rh-signing-client-stage`
calls, while keeping the managed-task-creates-internal-requests architecture. Simplify the managed
task by removing existing-signature checks.

## Current Architecture

1. **`rh-sign-image`** (managed task) - iterates snapshot components, queries Pyxis for existing
   signatures (`find_signatures`), groups references/digests by signing keys, batches them, creates
   internal-requests pointing to `simple-signing-pipeline`
2. **`simple-signing-pipeline`** (internal pipeline) - runs `collect-simple-signing-params` (reads
   UMB/Pyxis config from configmap) then `request-and-upload-signature`
3. **`request-and-upload-signature`** (internal task) - sends signing request via UMB using
   `pubtools-sign-msg-container-sign`, checks response, uploads signatures to Pyxis

## Proposed Architecture

```
rh-push-to-registry-redhat-io pipeline
  └── rh-sign-image (managed, simplified)
       ├── Iterate components, collect references/digests
       ├── Skip existing-signature checks (sign everything)
       └── Create internal-requests → direct-signing-pipeline
            └── direct-sign-and-upload (internal task)
                 ├── Mount keytab from konflux-release-signing-stage-sa
                 ├── Call rh-signing-client-stage to sign
                 └── Upload signatures to Pyxis
```

## Files

### New files

1. `pipelines/internal/direct-signing-pipeline/direct-signing-pipeline.yaml`
   - Copy of `simple-signing-pipeline`, stripped of `collect-simple-signing-params`
   - Wired to the new `direct-sign-and-upload` task
   - Passes through: references, manifest_digests, requester, signing_key_names, taskGitUrl,
     taskGitRevision

2. `tasks/internal/direct-sign-and-upload/direct-sign-and-upload.yaml`
   - Authenticates via Kerberos keytab (secret: `konflux-release-signing-stage-sa`)
   - Calls `rh-signing-client-stage` to sign container references
   - Uploads signatures to Pyxis (reuses `pubtools-pyxis-upload-signatures` pattern)
   - Image: `images.paas.redhat.com/chuazhan/sign-client-image` (same as RPM POC)

### Modified files

3. `tasks/managed/rh-sign-image/rh-sign-image.yaml`
   - Remove `find_signatures` parallel checks (skip existing-signature detection)
   - Remove memory throttling for signature lookups
   - Keep component iteration, skopeo inspect, batching logic
   - Change default request from `simple-signing-pipeline` to `direct-signing-pipeline`

### Untouched files

- `pipelines/internal/simple-signing-pipeline/simple-signing-pipeline.yaml` - left as-is
- `tasks/internal/request-and-upload-signature/request-and-upload-signature.yaml` - left as-is
- `tasks/internal/collect-simple-signing-params/collect-simple-signing-params.yaml` - left as-is

## Key Decisions

- **Staging infrastructure**: Uses `rh-signing-client-stage` + staging keytab
  (`konflux-release-signing-stage-sa`), same as the RPM signing POC
- **Signing client image**: `images.paas.redhat.com/chuazhan/sign-client-image` (shared with RPM POC)
- **Pyxis upload**: Retained - signatures are uploaded to Pyxis after direct signing
- **Existing signatures**: Not checked in POC (sign everything, optimize later)
- **Pipeline wiring**: `rh-push-to-registry-redhat-io` pipeline itself is NOT modified; it already
  points to `rh-sign-image` which will internally use the new pipeline name