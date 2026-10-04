

Readme · MD
# Highly Available 3-Tier Web Architecture on AWS
 
A production-style three-tier web architecture on AWS, designed with reliability, security, and observability in mind. The environment spans two Availability Zones inside a custom VPC (`172.17.0.0/16`) and isolates the presentation, application, and database tiers in separate subnets.
 
![Architecture diagram](./3-tier_architecture.png)
 
> **Note:** This environment was built manually through the AWS Console as a hands-on learning project. Rebuilding it with Infrastructure as Code (Terraform) is the planned next step (see [Future Improvements](#future-improvements)).
 
## Architecture Highlights
 
- **Edge and traffic flow:** Route 53 → CloudFront (with AWS WAF and an ACM-issued HTTPS certificate) → Internet Gateway → presentation ALB → application ALB → database. CloudFront serves cached content and forwards to the origin only on a cache miss.
- **Network isolation:** Each AZ has a public subnet (Bastion host / NAT Gateway) plus private subnets for the presentation, application, and database tiers. Database subnets have no route to the internet.
- **High availability:** The presentation and application tiers run in Auto Scaling Groups across 2 AZs behind separate Application Load Balancers. The database tier uses Amazon RDS with a primary in one AZ and synchronous replication to a standby in the other.
- **Security:** Per-tier security groups, private route tables (outbound access through NAT only), a Bastion host as the single administrative entry point, IAM roles for machine authentication, and WAF in front of CloudFront.
- **Observability:** CloudWatch metrics, logs, and alarms on ALB, ASG, RDS, and NAT Gateway, with SNS email notifications when alarms fire.
## Network Layout
 
| Subnet | CIDR | AZ | Purpose |
|---|---|---|---|
| Public | 172.17.1.0/24 | A | Bastion host |
| Public | 172.17.2.0/24 | B | NAT Gateway |
| Private (Presentation) | 172.17.3.0/24 | A | Frontend ASG |
| Private (Presentation) | 172.17.4.0/24 | B | Frontend ASG |
| Private (Application) | 172.17.5.0/24 | A | Backend ASG |
| Private (Application) | 172.17.6.0/24 | B | Backend ASG |
| Private (Database) | 172.17.7.0/24 | A | RDS primary |
| Private (Database) | 172.17.8.0/24 | B | RDS standby |
 
## Benefits of This Architecture
 
- **High availability:** Every tier runs across two AZs. ALBs route around unhealthy targets, Auto Scaling Groups replace failed instances, and RDS Multi-AZ fails over automatically.
- **Elastic scalability:** The presentation and application tiers scale out and in independently based on load, so each tier can be sized for its own traffic pattern.
- **Defense in depth:** Security is layered: WAF and HTTPS at the edge, tier-specific security groups, private subnets, a database with no internet route, and IAM roles instead of long-lived credentials.
- **Clear separation of concerns:** Each tier can be deployed, scaled, secured, and troubleshot on its own. A problem in the frontend tier does not directly affect the database.
- **Good observability:** CloudWatch alarms plus SNS email notifications give early warning on the key components (ALB, ASG, RDS, NAT).
- **Industry-standard pattern:** The design maps directly to the AWS Well-Architected reliability and security pillars, which makes it easy to reason about and extend.
## Drawbacks and Trade-offs
 
- **Cost:** This is the biggest downside. Even with low traffic, many resources bill continuously:
  - Two Application Load Balancers (hourly charge each)
  - A NAT Gateway (hourly charge plus per-GB data processing)
  - EC2 instances in two AZs for two tiers (ASG minimum capacity is always running)
  - RDS Multi-AZ, which runs a full standby instance that serves no traffic
  - WAF, and a Bastion host that is idle most of the time
- **Operational complexity:** Many moving parts (VPC, 8 subnets, route tables, security groups, 2 ALBs, 2 ASGs, RDS) mean more to configure, monitor, and keep consistent.
- **Manual provisioning:** The environment was built by hand in the AWS Console, so it is hard to reproduce and prone to configuration drift until it is codified in IaC.
- **Remaining single points of failure:** The single NAT Gateway (AZ B) is an AZ-level weakness (see the interview questions below).
- **Over-engineered for small workloads:** For a low-traffic site, this much infrastructure is more than needed. The design pays off when availability and independent scaling really matter.
- **Server management overhead:** EC2 instances still need OS patching, AMI updates, and capacity tuning, which a managed or serverless service would handle for us.
## Interview Questions May Be Asked
 
### 1. There is only one NAT Gateway (in AZ B), but both AZs' private route tables point to it. Isn't that a single point of failure?
 
Yes. It is an AZ-level single point of failure, and it was a deliberate cost trade-off for this learning project, since each NAT Gateway is billed per hour plus per GB processed.
 
**Impact if AZ B goes down:**
- Inbound user traffic is **not** affected. It goes through CloudFront and the ALBs, which are multi-AZ.
- Instances in AZ A's private subnets lose **outbound** internet access (OS patching, package downloads, calls to external APIs) until the NAT is restored.
- Cross-AZ traffic from AZ A to the NAT in AZ B also adds data transfer cost and latency.
**What I would change in production:** deploy one NAT Gateway per AZ and point each AZ's private route table to the NAT Gateway in the same AZ. This removes the cross-AZ dependency and keeps each AZ self-sufficient.
 
### 2. The diagram says "Master / Slave". Is this a Multi-AZ deployment or a read replica?

This project uses **RDS Multi-AZ**, so technically it is a **primary / standby** pair, not master/slave:
- The standby receives **synchronous** replication and exists purely for **failover**. It does not serve read traffic.
- If the primary or its AZ fails, RDS automatically promotes the standby and updates the DNS endpoint, so the application reconnects without a code change.
By contrast, a **read replica** uses **asynchronous** replication and is meant for **scaling reads**. It can lag behind the primary, and failover to it is not automatic by default. The two features solve different problems (availability vs. read scalability) and can be combined.
 
### 3. What happens if an entire Availability Zone fails?
 
- Route 53 / CloudFront keep working because they are global services.
- The ALBs stop routing to unhealthy targets in the failed AZ and continue to serve from the healthy AZ.
- Auto Scaling Groups launch replacement instances in the remaining AZ to restore capacity.
- RDS fails over to the standby in the surviving AZ.
- Known gap: the single NAT Gateway issue above if the failed AZ is AZ B.
### 4. Why two separate load balancers (presentation ALB and application ALB)?
 
It separates the tiers so each can scale, be secured, and fail independently. The application tier is only reachable from the presentation tier's security group, never directly from the internet. Each tier's ASG scales on its own load pattern.
 
### 5. Why use a Bastion host, and what are the risks?
 
It provides a single, controlled entry point for administrative SSH access to private instances, instead of exposing them. Risks: it is an internet-facing host that must be tightly restricted (source IP allow-list, key-only auth, patched). A more modern alternative is **AWS Systems Manager Session Manager**, which needs no inbound ports or SSH keys and gives audited access through IAM.
 
### 6. Why was this built manually, and how would you make it reproducible?
 
I built it by hand first to understand how each AWS component (VPC, route tables, security groups, ALB, ASG, RDS) connects. The weakness of manual setup is configuration drift and no repeatability. The next step is to codify the whole environment in Terraform, store state remotely (S3 + DynamoDB locking), and deploy through a CI pipeline.
 
## Possible Improvements
 
### 1. Remove the presentation tier: host the frontend on S3 + CloudFront
 
If the frontend is a static site (for example a React/Vue single-page app built into HTML/JS/CSS), it does not need EC2 instances, an ASG, or a presentation ALB at all.
 
**Proposed design:** Route 53 → CloudFront (+ WAF) → S3 bucket for static assets, and CloudFront forwards API paths (e.g. `/api/*`) to the application tier.
 
**Why it is better:**
- **Lower cost:** Removes the presentation-tier EC2 instances, the presentation ALB, and the NAT traffic they generate. S3 storage and CloudFront delivery are billed by usage and are typically far cheaper for static content.
- **Higher availability with less effort:** S3 is a regional service with built-in multi-AZ durability, and CloudFront is a globally distributed service. There are no instances to patch, replace, or scale.
- **Better performance:** Static files are cached at edge locations close to users instead of being served from EC2 in a single region.
- **Smaller attack surface:** The bucket stays private and is only readable by CloudFront through Origin Access Control (OAC). There are no frontend servers to harden or SSH into.
- **Simpler operations:** Deploying a new frontend version becomes uploading files to S3 and invalidating the CloudFront cache.
**Caveat:** This only works for static or client-side rendered frontends. If the presentation tier does server-side rendering (SSR), it still needs compute (EC2, containers, or Lambda). The application tier must also be reachable from CloudFront, for example through a public ALB restricted to CloudFront or a CloudFront VPC origin.
 
### 2. Other improvements
 
- **One NAT Gateway per AZ** to remove the AZ-level single point of failure.
- **Replace the Bastion host with SSM Session Manager** to remove inbound SSH exposure and get IAM-audited access.
- **Rebuild with Terraform** (modules per tier, remote state in S3 with locking) for repeatable, reviewable deployments.
- **Add a CI/CD pipeline** for application and infrastructure changes.
- **Cost optimization:** Use VPC endpoints (e.g. for S3) to reduce NAT data charges, right-size instances, and consider Savings Plans or Reserved Instances for steady-state workloads.
- **Stronger observability:** Add dashboards, application-level metrics and logs, and define SLOs with alerting tied to them.
### Roadmap (Go to further steps!)
 
- [ ] Run failure drills (terminate an instance, simulate AZ failure, trigger RDS failover) and document recovery times
- [ ] Rebuild the environment with Terraform
- [ ] Migrate the frontend to S3 + CloudFront and decommission the presentation tier
- [ ] One NAT Gateway per AZ
- [ ] Replace the Bastion host with SSM Session Manager
- [ ] Add a CI/CD pipeline
 
**Feedback and suggestions are always welcome. If you spot any mistakes or have ideas for improvement, feel free to let me know.**