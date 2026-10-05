# EC2 Access Methods — Interview Notes

## 1. Different Ways to Connect to EC2

EC2 connectivity depends on whether the requirement is **administrative access** or **application/network access**.

---

## 2. SSH — Linux EC2

Classic administrative access method.

~~~text
Laptop
  |
Internet
  |
Public IP / Elastic IP
  |
Security Group TCP 22
  |
EC2 Linux
~~~

Example:

~~~bash
ssh -i mykey.pem ec2-user@<public-ip>
~~~

For production, avoid exposing SSH broadly to the internet.

---

## 3. RDP — Windows EC2

Used for Windows administration.

~~~text
Laptop
  |
RDP TCP 3389
  |
Windows EC2
~~~

Access should normally be restricted to trusted sources or provided through private connectivity.

---

## 4. AWS Systems Manager Session Manager — Preferred Enterprise Method

Session Manager allows administrators to access EC2 without exposing SSH/RDP ports.

~~~text
Engineer
   |
Corporate SSO / IAM
   |
AWS Systems Manager
   |
SSM Agent
   |
Private EC2
~~~

Advantages:

- No public IP required
- No inbound port 22/3389 required
- No shared PEM files
- IAM/RBAC-based access
- Individual user accountability
- Auditable sessions
- Works well for private subnets

The EC2 instance needs an IAM role with Systems Manager permissions, commonly through:

~~~text
AmazonSSMManagedInstanceCore
~~~

The instance also needs network connectivity to Systems Manager endpoints through NAT or Interface VPC Endpoints.

### Interview answer

> For production private EC2 instances, I prefer Systems Manager Session Manager because it avoids public IPs and inbound SSH/RDP ports and provides IAM-based, auditable access.

---

## 5. EC2 Instance Connect

EC2 Instance Connect can push a temporary SSH public key to the instance.

~~~text
Engineer
   |
EC2 Instance Connect
   |
Temporary SSH Public Key
   |
EC2
~~~

This avoids permanently distributing the same SSH key to many users.

---

## 6. EC2 Instance Connect Endpoint

Useful for private EC2 instances without a public IP or bastion host.

~~~text
Engineer
   |
EC2 Instance Connect
   |
EC2 Instance Connect Endpoint
   |
Private VPC
   |
Private EC2
~~~

This provides private administrative connectivity to the instance.

---

## 7. Bastion / Jump Host

Traditional private-subnet access pattern.

~~~text
Engineer
   |
SSH
   |
Bastion Host
Public Subnet
   |
SSH
   |
Private EC2
~~~

Private EC2 Security Group can allow SSH only from the bastion Security Group.

~~~text
TCP 22
Source = Bastion Security Group
~~~

Session Manager is often preferable where available because it reduces bastion-host management.

---

## 8. VPN

### Client VPN

~~~text
Engineer Laptop
      |
AWS Client VPN
      |
VPC
      |
Private EC2
~~~

### Site-to-Site VPN

~~~text
Corporate Network
      |
Site-to-Site VPN
      |
AWS VPC
      |
Private EC2
~~~

The engineer/application can then access the EC2 private IP if routing and security rules allow it.

---

## 9. Direct Connect

Used for dedicated enterprise connectivity between on-premises networks and AWS.

~~~text
On-Prem Data Center
        |
AWS Direct Connect
        |
       VPC
        |
   Private EC2
~~~

Direct Connect is primarily hybrid network connectivity rather than an individual login mechanism.

---

## 10. Application Access Through Load Balancer

Applications and users should normally not connect directly to EC2 instances.

~~~text
User / Application
       |
Route 53
       |
ALB / NLB
       |
Target Group
       |
EC2-A  EC2-B  EC2-C
~~~

This provides a scalable and highly available application-access pattern.

---

# 11. Using PEM Files in a Large Organization

A shared PEM/private SSH key should **not** be distributed among many engineers.

Bad pattern:

~~~text
Engineer A ----\
Engineer B -----\
Engineer C ------> Same PEM Key ---> EC2
Engineer D -----/
~~~

Problems:

- Poor individual accountability
- Difficult offboarding
- Key rotation impacts everyone
- Shared-secret leakage risk
- Harder to enforce least privilege
- Difficult to audit who accessed the instance

---

## 12. Better SSH-Key Model

If SSH keys must be used, each engineer should have an individual key pair.

The engineer keeps the private key.

Only the public key is installed on the EC2 instance, for example in:

~~~text
~/.ssh/authorized_keys
~~~

Concept:

~~~text
Engineer A private key
        |
Public key A ----\
                  \
Engineer B public key ---> EC2 authorized_keys
                  /
Engineer C public key ---/
~~~

If Engineer B leaves:

~~~text
Remove Engineer B public key
~~~

Other engineers' keys do not need to be rotated.

---

## 13. Recommended Enterprise Model

For a large organization, prefer centralized identity instead of distributing PEM files.

~~~text
Corporate SSO
    |
IAM Identity Center
    |
IAM Role
    |
SSM Session Manager
    |
Private EC2
~~~

Benefits:

- Central user lifecycle
- No shared private keys
- Role-based access
- Easier onboarding/offboarding
- Better auditability
- No inbound SSH requirement
- Works with private EC2 instances

---

# 14. Interview Q&A

## Q1. Can we share one PEM file with many engineers?

Technically it may work, but it is poor enterprise security practice.

A shared private key removes individual accountability and creates key-rotation and offboarding problems.

---

## Q2. What should be done if SSH is still required?

Give each engineer an individual SSH key pair and install only their public key on the target instance.

Do not share the private key.

---

## Q3. What is the preferred method for production EC2 administration?

AWS Systems Manager Session Manager is usually preferred because it allows IAM/SSO-controlled access without public IPs or inbound SSH/RDP ports.

---

## Q4. How do you revoke one engineer's SSH access?

Remove only that engineer's public key from the authorized keys or centralized SSH-access mechanism.

With SSM, revoke the engineer's IAM/SSO role or permission instead.

---

## Q5. Do we need a bastion host if we use Session Manager?

Usually no.

Session Manager can provide administrative access to private EC2 instances without a bastion host, assuming the instance has the required IAM role, SSM agent, and network connectivity to Systems Manager.

---

## Q6. What is the difference between SSH access and application access to EC2?

Administrative access:

~~~text
SSH / RDP / SSM / Instance Connect
~~~

Application access:

~~~text
Route 53 -> ALB/NLB -> EC2 Target Group
~~~

Users should generally access the application through a load balancer instead of connecting directly to EC2.

---

# 15. Interview-Ready Summary

> In a large organization, I would not distribute a common PEM file among engineers. If SSH is mandatory, each engineer should have an individual SSH key and only the public key should be installed on the EC2 instance. For production environments, I prefer IAM/SSO-based access through AWS Systems Manager Session Manager because it provides centralized RBAC, auditability, easier onboarding/offboarding, and removes the need for shared private keys, public IPs, and open SSH ports.
