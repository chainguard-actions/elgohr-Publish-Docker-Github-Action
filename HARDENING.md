<!-- markdownlint-disable -->

# Hardening Report: elgohr--Publish-Docker-Github-Action/v5

> This file was generated automatically by the hardening agent.

**Policy SHA:** `374911c4be584df24e6da57af2d990570ef0f859`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **elgohr--Publish-Docker-Github-Action/v5** was hardened automatically. 3 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

The composite step 'docker-publish' (action.yml) runs entrypoint.sh, which writes `echo "tag=${FIRST_TAG}" >> "$GITHUB_OUTPUT"`. FIRST_TAG comes from INPUT_TAGS (inputs.tags), INPUT_NAME (inputs.name) or GITHUB_REF, via `FIRST_TAG=$(echo "${TAGS}" | cut -d ' ' -f1)`. `cut` works line by line, so newlines in the value survive. An attacker who controls that value can add extra key=value output lines. The same applies to `echo "digest=${DIGEST}" >> "$GITHUB_OUTPUT"`, where DIGEST is read from docker inspect or metadata.json for an image named by inputs. Neither write strips newlines first with `printf '%s' ... | tr -d '\n\r'`.

Locations:

- `entrypoint.sh:70`
- `entrypoint.sh:76`
- `action.yml:75`

### prior-fix (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. The previous harden changed action.yml without recording a finding for it, and this analysis reported nothing there. Re-apply its security changes unless the code no longer needs them. The previous patch:
```diff
diff --git a/action.yml b/action.yml
index 6d34270..48be96e 100644
--- a/action.yml
+++ b/action.yml
@@ -68,7 +68,24 @@ outputs:
     value: ${{ steps.docker-publish.outputs.digest }}
 runs:
   using: 'composite'
-  steps:      
+  steps:
     - id: docker-publish
       run: $GITHUB_ACTION_PATH/entrypoint.sh
       shell: bash
@@ -89,4 +106,4 @@ runs:
         INPUT_TAG_NAMES: ${{ inputs.tag_names }}
         INPUT_TAG_SEMVER: ${{ inputs.tag_semver }}
         INPUT_NO_PUSH: ${{ inputs.no_push }}
-        INPUT_PLATFORMS: ${{ inputs.platforms }}
\ No newline at end of file
+        INPUT_PLATFORMS: ${{ inputs.platforms }}

```

Locations:

- `action.yml`

### prior-fix (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. The previous harden changed entrypoint.sh without recording a finding for it, and this analysis reported nothing there. Re-apply its security changes unless the code no longer needs them. The previous patch:
```diff
diff --git a/entrypoint.sh b/entrypoint.sh
index 4f49f02..8ea4a64 100755
--- a/entrypoint.sh
+++ b/entrypoint.sh
@@ -1,4 +1,4 @@
-#!/bin/sh
+#!/bin/bash
 set -e
 
 main() {
@@ -29,12 +29,12 @@ main() {
   fi
 
   if uses "${INPUT_USERNAME}" && uses "${INPUT_PASSWORD}"; then
-    echo "${INPUT_PASSWORD}" | docker login -u ${INPUT_USERNAME} --password-stdin ${INPUT_REGISTRY}
+    echo "${INPUT_PASSWORD}" | docker login -u "${INPUT_USERNAME}" --password-stdin "${INPUT_REGISTRY}"
   fi
 
   FIRST_TAG=$(echo "${TAGS}" | cut -d ' ' -f1)
   DOCKERNAME="${INPUT_NAME}:${FIRST_TAG}"
-  BUILDPARAMS=""
+  BUILDPARAMS=()
   CONTEXT="."
 
   if uses "${INPUT_DOCKERFILE}"; then
@@ -67,13 +67,15 @@ main() {
     push
   fi
 
-  echo "tag=${FIRST_TAG}" >> "$GITHUB_OUTPUT"
+  safe_tag=$(printf '%s' "${FIRST_TAG}" | tr -d '\n\r')
+  echo "tag=${safe_tag}" >> "$GITHUB_OUTPUT"
   if uses "${INPUT_PLATFORMS}"; then
     DIGEST=$(jq -r '."containerimage.digest"' metadata.json)
   else 
-    DIGEST=$(docker inspect --format='{{index .RepoDigests 0}}' ${DOCKERNAME})
+    DIGEST=$(docker inspect --format='{{index .RepoDigests 0}}' "${DOCKERNAME}")
   fi
-  echo "digest=${DIGEST}" >> "$GITHUB_OUTPUT"
+  safe_digest=$(printf '%s' "${DIGEST}" | tr -d '\n\r')
+  echo "digest=${safe_digest}" >> "$GITHUB_OUTPUT"
 
   docker logout
 }
@@ -146,19 +148,19 @@ changeWorkingDirectory() {
 }
 
 useCustomDockerfile() {
-  BUILDPARAMS="${BUILDPARAMS} -f ${INPUT_DOCKERFILE}"
+  BUILDPARAMS+=("-f" "${INPUT_DOCKERFILE}")
 }
 
 addBuildArgs() {
   for ARG in $(echo "${INPUT_BUILDARGS}" | tr ',' '\n'); do
-    BUILDPARAMS="${BUILDPARAMS} --build-arg ${ARG}"
+    BUILDPARAMS+=("--build-arg" "${ARG}")
     echo "::add-mask::${ARG}"
   done
 }
 
 useBuildCache() {
   if docker pull "${DOCKERNAME}" 2>/dev/null; then
-    BUILDPARAMS="${BUILDPARAMS} --cache-from ${DOCKERNAME}"
+    BUILDPARAMS+=("--cache-from" "${DOCKERNAME}")
   fi
 }
 
@@ -183,23 +185,35 @@ useSnapshot() {
   local SHORT_SHA=$(echo "${GITHUB_SHA}" | cut -c1-6)
   local SNAPSHOT_TAG="${TIMESTAMP}${SHORT_SHA}"
   TAGS="${TAGS} ${SNAPSHOT_TAG}"
-  echo "snapshot-tag=${SNAPSHOT_TAG}" >> "$GITHUB_OUTPUT"
+  safe_snapshot_tag=$(printf '%s' "${SNAPSHOT_TAG}" | tr -d '\n\r')
+  echo "snapshot-tag=${safe_snapshot_tag}" >> "$GITHUB_OUTPUT"
 }
 
 build() {
-  local BUILD_TAGS=""
+  local BUILD_TAGS=()
   for TAG in ${TAGS}; do
-    BUILD_TAGS="${BUILD_TAGS}-t ${INPUT_NAME}:${TAG} "
+    BUILD_TAGS+=("-t" "${INPUT_NAME}:${TAG}")
   done
+  # Split INPUT_BUILDOPTIONS safely into an array using word splitting on the
+  # trusted-but-multi-word option string. We use eval-free approach via read.
+  local BUILDOPTS=()
+  if uses "${INPUT_BUILDOPTIONS}"; then
+    while IFS= read -r -d '' opt; do
+      BUILDOPTS+=("$opt")
+    done < <(xargs printf '%s\0' <<< "${INPUT_BUILDOPTIONS}")
+  fi
   if uses "${INPUT_PLATFORMS}"; then
-    local PLATFORMS="--platform ${INPUT_PLATFORMS}"
     local PUSHING="--push"
     if usesBoolean "${INPUT_NO_PUSH}"; then
       PUSHING=""
     fi
-    docker buildx build ${PUSHING} --metadata-file metadata.json ${PLATFORMS} ${INPUT_BUILDOPTIONS} ${BUILDPARAMS} ${BUILD_TAGS} ${CONTEXT}
+    if [ -n "${PUSHING}" ]; then
+      docker buildx build "${PUSHING}" --metadata-file metadata.json --platform "${INPUT_PLATFORMS}" "${BUILDOPTS[@]}" "${BUILDPARAMS[@]}" "${BUILD_TAGS[@]}" "${CONTEXT}"
+    else
+      docker buildx build --metadata-file metadata.json --platform "${INPUT_PLATFORMS}" "${BUILDOPTS[@]}" "${BUILDPARAMS[@]}" "${BUILD_TAGS[@]}" "${CONTEXT}"
+    fi
   else
-    docker build ${INPUT_BUILDOPTIONS} ${BUILDPARAMS} ${BUILD_TAGS} ${CONTEXT}
+    docker build "${BUILDOPTS[@]}" "${BUILDPARAMS[@]}" "${BUILD_TAGS[@]}" "${CONTEXT}"
   fi
 }
 

```

Locations:

- `entrypoint.sh`

## Iteration Notes

### Iteration 1

**Fixes applied:** github-env-injection, prior-fix

**Notes:**

entrypoint.sh: newlines are now stripped from the tag, digest and snapshot-tag values with printf | tr -d '\n\r' before they are written to GITHUB_OUTPUT. I re-applied the earlier patch: a bash shebang (this is a composite action on the runner, so bash is there), quoted docker login and inspect arguments, and bash arrays for BUILDPARAMS, BUILD_TAGS and the --push flag. I changed two things from that patch. First, buildoptions is now split on whitespace with `read -r -d '' -a` instead of xargs, because upstream expanded the env var unquoted and xargs would honor quotes in the value. Second, the registry argument for docker login is ${INPUT_REGISTRY:+"$INPUT_REGISTRY"}, so an empty registry adds no argument, as it did upstream. action.yml: re-applied the earlier patch, which only removed trailing whitespace and added a final newline. None of this has been run.

