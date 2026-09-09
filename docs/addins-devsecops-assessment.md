# AddIns DevSecOps Standards Assessment

## Executive Summary

This assessment reviews the AddIns ecosystem against the CI stable standards
and best-practices repository. It covers:

- branching and repository governance
- continuous integration and delivery
- deployment and rollback strategies
- artifact management and release integrity
- application and infrastructure security
- infrastructure as code (IaC)
- documentation and operational readiness
- compliance and DevSecOps maturity

The AddIns repositories already have a solid technical base:

- reusable GCF templates
- HashiCorp Vault and Azure Key Vault integration
- Enterprise Artifactory usage
- environment-specific credentials and runners
- modular pipeline definitions
- security scanning tools
- remote Terraform state
- manual controls for production operations

The primary weakness is **governance consistency**. Several important controls
are optional, manual, or implemented differently across repositories. The next
stage of maturity should establish a consistent control baseline before adding
more advanced deployment techniques.

## Recommended Branching Strategy

Do not introduce a permanent `develop` branch only because the standards
repository contains a GitFlow example. Trunk-based development remains the
better fit for the AddIns ecosystem:

- `main` is authoritative, production-ready, and protected.
- `feature/*` branches are short-lived and merge into `main` through merge requests.
- `release/*` branches are temporary stabilization branches.
- `hotfix/*` branches start from the current production version and merge back into `main`.
- production releases use immutable Semantic Versioning tags created from `main`.
- temporary release and hotfix branches are deleted after reconciliation.

### Branching in Simple English

The purpose of a branching strategy is to control how a change moves from a
developer's machine into production. A branch is not an environment. It is a
temporary place to prepare and review a change.

For normal work, the flow should be:

1. Update local `main` from the remote repository.
2. Create a short-lived branch such as `feature/add-health-check` or
   `fix/certificate-expiry-alert`.
3. Make the change and open a merge request into `main`.
4. Let the merge-request pipeline compile, test, and scan the change.
5. Obtain the required independent approvals.
6. Merge only when the pipeline and approvals are complete.
7. Delete the feature branch after the merge.

This keeps `main` close to production and prevents long-lived branches from
collecting changes that are difficult to merge or test together.

```text
main
  └── feature/small-change
	  └── merge request + tests + review
		  └── main
```

### When to Use a Release Branch

A `release/*` branch is optional and temporary. Use one only when a release
needs a short stabilization period while other development continues on
`main`.

For example:

1. Create `release/1.4` from a known commit on `main`.
2. Deploy that branch to PPR for final validation.
3. Allow only release fixes, version updates, or release documentation on it.
4. Merge every release fix back into `main` before production.
5. Create the production tag from the reconciled commit on `main`.
6. Delete `release/1.4` after the release.

Do not treat a release branch as a second permanent main branch. If production
contains a change that is absent from `main`, the next release can accidentally
remove that change.

### How Hotfixes Should Work

A hotfix is an urgent correction to the version currently running in
production:

1. Identify the exact production tag, for example `v1.4.0`.
2. Create `hotfix/1.4.1` from that tag.
3. Make the smallest safe correction.
4. Run the normal tests and security checks.
5. Merge the correction into `main`.
6. Create a new patch tag, such as `v1.4.1`, from the merged commit.
7. Deploy the existing artifact produced for that tag.
8. Delete the hotfix branch after confirming production health.

Never move or reuse `v1.4.0`. A tag should permanently identify one source
commit and one released artifact.

## How Release Tags Should Work

Use Semantic Versioning in the form `vMAJOR.MINOR.PATCH`:

- **MAJOR**, such as `v2.0.0`, is for an incompatible or breaking change.
- **MINOR**, such as `v1.5.0`, adds backwards-compatible functionality.
- **PATCH**, such as `v1.4.1`, contains a backwards-compatible fix.

Recommended tag controls:

- only an authorized release role can create protected `v*` tags
- the tagged commit must already exist on protected `main`
- the pipeline must have passed before the tag is created
- tags are annotated and include release context
- a tag cannot be deleted, moved, or recreated during normal operation
- the tag creates a release record and immutable artifact

Example release flow:

```text
Merge request approved
	↓
Change merged to main
	↓
main pipeline passes
	↓
Release owner creates v1.4.0
	↓
Pipeline publishes artifact + digest + SBOM
	↓
The same artifact is promoted DEV → PPR → PRD
```

The Git tag identifies the source. The artifact version identifies the package
or container. The digest proves that the deployed bytes are unchanged. A good
release record links all three.

## Build Once and Promote

Do not rebuild the application separately for each environment. A rebuild can
produce different bytes because dependencies, timestamps, or build tools may
have changed.

Instead:

1. Build and test the application once.
2. Publish the immutable output to Enterprise Artifactory.
3. Record its version and SHA-256 digest.
4. Deploy that artifact to DEV.
5. Promote the same digest to PPR after validation.
6. Promote the same digest to PRD after approval.

Environment-specific configuration should be supplied at deployment or runtime;
it should not require rebuilding the application. This makes it possible to say
with evidence that production contains exactly what was tested in PPR.

## How Rollback Should Work

Rollback is the controlled restoration of the last known-good version. It must
be designed before deployment and tested regularly. A rollback plan written
only after an incident is not a reliable control.

### Application Rollback

For a normal application release:

1. Record the currently deployed version and digest before changing anything.
2. Keep that artifact available in the approved registry or repository.
3. Deploy the new version without deleting the previous one.
4. Run technical health checks and a small business smoke test.
5. Move traffic only after those checks pass.
6. If validation fails, restore the previous version or traffic target.
7. Verify service health and record the rollback in the release evidence.

Rollback should deploy an existing known-good artifact. It should not rebuild
old source code during an incident.

### Azure Container Apps Rollback

For XML API on Azure Container Apps, use revisions as the recovery mechanism:

1. Capture the active revision and image digest.
2. Deploy the new image as a separate revision.
3. Wait for readiness and application health checks.
4. Send smoke-test traffic to the new revision where possible.
5. Shift production traffic only when validation succeeds.
6. Keep the previous healthy revision available for the agreed rollback window.
7. On failure, route traffic back to the previous revision.
8. Deactivate older revisions only after the observation window ends.

The current XML API deployment deactivates older revisions. That order should
change so the previous healthy revision remains available until the new release
has passed application-level validation.

### Database and Infrastructure Rollback

Application rollback does not automatically reverse database or infrastructure
changes:

- Prefer backwards-compatible database changes using expand-and-contract.
- Take backups and test restoration before destructive schema changes.
- Use a reviewed Terraform plan to move infrastructure forward to a known
  configuration; avoid treating `terraform destroy` as rollback.
- Document changes that cannot be reversed and define a forward-fix procedure.

Every release plan should answer four questions:

1. What exact condition triggers rollback?
2. Who is authorized to decide and execute it?
3. Which version or revision will be restored?
4. How will the team prove that service has recovered?

## Migration Backlog

### P0: Control Baseline

| Recommendation | Affected repositories | Current gap | Migration action | Suggested owner | Acceptance criteria |
|---|---|---|---|---|---|
| Enforce protected-branch governance | All | Only the lights-off repository has a visible `CODEOWNERS` file | Protect `main`; block direct pushes; require two approvals; disable author and committer approval; reset approvals after new commits; require Code Owner approval | GitLab migration team | A developer cannot push directly or self-approve. Two independent approvals and a successful pipeline are required. |
| Require merge-request pipelines | XML API, AddIn, IaC, deployment repositories | XML API allows every pipeline source and has no MR-specific gate in `xml-api/.gitlab-ci.yml` | Add `workflow:rules` for merge requests, the default branch, valid tags, schedules, and approved manual runs; make MR validation mandatory | Migration team and repository owners | Every MR automatically runs validation and cannot merge when validation fails. |
| Restrict production to approved SemVer tags | XML API, both CD stacks, DevOps, Certificate, IaC | XML API production jobs have no branch or tag restrictions; AddIn CD permits `main` or any tag | Permit production deployment only from protected tags matching `vMAJOR.MINOR.PATCH` and created from `main` | Migration team | Production jobs do not appear in feature, release, arbitrary-tag, or unprotected pipelines. |
| Restore automated testing | XML API, AddIn, XML API CD, Certificate, IaC | XML API disables unit and functional tests; AddIn primarily performs Black Duck scanning | Enable unit tests on every MR; add integration and smoke tests where applicable; remove permanent `*_TEST_DISABLED` settings | Development teams | An MR pipeline fails when a test fails; test reports and coverage are retained. |
| Make security gates mandatory | XML API, AddIn | Black Duck is manual; XML API Trivy is allowed to fail | Run SAST, SCA, and container scanning automatically; define approved severity thresholds and time-limited exception handling | AppSec and migration team | High or critical violations block merge or release unless a recorded, expiring waiver exists. |
| Build once and promote by digest | XML API, AddIn, CD stacks | Pipeline-generated `0.0.$CI_PIPELINE_IID` versions are not formal releases; mutable tag promotion remains possible | Publish one versioned artifact to Enterprise Artifactory; record its digest and SBOM; promote exactly that digest through DEV, PPR, and PRD | Migration team | Deployment evidence for DEV, PPR, and PRD shows the same SHA-256 digest and release version. |
| Implement rollback as a first-class job | XML API and deployment repositories | XML API deactivates previous ACA revisions and has no rollback job | Retain the previous healthy revision or image; validate the new revision before traffic cutover; add protected `rollback-production` jobs and a tested runbook | Platform and application owners | A rollback drill restores the previous version within the agreed recovery time without rebuilding it. |
| Protect production environments | Deployment repositories and IaC | Manual jobs exist, but a manual button alone does not provide separation of duties | Configure protected GitLab environments, authorized deployer groups, deployment approvals, and production runner restrictions | GitLab administrators | Only the release or deployment group can authorize production; developer credentials cannot deploy directly. |

### P1: Release Integrity and Engineering Consistency

| Recommendation | Affected repositories | Current gap | Migration action | Suggested owner | Acceptance criteria |
|---|---|---|---|---|---|
| Standardize MR descriptions and traceability | All | No MR templates were found | Add `.gitlab/merge_request_templates/Default.md` with description, Jira or CR links, risk, testing, deployment, rollback, evidence, and checklist sections | Migration team | Every MR contains a work-item URL, test evidence, impact assessment, and rollback information. |
| Add CODEOWNERS everywhere | All except lights-off | Only the lights-off repository has a visible `CODEOWNERS` file | Define default owners and specialist ownership for CI, security, Terraform, production configuration, and certificate paths | Repository owners | Sensitive changes require approval from the relevant Code Owner. |
| Standardize release metadata | All releasable repositories | No changelogs were found; XML API uses a snapshot Maven version lineage | Adopt Conventional Commits, automated SemVer calculation, annotated protected tags, release records, and a generated `CHANGELOG.md` | Development teams | A tag automatically produces a release, changelog, immutable artifact, SBOM, and pipeline evidence. |
| Strengthen Terraform controls | IaC | Remote Azure state and environment-specific keys are good; Wiz uses unpinned `ref: stable`; `.terraform.lock.hcl` is ignored; `terraform.exe` is stored in the repository | Pin all templates; commit provider lock files; obtain Terraform from approved tooling; run format, validation, security, and plan checks on MRs; protect apply, unlock, and destroy | Cloud platform and migration team | Each MR contains a reviewed plan; apply consumes that plan; state is encrypted, locked, versioned, access-controlled, and backed up. |
| Add concurrency and deployment safety | XML API, CD stacks, IaC | Parallel pipelines may target the same environment; retries can repeat state-changing jobs | Add `resource_group` per environment and region, sensible timeouts, interruptible non-deployment jobs, and non-interruptible production jobs | Migration team | Only one deployment or Terraform apply can modify a target environment at a time. |
| Establish release reconciliation | XML API and AddIn | Release branches can diverge from `main` | Automatically verify that each release or tag commit is contained in `main`; delete temporary release branches after production | Repository owners | No production release contains code that is absent from `main`. |

### P2: Operational Maturity

| Recommendation | Affected repositories | Current gap | Migration action | Suggested owner | Acceptance criteria |
|---|---|---|---|---|---|
| Improve API and operational documentation | XML API, Certificate, lights-off | XML API has useful runbooks but no discovered OpenAPI specification or changelog; lights-off retains a largely stock README | Generate OpenAPI where applicable; document architecture, support, SLOs, dependencies, deployment, rollback, disaster recovery, and ownership in each repository | Application owners | A new engineer can build, test, release, operate, and roll back the application using repository documentation. |
| Add deployment observability | Runtime repositories | Infrastructure has health probes, but release success is not tied to service-level signals | Add post-deployment smoke tests, metrics and log links, alert checks, and deployment annotations | SRE and operations | Deployment fails or rolls back when health or SLO checks fail; the release is visible in monitoring. |
| Introduce policy-as-code reporting | All | Controls are distributed across repositories and GitLab settings | Create a centrally versioned compliance component that checks protected references, pinned includes, required jobs, owners, tags, artifacts, and evidence | Migration and platform team | A compliance dashboard reports each repository against the standard and records approved exceptions. |

## Repository-Specific Focus

### XML API

Highest-priority improvements:

1. Enable merge-request pipelines.
2. Restore unit and functional tests.
3. Make security scans mandatory.
4. Restrict production deployment to approved protected tags.
5. Introduce safe Azure Container Apps revision rollback.

Retain the modular pipeline structure and pinned external includes. These are
good foundations for a standardized delivery pipeline.

### Yield Book AddIn

The AddIn needs a complete CI pipeline around:

1. dependency restoration
2. deterministic compilation
3. unit and integration testing
4. packaging and signing
5. SAST and SCA scanning
6. immutable artifact publication
7. controlled rollout and recovery packages

The current local pipeline is primarily a manually triggered Black Duck scan,
which does not provide sufficient build or release assurance.

### AddIn CD Stack

The existing branch rules are a useful start. Strengthen them by:

- allowing production only from protected SemVer tags
- using protected GitLab environments
- requiring deployment approvals
- verifying the promoted artifact digest
- preventing concurrent deployment to the same environment and region

### XML API CD Stack

The repository currently has a small Vault-oriented root pipeline. Confirm
whether it owns an independent deployment responsibility. Then either:

- align it with the common deployment policy; or
- retire duplicated deployment responsibilities and keep one authoritative path.

### Terraform Infrastructure

Remote Azure state and Wiz scanning are strong foundations. Prioritize:

- immutable versions for all imported templates
- committed provider lock files
- MR-based format, validate, security, and plan checks
- plan-to-apply integrity
- stronger apply, unlock, and destroy authorization
- documented state backup and recovery tests

### Certificate Monitor

Use the certificate monitor as a reference implementation for:

- restricted pipeline sources
- automated tests
- short-lived and inaccessible secret artifacts
- documented operational procedures
- scheduled execution with controlled manual runs

### Certificate Deployment

Add:

- branch and tag gates
- protected deployment environments
- certificate validation before promotion
- auditable version recovery and rollback
- post-deployment verification

### Lights-Off Operations

Preserve the existing `CODEOWNERS` configuration. Replace the stock README
content with documentation covering:

- schedule and trigger behavior
- ownership and support contacts
- test procedure
- credential boundaries
- expected failure modes
- recovery and rerun steps

### DevOps Repository

Use this repository as the centrally maintained, versioned pipeline-component
layer. Publish immutable tagged releases and prevent application repositories
from depending on moving branches such as `main`, `stable`, or `latest`.

## Controls Requiring GitLab Configuration Review

Some recommendations cannot be confirmed from repository files alone. The
migration team should inventory these settings through the GitLab API or
administration interface:

- protected branches and tags
- direct-push permissions
- required approval count
- author and committer approval restrictions
- approval reset after new commits
- Code Owner approval requirements
- successful-pipeline merge checks
- protected environments and deployment approvers
- runner scope and production runner access
- role permissions and separation of duties

Record the result in a repository-by-repository compliance matrix before
changing settings.

## Standards Adoption Caveat

Use the standards repository as policy guidance rather than copying every
example literally. Some examples should be corrected or modernized before use:

- regex such as `^release*` does not correctly represent `release/*`
- branch-based includes should be replaced with immutable version tags
- legacy `only` and `except` syntax should be replaced with `rules`
- naming examples should be reconciled where underscore and hyphen guidance differs

Production templates should implement the intent using current GitLab features
and validated regular expressions.

## Recommended Delivery Sequence

### Phase 1: Control Baseline

- protect branches, tags, and environments
- enforce approvals and Code Owners
- require merge-request pipelines
- restore automated tests
- make security scans blocking
- restrict production to approved SemVer tags

### Phase 2: Release Integrity

- build once and promote by digest
- produce SBOM and provenance evidence
- standardize SemVer and release records
- add protected deployment approvals
- implement and test rollback
- enforce Terraform plan-to-apply integrity

### Phase 3: Operational Maturity

- connect releases to health and SLO signals
- automate rollback criteria where appropriate
- implement compliance reporting
- exercise disaster recovery procedures
- reassess maturity and prioritize the next improvements

## Target Outcome

The target is an **Established** level of DevSecOps maturity for source control,
continuous integration, security, artifact management, deployment governance,
and audit evidence. Advanced progressive delivery should follow once every
production change is reviewed, tested, traceable, reproducible, recoverable,
and observable.