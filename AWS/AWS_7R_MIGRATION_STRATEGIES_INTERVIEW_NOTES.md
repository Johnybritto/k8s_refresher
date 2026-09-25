# AWS 7 Rs Migration Strategies — Interview Notes

The AWS **7 Rs** are a framework for deciding **how each application should move to the cloud**.  
The key interview point is that not every workload should be migrated in the same way.

---

## 1. Rehost — Lift and Shift

### What it means

Move the application largely **as-is** from the existing environment to AWS with minimal code or architecture changes.

### Typical example

```text
On-Prem VM
   |
   v
Amazon EC2
```

### Common AWS components

- Amazon EC2
- EBS
- VPC / Subnets
- Elastic Load Balancer
- Route 53
- CloudWatch
- AWS Application Migration Service

### When to use

- Fast data-center exit
- Stable legacy application
- Minimal change is preferred
- Tight migration deadline

### Advantages

- Fastest migration approach
- Low application-change risk
- Minimal retraining initially
- Good first step before later modernization

### Disadvantages

- Carries existing technical debt into AWS
- May not fully use cloud-native capabilities
- Can remain expensive if oversized VMs are moved directly
- Operational model may remain similar to on-prem
- Scalability and resilience may not improve significantly

### Interview answer

> “Rehost means moving the application as-is to AWS, usually from a VM to EC2, with minimal changes. It is fast and low risk, but the trade-off is that we also carry existing technical debt and may not get the full cloud benefit.”

---

## 2. Replatform — Lift, Tinker, and Shift

### What it means

Move the application while making **limited optimizations** without fundamentally redesigning it.

### Typical example

```text
Before:
App VM + DB VM

After:
EC2 App
   |
   v
Amazon RDS
```

Another example:

```text
VM-hosted application
        |
        v
Containerized app on ECS/EKS
```

### Common AWS components

- EC2
- ECS / EKS
- RDS / Aurora
- S3
- ElastiCache
- Elastic Load Balancer
- CloudWatch

### When to use

- Want some cloud benefits
- Full rewrite is not justified
- Reduce operational overhead
- Replace self-managed services with managed services

### Advantages

- Better cloud efficiency than rehosting
- Reduced management overhead
- Faster than a full refactor
- Can improve HA, backups, patching, and scalability

### Disadvantages

- Still retains much of the original architecture
- Some application changes and testing are required
- Can introduce new service dependencies
- May create an intermediate architecture that needs another modernization later

### Interview answer

> “Replatforming keeps the core application architecture mostly intact but replaces selected components with managed AWS services, for example moving a self-managed database to RDS. It gives more cloud benefit than rehosting but still does not fully modernize the application.”

---

## 3. Refactor / Re-architect

### What it means

Redesign the application to use cloud-native architecture and services.

### Typical example

```text
Before:
Monolithic Application
        |
        v
Single Database

After:
API Gateway
     |
Microservices
 |    |    |
EKS  ECS  Lambda
     |
Aurora / DynamoDB / SQS / EventBridge
```

### Common AWS components

- EKS / ECS
- Lambda
- API Gateway
- SQS / SNS
- EventBridge
- Step Functions
- Aurora
- DynamoDB
- ElastiCache
- S3
- CloudFront

### When to use

- Need high scalability
- Need better resilience
- Reduce technical debt
- Move from monolith to microservices
- Adopt event-driven or serverless architecture

### Advantages

- Maximum cloud-native benefit
- Better scalability and resilience
- Faster deployment potential
- Can reduce infrastructure management
- Easier to adopt automation and modern engineering practices

### Disadvantages

- Highest migration effort
- Highest development and testing cost
- Longer migration timeline
- Greater architectural complexity
- Requires strong application, platform, and cloud skills
- Higher migration risk if poorly planned

### Interview answer

> “Refactoring means redesigning the application to use cloud-native patterns such as microservices, EKS, Lambda, API Gateway, and managed databases. It provides the biggest long-term benefits, but it also has the highest cost, effort, and migration risk.”

---

## 4. Repurchase — Drop and Shop

### What it means

Replace the existing application with a commercial product or SaaS solution instead of migrating the existing software.

### Typical example

```text
Custom CRM
   |
   X
   |
Salesforce / SaaS CRM
```

### Common technologies

- SaaS products
- AWS Marketplace
- Vendor-hosted solutions
- Commercial off-the-shelf software

### When to use

- Existing application is outdated
- SaaS already meets the requirement
- Maintaining custom software is expensive
- Business wants to standardize on a commercial product

### Advantages

- Removes application maintenance burden
- Can reduce infrastructure operations
- Faster access to new features
- Vendor handles much of the platform lifecycle

### Disadvantages

- Vendor lock-in
- Subscription/licensing costs
- Limited customization
- Data migration can be difficult
- Integration with existing enterprise systems may be complex
- Business processes may need to change to fit the product

### Interview answer

> “Repurchase means replacing the existing application with a SaaS or commercial product. It can remove a lot of maintenance overhead, but the trade-offs are vendor lock-in, licensing cost, customization limits, and data migration complexity.”

---

## 5. Relocate

### What it means

Move an entire existing platform or virtualized environment with minimal workload-level changes.

### Typical example

```text
VMware On-Prem
      |
      v
VMware Cloud on AWS
```

### Common components / technologies

- VMware Cloud on AWS
- VMware HCX
- Direct Connect
- Site-to-Site VPN
- Existing VMware tooling

### When to use

- Large VMware estate
- Fast data-center exit
- Application changes are difficult
- Want to move the platform first and modernize later

### Advantages

- Very little application change
- Fast for large virtualized estates
- Existing operational skills can be reused
- Lower application migration risk

### Disadvantages

- Can be expensive
- Keeps legacy platform architecture
- Limited cloud-native benefit
- May simply move the same operational complexity into AWS
- Usually requires later modernization for better cloud economics

### Interview answer

> “Relocate is used when we move the underlying platform rather than redesign individual applications, for example VMware workloads to VMware Cloud on AWS. It minimizes application changes, but we keep much of the legacy architecture and may not gain the full cloud-native benefit.”

---

## 6. Retain

### What it means

Keep the application in the existing environment for now.

### Common reasons

- Compliance restrictions
- Licensing limitations
- Hardware dependencies
- Low business priority
- Migration cost is not justified
- Application depends heavily on systems that remain on-prem

### Common supporting AWS components

- Direct Connect
- Site-to-Site VPN
- Transit Gateway
- Route 53 Resolver
- Hybrid DNS

### Typical architecture

```text
On-Prem Application
        |
        | Direct Connect / VPN
        v
AWS Workloads
```

### Advantages

- Avoids unnecessary migration risk
- Can be the right decision for specialized or regulated workloads
- Allows migration program to focus on higher-value applications

### Disadvantages

- Continues on-prem infrastructure and support costs
- Requires hybrid networking and operations
- Can create latency and dependency between cloud and on-prem systems
- Legacy technology may remain longer than desired

### Interview answer

> “Retain means making a deliberate decision to keep an application where it is, usually because of compliance, dependencies, cost, or business priorities. The disadvantage is that we continue maintaining hybrid infrastructure and the associated operational complexity.”

---

## 7. Retire

### What it means

Decommission applications that are no longer needed instead of migrating them.

### Typical activities

- Confirm no active users or dependent systems
- Archive required business data
- Remove DNS entries
- Decommission servers
- Remove load balancers
- Stop backups
- Terminate licenses
- Update CMDB and documentation

### Common AWS components

- S3
- S3 Glacier
- AWS Backup
- CloudTrail / CloudWatch for validation

### Typical flow

```text
Legacy Application
       |
Usage / Dependency Analysis
       |
No Business Need
       |
Archive Required Data
       |
Decommission
```

### Advantages

- Removes unnecessary infrastructure cost
- Reduces security exposure
- Reduces maintenance and support effort
- Simplifies the application portfolio

### Disadvantages

- Risk of retiring an application with hidden dependencies
- Data retention requirements must be handled carefully
- Restoring an incorrectly retired application can be difficult
- Requires strong dependency and business validation

### Interview answer

> “Retire means identifying applications that no longer provide business value and safely decommissioning them instead of migrating them. The main risk is hidden dependencies, so dependency analysis and data-retention validation are critical before shutdown.”

---

# Quick Comparison

| Strategy | Meaning | Typical AWS Approach | Main Benefit | Main Disadvantage |
|---|---|---|---|---|
| Rehost | Move as-is | EC2, EBS, MGN | Fast migration | Carries technical debt |
| Replatform | Minor optimization | EC2 + RDS, ECS/EKS | Better cloud efficiency | Still partly legacy |
| Refactor | Redesign | EKS, Lambda, API GW, DynamoDB | Maximum modernization | High cost/effort |
| Repurchase | Replace | SaaS / Marketplace | Less maintenance | Vendor lock-in |
| Relocate | Move platform | VMware Cloud on AWS | Minimal app change | Limited cloud-native benefit |
| Retain | Keep where it is | VPN / DX / TGW | Avoid unnecessary migration | Hybrid complexity |
| Retire | Decommission | S3/Glacier archive | Removes cost | Hidden dependency risk |

---

# Strong Interview Summary

> “I would first perform application discovery and dependency analysis, then classify each workload using the 7 Rs. Stable legacy workloads may be rehosted to EC2, selected components may be replatformed to services such as RDS, strategic applications may be refactored onto EKS or serverless services, commercial applications may be repurchased as SaaS, VMware estates may be relocated, applications with constraints may be retained, and systems with no business value should be retired. The decision is based on business value, migration risk, technical complexity, cost, compliance, dependencies, and the amount of cloud-native benefit we want to achieve.”
