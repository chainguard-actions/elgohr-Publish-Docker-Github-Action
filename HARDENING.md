<!-- markdownlint-disable -->

# Hardening Report: elgohr--Publish-Docker-Github-Action/v5

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **elgohr--Publish-Docker-Github-Action/v5** was hardened automatically. 1 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

entrypoint.sh writes values derived from workflow-controlled (untrusted) inherited environment variables to $GITHUB_OUTPUT without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`).

1. Line ~57: `echo "tag=${FIRST_TAG}" >> "$GITHUB_OUTPUT"` — FIRST_TAG is derived from TAGS, which is set from INPUT_TAGS (mapped from inputs.tags by the calling workflow) or from GITHUB_REF/GITHUB_SHA (inherited env vars). An attacker can inject newlines via a crafted tag or branch name to poison GITHUB_OUTPUT.

2. Line ~62: `echo "digest=${DIGEST}" >> "$GITHUB_OUTPUT"` — DIGEST is obtained from `docker inspect` of DOCKERNAME, which includes INPUT_NAME (mapped from inputs.name). No sanitization before the write.

3. Line ~130: `echo "snapshot-tag=${SNAPSHOT_TAG}" >> "$GITHUB_OUTPUT"` — SNAPSHOT_TAG is composed of a timestamp and the first 6 characters of GITHUB_SHA (an inherited env var). While GITHUB_SHA is typically a hex string, it is still an unsanitized inherited env var written directly to GITHUB_OUTPUT.

All three writes should be preceded by: `safe=$(printf '%s' "$VAR" | tr -d '\n\r')` before the echo.

Locations:

- `entrypoint.sh:57`
- `entrypoint.sh:62`
- `entrypoint.sh:130`

## Iteration Notes

### Iteration 1

**Fixes applied:** github-env-injection

**Notes:**

Fixed three GITHUB_OUTPUT injection vulnerabilities in entrypoint.sh:
1. Line ~57: Added `safe_tag=$(printf '%s' "${FIRST_TAG}" | tr -d '\n\r')` before writing tag to GITHUB_OUTPUT.
2. Line ~62: Added `safe_digest=$(printf '%s' "${DIGEST}" | tr -d '\n\r')` before writing digest to GITHUB_OUTPUT.
3. Line ~130 (useSnapshot function): Added `safe_snapshot_tag=$(printf '%s' "${SNAPSHOT_TAG}" | tr -d '\n\r')` before writing snapshot-tag to GITHUB_OUTPUT.
All three writes now use the sanitized variables instead of the raw values, preventing newline injection via crafted tag names, branch names, or other workflow-controlled inputs.

### Iteration 1

**Fixes applied:** script-injection

**Notes:**

Fixed all four script-injection vulnerabilities in entrypoint.sh:
1. Quoted INPUT_USERNAME and INPUT_REGISTRY in docker login command (line 32)
2. Converted BUILDPARAMS from a string to a bash array; useCustomDockerfile() now uses BUILDPARAMS+=("-f" "${INPUT_DOCKERFILE}") with proper quoting (line ~112)
3. INPUT_BUILDOPTIONS is now safely split into a BUILDOPTS bash array using xargs printf '%s\0' (no raw word-splitting), and expanded as "${BUILDOPTS[@]}" in both docker buildx build and docker build calls (lines ~148, ~150)
4. Changed shebang from #!/bin/sh to #!/bin/bash to enable array support
5. Also fixed addBuildArgs() and useBuildCache() to use array syntax for BUILDPARAMS
6. BUILD_TAGS converted to array for consistent safe expansion
7. PUSHING flag handled via conditional branch to avoid passing empty string as a positional argument

