# PR and release flow

This document summarizes how pull requests and releases are automated for `terraform-provider-rhcs`. For contributor conventions (commits, hooks), see `[CONTRIBUTE.md](../CONTRIBUTE.md)` at the repository root.

**OpenShift CI / Prow** (presubmit names, profiles, environment, credentials): see **[OpenShift CI / Prow](openshift-ci-prow.md)**.

## PR flow

### Pull request timeline

Follow this path from local work through merge. Workflow files: `[.github/workflows/check-commit-format.yml](../.github/workflows/check-commit-format.yml)`, `[.github/workflows/check-pull-request.yaml](../.github/workflows/check-pull-request.yaml)`.

1. **Before / while you work (your machine)** — You change code, optionally run `make commits/check` so commits match the required format, and `make pre-push-checks` or `make basic-checks` so you catch fmt/build/lint/unit/subsystem issues early.
2. **Push branch & open PR** — GitHub receives the PR; **two CI systems** can start independently from here—nothing waits on the other first.
3. **GitHub Actions (runs automatically on PR to `main`)** — **Validate commit messages** → `make commits/check`. **Pre-push checks** → `make pre-push-checks` (fmt-check → build → codegen check → lint → coverage on touched files → `make test`). **Test matrix** → `make test` again on Ubuntu / macOS / Windows (**unit + subsystem**).
4. **Prow / OpenShift CI (runs in parallel; webhook-driven)** — After the PR is trusted for CI (`ok-to-test` label or org member), OpenShift CI runs presubmits (see **[OpenShift CI / Prow](openshift-ci-prow.md)**). Definitions live in [openshift/release](https://github.com/openshift/release).
5. **Optional / org bots (can appear anytime on the PR)** — **CodeRabbit**, **DCO**, **Renovate** (if applicable), and similar—separate from `make` targets and Prow test pods.
6. **Review** — Humans review; contributors or bots use `/lgtm`, `/approve`, or equivalent labels via the GitHub UI.
7. **Tide (merge gate)** — **Tide** waits until **required** checks are green and required labels (for example `lgtm`, `approved`) are present, then merges (this repo uses **rebase**). Configuration: `_prowconfig.yaml` for `terraform-redhat/terraform-provider-rhcs` in openshift/release.
8. **After merge** — **Konflux** / `.tekton` does **not** run on the PR itself; it runs on **tag push `v*`** for release image builds (see **Release flow**). **Prow periodics** can keep exercising `main` on a schedule.

## Release flow

**Konflux configuration** lives in the repository under `.tekton/`. The active pipeline is declared in `[terraform-provider-rhcs-push.yaml](../.tekton/terraform-provider-rhcs-push.yaml)` as a **PipelineRun** for Pipelines as Code (Konflux / App Studio).

**Trigger:** the pipeline runs on `push` when the target matches a **version tag** (`refs/tags/v…`), not on ordinary branch pushes or on pull requests.

**Files involved:**

- `**[.tekton/terraform-provider-rhcs-push.yaml](../.tekton/terraform-provider-rhcs-push.yaml)`** — PipelineRun spec: clone, dependency prefetch (gomod/Cachi2), container image build (`Dockerfile`), image index, security and compliance tasks (for example SBOM, vulnerability and SAST scans), using Tekton task bundles from the Konflux catalog.

**Result:** a **container image** is built and published (for example to `quay.io/redhat-user-workloads/.../terraform-provider-rhcs` at the tagged revision), with supply-chain oriented checks executed as part of the pipeline. Tagging `v`* also triggers `**[.github/workflows/update-changelog.yml](../.github/workflows/update-changelog.yml)`**, which can open a PR to refresh `CHANGELOG.md` for that release.