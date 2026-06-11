<!-- markdownlint-disable -->

# Hardening Report: elgohr--Publish-Docker-Github-Action/v4

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **elgohr--Publish-Docker-Github-Action/v4** was hardened automatically. 2 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

entrypoint.sh writes untrusted values to $GITHUB_OUTPUT without the required sanitization (printf '%s' ... | tr -d '\n\r'). Three writes are affected: (1) `echo "tag=${FIRST_TAG}" >> "$GITHUB_OUTPUT"` — FIRST_TAG is derived from INPUT_TAGS (user-supplied via inputs.tags) or from GITHUB_REF/GITHUB_SHA; (2) `echo "digest=${DIGEST}" >> "$GITHUB_OUTPUT"` — DIGEST comes from docker inspect on an attacker-influenced image name; (3) `echo "snapshot-tag=${SNAPSHOT_TAG}" >> "$GITHUB_OUTPUT"` — SNAPSHOT_TAG is derived from GITHUB_SHA. None of these writes are preceded by newline-stripping sanitization, allowing a malicious value containing newlines to inject arbitrary key=value pairs into the output context.

Locations:

- `entrypoint.sh:56`
- `entrypoint.sh:60`
- `entrypoint.sh:107`

### script-injection (severity: high)

Rule (b): Multiple env vars holding workflow-controllable input values are expanded unquoted inside shell commands in entrypoint.sh, allowing shell metacharacter injection. Affected lines: (1) `echo "${INPUT_PASSWORD}" | docker login -u ${INPUT_USERNAME} --password-stdin ${INPUT_REGISTRY}` — INPUT_USERNAME and INPUT_REGISTRY are unquoted; (2) `docker buildx build --push --metadata-file metadata.json ${PLATFORMS} ${INPUT_BUILDOPTIONS} ${BUILDPARAMS} ${BUILD_TAGS} ${CONTEXT}` — INPUT_BUILDOPTIONS and other vars are unquoted; (3) `docker build ${INPUT_BUILDOPTIONS} ${BUILDPARAMS} ${BUILD_TAGS} ${CONTEXT}` — INPUT_BUILDOPTIONS and other vars are unquoted. All INPUT_* variables are set from inputs.* expressions in action.yml and are therefore workflow-controllable. An attacker-supplied value containing shell metacharacters (`;`, `|`, `&`, `$(...)`, etc.) would be interpreted by the shell.

Locations:

- `entrypoint.sh:31`
- `entrypoint.sh:121`
- `entrypoint.sh:123`

## Iteration Notes

### Iteration 1

**Fixes applied:** github-env-injection, script-injection

**Notes:**

Fixed all findings in entrypoint.sh:

1. github-env-injection (3 locations): Added newline sanitization before all three $GITHUB_OUTPUT writes using `printf '%s' "$VAR" | tr -d '\n\r'`. Applied to FIRST_TAG (→ SAFE_FIRST_TAG), DIGEST (→ SAFE_DIGEST), and SNAPSHOT_TAG (→ SAFE_SNAPSHOT_TAG).

2. script-injection (3 locations):
   - docker login (line 31): Quoted INPUT_USERNAME and INPUT_REGISTRY with double quotes.
   - build() function (lines 121/123): Restructured to use `set --` to safely accumulate arguments. INPUT_BUILDOPTIONS tokens are iterated in a for loop so each token becomes a separately-quoted argument (preventing shell metacharacter interpretation). INPUT_PLATFORMS is now quoted. CONTEXT is now quoted. All arguments passed to docker via "$@".

