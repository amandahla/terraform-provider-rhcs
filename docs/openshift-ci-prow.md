# OpenShift CI / Prow (`terraform-provider-rhcs`)

This document is **reference material** for OpenShift CI / Prow presubmits for `terraform-redhat/terraform-provider-rhcs` on branch `main`. For the end-to-end PR and release narrative, see **[PR and release flow](pr-and-release-flow.md)**.

You can see the status of all Terraform RHCS provider CI jobs (including runs for each pull request) on Prow: [prow.ci.openshift.org — `terraform-redhat/terraform-provider-rhcs](https://prow.ci.openshift.org/?repo=terraform-redhat%2Fterraform-provider-rhcs)`.

Job definitions live in [openshift/release](https://github.com/openshift/release):


| What                                                         | Path                                                                                                                                                                                                                                              |
| ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Presubmit specs (context names, `always_run`, `optional`, …) | `ci-operator/jobs/terraform-redhat/terraform-provider-rhcs/terraform-redhat-terraform-provider-rhcs-main-presubmits.yaml`                                                                                                                         |
| Periodic jobs (scheduled on `main`)                          | `ci-operator/jobs/terraform-redhat/terraform-provider-rhcs/terraform-redhat-terraform-provider-rhcs-main-periodics.yaml`                                                                                                                          |
| `ci-operator` test bodies (`--target=`)                      | `ci-operator/config/terraform-redhat/terraform-provider-rhcs/terraform-redhat-terraform-provider-rhcs-main.yaml`, `terraform-redhat-terraform-provider-rhcs-main__e2e-presubmits.yaml`, `terraform-redhat-terraform-provider-rhcs-main__e2e.yaml` |
| E2e step scripts                                             | `ci-operator/step-registry/rhcs/e2e/tests/`, `ci-operator/step-registry/rhcs/e2e/general-tests/`, workflow `rhcs-aws-sts` under `ci-operator/step-registry/rhcs/aws/sts/`                                                                         |
| Tide                                                         | `core-services/prow/02_config/terraform-redhat/terraform-provider-rhcs/_prowconfig.yaml`                                                                                                                                                          |


## Presubmits when the PR has `ok-to-test`

Once a PR has the `ok-to-test` label (maintainer `/ok-to-test` on untrusted contributions), OpenShift CI runs:

- `ci/prow/unit`
- `ci/prow/e2e-images`
- `ci/prow/e2e-presubmits-images`
- `ci/prow/e2e-presubmits-rosa-hcp-advanced-critical-high-presubmit` — Profile: `rosa-hcp-ad`
- `ci/prow/e2e-presubmits-rosa-hcp-private-critical-high-presubmit` — Profile: `rosa-hcp-pl`
- `ci/prow/e2e-presubmits-rosa-sts-advanced-critical-high-presubmit` — Profile: `rosa-sts-ad`
- `ci/prow/e2e-presubmits-rosa-sts-private-critical-high-presubmit` — Profile: `rosa-sts-pl`

**Note:** Trusted org members often skip the `ok-to-test` label, but merging still requires green required checks and Tide labels such as `lgtm` and `approved`.

### How to change jobs

Edit presubmit job YAML and `ci-operator` targets in [openshift/release](https://github.com/openshift/release) under `ci-operator/jobs/terraform-redhat/terraform-provider-rhcs/` and `ci-operator/config/terraform-redhat/terraform-provider-rhcs/`.

## What each check does

- `ci/prow/unit` — Runs `make unit-test` (Ginkgo on `provider` and `internal/...`) in the source build image; no live AWS or OCM cluster.
- `ci/prow/e2e-images` — Builds the `rhcs-tf-e2e` CI image (ci-operator `[images]` target, `e2e` variant) from `[build/ci-tf-e2e.Dockerfile](../build/ci-tf-e2e.Dockerfile)`; validates the image pipeline, not `tests/e2e` Ginkgo.
- `ci/prow/e2e-presubmits-images` — Same image build for the `e2e-presubmits` variant so AWS presubmit pods use the matching `rhcs-tf-e2e` image.
- `ci/prow/e2e-presubmits-rosa-hcp-advanced-critical-high-presubmit` — AWS FVT on **ROSA HCP advanced** profile; **Critical/High** day1-post + day2 Ginkgo filter against **staging OCM** (**rhcs-e2e-tests** step). Profile: `rosa-hcp-ad`.
- `ci/prow/e2e-presubmits-rosa-hcp-private-critical-high-presubmit` — Same **Critical/High** filter on **ROSA HCP private** topology. Profile: `rosa-hcp-pl`.
- `ci/prow/e2e-presubmits-rosa-sts-advanced-critical-high-presubmit` — Same **Critical/High** filter on **ROSA STS advanced** (classic-style) topology. Profile: `rosa-sts-ad`.
- `ci/prow/e2e-presubmits-rosa-sts-private-critical-high-presubmit` — Same **Critical/High** filter on **ROSA STS private** topology. Profile: `rosa-sts-pl`.

Additional presubmits (optional suites) are triggered with `/test …`.

For AWS `ci/prow/e2e-presubmits-`* jobs, **ci-operator** `cluster_profile: oex-aws-qe` selects **Boskos** / CI AWS account wiring—how `ci-pull-credentials`, `ocm-token`, `.awscred`, and related secrets are mounted.

That is separate from the `CLUSTER_PROFILE` values used for the four critical presubmits above (`rosa-hcp-ad`, `rosa-hcp-pl`, `rosa-sts-ad`, `rosa-sts-pl`), which choose the RHCS / Terraform scenario (see `tests/ci/profiles/` in this repo).

## Periodic jobs

Scheduled jobs on `main` live in openshift/release under `ci-operator/jobs/terraform-redhat/terraform-provider-rhcs/terraform-redhat-terraform-provider-rhcs-main-periodics.yaml`. They run AWS end-to-end style suites on a **cron**, not on every pull request.

Use the existing scheduled jobs on `main` in openshift/release (`terraform-redhat-terraform-provider-rhcs-main-periodics.yaml`) as the long-term AWS check after merge. Do not require full AWS presubmit on every Renovate PR.

Those scheduled jobs send results to Slack channel `#tf-provider-qe-ci`.

## CI environment

### OCM / API URL

Presubmit AWS FVT jobs use staging OCM at `https://api.stage.openshift.com`

### Prow cluster

The `cluster:` field on presubmits (for example `build09`) selects which OpenShift CI build cluster schedules the ci-operator pod (scheduling, compute, and CI integrations—not the ROSA cluster under test).

Presubmits carry `ci-operator.openshift.io/cloud: aws` because the workflow leases cloud capacity and consumes cloud credentials.

The ci-operator talks to Boskos (see `--lease-server-credentials-file=/etc/boskos/credentials` on the job pod) to obtain a quota slice tied to `cluster_profile: oex-aws-qe` (resource type such as `oex-aws-qe-quota-slice` in `core-services/prow/02_config/_boskos.yaml`), instead of each job encoding a single static AWS pool. 

`build09` is only the build-farm cluster that runs that pod; Boskos still governs which leased slice (and thus which credential tree under `CLUSTER_PROFILE_DIR`) the test step sees.

### AWS account

The numeric AWS account and static mapping for `oex-aws-qe` are not published in public job YAML; they live in the OpenShift CI secret store (materialized via bootstrap secret names such as `cluster-secrets-oex-aws-qe` in `core-services/ci-secret-bootstrap/_config.yaml`).

**AWS region**  
`REGION` is set **per presubmit** in `terraform-redhat-terraform-provider-rhcs-main__e2e-presubmits.yaml` (for example `us-west-2`, `ap-northeast-1`, `us-east-1` depending on the job); the e2e scripts may also align with the leased resource when `REGION` is not overridden.

**Image definition**  

- **Unit / `src` build:** `terraform-provider-rhcs/.ci-operator.yaml` → `build_root_image` (`ocp` `builder`, e.g. `rhel-9-golang-…`) defines the builder image for compiling and `make unit-test`.  
- **AWS FVT step image:** `build/ci-tf-e2e.Dockerfile` in this repo; the `rhcs-tf-e2e` image is declared under `images:` in `terraform-redhat-terraform-provider-rhcs-main__e2e.yaml` / `terraform-redhat-terraform-provider-rhcs-main__e2e-presubmits.yaml` (openshift/release); **rhcs-e2e-tests** runs `from: rhcs-tf-e2e`.

## CI credentials

The **rhcs-e2e-tests** / **rhcs-e2e-general-tests** scripts (openshift/release `ci-operator/step-registry/rhcs/e2e/...`) read **OCM** `RHCS_TOKEN` from `${CLUSTER_PROFILE_DIR}/ocm-token` and **AWS** credentials from `${CLUSTER_PROFILE_DIR}/.awscred` via `AWS_SHARED_CREDENTIALS_FILE`; shared-VPC scenarios may also use `${CLUSTER_PROFILE_DIR}/.awscred_shared_account`. The presubmit pod spec mounts `/secrets/ci-pull-credentials`, **Boskos** credentials, registry pull secrets, and related volumes so **ci-operator** can populate `CLUSTER_PROFILE_DIR` for `cluster_profile: oex-aws-qe`.