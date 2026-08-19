<!-- markdownlint-disable -->

# Hardening Report: elgohr--Publish-Docker-Github-Action/v4

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **elgohr--Publish-Docker-Github-Action/v4** was hardened automatically. 3 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Multiple `uses:` references in workflow files use mutable tags or branch names instead of pinned 40-character SHA commit hashes, making the action vulnerable to supply-chain attacks if the referenced action is compromised or the tag is moved.

In .github/workflows/assign.yml:
- `uses: pozil/auto-assign-issue@v1` (tag)

In .github/workflows/release.yml:
- `uses: actions/checkout@v3` (tag, appears 4 times)
- `uses: elgohr/Publish-Docker-Github-Action@main` (branch, appears 3 times)
- `uses: docker/setup-buildx-action@v2` (tag)

Locations:

- `.github/workflows/assign.yml:9`
- `.github/workflows/release.yml:14`
- `.github/workflows/release.yml:24`
- `.github/workflows/release.yml:26`
- `.github/workflows/release.yml:38`
- `.github/workflows/release.yml:40`
- `.github/workflows/release.yml:52`
- `.github/workflows/release.yml:54`
- `.github/workflows/release.yml:56`
- `.github/workflows/release.yml:74`

### missing-permissions (severity: medium)

Two workflow files are missing `permissions:` blocks on jobs that have no job-level permissions and no top-level permissions key, meaning those jobs run with the default (potentially broad) token permissions.

- `.github/workflows/assign.yml`: No top-level `permissions:` key and the `auto-assign` job has no job-level `permissions:` block.
- `.github/workflows/release.yml`: No top-level `permissions:` key and the `release` job has no job-level `permissions:` block (other jobs do have job-level permissions).

Locations:

- `.github/workflows/assign.yml:1`
- `.github/workflows/release.yml:73`

### github-env-injection (severity: high)

In entrypoint.sh, three values derived from user-controlled or workflow-controlled inputs are written to $GITHUB_OUTPUT without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`). An attacker can inject newlines into these values to poison subsequent steps that read from GITHUB_OUTPUT.

1. `echo "tag=${FIRST_TAG}" >> "$GITHUB_OUTPUT"` — FIRST_TAG is derived from INPUT_TAGS (which maps to `inputs.tags`, a user-supplied input) or from GITHUB_REF. No sanitization is applied before the write.

2. `echo "digest=${DIGEST}" >> "$GITHUB_OUTPUT"` — DIGEST is derived from docker inspect output, which can include image names constructed from user-controlled inputs. No sanitization is applied.

3. `echo "snapshot-tag=${SNAPSHOT_TAG}" >> "$GITHUB_OUTPUT"` — SNAPSHOT_TAG is derived from GITHUB_SHA (workflow-controlled). No sanitization is applied before the write.

Locations:

- `entrypoint.sh:62`
- `entrypoint.sh:66`
- `entrypoint.sh:130`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, missing-permissions, github-env-injection

**Notes:**

Fixed all three findings: (1) Pinned all 8 unpinned action references in assign.yml and release.yml to full 40-char SHA hashes with tag comments preserved. (2) Added minimal permissions blocks to assign.yml (issues: write) and release.yml release job (contents: write). (3) Sanitized all three GITHUB_OUTPUT writes in entrypoint.sh using printf + tr -d '\n\r' to prevent newline injection attacks on FIRST_TAG, DIGEST, and SNAPSHOT_TAG values.

