# Corporate VPN to AWS — Common Enterprise Flow

```text
User Laptop
    |
    | Company VPN
    v
Corporate VPN Gateway
    |
    v
Corporate Network
    |
    | Site-to-Site VPN
    | or Direct Connect
    v
AWS Transit Gateway / Virtual Private Gateway
    |
    v
AWS VPC
    |
    v
EC2 / Internal ALB / Private Service
```

## Minimal Explanation

1. The user first connects to the **company VPN** and enters the corporate network.
2. The corporate network reaches AWS through **Site-to-Site VPN** or **AWS Direct Connect**.
3. On the AWS side, traffic typically enters through a **Transit Gateway** or **Virtual Private Gateway**.
4. AWS routing then sends the traffic into the required **VPC/subnet**.
5. The user can then access private AWS resources such as **EC2 instances, internal load balancers, or private services**.
