<!-- markdownlint-disable -->

# Hardening Report: elgohr--Publish-Docker-Github-Action/v4

> This file was generated automatically by the hardening agent.

**Policy SHA:** `374911c4be584df24e6da57af2d990570ef0f859`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **elgohr--Publish-Docker-Github-Action/v4** was hardened automatically. 3 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

entrypoint.sh, which the composite step in action.yml runs, writes `echo "tag=${FIRST_TAG}" >> "$GITHUB_OUTPUT"` without the `printf '%s' ... | tr -d '\n\r'` step first. FIRST_TAG comes from INPUT_TAGS (inputs.tags) via `echo "${INPUT_TAGS}" | sed ... | cut -d ' ' -f1`, or from INPUT_NAME (inputs.name) via cut -d':' -f2. Neither path removes newlines. A value with an embedded newline can therefore inject extra key=value entries into the step outputs.

Locations:

- `entrypoint.sh:70`
- `action.yml:69`

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
index fbdad83..1607792 100755
--- a/entrypoint.sh
+++ b/entrypoint.sh
@@ -67,14 +67,16 @@ main() {
   fi
 
   echo "::set-output name=tag::${FIRST_TAG}"
-  echo "tag=${FIRST_TAG}" >> "$GITHUB_OUTPUT"
+  safe_first_tag=$(printf '%s' "${FIRST_TAG}" | tr -d '\n\r')
+  echo "tag=${safe_first_tag}" >> "$GITHUB_OUTPUT"
   if uses "${INPUT_PLATFORMS}"; then
     DIGEST=$(jq -r '."containerimage.digest"' metadata.json)
   else 
     DIGEST=$(docker inspect --format='{{index .RepoDigests 0}}' ${DOCKERNAME})
   fi
   echo "::set-output name=digest::${DIGEST}"
-  echo "digest=${DIGEST}" >> "$GITHUB_OUTPUT"
+  safe_digest=$(printf '%s' "${DIGEST}" | tr -d '\n\r')
+  echo "digest=${safe_digest}" >> "$GITHUB_OUTPUT"
 
   docker logout
 }
@@ -185,7 +187,8 @@ useSnapshot() {
   local SNAPSHOT_TAG="${TIMESTAMP}${SHORT_SHA}"
   TAGS="${TAGS} ${SNAPSHOT_TAG}"
   echo "::set-output name=snapshot-tag::${SNAPSHOT_TAG}"
-  echo "snapshot-tag=${SNAPSHOT_TAG}" >> "$GITHUB_OUTPUT"
+  safe_snapshot_tag=$(printf '%s' "${SNAPSHOT_TAG}" | tr -d '\n\r')
+  echo "snapshot-tag=${safe_snapshot_tag}" >> "$GITHUB_OUTPUT"
 }
 
 build() {

```

Locations:

- `entrypoint.sh`

## Iteration Notes

### Iteration 1

**Fixes applied:** github-env-injection, prior-fix

**Notes:**

entrypoint.sh: the tag, digest and snapshot-tag values now go through printf '%s' | tr -d '\n\r' before they are written to $GITHUB_OUTPUT. This re-applies the previous patch. Each value is captured first and cleaned afterwards, so if the command producing it fails, set -e still stops the script. action.yml: re-applied the previous changes, which were only cosmetic (removed trailing whitespace after 'steps:' and added a newline at the end of the file). action.yml had no security issue to fix. Nothing was run to test these changes. The deprecated ::set-output lines were left exactly as upstream wrote them.

