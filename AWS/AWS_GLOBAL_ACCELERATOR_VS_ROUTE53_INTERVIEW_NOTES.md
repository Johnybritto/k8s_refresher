# AWS Global Accelerator vs Route 53 — Interview Notes

## 1. What Is AWS Global Accelerator?

AWS Global Accelerator is a global traffic-routing service that provides static anycast IP addresses and directs user traffic over the AWS global network to healthy application endpoints.

Typical endpoints include:
- Application Load Balancer (ALB)
- Network Load Balancer (NLB)
- EC2 instances
- Elastic IP addresses

High-level flow:

```text
Users
  |
  v
Global Accelerator
Static Anycast IPs
  |
  +-----------> Region A ALB
  |
  +-----------> Region B ALB
```

The user connects to the same Global Accelerator IPs, while AWS decides which healthy regional endpoint should receive the traffic.

---

# 2. Route 53 vs Global Accelerator

The most important distinction:

```text
Route 53
= DNS-based routing

Global Accelerator
= Network-level traffic steering
```

| Route 53 | Global Accelerator |
|---|---|
| DNS service | Global network traffic service |
| Returns an endpoint/IP through DNS | Uses static anycast IPs |
| DNS TTL and caching influence failover | Does not depend on DNS propagation for backend switching |
| Supports many DNS routing policies | Focuses on fast endpoint steering |
| Works with AWS and non-AWS destinations | Primarily for supported AWS regional endpoints |
| Useful for public/private DNS | Not a DNS hosting service |

Route 53 supports routing policies such as:
- Simple
- Weighted
- Failover
- Latency-based
- Geolocation
- Geoproximity
- Multivalue

Global Accelerator supports:
- Health-based endpoint selection
- Static anycast IPs
- Endpoint weights
- Regional traffic dials
- Traffic over AWS global backbone

---

# 3. Why Global Accelerator Failover Can Be Faster

## Route 53

```text
Client
  |
DNS Query
  |
Route 53
  |
Region-A ALB
```

If Region A fails:

```text
Health Check Fails
      |
Route 53 Changes DNS Answer
      |
Old DNS May Still Be Cached
      |
TTL Expires
      |
Client Resolves Region B
```

DNS caching can affect how quickly all clients see the new destination.

## Global Accelerator

```text
Client
  |
Static Accelerator IP
  |
AWS Edge
  |
Region-A ALB
```

If Region A fails:

```text
Same Destination IP
      |
Global Accelerator
      |
Changes Backend Routing
      |
Region-B ALB
```

The client keeps using the same accelerator IP, while AWS changes which healthy backend receives the traffic.

---

# 4. Multi-Region EKS with Global Accelerator

Instead of:

```text
User
 |
Route 53 Failover
 |
+---- Region A ALB
|
+---- Region B ALB
```

you can use:

```text
                     Route 53
                        |
                apps.example.com
                        |
                        v
               Global Accelerator
                Static Anycast IPs
                   /          \
                  /            \
                 v              v
          Region A ALB      Region B ALB
                |                |
              EKS-A            EKS-B
```

Route 53 still provides the friendly DNS name:

```text
apps.example.com
```

Global Accelerator performs the global traffic steering.

---

# 5. Main Use Cases

## Use Case 1 — Faster Multi-Region Failover

Very relevant for EKS:

```text
Global Accelerator
      |
   ---------
   |       |
ALB-A    ALB-B
 |         |
EKS-A    EKS-B
```

If Region A becomes unhealthy, Global Accelerator can steer traffic to Region B.

---

## Use Case 2 — Static Public IPs

Useful when customers or firewalls require allow-listing.

```text
Customer Firewall
Allow:
Static IP-1
Static IP-2
```

Global Accelerator provides stable anycast entry IPs.

This is especially useful when your underlying endpoints are ALBs whose backing IPs can change.

---

## Use Case 3 — Global Performance

```text
User
 |
Nearest AWS Edge
 |
AWS Global Backbone
 |
Regional Application
```

Traffic enters AWS's network closer to the client instead of traversing the public internet for most of the path.

---

## Use Case 4 — TCP / UDP Applications

Global Accelerator works well beyond only HTTP/HTTPS use cases.

Examples:
- Gaming
- Voice / real-time applications
- TCP services
- UDP services
- Low-latency applications

---

## Use Case 5 — Controlled Regional Traffic Shift

Global Accelerator supports endpoint weighting and regional traffic dials.

Example:

```text
Region A = 90%
Region B = 10%
```

Then:

```text
Region A = 50%
Region B = 50%
```

Finally:

```text
Region A = 0%
Region B = 100%
```

Useful for:
- Regional migration
- Blue/green cutover
- DR testing
- Gradual traffic movement

---

# 6. When to Prefer Route 53

Use Route 53 when you mainly need:
- DNS hosting
- DNS failover
- Weighted routing
- Latency routing
- Geolocation routing
- Private DNS
- Routing to AWS and non-AWS destinations
- Simpler / lower-cost DNS-based traffic management

Example:

```text
Europe Users -> Frankfurt
India Users  -> Mumbai
US Users     -> Virginia
```

Route 53 is especially useful when the routing policy itself is DNS-driven.

---

# 7. When to Prefer Global Accelerator

Use Global Accelerator when you need:
- Faster regional failover
- Static public IPs
- Global network performance
- TCP/UDP acceleration
- Stable endpoint addressing
- Controlled traffic movement between regions

---

# 8. Can Route 53 and Global Accelerator Be Used Together?

Yes.

This is a common enterprise pattern:

```text
                 Route 53
                    |
             apps.example.com
                    |
                    v
            Global Accelerator
              /           \
             /             \
        Region A         Region B
           ALB              ALB
            |                |
           EKS              EKS
```

Route 53:

```text
DNS Resolution
apps.example.com
```

Global Accelerator:

```text
Health-based Routing
Fast Regional Failover
Static IPs
Global Network Optimization
```

---

# 9. Global Accelerator Does Not Solve Application State

Global Accelerator can move traffic:

```text
Region A
   |
   v
Region B
```

But it does not automatically handle:
- Database promotion
- S3 replication
- Queue replication
- Redis/cache
- User sessions
- EKS capacity
- External dependencies

So traffic failover must still be coordinated with DR of stateful components.

---

# 10. Route 53 vs Global Accelerator — Quick Interview Table

| Requirement | Route 53 | Global Accelerator |
|---|---|---|
| DNS hosting | Yes | No |
| Private DNS | Yes | No |
| Static global IPs | No | Yes |
| DNS failover | Yes | Not DNS-based |
| Network-level failover | No | Yes |
| Latency routing | Yes | Yes, via accelerator endpoint selection |
| Geolocation routing | Yes | No direct equivalent |
| Weighted DNS routing | Yes | No, but endpoint weights / traffic dials exist |
| TCP/UDP optimization | Limited to DNS decision | Yes |
| AWS backbone acceleration | No | Yes |
| Multi-region ALB failover | Yes | Yes |
| Non-AWS destinations | Yes | Limited / endpoint-type dependent |

---

# 11. Interview-Ready Answer

> “Route 53 is primarily a DNS and DNS-routing service, while AWS Global Accelerator is a network-level global traffic-steering service. Global Accelerator provides static anycast IP addresses, brings traffic onto the AWS global network close to the user, evaluates endpoint health and can direct traffic between regional ALBs, NLBs, EC2 instances or Elastic IPs. I would use Route 53 when I need DNS policies such as weighted, latency, geolocation or failover routing. I would use Global Accelerator when I need faster regional failover, static IPs, improved global network performance or TCP/UDP traffic. In a multi-region EKS design, Route 53 can resolve the application hostname to Global Accelerator, and Global Accelerator can then route traffic to the healthy regional ALB.”
