# PCF / TAS to Kubernetes Migration Guide

> **Purpose:** Interview-ready and implementation-oriented notes for migrating applications from PCF/TAS to Kubernetes without turning this into a huge textbook.
>
> **Scope:** Application team + Platform/Infrastructure team + PCF marketplace/service migration + cutover/rollback + consolidated follow-up Q&A.

---

# 1. End-to-End Migration Flow

```text
                     PCF / TAS
                         |
                         v
             +-----------------------+
             | 1. DISCOVERY          |
             | Apps / Routes /       |
             | Services / Buildpacks |
             | Env / Dependencies    |
             +-----------+-----------+
                         |
                         v
             +-----------------------+
             | 2. ASSESSMENT         |
             | Simple / Medium /     |
             | Complex / Retire      |
             +-----------+-----------+
                         |
               +---------+---------+
               |                   |
               v                   v
      INFRASTRUCTURE TRACK     APPLICATION TRACK
      --------------------     -----------------
      K8s Cluster              Containerize
      Networking               Config / Secrets
      Ingress / DNS            Health probes
      Registry                 Service migration
      IAM/RBAC                 CI/CD
      Observability            K8s manifests/Helm
      Security                 Testing
               |                   |
               +---------+---------+
                         |
                         v
                +----------------+
                | POC / PILOT     |
                +--------+-------+
                         |
                         v
               MIGRATION WAVES
           Dev -> QA -> Perf -> Prod
                         |
                         v
             PCF + K8s coexistence
                         |
                         v
                    CUTOVER
                         |
                         v
                 Stabilization
                         |
                         v
                PCF Decommission
```

## Main principle

PCF migration is **not** simply:

```text
cf push  ->  kubectl apply
```

PCF automatically provides many platform capabilities. In Kubernetes we have to deliberately rebuild or replace them.

---

# 2. Phase 0 - Decide the Target Architecture

Before migrating applications, define the future state.

```text
PCF/TAS
   |
   +--> Target Kubernetes
          |
          +-- EKS / AKS / OpenShift / TKG
          +-- Container Registry
          +-- Ingress / Gateway
          +-- DNS
          +-- Secrets / Vault
          +-- Observability
          +-- CI/CD / GitOps
          +-- Managed Data Services
```

## Checkpoint

Before migration begins, answer:

- Where will workloads run?
- How will users reach applications?
- Where will container images live?
- Where will secrets live?
- How will TLS certificates be handled?
- How will apps be deployed?
- How will logs, metrics and traces be collected?
- How will Redis, RabbitMQ and databases be provided?
- How will RBAC and workload identity work?
- What is the rollback strategy?

---


# 2A. Target Placement and Cluster Strategy Decisions

These questions should be answered **before application migration waves are finalized**.

## Q. Should the workload run on-premises or in public cloud?

Use a workload-by-workload decision, not a single blanket rule.

| Decision Area | Favors On-Premises | Favors Public Cloud |
|---|---|---|
| Data residency / regulation | Strict local or internal hosting requirement | Cloud region/compliance model is acceptable |
| Latency | Heavy dependency on low-latency on-prem systems | Users/dependencies are already cloud-based |
| Legacy dependency | Mainframe, appliance, internal middleware tightly coupled on-prem | Dependencies can be reached reliably over private connectivity |
| Elasticity | Stable/predictable demand | Bursty or rapidly changing demand |
| Managed services | Organization must self-host | Need managed DB, Redis, messaging, observability, etc. |
| Operations | Strong internal DC/platform capability already exists | Want to reduce infrastructure operations |
| DR / multi-region | Existing second DC strategy | Cloud regions/AZs make DR easier to implement |
| Cost model | Existing capacity is already paid for and well utilized | Consumption model and faster provisioning are beneficial |

### Practical migration checkpoint

Ask:

```text
Does the application have data-residency constraints?
        |
        +-- Yes --> Can the target cloud satisfy them?
        |              |
        |              +-- No --> On-prem/private K8s
        |
        +-- No
             |
             v
Does it depend heavily on low-latency on-prem systems?
        |
        +-- Yes --> Prefer on-prem initially OR prove private-link latency
        |
        +-- No
             |
             v
Evaluate cloud cost, managed services, DR, scalability and operating model
```

### Important migration pattern

An application can first move:

```text
PCF on-prem
    |
    v
Kubernetes on-prem
```

and later:

```text
Kubernetes on-prem
    |
    v
EKS / AKS / OpenShift cloud
```

This can reduce risk because **platform migration and data-center/cloud migration are separated**.

---

## Q. Should the workload use a shared cluster or dedicated cluster?

### Shared cluster

```text
Shared Kubernetes Cluster
   |
   +--> Namespace: Team-A
   +--> Namespace: Team-B
   +--> Namespace: Team-C
```

Use shared clusters when workloads have similar:

- Security classification
- Availability requirements
- Network requirements
- Upgrade cadence
- Compliance requirements
- Resource profile

Benefits:

- Better infrastructure utilization
- Lower cost
- Less cluster sprawl
- Easier centralized operations

Controls required:

```text
Namespaces
RBAC
NetworkPolicy
ResourceQuota
LimitRange
Pod Security / admission policies
Workload identity
Separate secrets
Observability boundaries
```

### Dedicated cluster

```text
Application / Business Unit
          |
     Dedicated Cluster
```

Consider a dedicated cluster when there is:

- Strong regulatory or security isolation requirement
- Very high criticality
- Large resource consumption
- Special networking requirements
- Different Kubernetes/version/upgrade lifecycle
- Noisy-neighbor concern
- GPU/specialized nodes
- Strict blast-radius requirement
- Business-unit/platform ownership separation

### Decision rule

Do **not** create a dedicated cluster simply because an application team asks for one.

Use:

```text
Can namespace-level controls provide enough isolation?
        |
        +-- Yes --> Shared cluster
        |
        +-- No --> Dedicated cluster
```

---

## Q. One cluster per environment or multiple environments in one cluster?

For enterprise production platforms, a common model is:

```text
Non-Prod Cluster(s)
   +--> DEV
   +--> QA

Production Cluster(s)
   +--> PROD
```

Avoid putting critical production and development workloads in the same failure domain unless there is a clear reason and strong isolation.

For highly critical platforms:

```text
DEV Cluster
TEST Cluster
PROD Cluster
DR / Secondary PROD Cluster
```

The exact number should depend on scale, security, operational overhead and blast radius.

---

## Q. Should all migrated PCF applications go to the same Kubernetes cluster?

No.

During discovery, applications should be grouped according to:

```text
Business criticality
Security classification
Data sensitivity
Region / residency
Network dependency
Availability requirement
Resource profile
Business domain
Failure isolation requirement
```

Example:

```text
PCF Estate
   |
   +--> Standard apps ---------> Shared K8s Cluster
   |
   +--> Payment workloads -----> Dedicated PCI/Secure Cluster
   |
   +--> Data-heavy workloads --> Separate data/platform cluster
   |
   +--> Low-latency legacy ----> On-prem K8s
   |
   +--> Cloud-native apps -----> EKS / AKS
```

---

## Q. What other target-platform decisions must be answered?

Before migration execution, explicitly decide:

- On-premises vs public cloud vs hybrid
- Region(s) and availability zones
- Shared vs dedicated clusters
- Production vs non-production separation
- Single-region vs multi-region
- Public vs private cluster/API endpoint
- Ingress/load-balancing model
- East-west network model
- Internet egress model
- Connectivity to on-prem databases and services
- Identity/RBAC/workload identity
- Secrets platform
- Container registry
- Logging/metrics/tracing platform
- Backup and DR
- Cluster and node upgrade ownership
- Kubernetes version lifecycle
- Node pool strategy
- Autoscaling strategy
- Tenant/resource quota model
- Cost ownership / chargeback

These decisions form part of the **future-state platform architecture**, not something to decide after applications are already being migrated.

# 3. Phase 1 - PCF Discovery

For every application collect:

```text
Application
├── Application name
├── Business owner
├── Source repository
├── Language / runtime
├── Buildpack
├── Instances
├── Memory / disk
├── Routes
├── Environment variables
├── Bound marketplace services
├── Autoscaling settings
├── Internal dependencies
├── External dependencies
├── Certificates
├── Authentication / SSO
├── Persistent storage
├── Scheduled jobs
├── Logs / monitoring
└── DR / HA requirements
```

Useful discovery areas:

```text
cf apps
cf app <app>
cf env <app>
cf routes
cf services
cf service <service>
cf events <app>
manifest.yml
pipeline configuration
application.yml
bootstrap.yml
VCAP_SERVICES usage
VCAP_APPLICATION usage
```

---

# 4. Create an Application Migration Inventory

Example:

| Application | Runtime | PCF Services | Complexity | Migration Approach |
|---|---|---|---|---|
| customer-ui | NodeJS | None | Simple | Replatform |
| order-api | Spring Boot | Redis | Medium | Replatform |
| payment-api | Java | RabbitMQ + DB | Complex | Refactor |
| legacy-app | Java / WebLogic | DB + NFS | Very Complex | Assess / Rearchitect |
| unused-api | - | - | - | Retire |

## Suggested classification

### Simple
- Stateless
- Modern runtime
- No PCF marketplace dependency
- Containerize and deploy

### Medium
- Redis / DB / simple external dependency
- Small configuration changes
- May require autoscaling / probes

### Complex
- RabbitMQ + DB + SSO + multiple integrations
- Code and configuration changes
- Strong PCF-specific dependencies

### Very Complex
- Legacy runtime
- Shared filesystem
- Multiple tightly coupled services
- Hard-coded platform assumptions
- May need modernization or redesign

---

# 5. Application Team Migration Track

## Step 1 - Remove PCF-specific dependencies

Look for:

```text
VCAP_SERVICES
VCAP_APPLICATION
CF_INSTANCE_INDEX
CF_INSTANCE_IP
CF_INSTANCE_PORTS
PCF-specific service-binding libraries
PCF-specific environment assumptions
```

Replace with platform-neutral configuration.

---

## Step 2 - Containerize the application

```text
Source
  |
  v
Build
  |
  v
Artifact
  |
  v
Dockerfile
  |
  v
Container Image
  |
  v
Registry
```

Important checks:

- Run as non-root
- Minimal base image
- Vulnerability scan
- CPU requests/limits
- Memory requests/limits
- Application port
- Graceful shutdown

---

## Step 3 - Add Kubernetes health probes

```text
startupProbe  -> Can the application start successfully?
readinessProbe -> Can Kubernetes send traffic to this pod?
livenessProbe  -> Is the application still healthy?
```

---

## Step 4 - Externalize application configuration

PCF environment variables become Kubernetes configuration.

```text
ConfigMap
   |
   v
Deployment -> Pod
```

Do not hard-code environment-specific settings inside the image.

---

## Step 5 - Secrets migration

Possible PCF source:

```text
CredHub / PCF Service Binding
```

Possible Kubernetes target:

```text
AWS Secrets Manager
Azure Key Vault
HashiCorp Vault
External Secrets Operator
Kubernetes Secret (where appropriate)
```

Preferred enterprise model:

```text
External Vault
    |
    v
External Secrets / CSI
    |
    v
Pod
```

---

# 6. Infrastructure / Platform Team Migration Track

## Cluster foundation

Prepare:

- Kubernetes control plane
- Worker/node pools
- Multi-AZ design
- Node sizing
- Cluster autoscaling
- Namespace strategy
- Resource quotas
- PodDisruptionBudgets
- Upgrade strategy
- Backup / restore

## Networking

Prepare:

- VPC / VNet
- Public/private subnets
- CNI
- Pod CIDR
- Service CIDR
- Security Groups / NSGs / firewalls
- NetworkPolicy
- Private endpoints
- On-prem connectivity
- Proxy / egress
- DNS

## Ingress traffic path

```text
DNS
 |
 v
WAF / External LB / F5
 |
 v
Ingress Controller / Gateway
 |
 v
Kubernetes Service
 |
 v
Pods
```

## Platform capabilities

```text
Container Registry
Secrets / Vault
Certificate management
IAM / RBAC
Ingress
DNS automation
Observability
Logging
Tracing
Backup
Policy enforcement
GitOps
CI/CD
```

---

# 7. PCF Marketplace Services -> Kubernetes / Cloud Services

| PCF Capability / Service | Typical Target |
|---|---|
| Redis | Managed Redis |
| RabbitMQ | Managed RabbitMQ / RabbitMQ Operator |
| MySQL | RDS / Azure Database / managed DB |
| PostgreSQL | Managed PostgreSQL |
| CredHub | Key Vault / Secrets Manager / Vault |
| PCF Metrics | Prometheus / Grafana / Dynatrace / AppDynamics |
| PCF Logging | Fluent Bit / OpenTelemetry / Splunk |
| Scheduler | Kubernetes CronJob |
| Autoscaler | HPA / KEDA |
| Routes | Ingress / Gateway |
| Service Registry | Kubernetes DNS / external service registry |

---

# 8. Redis Migration

## Existing

```text
PCF App
   |
VCAP_SERVICES
   |
PCF Redis
```

## Target

```text
Kubernetes Pod
      |
      v
Managed Redis
```

## Migration steps

1. Provision target Redis.
2. Establish network connectivity.
3. Configure TLS and authentication.
4. Replace VCAP_SERVICES parsing.
5. Inject Redis endpoint/credentials through secrets/config.
6. Test connectivity.
7. Decide whether data migration is required.
8. Validate TTL/session behaviour.
9. Cut over application traffic.

### Does Redis data need migration?

If Redis is only cache:

```text
Usually no data migration is required.
Provision new Redis and allow cache to warm again.
```

If Redis contains:

- Session state
- Workflow state
- Durable business data
- Queue-like state

then a proper migration/cutover strategy is required.

---

# 9. RabbitMQ Migration

## Existing

```text
Application
    |
VCAP_SERVICES
    |
PCF RabbitMQ
```

## Target

```text
Producer Pods
     |
     v
Managed RabbitMQ / RabbitMQ Cluster
     |
     v
Consumer Pods
```

## Migration checklist

- Create broker/cluster
- Create vhost
- Create users
- Create exchanges
- Create queues
- Create bindings
- Configure DLQ
- Configure TLS
- Configure HA policies
- Validate networking
- Update application connection configuration

## Existing messages

Do not simply switch brokers if old messages are still queued.

Simple drain model:

```text
Old App -> Old RabbitMQ
                |
          Drain queues
                |
        Queue Depth = 0

Deploy new app
      |
      v
New RabbitMQ
```

For complex migrations use capabilities such as:

- Federation
- Shovel
- Dual publish where safe
- Controlled producer/consumer migration

Checkpoint before cutover:

```text
Queues drained?
No important unacked messages?
DLQ checked?
Consumers healthy?
Publisher confirms working?
```

---

# 10. Database Migration

Do not automatically migrate the database at the same time as the application.

A lower-risk pattern is:

```text
PCF App
   |
   v
Existing Database
```

becomes:

```text
Kubernetes App
      |
      v
Existing Database
```

Then database modernization happens separately.

If DB migration is required:

```text
Source DB
   |
CDC / Replication
   |
Target DB
   |
Validation
   |
Application Cutover
```

---

# 11. Other Common Service Migrations

## Shared filesystem / NFS

Kubernetes container filesystem is ephemeral.

Possible targets:

```text
Object Storage
EFS / Azure Files
PersistentVolume
Database
```

## Scheduled jobs

```text
PCF Scheduler
     |
     v
Kubernetes CronJob
```

## Autoscaling

```text
PCF Autoscaler
     |
     +--> HPA
     +--> KEDA
     +--> Cluster Autoscaler
```

KEDA is useful for event-driven workloads, e.g. RabbitMQ queue depth.

---

# 12. CI/CD Migration

Old pattern:

```text
Git
 |
Jenkins / Concourse
 |
cf push
```

Target pattern:

```text
Git
 |
Build
 |
Unit Test
 |
SAST
 |
Container Build
 |
Image Scan
 |
Registry
 |
Helm / Kustomize
 |
GitOps Repo
 |
ArgoCD / Flux
 |
Kubernetes
```

---

# 13. Move2Kube - Where It Helps

Move2Kube can help with:

```text
CF runtime metadata
      |
      v
   COLLECT
      |
Source + manifest.yml
      |
      v
     PLAN
      |
      v
  TRANSFORM
      |
      v
Dockerfile + Kubernetes YAML
```

It can accelerate conversion of Cloud Foundry source/manifest/runtime metadata into candidate Kubernetes artifacts.

## Important

Treat generated artifacts as a **starting point**, not production-ready output.

Review:

- Security
- Resource limits
- Health probes
- Secret handling
- Network policy
- Service migration
- Observability
- HA / DR
- Image hardening
- Production deployment standards

---

# 14. POC Before Factory Migration

Select at least:

```text
1 Simple app

1 Medium app
   +
 Redis

1 Complex app
   +
 RabbitMQ
   +
 Database
   +
 External API
```

The POC should prove:

- Build
- Containerization
- Deployment
- Networking
- DNS
- Secrets
- Service connectivity
- Monitoring
- Autoscaling
- Security
- Rollback
- Operational support

---

# 15. Migration Factory

After POC, standardize reusable templates:

```text
Dockerfile
Helm chart
Deployment
Service
Ingress
ConfigMap
Secret integration
HPA
NetworkPolicy
PodDisruptionBudget
ServiceAccount
Logging
Monitoring
CI/CD
ArgoCD Application
```

Migration factory:

```text
App Source
   |
Assessment
   |
Template Generation
   |
App Modifications
   |
Pipeline
   |
DEV -> QA -> PERF -> PROD
```

---

# 16. Migration Waves

Example:

```text
Wave 0 - Platform POC
Wave 1 - Stateless/simple applications
Wave 2 - Apps using Redis
Wave 3 - Apps using database
Wave 4 - RabbitMQ/event-driven apps
Wave 5 - Complex/legacy workloads
```

Migrate logical application families together where possible.

---

# 17. Testing Before Production

Every migrated application should pass:

- Functional testing
- Integration testing
- Performance testing
- Security testing
- Resilience testing
- Failover testing
- Autoscaling testing
- Observability validation
- Backup/restore validation
- DR validation

Compare PCF and Kubernetes baselines:

```text
Latency
Throughput
Memory
CPU
Error rate
Response time
Connection pool usage
```

---

# 18. Production Cutover Methods

There are two main approaches.

---

## Method A - DNS Cutover

### Before

```text
app.company.com
      |
      | A record
      v
PCF Load Balancer
      |
   Gorouter
      |
   PCF App
```

### Kubernetes prepared separately

```text
app-k8s.company.com
       |
       v
K8s Load Balancer
       |
    Ingress
       |
    Service
       |
     Pods
```

### Cutover steps

1. Keep PCF running.
2. Deploy Kubernetes app.
3. Validate Kubernetes using temporary DNS.
4. Reduce DNS TTL in advance, e.g. 300 seconds.
5. Smoke/performance test.
6. Change application A record from PCF VIP to K8s VIP.
7. Monitor closely.
8. Keep PCF for rollback.
9. Retire PCF only after stabilization.

Rollback:

```text
app.company.com
     |
K8s VIP has issue
     |
Change DNS back
     |
PCF VIP
```

---

# 19. Load Balancer Cutover - Preferred for Controlled Migration

Instead of repeatedly changing DNS, keep the same public application DNS and VIP.

```text
                     app.company.com
                           |
                       DNS A record
                           |
                           v
                  Common F5 / Load Balancer
                     /               \
                    /                 \
                   v                   v
          PCF Backend Pool      K8s Backend Pool
          ----------------      ----------------
          Gorouter VM 1         K8s Ingress endpoint
          Gorouter VM 2               |
          Gorouter VM 3           Ingress Controller
                 |                     |
                 v                     v
              PCF App               Service
                                      |
                                     Pods
```

## Traffic shift

```text
Stage 1
PCF 100%
K8s   0%

Stage 2
PCF 90%
K8s 10%

Stage 3
PCF 50%
K8s 50%

Stage 4
PCF  0%
K8s 100%
```

Rollback is simply:

```text
PCF 100%
K8s   0%
```

No DNS TTL wait.

---

# 20. Important Point About *.apps VIP

Suppose PCF currently has:

```text
*.apps.company.com
       |
       v
     F5 VIP
       |
    Gorouters
```

Do **not** blindly switch the entire wildcard VIP if hundreds of applications still run on PCF.

A better pattern is L7 host-based routing:

```text
                       *.apps.company.com
                              |
                             F5
                              |
             +----------------+----------------+
             |                |                |
             v                v                v

app1.apps.company.com  app2.apps.company.com  app3.apps.company.com
        |                      |                      |
       PCF                  Migration                K8s
        |                  /        \                |
    Gorouter           Gorouter   K8s Ingress    K8s Ingress
```

This allows application-by-application migration.

---

# 21. Backend Pool - Definition

A **backend pool** is simply the group of servers/endpoints behind a load balancer that can receive traffic.

```text
Client
  |
  v
Load Balancer
  |
  v
Backend Pool
  |
  +--> Server 1
  +--> Server 2
  +--> Server 3
```

PCF example:

```text
*.apps.company.com
       |
       v
     F5 VIP
       |
       v
  PCF Backend Pool
       |
       +--> Gorouter VM1
       +--> Gorouter VM2
       +--> Gorouter VM3
```

Kubernetes example:

```text
F5 VIP
  |
  +--> PCF Pool
  |      +--> Gorouter1
  |      +--> Gorouter2
  |
  +--> K8s Pool
         +--> Ingress endpoint 1
         +--> Ingress endpoint 2
```

Terminology:

```text
F5                     -> Pool
Azure LB/App Gateway   -> Backend Pool
AWS ALB/NLB            -> Target Group
```

Mental model:

```text
VIP          = Front door
Backend Pool = Destinations behind the front door
```

---

# 22. Can the Same DNS Name Point to Different IPs?

Yes.

Example:

```text
app.company.com  A  10.10.10.100
app.company.com  A  10.20.20.100
```

But plain DNS does not provide precise migration control because clients/resolvers may use either IP and cache responses.

For controlled migration use:

```text
Same DNS name
   |
   +--> Weighted DNS
   |
   or
   |
   +--> Common Load Balancer
          |
          +--> PCF
          +--> Kubernetes
```

---

# 23. Should Gorouter VMs and K8s Nodes Be Put in the Same Pool?

Normally **no**.

Do not model it as:

```text
*.apps VIP
   |
Same Pool
   |
   +--> Gorouter VMs
   +--> Raw K8s Nodes
```

Better:

```text
*.apps VIP
   |
F5 / Common LB
   |
   +--> PCF Pool
   |      |
   |   Gorouter VMs
   |
   +--> K8s Pool
          |
       Ingress endpoint
          |
       Services
          |
         Pods
```

The Kubernetes equivalent of the PCF Gorouter function is the **Ingress Controller/Gateway**, not the worker node itself.

Technically F5 can target K8s nodes if ingress is exposed via NodePort:

```text
F5
 |
 +--> worker1:32443
 +--> worker2:32443
 +--> worker3:32443
          |
      Ingress Controller
```

But in architecture discussions it is clearer to say:

> F5 sends traffic to the Kubernetes ingress endpoints.

---

# 24. Same Hostname on PCF and Kubernetes

Both platforms can be configured to serve the same application hostname during migration.

PCF:

```text
Route:
app.company.com
```

Kubernetes:

```yaml
spec:
  rules:
  - host: app.company.com
```

Then:

```text
Host: app.company.com
        |
        v
Common Load Balancer
      /       \
     /         \
PCF Gorouter   K8s Ingress
     |             |
  PCF App        Service
                   |
                  Pods
```

The LB decides which platform receives the request.

---

# 25. Health Checks During Cutover

The LB should check both platforms.

```text
Common LB
   |
   +---- /health ---> PCF
   |
   +---- /health ---> Kubernetes
```

If Kubernetes becomes unhealthy:

```text
K8s health check FAIL
        |
        v
Remove / disable K8s backend
        |
        v
PCF continues serving traffic
```

---

# 26. Session Handling During Dual-Run

If the application stores local session state, this can break:

```text
Request 1 -> PCF
Request 2 -> K8s
Request 3 -> PCF
```

Prefer shared external session state:

```text
PCF App ----\
             +--> Shared Redis
K8s App ----/
```

Sticky sessions can be used temporarily, but shared stateless/session architecture is better.

---

# 27. Shared Database / Messaging During Application Cutover

To reduce migration complexity, keep application cutover separate from data-service migration when possible.

Example:

```text
PCF App ----\
             +--> Same Database
K8s App ----/
```

And where application/message semantics allow:

```text
PCF App ----\
             +--> Same RabbitMQ
K8s App ----/
```

This avoids migrating application, database and messaging all at exactly the same moment.

---

# 28. TLS Options with Common Load Balancer

### TLS termination / re-encryption

```text
Client
  |
HTTPS
  |
Common LB
  |
HTTPS
  |
PCF / Kubernetes
```

### TLS passthrough

```text
Client
  |
HTTPS
  |
Common LB
  |
TLS passthrough
  |
Gorouter / Ingress
```

Choose based on enterprise security and certificate ownership.

---

# 29. Final Migration Completion Checklist

An application is **not migrated** merely because:

```text
kubectl get pods
Running
```

Complete only after:

- [ ] Container image created
- [ ] Kubernetes manifests/Helm created
- [ ] Configuration externalized
- [ ] Secrets externalized
- [ ] PCF service dependencies replaced
- [ ] Network connectivity validated
- [ ] DNS configured
- [ ] TLS configured
- [ ] Health probes configured
- [ ] Resource requests/limits configured
- [ ] Autoscaling configured
- [ ] Logging configured
- [ ] Metrics configured
- [ ] Alerting configured
- [ ] CI/CD configured
- [ ] Backup/restore verified
- [ ] Performance tested
- [ ] Security tested
- [ ] Rollback tested
- [ ] Production cutover completed
- [ ] Stabilization period completed

Then:

```text
Unbind PCF services
      |
Stop PCF application
      |
Observe
      |
Delete PCF application
      |
Delete unused service instances
      |
Remove routes
      |
Decommission PCF capacity
```

---

# 30. Consolidated Interview Q&A

## Q1. How would you migrate applications from PCF to Kubernetes?

**Answer:**

We first discover the PCF estate including applications, buildpacks, routes, environment variables, service bindings, dependencies, sizing and availability requirements. We classify applications by complexity and decide whether each workload can be replatformed, needs refactoring or should be retired.

In parallel, the platform team builds the Kubernetes landing zone including networking, ingress, DNS, IAM/RBAC, registry, secrets, observability, CI/CD and managed backing services.

The application team removes PCF-specific dependencies such as VCAP service bindings, containerizes the application, externalizes configuration and secrets, adds health probes and resource settings, and creates Helm/Kubernetes deployment artifacts.

PCF marketplace services are migrated separately, such as Redis to managed Redis, RabbitMQ to a managed/operator-based platform, CredHub to Vault/Key Vault/Secrets Manager, scheduler jobs to CronJobs and PCF autoscaling to HPA/KEDA.

We prove the patterns with simple, medium and complex POCs, create reusable templates, then migrate applications in waves. PCF and Kubernetes coexist during production cutover so traffic can be shifted gradually and rolled back if required.

---

## Q2. Can the same DNS name point to multiple IPs?

Yes.

```text
app.company.com -> 10.10.10.100
app.company.com -> 10.20.20.100
```

However plain multi-A-record DNS is not ideal for controlled migration because traffic distribution and caching are not precise.

Prefer weighted DNS or a common load balancer.

---

## Q3. Can PCF and Kubernetes use the same application DNS name?

Yes.

Both can be configured for:

```text
app.company.com
```

The common load balancer determines whether the request is sent to PCF Gorouter or Kubernetes Ingress.

---

## Q4. Can I keep the existing *.apps VIP?

Yes, if the existing load balancer/F5 remains the common front door.

```text
*.apps.company.com
        |
      F5 VIP
        |
   +----+----+
   |         |
PCF Pool   K8s Pool
```

The DNS and VIP can remain unchanged while routing policies move traffic platform by platform or application by application.

---

## Q5. Should I add Kubernetes worker nodes to the same pool as Gorouters?

Normally no.

Use separate backend pools:

```text
PCF Pool -> Gorouters
K8s Pool -> Ingress endpoints
```

Raw K8s nodes are only direct targets when the chosen architecture exposes ingress through NodePort or a similar model.

---

## Q6. What is a backend pool?

A backend pool is the set of backend endpoints behind a load balancer.

```text
VIP = Front door

Backend Pool = Where traffic is forwarded
```

For PCF, backend members are typically Gorouter VMs.

For Kubernetes, backend members should normally represent ingress endpoints.

---

## Q7. How is load-balancer-based cutover performed?

```text
100% PCF / 0% K8s
      |
90% PCF / 10% K8s
      |
50% PCF / 50% K8s
      |
0% PCF / 100% K8s
```

Monitor health, errors, latency, DB, Redis and RabbitMQ throughout.

Rollback simply sends 100% back to PCF.

---

## Q8. Why is load-balancer cutover better than DNS cutover?

Advantages:

- No DNS TTL waiting
- Faster rollback
- Controlled percentage-based migration
- Same application URL
- Easier canary validation
- PCF and Kubernetes can run in parallel

DNS cutover is simpler, but LB-based cutover gives more traffic control.

---

## Q9. What happens after all applications migrate?

Once every application behind the PCF route/domain has moved and the stabilization period is complete:

```text
*.apps.company.com
       |
      F5
       |
       v
Kubernetes Ingress
       |
    Services
       |
      Pods
```

Then PCF Gorouter backends/pools can be removed and the old PCF application/service capacity can be decommissioned.

---

# 31. Short Interview Pitch

> We start with PCF discovery and dependency mapping, then classify applications by migration complexity. In parallel, the platform team builds the Kubernetes landing zone including networking, ingress, DNS, IAM, secrets, observability, CI/CD and managed backing services. Application teams containerize workloads, remove VCAP/PCF dependencies, externalize configuration and secrets, add probes and resource controls, and migrate marketplace services such as Redis and RabbitMQ. We prove the approach with simple, medium and complex POCs, standardize templates and then migrate in waves. For production cutover, we can retain the existing F5/VIP, keep PCF Gorouters and Kubernetes ingress endpoints in separate backend pools, and gradually shift traffic from PCF to Kubernetes while preserving the same application hostname. Rollback simply shifts traffic back to PCF until the application is stable.
