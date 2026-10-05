# AWS 3-Tier Architecture — Improvements

This document describes the improved architecture: what is added or changed at the edge, the network layer, the application and database tiers and the operations/security layer.

---

## 1. Overview

| Item               | Value                                             |
| ------------------ | ------------------------------------------------- |
| VPC CIDR           | `172.17.0.0/16`                                   |
| Availability Zones | 2 (AZ A, AZ B), mirrored layout                   |
| Subnets            | 8 total: 4 public + 4 private, each `/24`         |
| Entry point        | CloudFront (single entry for all traffic)         |
| Static content     | S3, served through CloudFront                     |
| Dynamic content    | CloudFront → Application ALB → Auto Scaling group |
| Database           | Amazon RDS, Master (AZ A) + Slave (AZ B)          |

### Subnet map

| Subnet    | CIDR            | AZ  | Contents                                            |
| --------- | --------------- | --- | --------------------------------------------------- |
| Public 1  | `172.17.1.0/24` | A   | Bastion                                             |
| Public 3  | `172.17.3.0/24` | A   | NAT Gateway + EIP                                   |
| Private 5 | `172.17.5.0/24` | A   | Application tier (Auto Scaling group, backend code) |
| Private 7 | `172.17.7.0/24` | A   | Database tier (Master)                              |
| Public 2  | `172.17.2.0/24` | B   | NAT Gateway + EIP                                   |
| Public 4  | `172.17.4.0/24` | B   | Bastion                                             |
| Private 6 | `172.17.6.0/24` | B   | Application tier (Auto Scaling group, backend code) |
| Private 8 | `172.17.8.0/24` | B   | Database tier (Slave)                               |

### Diagram Workflow

![Architecture diagram](./3-tier-improvements.png)
---

## 2. Request Flow

```
User
  │  DNS lookup
  ▼
Route 53 ──► (DNS resolver returns CloudFront)
  │
  ▼  HTTPS
WAF ──► CloudFront
            │
            ├── static resources ──► S3 (responds directly, no backend involved)
            │
            └── backend requests ──► Internet Gateway ──► Application ALB
                                                              │
                                                              ▼
                                                  Auto Scaling group (AZ A / AZ B)
                                                              │  Read / Write
                                                              ▼
                                                    RDS Master (AZ A)
                                                              │  Data synchronization
                                                              ▼
                                                    RDS Slave (AZ B)
```

The response travels back along the same path (marked "Response" in the diagram).

---

## 3. Improvements

### 3.1 Edge layer

| Component             | Improvement                                              | Benefit                                                            |
| --------------------- | -------------------------------------------------------- | ------------------------------------------------------------------ |
| **Route 53**          | DNS entry for the domain, pointing to CloudFront         | Single, managed DNS layer                                          |
| **CloudFront**        | Single entry point for all user traffic                  | Caching, lower latency, hides the origin                           |
| **WAF**               | Placed in front of CloudFront                            | Blocks attacks from the internet before they reach the VPC         |
| **HTTPS certificate** | Provided for CloudFront (HTTPS end to end from the user) | Encrypted traffic                                                  |
| **S3**                | Static resources are served straight from S3             | Static requests never hit the backend, which reduces load and cost |

**Routing rule:** if a request only needs static resources, CloudFront answers from S3. Only requests that need backend logic are forwarded to the backend service (via the Application ALB).

### 3.2 Network layer

- **Internet Gateway (IGW)** is the VPC's door to the internet; public subnets route `0.0.0.0/0 → igw`.
- **One NAT Gateway + EIP per AZ** (in public subnets `172.17.3.0/24` and `172.17.2.0/24`), so each AZ has its own outbound path and one AZ failing does not cut off the other.
- **One Bastion per AZ** (public subnets `172.17.1.0/24` and `172.17.4.0/24`). The bastion can reach all machines and is the only administrative entry point.
- **Application tier and database tier are in private subnets**, with no public IPs.
- Database subnets have **no internet route** at all.

### 3.3 Route tables

| Route table         | Used by              | Routes                                                    |
| ------------------- | -------------------- | --------------------------------------------------------- |
| `pub-route`         | Public subnets       | `0.0.0.0/0 → igw`, `172.17.0.0/16 → local`                |
| `private-rt` (AZ A) | Private app subnet A | `172.17.0.0/16 → local`, `0.0.0.0/0 → nat-gateway (AZ A)` |
| `private-rt` (AZ B) | Private app subnet B | `172.17.0.0/16 → local`, `0.0.0.0/0 → nat-gateway (AZ B)` |
| `db-rt`             | Database subnets     | `172.17.0.0/16 → local` only                              |

Private application subnets reach the internet (patches, external APIs) via **NAT Gateway → IGW**. Each AZ uses the NAT Gateway in its own AZ.

### 3.4 Security

| Control                      | Purpose                                                                                |
| ---------------------------- | -------------------------------------------------------------------------------------- |
| WAF                          | Prevents cyber attacks from the internet                                               |
| HTTPS certificate            | Encrypts traffic between users and CloudFront                                          |
| Security Group (Application) | Controls what can reach the backend instances                                          |
| Security Group (Database)    | Controls what can reach RDS (application tier only)                                    |
| IAM machine authentication   | Instances authenticate to AWS services through IAM roles instead of stored credentials |
| Bastion host                 | Single controlled path for administrator access                                        |
| Private subnets              | App and DB tiers are not directly reachable from the internet                          |

### 3.5 Monitoring and alerting

- **Amazon CloudWatch** collects monitoring data, logs and alarms for **ALB, ASG, RDS and NAT**.
- **Amazon SNS** sends an email when an alarm fires.

---

## 4. Summary of changes

1. CloudFront becomes the single entry point, with WAF and HTTPS in front.
2. Static resources are served from S3, so they bypass the backend.
3. Route 53 handles DNS for the domain.
4. Everything is mirrored across two AZs: NAT Gateway, Bastion, application tier, and database tier.
5. The database has a Master/Slave pair with data synchronization across AZs.
6. Application and database tiers are fully private; the database has no internet route.
7. CloudWatch + SNS provide monitoring and email alerts.
8. IAM provides machine authentication.
