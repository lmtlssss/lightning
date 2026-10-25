# Release authority boundary

This repository builds release candidates in GitHub Actions. Release signing
and publication are separate privileges from the ability to contribute code.

The workflow changes in this branch add **job-level checks and environment
names**. GitHub does **not** create reviewer restrictions merely because a
workflow names an environment. Repository administrators must complete the
following settings before treating the release boundary as enforced.

## Required repository settings

1. In **Settings → Environments**, create three tag-release environments; coordinate the fourth Cargo environment with PR #9663. Require at least
   one independent release captain as reviewer and **prevent self-review**.
   Limit deployments to the specified protected refs:

   | Environment | Allowed refs | Environment-scoped secrets |
   | --- | --- | --- |
   | `release-signing` | Release tags `v*` only | `GPG_PRIVATE_KEY`, `GPG_PASSPHRASE` |
   | `release-docker` | Release tags `v*` only | `DOCKER_USERNAME`, `DOCKER_PASSWORD` |
   | `release-pypi` | Release tags `v*` only | `PYPI_API_TOKEN`, `TEST_PYPI_API_TOKEN` |
   | `release-cargo` (PR #9663) | Protected `master` only | OIDC; no registry API token |

   Use selected branch/tag patterns and verify the exact deployment policy
   with a controlled workflow run. Each environment should contain only its
   listed secrets. **Remove the repository/organization-level copies** of
   these release secrets after migration. Otherwise an actor who can edit a
   workflow on a branch can simply bypass job-level guards and read the
   broadly scoped copies.

2. In **Settings → Rules → Rulesets**, restrict creation of all release
   tags matching `v*` to designated release captains. Retain restrictions
   on tag updates and deletion. A rule that blocks updates alone does not
   prevent an ordinary write actor from creating a new release tag.

3. Review **Actions actor policies**. Permit release workflow execution only
   to the release-maintainer group where available. A manually selected
   branch/tag must not grant signing or publication authority.

4. Ensure the trusted `master` branch retains required independent review,
   CI and no-force-push rules. Treat any modification to release workflows,
   helper scripts or environment names as a security-sensitive change.

## Workflow behavior

- `workflow_dispatch` on a branch can still run the unsigned release binary
  and PyPI build jobs for release engineering. Docker build is excluded
  because it pushes images **by digest** before the final Docker tag step.
- The top-level tagged release runs the existing build matrix and publication
  path. Jobs that import signing/publication secrets require the appropriate
  environment, a release-tag ref and a version matching that tag.
- Cargo publishing is separately patched in PR #9663; this branch leaves `crate-io.yml` unchanged.
- The caller no longer passes `secrets: inherit` to reusable workflows.
  Secrets are resolved only inside the credential-bearing called job.
- `skip_validation` remains available for unsigned branch testing. Tagged
  release runs reject it, including manual dispatch against a tag.

## Acceptance gate before merging

Use only **dummy credentials** in a fork and disable all external publication.
Prove that a manual branch dispatch completes its unsigned build but skips
every credential-bearing job, including the Docker **digest** push. Verify
that a tagged run cannot enter any protected environment until an independent
reviewer approves it, and that an ordinary write actor cannot create `v*`
tags. Check the separate Cargo branch guard. Confirm legitimate tagged
releases still run after approval.

The code patch alone does not configure GitHub's private environment
protections or actor policies. This checklist is an operational part of the
fix; the PR must remain a draft until these settings are accepted and tested.


## Parallel submission

This branch does not change Cargo workflows, keeping it independent of upstream PR #9663. Both proposals require separate maintainer configuration and coordinated review. Never reintroduce a durable Cargo token.
