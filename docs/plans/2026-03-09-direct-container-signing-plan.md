# Direct Container Signing POC Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Replace UMB/robosignatory container signing with direct `rh-signing-client-stage` calls while keeping the internal-request architecture and Pyxis upload.

**Architecture:** The managed `rh-sign-image` task iterates snapshot components, collects references/digests (without checking existing signatures), and creates internal-requests. A new `direct-signing-pipeline` internal pipeline calls a `direct-sign-and-upload` task that authenticates via Kerberos keytab and signs containers with `rh-signing-client-stage`, then uploads signatures to Pyxis.

**Tech Stack:** Tekton Pipelines/Tasks (YAML), Bash scripts, `rh-signing-client-stage`, `pubtools-pyxis-upload-signatures`, Kerberos keytab auth

---

### Task 1: Create the direct-sign-and-upload internal task

**Files:**
- Create: `tasks/internal/direct-sign-and-upload/direct-sign-and-upload.yaml`

**Reference files to study first:**
- `tasks/internal/request-and-upload-signature/request-and-upload-signature.yaml` (current signing task, reuse Pyxis upload step pattern)
- `tasks/internal/send-rpm-siging-request/send-rpm-siging-request.yaml` (RPM POC, reuse keytab auth pattern — on `staging-rpm-sign` branch)

**Step 1: Create the task file**

Create `tasks/internal/direct-sign-and-upload/direct-sign-and-upload.yaml` with:

```yaml
---
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: direct-sign-and-upload
  annotations:
    tekton.dev/pipelines.minVersion: "0.12.1"
    tekton.dev/tags: release
spec:
  description: |-
    Tekton task to directly sign container images using rh-signing-client
    and upload signatures to Pyxis.
    - This task is meant to be used in an internal pipeline that can be triggered frequently
      and is expected to complete as quickly as possible.
  params:
    - description: |
        List of space separated manifest digests for the signed content, usually in the format sha256:xxx
      name: manifest_digests
      type: string
    - description: Name of the user that requested the signing, for auditing purposes
      name: requester
      type: string
    - description: |
        List of space separated docker references for the signed content,
        e.g. registry.com/ns/image:v4.9 registry.com/ns/image:v4.10
      name: references
      type: string
    - description: Space separated signing key names that the content is signed with
      name: sig_key_names
      default: containerisvsign
      type: string
    - description: A docker image with rh-signing-client needed for the signing
      name: pipeline_image
      default: "images.paas.redhat.com/chuazhan/sign-client-image:latest"
      type: string
    - description: Kubernetes secret name that contains the Pyxis SSL files
      name: pyxis_ssl_cert_secret_name
      type: string
    - description: The key within the Kubernetes secret that contains the Pyxis SSL cert
      name: pyxis_ssl_cert_file_name
      type: string
    - description: The key within the Kubernetes secret that contains the Pyxis SSL key
      name: pyxis_ssl_key_file_name
      type: string
    - default: https://pyxis.engineering.redhat.com
      description: Pyxis instance to upload the signature to
      name: pyxis_url
      type: string
    - name: caTrustConfigMapName
      type: string
      description: The name of the ConfigMap to read CA bundle data from
      default: trusted-ca
    - name: caTrustConfigMapKey
      type: string
      description: The name of the key in the ConfigMap that contains the CA bundle data
      default: ca-bundle.crt
  volumes:
    - name: secret-volume
      secret:
        secretName: "konflux-release-signing-stage-sa"
    - name: pyxis-ssl-cert-secret
      secret:
        secretName: $(params.pyxis_ssl_cert_secret_name)
    - name: trusted-ca
      configMap:
        name: $(params.caTrustConfigMapName)
        items:
          - key: $(params.caTrustConfigMapKey)
            path: ca-bundle.crt
        optional: true
  stepTemplate:
    volumeMounts:
      - name: trusted-ca
        mountPath: /mnt/trusted-ca
        readOnly: true
  steps:
    - name: direct-sign
      image: "$(params.pipeline_image)"
      computeResources:
        limits:
          memory: 512Mi
        requests:
          memory: 512Mi
          cpu: 200m
      workingDir: "$(workspaces.data.path)"
      env:
        - name: REFERENCES
          value: $(params.references)
        - name: MANIFEST_DIGESTS
          value: $(params.manifest_digests)
        - name: SIG_KEY_NAMES
          value: $(params.sig_key_names)
        - name: REQUESTER
          value: $(params.requester)
      volumeMounts:
        - mountPath: /etc/secret
          name: secret-volume
          readOnly: true
      script: |
        #!/usr/bin/env bash
        set -ex

        # Decode keytab
        base64 --decode /etc/secret/keytab > ./signing.keytab

        # List available keys for debugging
        rh-signing-client-stage --keytab ./signing.keytab --list-keys

        # Sign each reference
        read -r -a reference_array <<< "${REFERENCES}"
        read -r -a digest_array <<< "${MANIFEST_DIGESTS}"
        read -r -a key_array <<< "${SIG_KEY_NAMES}"

        for key_name in "${key_array[@]}"; do
          for i in "${!reference_array[@]}"; do
            ref="${reference_array[$i]}"
            digest="${digest_array[$i]}"
            echo "Signing reference=${ref} digest=${digest} key=${key_name}"
            rh-signing-client-stage \
              --keytab ./signing.keytab \
              --key "${key_name}" \
              --reference "${ref}" \
              --digest "${digest}"
          done
        done

        # Clean up keytab
        rm -f ./signing.keytab
    - name: upload-signature
      image: "quay.io/konflux-ci/release-service-utils:13e379cb498293f8f7b8b9c84c57d9e8ab141be2"
      computeResources:
        limits:
          memory: 56Mi
          cpu: 25m
        requests:
          memory: 56Mi
          cpu: 25m
      workingDir: "$(workspaces.data.path)"
      volumeMounts:
        - name: pyxis-ssl-cert-secret
          mountPath: /mnt/pyxis_ssl_cert_secret
          readOnly: true
      securityContext:
        runAsUser: 1001
      env:
        - name: PYXIS_CERT_PATH
          value: "/tmp/pyxisCert"
        - name: PYXIS_KEY_PATH
          value: "/tmp/pyxisKey"
        - name: pyxis_url
          value: $(params.pyxis_url)
      script: |
        #!/usr/bin/env bash

        PyxisCert="$(cat "/mnt/pyxis_ssl_cert_secret/$(params.pyxis_ssl_cert_file_name)")"
        PyxisKey="$(cat "/mnt/pyxis_ssl_cert_secret/$(params.pyxis_ssl_key_file_name)")"

        echo "${PyxisCert:?}" > "${PYXIS_CERT_PATH}"
        echo "${PyxisKey:?}" > "${PYXIS_KEY_PATH}"
        set -x

        SIGNATURE_FILE="$(workspaces.data.path)/signing_response.json"
        if [ -f "${SIGNATURE_FILE}" ]; then
          pubtools-pyxis-upload-signatures \
            --pyxis-server "${pyxis_url}" \
            --pyxis-ssl-crtfile "${PYXIS_CERT_PATH}" \
            --pyxis-ssl-keyfile "${PYXIS_KEY_PATH}" \
            --request-threads 5 \
            --signatures @"${SIGNATURE_FILE}"
        else
          echo "No signature data file found, skipping Pyxis upload"
        fi
  workspaces:
    - name: data
```

Note: The exact `rh-signing-client-stage` CLI flags for container signing may differ from RPM signing. The flags above (`--reference`, `--digest`) are placeholders — verify against the actual tool's `--help` output during testing and adjust accordingly.

**Step 2: Commit**

```bash
git add tasks/internal/direct-sign-and-upload/direct-sign-and-upload.yaml
git commit -s -S -m "feat: add direct-sign-and-upload internal task for container signing POC"
```

---

### Task 2: Create the direct-signing-pipeline internal pipeline

**Files:**
- Create: `pipelines/internal/direct-signing-pipeline/direct-signing-pipeline.yaml`

**Reference files:**
- `pipelines/internal/simple-signing-pipeline/simple-signing-pipeline.yaml` (copy and modify)

**Step 1: Create the pipeline file**

Copy `simple-signing-pipeline.yaml` and modify:
- Remove the `collect-simple-signing-params` task entirely
- Replace `request-and-upload-signature` taskRef with `direct-sign-and-upload`
- Remove all UMB-related param pass-throughs
- Add `pipeline_image` param
- Keep: `manifest_digests`, `references`, `requester`, `signing_key_names`, `taskGitUrl`,
  `taskGitRevision` params
- Add Pyxis params: `pyxis_ssl_cert_secret_name`, `pyxis_ssl_cert_file_name`,
  `pyxis_ssl_key_file_name`, `pyxis_url`

```yaml
---
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: direct-signing-pipeline
  annotations:
    tekton.dev/pipelines.minVersion: "0.12.1"
    tekton.dev/tags: release
spec:
  description: |-
    Tekton pipeline for direct container signing using rh-signing-client.
    It is meant to be used by the rh-sign-image task, not as a standalone
    managed pipeline.
  params:
    - name: manifest_digests
      description: Space separated manifest digests for the signed content
      type: string
    - name: references
      description: >-
        Space separated docker references for the signed content,
        e.g. registry.redhat.io/redhat/community-operator-index:v4.9
      type: string
    - name: requester
      description: Name of the user that requested the signing, for auditing purposes
      type: string
    - name: signing_key_names
      description: Space separated list of signing key names to use
      type: string
    - name: pipeline_image
      description: An image with rh-signing-client needed for the signing
      default: "images.paas.redhat.com/chuazhan/sign-client-image:latest"
      type: string
    - name: config_map_name
      description: A config map name with configuration
      default: hacbs-signing-pipeline-config
      type: string
    - name: taskGitUrl
      description: The url to the git repo where the release-service-catalog tasks to be used are stored
      default: https://github.com/konflux-ci/release-service-catalog.git
      type: string
    - name: taskGitRevision
      description: The revision in the taskGitUrl repo to be used
      type: string
  workspaces:
    - name: pipeline
  tasks:
    - name: collect-signing-params
      retries: 5
      taskRef:
        resolver: "git"
        params:
          - name: url
            value: $(params.taskGitUrl)
          - name: revision
            value: $(params.taskGitRevision)
          - name: pathInRepo
            value: tasks/internal/collect-simple-signing-params/collect-simple-signing-params.yaml
      params:
        - name: config_map_name
          value: $(params.config_map_name)
    - name: direct-sign-and-upload
      retries: 2
      taskRef:
        resolver: "git"
        params:
          - name: url
            value: $(params.taskGitUrl)
          - name: revision
            value: $(params.taskGitRevision)
          - name: pathInRepo
            value: tasks/internal/direct-sign-and-upload/direct-sign-and-upload.yaml
      params:
        - name: manifest_digests
          value: $(params.manifest_digests)
        - name: references
          value: $(params.references)
        - name: requester
          value: $(params.requester)
        - name: sig_key_names
          value: $(params.signing_key_names)
        - name: pipeline_image
          value: $(params.pipeline_image)
        - name: pyxis_ssl_cert_secret_name
          value: $(tasks.collect-signing-params.results.pyxis_ssl_cert_secret_name)
        - name: pyxis_ssl_cert_file_name
          value: $(tasks.collect-signing-params.results.pyxis_ssl_cert_file_name)
        - name: pyxis_ssl_key_file_name
          value: $(tasks.collect-signing-params.results.pyxis_ssl_key_file_name)
        - name: pyxis_url
          value: $(tasks.collect-signing-params.results.pyxis_url)
      workspaces:
        - name: data
          workspace: pipeline
          subPath: signing
      runAfter:
        - collect-signing-params
```

Note: We keep `collect-simple-signing-params` to get the Pyxis connection details (secret names,
URL) from the configmap. Only the UMB-related results are ignored.

**Step 2: Commit**

```bash
git add pipelines/internal/direct-signing-pipeline/direct-signing-pipeline.yaml
git commit -s -S -m "feat: add direct-signing-pipeline internal pipeline for container signing POC"
```

---

### Task 3: Simplify the rh-sign-image managed task

**Files:**
- Modify: `tasks/managed/rh-sign-image/rh-sign-image.yaml`

**What to change in the `sign-image` step script (lines 170-651):**

1. **Remove** the `find_signatures` first pass (lines ~264-413): delete the arrays
   `find_signatures_files`, `component_data`, `FIND_SIGNATURES_SUCCESS`,
   `FIND_SIGNATURES_JOB_COUNT`, the entire first-pass loop that starts find_signatures jobs, the
   wait-for-jobs block, and the output/error printing block.

2. **Remove** the Pyxis GraphQL URL setup (lines ~233-249) and the pyxis cert/key writing
   (lines ~251-257) — these were only needed for `find_signatures`.

3. **Remove** the second-pass signature existence checks (lines ~423-487): the
   `for component_info in "${component_data[@]}"` loop that checks `grep -q` against
   `/tmp/${manifest_digest}`. Replace with direct collection of references/digests/keys.

4. **Remove** the memory throttling setup (lines ~179-184, ~186-191): `source memory-throttle.sh`,
   `log_memory_throttle_status`, `RUNNING_JOBS`, `BURST_SIZE`, `STABILIZATION_DELAY`, and all
   `wait_for_memory` / burst sleep calls.

5. **Change** the default request name from `simple-signing-pipeline` to `direct-signing-pipeline`
   (line ~227).

6. **Remove** params that are no longer needed: `pyxisServer`, `pyxisSecret`, `batchLimit`,
   `concurrentLimit`, `signRegistryAccessPath` — and their corresponding volume mounts
   (`pyxis-secret-vol`).

7. **Simplify** the signing loop: iterate components, collect all references/digests, create
   internal-requests for the direct-signing-pipeline with the collected data. Keep the batching
   logic for the internal-request creation.

**Simplified script structure:**

```bash
#!/usr/bin/env bash
set -ex

SNAPSHOT_PATH=$(params.dataDir)/$(params.snapshotPath)
TASK_LABEL="internal-services.appstudio.openshift.io/group-id"
TASK_ID=$(context.taskRun.uid)
PIPELINERUN_LABEL="internal-services.appstudio.openshift.io/pipelinerun-uid"

DATA_FILE="$(params.dataDir)/$(params.dataPath)"
if [ ! -f "${DATA_FILE}" ] ; then
    echo "No valid data file was provided."
    exit 1
fi
RPA_FILE="$(params.dataDir)/$(params.releasePlanAdmissionPath)"
if [ ! -f "${RPA_FILE}" ] ; then
    echo "No valid rpa file was provided."
    exit 1
fi

REQUESTTYPE=$(jq -r '.requestType // "internal-request"' "${DATA_FILE}")
service_account_name=$(jq -r '.spec.pipeline.serviceAccountName // "release-service-account"' "${RPA_FILE}")
if [ "${REQUESTTYPE}" == "internal-pipelinerun" ] ; then
  requestType=internal-pipelinerun
  EXTRA_ARGS=(
  --service-account "${service_account_name}"
  )
else
  requestType=internal-request
  EXTRA_ARGS=()
fi
request=$(jq -r '.sign.request // "direct-signing-pipeline"' "${DATA_FILE}")
config_map_name=$(jq -r '.sign.configMapName // "signing-config-map"' "${DATA_FILE}")

defaultPushSourceContainer=$(jq -r \
  '.mapping.defaults.pushSourceContainer | if . == null then true else . end' "${DATA_FILE}")

# Gather signing keys from configmap
jqquery='.data|if has ("SIG_KEY_NAMES")
then (.SIG_KEY_NAMES|split(",")|.[]|gsub("^\\s+|\\s+$";"")) else .SIG_KEY_NAME end'
configMapJson=$(kubectl get "cm/${config_map_name:?}" -ojson)
SIG_KEY_NAMES=$(jq -er "$jqquery" <<< "${configMapJson}")

COMPONENTS_LENGTH=$(jq '.components |length' "${SNAPSHOT_PATH}")
declare -a to_sign_references=()
declare -a to_sign_digests=()

for (( i = 0; i < COMPONENTS_LENGTH; i++ )); do
    component=$(jq -c --argjson i "$i" '.components[$i]' "${SNAPSHOT_PATH}")
    referenceContainerImage=$(jq -r '.containerImage' <<< "$component")

    # check if multi-arch
    RAW_OUTPUT=$(skopeo inspect --retry-times 3 --no-tags --raw "docker://${referenceContainerImage}")
    manifest_digests="${referenceContainerImage#*@}"
    MEDIATYPE="$(jq -r '.mediaType' <<< "$RAW_OUTPUT")"
    if [ "$MEDIATYPE" != "application/vnd.oci.image.manifest.v1+json" ] && \
       [ "$MEDIATYPE" != "application/vnd.docker.distribution.manifest.v2+json" ]; then
      nested_digests=$(jq -r '[.manifests[].digest] | join(" ")' <<< "$RAW_OUTPUT")
      manifest_digests="$manifest_digests $nested_digests"
    fi
    echo "MANIFEST DIGESTS: ${manifest_digests}"

    sourceContainerDigest=
    if [[ $(jq -r '.pushSourceContainer' <<< "$component") == "true" ]] || \
       [[ $(jq 'has("pushSourceContainer")' <<< "$component") == "false" \
        && ${defaultPushSourceContainer} == "true" ]] ; then
      source_repo=${referenceContainerImage%%@sha256:*}
      source_reference_tag=sha256-${referenceContainerImage#*@sha256:}.src
      sourceContainer="${source_repo}:${source_reference_tag}"

      AUTH_FILE=$(mktemp)
      select-oci-auth "${sourceContainer}" > "$AUTH_FILE"
      sourceContainerDigest=$(oras resolve --registry-config "$AUTH_FILE" "${sourceContainer}")
    fi

    NUM_REPOS=$(jq -c '.repositories | length' <<< "$component")
    for ((j = 0; j < NUM_REPOS; j++)); do
      repo=$(jq -c --argjson j "$j" '.repositories[$j]' <<< "$component")
      TAGS=$(jq -r '.tags | join(" ")' <<< "$repo")
      rh_registry_repo=$(jq -er '."rh-registry-repo"' <<< "$repo")

      for manifest_digest in $manifest_digests; do
        for tag in ${TAGS}; do
          to_sign_references+=("${rh_registry_repo}:${tag}")
          to_sign_digests+=("${manifest_digest}")
        done
      done

      # Source container signing
      if [ "${sourceContainerDigest}" != "" ] ; then
        for tag in ${TAGS}; do
          to_sign_references+=("${rh_registry_repo}:${tag}-source")
          to_sign_digests+=("${sourceContainerDigest}")
        done
      fi
    done
done

# Create internal-requests with batching
BATCH_LIMIT=16384
REQUEST_COUNT=0
references_batch=""
digests_batch=""

for i in "${!to_sign_references[@]}"; do
  reference="${to_sign_references[$i]}"
  digest="${to_sign_digests[$i]}"

  new_references_batch="${references_batch}${reference} "
  new_digests_batch="${digests_batch}${digest} "

  if [[ ${#new_references_batch} -gt $BATCH_LIMIT || ${#new_digests_batch} -gt $BATCH_LIMIT ]]; then
    echo "Creating ${requestType} to sign images: ${references_batch}"
    ${requestType} \
      --pipeline "${request}" \
      -p references="${references_batch}" \
      -p signing_key_names="${SIG_KEY_NAMES}" \
      -p manifest_digests="${digests_batch}" \
      -p config_map_name="${config_map_name}" \
      -p requester="$(params.requester)" \
      -p taskGitUrl="$(params.taskGitUrl)" \
      -p taskGitRevision="$(params.taskGitRevision)" \
      -l ${TASK_LABEL}="${TASK_ID}" \
      -l ${PIPELINERUN_LABEL}="$(params.pipelineRunUid)" \
      -t "$(params.requestTimeout)" --pipeline-timeout "0h30m0s" --task-timeout "0h25m0s" \
      "${EXTRA_ARGS[@]}" -s true &
    ((++REQUEST_COUNT))

    references_batch="${reference} "
    digests_batch="${digest} "
  else
    references_batch="${new_references_batch}"
    digests_batch="${new_digests_batch}"
  fi
done

# Send remaining batch
if [[ ${#references_batch} -gt 0 ]]; then
  echo "Creating ${requestType} to sign images: ${references_batch}"
  ${requestType} \
    --pipeline "${request}" \
    -p references="${references_batch}" \
    -p signing_key_names="${SIG_KEY_NAMES}" \
    -p manifest_digests="${digests_batch}" \
    -p config_map_name="${config_map_name}" \
    -p requester="$(params.requester)" \
    -p taskGitUrl="$(params.taskGitUrl)" \
    -p taskGitRevision="$(params.taskGitRevision)" \
    -l ${TASK_LABEL}="${TASK_ID}" \
    -l ${PIPELINERUN_LABEL}="$(params.pipelineRunUid)" \
    -t "$(params.requestTimeout)" --pipeline-timeout "0h30m0s" --task-timeout "0h25m0s" \
    "${EXTRA_ARGS[@]}" -s true &
  ((++REQUEST_COUNT))
fi

echo "Waiting for ${REQUEST_COUNT} signing requests to complete..."
wait
echo "done"
```

**Params to remove from the task spec:**
- `pyxisServer`
- `pyxisSecret`
- `batchLimit`
- `concurrentLimit`
- `signRegistryAccessPath`

**Volumes to remove:**
- `pyxis-secret-vol`

**Params to keep:**
- `snapshotPath`, `dataPath`, `releasePlanAdmissionPath`, `requester`, `requestTimeout`,
  `pipelineRunUid`, all trusted artifacts params, `taskGitUrl`, `taskGitRevision`,
  `caTrustConfigMapName`, `caTrustConfigMapKey`

**Also update the pipeline** `rh-push-to-registry-redhat-io.yaml` to stop passing removed params:
- Remove `pyxisServer` param pass-through (line ~428)
- Remove `pyxisSecret` param pass-through (line ~430)
- Remove `signRegistryAccessPath` param pass-through (line ~432)

**Step 1: Modify rh-sign-image task**

Apply all changes described above.

**Step 2: Update rh-push-to-registry-redhat-io pipeline**

Remove the params that no longer exist from the `rh-sign-image` task invocation.

**Step 3: Verify YAML validity**

Run: `yamllint tasks/managed/rh-sign-image/rh-sign-image.yaml pipelines/managed/rh-push-to-registry-redhat-io/rh-push-to-registry-redhat-io.yaml`

Expected: No errors (or only pre-existing warnings)

**Step 4: Commit**

```bash
git add tasks/managed/rh-sign-image/rh-sign-image.yaml \
       pipelines/managed/rh-push-to-registry-redhat-io/rh-push-to-registry-redhat-io.yaml
git commit -s -S -m "feat: simplify rh-sign-image for direct signing POC

Remove find_signatures checks, memory throttling, and Pyxis secret
handling from managed task. Point to direct-signing-pipeline instead
of simple-signing-pipeline."
```

---

### Task 4: Validate all files together

**Step 1: Run yamllint on all changed files**

```bash
yamllint tasks/internal/direct-sign-and-upload/direct-sign-and-upload.yaml \
         pipelines/internal/direct-signing-pipeline/direct-signing-pipeline.yaml \
         tasks/managed/rh-sign-image/rh-sign-image.yaml \
         pipelines/managed/rh-push-to-registry-redhat-io/rh-push-to-registry-redhat-io.yaml
```

Expected: No errors

**Step 2: Verify param consistency**

Manually check that:
- `direct-signing-pipeline` passes all required params to `direct-sign-and-upload`
- `rh-sign-image` passes all required params to the internal-request for `direct-signing-pipeline`
- `rh-push-to-registry-redhat-io` passes all required params to `rh-sign-image`

**Step 3: Fix any issues found and commit**

```bash
git add -u
git commit -s -S -m "fix: address validation issues in direct container signing POC"
```