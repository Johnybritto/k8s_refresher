# GitLab CI/CD Interview Refresher

Focused interview notes for Principal / Senior Platform / DevOps roles.

---

## 1. GitLab CI/CD Architecture

Basic flow:

```text
Developer
   ↓
GitLab Repository
   ↓
.gitlab-ci.yml
   ↓
GitLab Pipeline
   ↓
GitLab Runner
   ↓
Build / Test / Scan / Package
   ↓
Artifact or Container Registry
   ↓
Deploy to Kubernetes / VM / Cloud
```

### Main components

1. **GitLab Repository**
   - Stores source code.
   - Stores `.gitlab-ci.yml`.
   - Merge Requests can trigger validation pipelines.

2. **`.gitlab-ci.yml`**
   - Pipeline definition as code.
   - Defines stages, jobs, variables, rules, dependencies and artifacts.

Typical stages:

```yaml
stages:
  - build
  - test
  - security
  - package
  - deploy
```

3. **GitLab Pipeline**
   - Created by push, merge request, tag, schedule, API call or manual trigger.

4. **GitLab Runner**
   - GitLab orchestrates the pipeline.
   - Runner actually executes the commands.

5. **Registry / Artifact Store**
   - Stores Docker images and build outputs.

---

## 2. Designing GitLab CI/CD for Many Microservices

Assume each microservice has its own repository:

```text
user-service
payment-service
order-service
notification-service
```

Each repo contains:

```text
Dockerfile
.gitlab-ci.yml
application code
```

### Recommended flow

```text
Code
 ↓
Build
 ↓
Test
 ↓
Security Scan
 ↓
Create Docker Image
 ↓
Push Image to Registry
 ↓
Promote / Deploy
```

### Important design principles

- Each microservice should deploy independently.
- Do not duplicate large pipeline YAML across 100 repositories.
- Use reusable CI templates.
- Build the image once.
- Promote the same immutable image through all environments.
- Integrate security before deployment.
- Prefer GitOps for Kubernetes CD.

### Interview answer

> I would give each microservice an independent pipeline but centralize common pipeline logic using reusable GitLab templates. Autoscaling runners execute build, test and security jobs. Each service builds an immutable container image once, pushes it to the registry, and promotes the same artifact across environments. For Kubernetes deployment, I would separate CI from CD and use ArgoCD or Flux for GitOps-based deployment.

---

## 3. Reusable GitLab Templates

Instead of duplicating pipeline logic:

```text
service-a/.gitlab-ci.yml
service-b/.gitlab-ci.yml
service-c/.gitlab-ci.yml
```

Create a central repository:

```text
platform-ci-templates/
├── build.yml
├── test.yml
├── security.yml
└── deploy.yml
```

Then a service includes it:

```yaml
include:
  - project: platform/ci-templates
    file: build.yml
```

### Why use reusable templates?

- Standardization.
- Central security controls.
- Less duplicated YAML.
- Easier upgrades.
- Update one template instead of 100 repositories.

---

## 4. Sample `build.yml`

```yaml
build_image:
  stage: build

  image: docker:27

  services:
    - docker:27-dind

  variables:
    DOCKER_TLS_CERTDIR: ""

  script:
    - echo "Logging in to GitLab Container Registry"
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" "$CI_REGISTRY"

    - echo "Building Docker image"
    - docker build -t "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA" .

    - echo "Pushing Docker image"
    - docker push "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA"

  rules:
    - if: '$CI_COMMIT_BRANCH'
```

### How `docker build` uses the Dockerfile

This command:

```bash
docker build -t "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA" .
```

uses the current directory (`.`) as build context and automatically looks for:

```text
Dockerfile
```

Example repo:

```text
payment-service/
├── Dockerfile
├── app.py
├── requirements.txt
└── .gitlab-ci.yml
```

For a custom Dockerfile:

```bash
docker build -f docker/Dockerfile.prod -t payment-service:abc123 .
```

Relationship:

```text
build.yml
   ↓
docker build
   ↓
Dockerfile
   ↓
Docker image
```

---

## 5. GitLab Runners

### Key rule

> GitLab orchestrates the pipeline; the Runner executes the job.

Runner executes commands such as:

```text
docker build
pytest
trivy scan
terraform plan
```

### Runner scope

- **Instance runner** — available across the GitLab instance.
- **Group runner** — available to projects in a group.
- **Project runner** — dedicated to a specific project.

### Executor types to remember

- Shell executor.
- Docker executor.
- Kubernetes executor.

---

## 6. Shell Executor

Commands execute directly on the runner host OS.

```text
GitLab
  ↓
Runner
  ↓
Host OS
  ↓
Execute shell commands
```

Example:

```yaml
build:
  script:
    - python test.py
    - docker build -t app:v1 .
```

### Advantages

- Simple.
- Fast.
- Easy to troubleshoot.

### Risks

- Poorer isolation.
- Jobs can affect the host.
- Dependency conflicts between jobs.

---

## 7. Docker Executor

Runner creates a temporary Docker container for the job.

```text
GitLab
  ↓
Runner
  ↓
Docker Engine API
  ↓
Create container
  ↓
Run job
  ↓
Delete container
```

Example:

```yaml
test:
  image: python:3.12
  script:
    - pip install -r requirements.txt
    - pytest
```

Runner roughly performs:

```text
Pull python:3.12
Create container
Mount working directory
Inject CI variables
Run commands
Collect logs/artifacts
Delete container
```

The job does not need to manually execute `docker run`; the Docker executor handles this.

---

## 8. Kubernetes Executor

Runner creates a temporary Kubernetes Pod for every job.

```text
GitLab
  ↓
GitLab Runner
  ↓
Kubernetes API Server
  ↓
Create Pod
  ↓
Scheduler selects node
  ↓
Kubelet starts containers
  ↓
Job executes
  ↓
Pod deleted
```

If many pipelines run simultaneously:

```text
job-pod-1
job-pod-2
job-pod-3
...
```

### Advantages

- Strong job isolation.
- Parallel execution.
- Autoscaling.
- Ephemeral clean environments.
- Kubernetes resource controls.

### Important distinction

> Docker executor talks to the Docker daemon. Kubernetes executor talks to the Kubernetes API server.

The Kubernetes scheduler, not GitLab Runner, decides which node runs the Pod.

---

## 9. Where Executor Configuration Is Defined

Executor type is configured on the Runner, commonly in `config.toml`.

### Docker executor

```toml
[[runners]]
  name = "docker-runner"
  url = "https://gitlab.example.com"
  token = "xxxxx"
  executor = "docker"

  [runners.docker]
    image = "alpine:latest"
    privileged = true
```

Critical line:

```toml
executor = "docker"
```

### Kubernetes executor

```toml
[[runners]]
  name = "k8s-runner"
  url = "https://gitlab.example.com"
  token = "xxxxx"
  executor = "kubernetes"

  [runners.kubernetes]
    namespace = "gitlab-runner"
    image = "alpine:latest"
```

Critical line:

```toml
executor = "kubernetes"
```

### Remember

```text
config.toml
   ↓
HOW should the job run?
Shell / Docker / Kubernetes

.gitlab-ci.yml
   ↓
WHAT should the job run?
Image + commands
```

---

## 10. Shared vs Dedicated Runners

### Shared runners

```text
Project A ─┐
Project B ─┼──> Shared Runner Pool
Project C ─┘
```

Good for:

- Normal build/test workloads.
- High utilization.
- Cost efficiency.
- Similar workloads.

Risks:

- Noisy neighbors.
- Less isolation.

### Dedicated runners

```text
Project A ───> Dedicated Runner A
Project B ───> Dedicated Runner B
```

Use for:

- Production deployment jobs.
- Sensitive workloads.
- Compliance requirements.
- Special tooling.
- Restricted network connectivity.
- Heavy builds requiring guaranteed capacity.

### Interview answer

> I would use shared autoscaling runners for normal CI workloads but dedicated runners for sensitive operations such as production deployments, restricted-network or air-gapped packaging, and privileged workloads.

---

## 11. Runner Tags

Runner tags route jobs to specific runner pools.

Example runner tags:

```text
linux
windows
prod
kubernetes
gpu
secure
```

Example job:

```yaml
deploy_prod:
  tags:
    - prod
    - kubernetes
  script:
    - ./deploy.sh
```

The job runs only on a matching runner.

---

## 12. Cache

Cache speeds up builds by reusing downloaded dependencies or generated files.

### Python example

```yaml
test:
  image: python:3.12

  cache:
    key: python-deps
    paths:
      - .cache/pip/

  script:
    - pip install --cache-dir .cache/pip -r requirements.txt
    - pytest
```

Flow:

```text
First pipeline
   ↓
Download dependencies
   ↓
Store cache
   ↓
Next pipeline
   ↓
Restore cache
   ↓
Reuse dependencies
```

### Cache key based on dependency file

```yaml
cache:
  key:
    files:
      - requirements.txt
  paths:
    - .cache/pip/
```

If `requirements.txt` changes, a new cache can be created.

### Common cache examples

- Maven `.m2/repository/`.
- npm `.npm/`.
- pip cache.

---

## 13. Artifacts vs Cache

### Cache

Used to speed up future builds.

Examples:

- Maven dependencies.
- npm packages.
- Python packages.

### Artifacts

Job outputs that must be passed to another job or retained.

Examples:

- JAR.
- Binary.
- Test report.
- Coverage report.

### Interview rule

> Cache is for performance; artifacts are job outputs.

---

## 14. Passing Artifacts Between Jobs

```yaml
stages:
  - build
  - test

build_app:
  stage: build
  script:
    - mkdir output
    - echo "my compiled application" > output/app.txt

  artifacts:
    paths:
      - output/app.txt
    expire_in: 1 day


test_app:
  stage: test

  needs:
    - job: build_app
      artifacts: true

  script:
    - cat output/app.txt
    - echo "Running tests using the artifact"
```

Flow:

```text
build_app
   ↓
creates artifact
   ↓
GitLab stores artifact
   ↓
test_app starts
   ↓
GitLab downloads artifact
   ↓
test_app uses artifact
```

---

## 15. `needs`

`needs` allows a job to start as soon as required dependencies complete instead of waiting for every job in the previous stage.

Example:

```yaml
test_app:
  needs:
    - job: build_app
      artifacts: true
```

Useful for reducing overall pipeline duration.

---

## 16. Pipeline Triggers

Pipelines can be triggered by:

- Push.
- Merge Request.
- Git tag.
- Schedule.
- API call.
- Manual action.

---

## 17. `rules`

`rules` determines whether a job should run.

Example:

```yaml
deploy_prod:
  stage: deploy
  rules:
    - if: '$CI_COMMIT_TAG'
```

This could restrict production deployment to tagged releases.

Older GitLab pipelines may also use `only` / `except`.

---

## 18. Variables and Secrets

Do not hardcode secrets in `.gitlab-ci.yml`.

Use:

- GitLab CI/CD variables.
- Masked variables.
- Protected variables.
- HashiCorp Vault.
- Cloud secret managers.

Typical flow:

```text
GitLab Runner
     ↓
Authenticate to Vault
     ↓
Retrieve temporary secret
     ↓
Execute deployment
```

---

## 19. Security Pipeline

Security should be part of CI/CD before deployment.

```text
Source Code
 ↓
SAST
 ↓
Unit Tests
 ↓
Dependency Scan
 ↓
Docker Build
 ↓
Container Scan
 ↓
SBOM
 ↓
Image Signing
 ↓
Registry
 ↓
Deployment
```

### SAST

Scans source code for security weaknesses.

### DAST

Tests the running application for security vulnerabilities.

### SCA / Dependency scanning

Checks third-party packages and libraries for known vulnerabilities.

### Container scanning

Tools such as Trivy or Snyk scan container images.

### SBOM

Software Bill of Materials: inventory of software components shipped inside an application.

Example:

```text
payment-service
├── Python
├── OpenSSL
├── requests
└── nginx
```

If a critical CVE appears later, the SBOM helps identify affected software releases quickly.

### Key principle

> Security gates should prevent vulnerable artifacts from being promoted to production.

---

## 20. Promotion Pipeline

Promotion means moving the same tested artifact through environments without rebuilding it.

```text
Code Commit
   ↓
Build Image
   ↓
payment-service:abc123
   ↓
DEV
   ↓
QA
   ↓
STAGING
   ↓
PRODUCTION
```

### Build once

Good:

```text
payment-service:abc123
        ↓
DEV
        ↓
QA
        ↓
STAGING
        ↓
PROD
```

Bad:

```text
DEV → rebuild
QA → rebuild
PROD → rebuild
```

Each rebuild could create a slightly different artifact.

### Example GitLab promotion pipeline

```yaml
stages:
  - build
  - deploy_dev
  - deploy_qa
  - deploy_staging
  - deploy_prod

variables:
  IMAGE_TAG: $CI_COMMIT_SHORT_SHA

build_image:
  stage: build
  script:
    - docker build -t "$CI_REGISTRY_IMAGE:$IMAGE_TAG" .
    - docker push "$CI_REGISTRY_IMAGE:$IMAGE_TAG"

deploy_dev:
  stage: deploy_dev
  script:
    - echo "Deploying $IMAGE_TAG to DEV"

deploy_qa:
  stage: deploy_qa
  script:
    - echo "Deploying $IMAGE_TAG to QA"
  needs:
    - deploy_dev

deploy_staging:
  stage: deploy_staging
  script:
    - echo "Deploying $IMAGE_TAG to STAGING"
  needs:
    - deploy_qa

deploy_prod:
  stage: deploy_prod
  script:
    - echo "Deploying $IMAGE_TAG to PROD"
  needs:
    - deploy_staging
  when: manual
```

### Key interview phrase

> Build once, promote the same immutable artifact across environments.

---

## 21. Protected Branches, Tags and Environments

Use protection around sensitive operations.

Examples:

- Only authorized users merge to `main`.
- Only approved users create production release tags.
- Production environment is protected.
- Production secrets are protected variables.
- Production deployment requires approval/manual gate.

---

## 22. Merge Request Pipelines

Use MR pipelines to validate code before merging.

Typical checks:

```text
Merge Request
   ↓
Build
   ↓
Unit test
   ↓
Static analysis
   ↓
Security scan
   ↓
Policy checks
   ↓
Merge allowed
```

---

## 23. Parent/Child Pipelines

Useful when a system has many components.

```text
Parent Pipeline
      │
 ┌────┼─────┐
 ↓    ↓     ↓
API  UI   Database
```

Each component can have its own child pipeline.

Multi-project pipelines can also orchestrate pipelines across separate repositories.

---

## 24. CI vs GitOps CD

A mature Kubernetes architecture separates CI from CD.

### CI

```text
GitLab CI
   ↓
Build
   ↓
Test
   ↓
Scan
   ↓
Package
   ↓
Push image
```

### CD

```text
GitOps Repository
       ↓
ArgoCD / Flux
       ↓
Kubernetes
       ↓
Detect drift and reconcile
```

Full flow:

```text
Developer
 ↓
GitLab
 ↓
CI
 ↓
Container Registry
 ↓
Update GitOps Repository
 ↓
ArgoCD / Flux
 ↓
Kubernetes
```

### Why GitOps?

- Git is source of truth.
- Audit trail.
- Easier rollback.
- Drift detection.
- Safer production access.
- CI runner does not necessarily need direct production cluster credentials.

### Interview phrase

> CI creates the artifact; GitOps handles deployment.

---

## 25. GitLab Registry Lifecycle

Important areas:

- Immutable image tags.
- Versioning.
- Retention policies.
- Cleanup of stale images.
- Vulnerability scanning.
- Cross-region replication where required.

Typical tagging options:

```text
payment-service:2.4.0
payment-service:a81fd23c
```

Commit-SHA tags improve traceability between source and deployed image.

---

## 26. Pipeline Failure Handling

Know these concepts:

- Retry transient jobs.
- `allow_failure` for non-blocking checks where justified.
- Fail fast on mandatory security/test failures.
- Keep deployment rollback strategy.
- Preserve logs and test artifacts for troubleshooting.
- Avoid automatic retry of destructive deployment operations unless safe/idempotent.

---

## 27. GitLab CI Observability

Treat CI infrastructure itself as a platform.

Monitor:

- Runner health.
- Pending job queue time.
- Pipeline duration.
- Job failure rate.
- Runner CPU/memory.
- Build concurrency.
- Artifact upload/download latency.
- Registry latency.
- Cache hit/miss effectiveness.

Principal-level idea:

> CI infrastructure should have its own availability and performance SLOs.

---

## 28. GitLab Runner Overload Troubleshooting

If CI jobs are waiting too long:

1. Check queued/pending job duration.
2. Check runner availability.
3. Check CPU/memory saturation.
4. Check configured concurrency.
5. Check whether jobs are blocked by runner tags.
6. Check Docker image pull times.
7. Check artifact transfer latency.
8. Check cache efficiency.
9. Scale runner pool.
10. Consider Kubernetes executor/autoscaling runners.
11. Separate heavyweight or sensitive workloads onto dedicated runners.

---

## 29. Air-Gapped Deployment Packaging with GitLab

Relevant for environments without Internet access.

Connected environment:

```text
Source
 ↓
GitLab CI
 ↓
Build containers
 ↓
Security scan
 ↓
Generate SBOM
 ↓
Sign artifacts
 ↓
Create release bundle
 ↓
Checksums
 ↓
Offline transfer
```

Air-gapped environment:

```text
Verify signature/checksum
 ↓
Load images into private registry
 ↓
Install Helm charts/manifests
 ↓
Deploy Kubernetes workloads
```

Possible release bundle:

```text
release-4.2.0/
├── images/
├── helm/
├── manifests/
├── dependencies/
├── sbom/
├── checksums/
├── certificates/
├── install.sh
├── upgrade.sh
└── rollback.sh
```

---

## 30. GitLab CI/CD in Multi-Region AWS Deployment

Build the application once:

```text
Developer
   ↓
GitLab
   ↓
Build → Test → Scan
   ↓
Docker Image
   ↓
ECR
```

Replicate image across regions if needed:

```text
ECR Region A
    ↓
Cross-region replication
    ↓
ECR Region B
```

GitOps deployment:

```text
GitLab CI
   ↓
ECR
   ↓
GitOps Repository
   │
   ├── ArgoCD → EKS Region A
   └── ArgoCD → EKS Region B
```

Safer rollout:

```text
Deploy Region B
   ↓
Smoke test / observe
   ↓
Deploy Region A
```

This reduces blast radius compared with deploying every region simultaneously.

---

## 31. Five Core GitLab Ideas to Remember

```text
Runner
   = executes jobs

Reusable Template
   = standardized pipeline logic

Security
   = scan and enforce controls before deployment

Promotion
   = same immutable artifact moves through environments

GitOps
   = Git controls desired deployment state
```

---

## 32. Principal-Level Summary Answer

If asked:

**How would you design GitLab CI/CD for many microservices?**

Answer:

> I would design independent pipelines for each microservice while centralizing common build, test, security and packaging logic through reusable GitLab templates. I would use appropriately scoped runners, preferably autoscaling Kubernetes runners for high-volume workloads, and dedicated runners for sensitive or restricted operations. Each service would build an immutable artifact once, run quality and security gates, publish it to the registry, and promote the same artifact across environments. For Kubernetes deployments I would separate CI from CD and use GitOps with ArgoCD or Flux, with protected environments and approval gates around production.

---

## 33. Rapid Interview Checklist

Before the interview, be able to explain clearly:

- What GitLab Runner does.
- Runner vs executor.
- Shell vs Docker vs Kubernetes executor.
- Where executor is configured (`config.toml`).
- Shared vs dedicated runners.
- Runner tags.
- Reusable templates.
- Pipeline triggers and `rules`.
- Cache vs artifacts.
- Passing artifacts with `needs`.
- Build once / promote same artifact.
- Protected branches/tags/environments.
- CI/CD variables and secrets.
- SAST/DAST/dependency/container scanning.
- SBOM.
- GitLab registry lifecycle.
- Parent/child pipelines.
- CI vs GitOps CD.
- GitOps with ArgoCD/Flux.
- Runner/pipeline observability.
- Air-gapped release packaging.
- Multi-region deployment workflow.
