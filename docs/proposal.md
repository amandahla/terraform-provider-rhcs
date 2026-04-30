# Proposals (CI and merge rules)

These are ideas for the team. They only become real after people agree and after someone changes other repos ([openshift/release](https://github.com/openshift/release) and Tide config). This file alone does not change CI.


| #   | Topic                                                                                |
| --- | ------------------------------------------------------------------------------------ |
| 1   | [Renovate dependency PRs](#proposal-1-renovate-dependency-pull-requests)             |
| 2   | [Prow presubmits and optional tests](#proposal-2-prow-presubmits-and-optional-tests) |


---

## Proposal 1: Renovate dependency pull requests

Status: proposal (needs team agreement and CI config updates).

### Background

[Renovate](../renovate.json) opens pull requests that update libraries (`go.mod` groups: Terraform plugins, AWS SDK, OCM, Kubernetes, etc.). Those PRs often get the `ok-to-test` label. Then OpenShift CI can run all presubmit jobs, including long AWS end-to-end jobs (see [OpenShift CI / Prow](openshift-ci-prow.md)).

Those AWS jobs help quality, but they are slow, cost money, and sometimes fail for reasons unrelated to the dependency change. That makes simple dependency updates hard to merge.

### Idea

For Renovate PRs that change dependencies only (no provider code in the same PR):

1. Say it is OK to merge when `make test` passes (unit + subsystem tests), plus the usual checks from `[make pre-push-checks](../hack/run-checks.sh)` and GitHub Actions (see [PR and release flow](pr-and-release-flow.md)).
2. Use this mainly for small updates (minor and patch versions). Large or risky updates (for example HTTP, TLS, or OCM-related bumps) should get extra review and may need full AWS presubmit runs before merge.
3. Use the existing scheduled jobs on `main` in openshift/release (`terraform-redhat-terraform-provider-rhcs-main-periodics.yaml`) as the long-term AWS check after merge. Do not require full AWS presubmit on every Renovate PR.

Those scheduled jobs send results to Slack channel `#tf-provider-qe-ci`.

### Why this may be OK

- Unit tests check provider logic with mocks.
- Subsystem tests run real Terraform against a fake OCM API (no real AWS bill).
- Scheduled jobs on `main` still run real AWS tests over time, if the team reacts when Slack reports a failure.

### Risks

- Real OCM or AWS behavior may differ from mocks. Some bugs might show up only in scheduled jobs or in production.
- Scheduled jobs do not run at merge time; there can be a delay before the next run.
- If nobody watches `#tf-provider-qe-ci`, a bad merge may stay on `main` until someone notices elsewhere.
- AWS-style tests fail often today (environment noise, product issues, or unstable tests). Green does not always mean safe; red does not always mean the Renovate change broke something. Root causes still need work.

What helps: limit this policy to small bumps; watch Slack; revert or fix when a failure matches a recent merge; run `/test …` on a dependency PR when reviewers want AWS signal; improve test stability over time.

### How to implement

- Tide only merges when required checks pass. Changing what Renovate PRs need means editing openshift/release (including `_prowconfig.yaml`). This document does not do that by itself.
- You can add a GitHub label for Renovate PRs and write clear reviewer rules (for example: dependency-only + green `make test` + normal checks ⇒ OK for small bumps).
- If Proposal 2 is adopted (fewer required Prow checks, GitHub Actions carries unit + subsystem), the same idea still fits: dependency PRs rely on GitHub Actions tests; heavy Prow jobs stay optional unless someone runs `/test`.

### Links

- [Renovate config](../renovate.json)
- [PR and release flow](pr-and-release-flow.md)
- [OpenShift CI / Prow](openshift-ci-prow.md)

---

## Proposal 2: Prow presubmits and optional tests

Status: proposal (needs team agreement and edits in openshift/release + Tide).

### Background

Running every AWS e2e presubmit on every pull request costs a lot of time and money. Prow already supports optional jobs, `/test` comments, path filters (`run_if_changed`, `skip_if_only_changed`), and a short list of required checks in Tide. This proposal uses a simple split: few required jobs, heavy jobs on demand, scheduled jobs for wide coverage.

### Idea

1. Stop blocking merge on `ci/prow/unit`. GitHub Actions already runs `make test` (unit + subsystem) and `make pre-push-checks` ([workflow](../.github/workflows/check-pull-request.yaml)). `ci/prow/unit` runs only `make unit-test` in OpenShift CI ([details](openshift-ci-prow.md)), so it repeats part of GitHub. Remove it from required Tide checks in `_prowconfig.yaml`, mark that presubmit optional in openshift/release, and use `/test` when reviewers want that Linux/ci-operator run anyway.
2. Keep a short required list for merge (exact job names under `ci/prow/…`). Good candidates: image build jobs if releases depend on them (`ci/prow/e2e-images`, `ci/prow/e2e-presubmits-images`). Large AWS profile jobs should usually not block merge unless product policy says otherwise. Changes under shared code (`internal/`, `provider/common/`, `go.mod`, or code that affects both Classic and HCP) may need more jobs or a reviewer rule to run `/test` before merge.
3. Optional presubmits (for example HCP and STS critical/high profiles, upgrades, extra profiles, and `ci/prow/unit` when wanted) should be listed in docs with their `/test` command from openshift/release ([OpenShift CI / Prow](openshift-ci-prow.md)). External contributors still need `ok-to-test` (or org rules) before CI runs.
4. Scheduled jobs on `main` already exist. Review the list: remove overlap, fix gaps, adjust schedule, assign owners. Treat `#tf-provider-qe-ci` as something the team watches. Reliability work on e2e tests (see Proposal 1) makes those results easier to trust.

Later option: add path rules so Classic-only changes do not always start HCP-only jobs (and the reverse), with clear rules for shared paths that need both.

### Risks

- If required checks are too small and reviewers skip `/test`, you might merge risky changes without enough AWS signal.
- Reviewers must remember to type `/test` when needed; clear docs and later path rules can help.
- Tide and branch protection must match the same required job list, or merge rules will confuse people.

### How to implement

- Edit openshift/release job YAML and `core-services/prow/02_config/.../terraform-redhat/terraform-provider-rhcs/_prowconfig.yaml` together. Marking a job `optional: true` does not help if Tide still lists it as required.

### Links

- [OpenShift CI / Prow](openshift-ci-prow.md)
- [openshift/release](https://github.com/openshift/release)

