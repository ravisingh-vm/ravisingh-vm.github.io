# Azure Container Apps Production Deployment Strategy

## Purpose

This plan gives the migration team a practical production deployment strategy
for XML API on Azure Container Apps (ACA). The objective is to prevent planned
release downtime and make rollback a fast traffic operation instead of waiting
for an old container to start again.

DEV can continue using a simpler deployment flow. This recommendation is for
**PPR validation and production rollout**, with the strongest controls applied
to PRD.

## Executive Recommendation

Use **blue-green deployment with ACA multiple revision mode** in production:

- the current production revision stays active with 100% traffic
- the candidate revision starts with 0% production traffic
- the team validates the candidate through a revision label URL
- traffic moves to the candidate only after technical and business checks pass
- the previous revision stays active and warm at 0% during an observation window
- rollback sends 100% traffic back to the previous revision

Start with an atomic `100/0` to `0/100` traffic switch. Do not introduce a
weighted canary until the team proves that XML API is stateless, or that session
state and request affinity work safely across revisions.

```text
Before release
  blue/current: 100% traffic, active and healthy
  green/new:       0% traffic, not deployed

Candidate validation
  blue/current: 100% production traffic
  green/new:       0% production traffic, tested through candidate URL

Production cutover
  blue/current:   0% traffic, active and warm
  green/new:    100% traffic, active and monitored

Rollback
  blue/current: 100% traffic
  green/new:      0% traffic
```

## Why the Current Rollout Takes Time

The present configuration and deployment script have four important limitations:

1. All environments use `revision_mode = "Single"`.
2. The readiness probe interval is `240` seconds.
3. The deployment script waits a fixed 20 seconds instead of checking real ACA state.
4. The script deactivates every older revision immediately after deployment.

Single revision mode itself supports zero-downtime replacement: ACA keeps the
old revision serving traffic until the new revision is ready. However, a
four-minute probe interval can delay readiness detection, and immediate cleanup
means the previous revision is no longer warm when rollback is needed.

## Target Production Configuration

### Revision and Replica Settings

Recommended starting point:

```hcl
revision_mode = "Multiple"

min_replicas = 2
max_replicas = 4
```

Use at least two production replicas so one replica can fail or restart without
making the service unavailable. Final replica counts must be confirmed using
load tests, CPU and memory observations, startup time, and expected peak traffic.

The migration team must also confirm that the ACA environment and quota can run
both blue and green revisions at the same time. During deployment, capacity can
temporarily approach twice the normal minimum.

### Cost Impact

This strategy can increase ACA compute cost, but the increase depends on how
long both revisions remain active and how many minimum replicas are configured.

- Moving the normal production baseline from one minimum replica to two increases
  the always-on replica cost.
- During blue-green deployment, both old and new revisions run at the same time.
  Compute usage can temporarily approach twice the normal production capacity.
- Keeping the previous revision warm at 0% traffic still consumes compute when
  it has minimum replicas assigned.
- After the observation and rollback window, deactivating the old revision returns
  usage to the new steady-state baseline.
- Inactive ACA revisions do not consume running replica compute, but reactivating
  one introduces cold-start time during rollback.

Cost-balanced recommendation:

1. Measure the current production CPU, memory, replica count, and startup time.
2. Use two minimum replicas where the availability requirement justifies it.
3. Keep both revisions warm only for a defined 30-to-60-minute observation window.
4. Deactivate the previous revision after release acceptance while retaining its
   immutable image and revision history.
5. Confirm ACA quota and estimate the temporary overlap cost before production adoption.
6. Review actual cost and utilization after the first three production releases.

The trade-off is a controlled, temporary cost increase in exchange for no planned
cutover downtime and rollback in seconds rather than waiting for a cold revision.
The migration team should present the expected steady-state and deployment-window
costs alongside the reliability benefit before final approval.

### Traffic Management

The current Terraform module models one `latest_revision` traffic rule. Extend
or upgrade it to support a list of explicit revision or label rules whose total
weight is exactly 100%.

Conceptual target:

```hcl
traffic = [
  {
    revision_name = "xml-api--v1-5-0"
    percentage    = 100
    label         = "production"
  },
  {
    revision_name = "xml-api--v1-6-0"
    percentage    = 0
    label         = "candidate"
  }
]
```

Use stable labels:

- `production` points to the revision receiving user traffic
- `candidate` points to the revision undergoing release validation

The deployment pipeline should control traffic during a release. Terraform
should establish the capability and baseline configuration, but it should not
unexpectedly reset traffic during an active deployment or rollback.

### Health Probes

Replace the current `240`-second readiness interval with dedicated startup,
readiness, and liveness probes. Initial values should be validated with measured
application startup data.

Recommended starting values:

| Probe | Initial delay | Interval | Timeout | Failure threshold | Purpose |
|---|---:|---:|---:|---:|---|
| Startup | 5 seconds | 5 seconds | 3 seconds | 60 | Give the JVM and application up to five minutes to initialize |
| Readiness | 5 seconds | 5 seconds | 3 seconds | 6 | Decide whether a replica can receive traffic |
| Liveness | 30 seconds | 10 seconds | 3 seconds | 3 | Restart a genuinely stuck application process |

Enable Spring Boot probe groups:

```yaml
management:
  endpoint:
    health:
      probes:
        enabled: true
  endpoints:
    web:
      exposure:
        include: health,info,metrics
```

Use these endpoints:

```text
Startup:   /actuator/health/liveness
Liveness:  /actuator/health/liveness
Readiness: /actuator/health/readiness
```

Liveness should test the application process, not every remote dependency. A
temporary database or downstream outage must not cause all replicas to restart
continuously. Readiness can include dependencies that must be available before
the replica receives requests.

The current Terraform module exposes only one readiness probe. The migration
team must extend the module or adopt a supported module version that provides
all three probe types.

### Port Consistency

PPR and PRD use port `8080`, matching the Docker image and Spring profiles. DEV
currently declares ACA ingress and probes on `3000`, while the application files
declare `8080`. DEV is outside this production plan, but the discrepancy should
still be confirmed to avoid carrying a hidden configuration error into shared
modules.

## Production Pipeline Design

Use separate, auditable jobs rather than one script that updates, waits, and
cleans up everything.

```text
release-precheck
      ↓
deploy-candidate
      ↓
wait-for-candidate
      ↓
validate-candidate
      ↓
approve-production-cutover
      ↓
switch-production-traffic
      ↓
observe-production
      ↓
accept-release
      ↓
retire-old-revision (delayed/manual)
```

### 1. Release Precheck

The pipeline must verify:

- release tag is protected and matches `vMAJOR.MINOR.PATCH`
- tagged commit is contained in protected `main`
- required build, test, and security pipelines passed
- image exists in the production ACR
- image digest matches the approved PPR artifact
- SBOM and release evidence are available
- no other production deployment is running
- both production regions have sufficient ACA quota

Use a GitLab `resource_group` per application, environment, and region to prevent
concurrent production deployments.

### 2. Capture Current State

Before deployment, record as a pipeline artifact:

- current production revision
- current image tag and digest
- current traffic weights and labels
- replica count and health state
- deployment timestamp and release tag

This file becomes the rollback input. Do not discover the rollback target by
guessing which revision looks older during an incident.

### 3. Deploy Candidate

Create a unique revision suffix derived from the release version and short
commit SHA, for example:

```text
v1-6-0-a1b2c3d
```

Deploy the candidate revision with 0% production traffic. Assign the `candidate`
label so automated validation can call its stable label URL.

### 4. Wait for Real State

Remove fixed sleeps such as `sleep 20`. Poll ACA until all conditions are true:

- provisioning state is `Provisioned`
- running state is `Running`
- desired minimum replicas are available
- all replicas pass startup and readiness probes

Use a bounded timeout. If the timeout expires, fail the deployment while the
old production revision continues serving 100% traffic.

### 5. Validate Candidate

Run two levels of validation against the candidate URL:

**Technical checks**

- readiness endpoint returns success
- expected release version is reported
- database connection succeeds
- required storage and secrets are accessible
- no repeated container restarts occur
- logs contain no startup errors

**Business smoke checks**

- submit one representative XML API request
- verify authentication and authorization
- verify a response from required backend services
- confirm no material latency regression

The smoke test must be read-only or use controlled test data.

### 6. Approve and Switch Traffic

Use a protected GitLab production environment and an authorized release or
operations group. The developer who authored the release should not be the only
approver.

After approval, switch traffic atomically:

```text
old revision = 0%
new revision = 100%
```

Move the `production` label to the new revision. Keep the previous revision
active at 0%.

### 7. Observe Production

Use a defined observation window, initially 30 to 60 minutes. Monitor:

- HTTP 5xx and timeout rate
- request latency and throughput
- readiness failures and replica restarts
- CPU and memory saturation
- database connection pool behavior
- XML API business smoke checks
- downstream Locator, Book, Bridge, storage, and identity failures

The pipeline should publish direct links to the relevant dashboards and logs.

### 8. Accept and Retire

After the observation window and release-owner acceptance:

- record the final revision and digest
- retain release evidence
- leave at least one previous known-good revision in ACA history
- deactivate the warm previous revision only after the rollback window
- clean up older revisions through a separate controlled job

Do not run old-revision cleanup inside the deployment job.

## Production Rollback Strategy

Rollback must be a protected pipeline job, available without rebuilding code.

### Rollback Trigger Conditions

Agree measurable triggers before release. Examples:

- readiness or business smoke test fails after cutover
- HTTP 5xx rate exceeds the agreed threshold
- critical endpoint latency breaches its SLO
- repeated replica restarts occur
- authentication or a critical downstream integration fails
- incident commander or release owner declares rollback

### Warm Revision Rollback

When the previous revision is still active at 0%:

1. Validate that the previous revision is running and ready.
2. Set previous revision traffic to 100%.
3. Set candidate revision traffic to 0%.
4. Move the `production` label back to the previous revision.
5. Run technical and business recovery checks.
6. Record the restored revision, image digest, operator, and reason.
7. Keep the failed revision at 0% for diagnosis.

This rollback should take seconds because it changes routing rather than waiting
for an inactive revision to cold-start.

### Cold Revision Fallback

If the previous revision was already deactivated:

1. Reactivate the exact revision captured by the precheck job.
2. Wait for its replicas and probes to become healthy.
3. Switch traffic only after validation.

This is slower and is why the previous revision should remain warm during the
rollback window.

### Database and Configuration Compatibility

Blue-green rollback is safe only when the previous application can still work
with the current database and configuration.

- Use expand-and-contract database migrations.
- Add backwards-compatible schema changes before removing old fields.
- Delay destructive schema changes until old application revisions are retired.
- Treat irreversible changes as forward-fix releases with tested recovery plans.
- Confirm that secrets and application-scope ACA settings remain compatible with
  both blue and green revisions.

ACA application-scope changes, including secrets and ingress configuration, can
affect all revisions. They require separate risk analysis from an image-only
revision deployment.

## Multi-Region Production Rollout

XML API runs in East US 2 and Central US. Do not update both regions at the same
time.

Recommended sequence:

1. Confirm Front Door or upstream routing and regional failover are healthy.
2. Deploy and validate the candidate in the first production region.
3. Switch that region and observe it for an agreed period.
4. Continue only if regional health and business checks pass.
5. Deploy and validate the second region.
6. Switch the second region and run global validation.

If business routing permits, reduce traffic to the region being changed while
the other region carries more load. Confirm the remaining region has sufficient
capacity before doing this.

## Migration Team Work Plan

### Workstream 1: Application Health Contract

**Owner:** XML API development team  
**Support:** Migration team and operations

Deliverables:

- enable Spring Boot liveness and readiness groups
- define what readiness checks and what liveness checks
- add a lightweight authenticated or controlled business smoke test
- expose reliable version information for deployment verification
- measure normal and worst-case startup time

Acceptance criteria:

- each endpoint returns the expected status under healthy and failed dependency conditions
- temporary downstream failure does not create a liveness restart loop
- readiness blocks traffic until the application can serve a representative request

### Workstream 2: Terraform and ACA Capability

**Owner:** Migration or cloud platform team  
**Support:** XML API and Terraform repository owners

Deliverables:

- add startup, readiness, and liveness probe support to the Terraform module
- enable multiple revision mode in PPR first, then PRD
- support explicit revision or label traffic rules
- configure production minimum replicas and confirm quota
- prevent Terraform from overwriting pipeline-managed release traffic
- document the ownership boundary between Terraform and deployment automation

Acceptance criteria:

- both revisions can run simultaneously
- candidate receives 0% main ingress traffic and is reachable by label URL
- traffic can be switched and restored without recreating either revision
- Terraform plan after deployment does not unexpectedly revert traffic

### Workstream 3: Deployment Pipeline

**Owner:** Migration team  
**Support:** GitLab administrators and application owners

Deliverables:

- split deployment into precheck, deploy, wait, validate, approve, switch, observe, and retire jobs
- capture rollback metadata before deployment
- use unique revision suffixes
- remove fixed sleeps and poll real ACA state
- add candidate technical and business tests
- add GitLab `resource_group` locking
- protect production environments and restrict deployers

Acceptance criteria:

- a failed candidate receives no production traffic
- the pipeline cannot run two deployments against one production target
- every release records source tag, commit, artifact digest, revisions, approvers, and checks

### Workstream 4: Rollback Automation

**Owner:** Migration team and production operations  
**Support:** XML API team

Deliverables:

- add a protected `rollback-production` job
- consume captured rollback metadata rather than selecting a revision heuristically
- restore traffic and labels to the previous revision
- run recovery verification
- preserve the failed revision for investigation

Acceptance criteria:

- a rollback drill restores the previous warm revision within the agreed RTO
- no image rebuild is required
- restored version and digest are visible in pipeline evidence and monitoring

### Workstream 5: Observability and Operations

**Owner:** SRE or production operations  
**Support:** Migration and application teams

Deliverables:

- define release health thresholds and rollback triggers
- add dashboard and log links to deployment jobs
- annotate monitoring with release version and revision
- create alerts for readiness failures, restarts, 5xx errors, and latency
- document operator roles and escalation path

Acceptance criteria:

- operators can identify the active version in each region
- a release problem is visible before the observation window ends
- rollback authority and escalation contacts are documented

## Phased Implementation

### Phase 1: Correct the Existing Safety Gaps

Priority actions:

1. Reduce readiness probe interval from 240 seconds to approximately 5 seconds.
2. Replace fixed sleeps with bounded ACA state polling.
3. Stop deactivating older revisions inside the deployment job.
4. Capture current revision and digest before every deployment.
5. Add post-deployment technical and business smoke tests.
6. Add production pipeline concurrency protection.

This phase improves the current Single mode workflow while the team prepares
multiple revision blue-green capability.

### Phase 2: Prove Blue-Green in PPR

1. Extend the Terraform module for multiple revisions, labels, traffic, and probes.
2. Run current and candidate revisions simultaneously in PPR.
3. Validate candidate label access and traffic switching.
4. Perform repeated forward deployment and rollback drills.
5. Measure cutover time, rollback time, startup time, and capacity.
6. Resolve session, database, secret, and downstream compatibility issues.

Do not enable the production design until PPR rollback drills consistently meet
the agreed recovery objective.

### Phase 3: Controlled Production Introduction

1. Enable the capability in one production region.
2. Perform a low-risk release with extended observation.
3. Demonstrate warm rollback in production under change control.
4. Review evidence and incidents with application, migration, and operations teams.
5. Enable the second region after acceptance.
6. Standardize the pattern as a reusable deployment component.

### Phase 4: Optional Progressive Delivery

Only after blue-green deployment is stable, consider weighted canary traffic:

```text
old 95% / new 5%
old 75% / new 25%
old 50% / new 50%
old  0% / new 100%
```

Canary requires reliable automated metrics, compatible sessions, sufficient
traffic for meaningful comparison, and automatic halt or rollback thresholds.
It is not required to achieve zero planned deployment downtime.

## Definition of Done

The production deployment strategy is complete when:

- no planned deployment requires an endpoint outage
- old production remains available until the candidate is ready
- the candidate receives no production traffic before validation and approval
- production runs at least two replicas per active serving revision
- probe timings are based on measured startup behavior
- both regions deploy sequentially, not simultaneously
- rollback uses the captured previous revision and artifact digest
- warm rollback meets the agreed recovery objective
- database and configuration changes remain backwards-compatible during the rollback window
- release and rollback drills are documented with evidence
- migration, application, and operations teams have clear ownership

## Recommended Decision

Approve **multiple-revision blue-green deployment** as the target production
strategy. Deliver it incrementally: correct the current probe and cleanup gaps,
prove the complete workflow in PPR, introduce it to one production region, and
expand only after a successful deployment and rollback exercise.