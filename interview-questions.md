# AWS Interview Questions

Important interview questions for each module, grouped by level:
- 🟢 **Basic**: definitions, what interviewers ask first
- 🟡 **Intermediate**: how things work and when to use what
- 🔴 **Scenario**: "how would you design / fix this?"

**How to practice:** read the question, answer **out loud** first, then open the answer. In scenario questions, interviewers care more about your reasoning (trade-offs, security, cost, availability) than about one "correct" answer.

## Modules
1. [Cloud Fundamentals](#module-1--cloud-fundamentals)
2. [IAM](#module-2--iam)
3. [Compute](#module-3--compute)
4. [Storage](#module-4--storage)

---

## Module 1 — Cloud Fundamentals

### 🟢 Basic

**Q1. What is cloud computing? What are its main benefits?**
<details><summary>Answer</summary>

On-demand delivery of IT resources (compute, storage, databases, networking) over the internet with pay-as-you-go pricing. Benefits: no upfront hardware cost (CapEx → OpEx), scale up/down on demand, faster time to market, global reach in minutes, no data center maintenance, economies of scale.
</details>

**Q2. Explain IaaS, PaaS and SaaS with AWS examples.**
<details><summary>Answer</summary>

- **IaaS**: you get raw infrastructure and manage the OS and above. *EC2, EBS, VPC.*
- **PaaS**: you manage only your code/data; the platform is managed. *Elastic Beanstalk, App Runner* (RDS and Lambda are often described as managed/PaaS-like).
- **SaaS**: complete software you just use. *Amazon WorkMail, Gmail, Salesforce.*
</details>

**Q3. What is the difference between a Region, an Availability Zone and an Edge Location?**
<details><summary>Answer</summary>

- **Region**: a separate geographic area (e.g. `ap-south-1`), fully isolated from other Regions.
- **AZ**: one or more isolated data centers inside a Region, with independent power/network, connected by low-latency links. Each Region has 3+ AZs.
- **Edge Location**: a point of presence used by CloudFront/Route 53 to serve content close to users.
</details>

**Q4. What is the Shared Responsibility Model?**
<details><summary>Answer</summary>

AWS is responsible for security **of** the cloud (physical data centers, hardware, network, virtualization). The customer is responsible for security **in** the cloud (data, IAM, encryption choices, OS patching on EC2, security groups, application code). The split shifts with the service: on EC2 you patch the OS; on RDS or Lambda AWS does.
</details>

### 🟡 Intermediate

**Q5. How do you choose an AWS Region?**
<details><summary>Answer</summary>

Four factors: **compliance/data residency** (legal requirements come first), **latency** to users, **service availability** (not every service is in every Region), and **pricing** (varies by Region).
</details>

**Q6. What is the difference between high availability, fault tolerance, scalability and elasticity?**
<details><summary>Answer</summary>

- **High availability**: the system stays up with minimal downtime (e.g. running in multiple AZs).
- **Fault tolerance**: the system keeps working with **zero** interruption even when a component fails (more redundancy, higher cost).
- **Scalability**: ability to handle more load by adding resources (vertical = bigger machine, horizontal = more machines).
- **Elasticity**: automatically scaling up **and down** to match demand, so you pay only for what you need.
</details>

**Q7. Explain the EC2 pricing models. When would you use each?**
<details><summary>Answer</summary>

- **On-Demand**: no commitment, highest price. Short-term or unpredictable workloads.
- **Reserved Instances / Savings Plans**: 1- or 3-year commitment, up to ~72% off. Steady, predictable workloads. Savings Plans are more flexible (commit to $/hour).
- **Spot**: up to ~90% off, but can be interrupted with a 2-minute warning. Fault-tolerant batch jobs, CI, data processing.
- **Dedicated Hosts**: a physical server for you. Licensing (per-socket/core) or compliance needs.

A common real answer: mix them. Savings Plans for the baseline, On-Demand for peaks, Spot for batch jobs.
</details>

**Q8. What do you pay for in AWS? Is data transfer charged?**
<details><summary>Answer</summary>

Mainly compute time, storage, and **data transfer out** (to the internet, and across Regions/AZs). Data transfer **in** to AWS is generally free.
</details>

### 🔴 Scenario

**Q9. Your company's AWS bill suddenly doubled. How do you investigate?**
<details><summary>Answer</summary>

1. Open **Cost Explorer**: group by service, Region, account and usage type to find what grew.
2. Check **cost allocation tags** to find which team/project owns it.
3. Common culprits: forgotten instances, unattached EBS volumes, old snapshots, NAT Gateway data processing, data transfer out, unused Elastic IPs/public IPv4, a misconfigured Lambda loop, logs with no retention.
4. Prevent it next time: **AWS Budgets** with alerts, **Cost Anomaly Detection**, tagging policies, and **Trusted Advisor / Compute Optimizer** for rightsizing.
</details>

**Q10. Why would a company move from on-premises to AWS, and what are the risks?**
<details><summary>Answer</summary>

Why: lower upfront cost, faster provisioning, elasticity, global reach, managed services, better DR options. Risks: cost surprises without governance, security misconfiguration (shared responsibility), vendor lock-in, skill gaps, data migration and compliance challenges. Mitigate with a landing zone (multi-account setup, guardrails), cost controls and training.
</details>

---

## Module 2 — IAM

### 🟢 Basic

**Q1. What is IAM? Is it regional or global?**
<details><summary>Answer</summary>

Identity and Access Management controls who (authentication) can do what on which resources (authorization). It's a **global** service and it's free.
</details>

**Q2. What are users, groups, roles and policies?**
<details><summary>Answer</summary>

- **User**: an identity with long-term credentials (password, access keys).
- **Group**: a collection of users; permissions attached to it apply to all members. Groups can't be nested.
- **Role**: an identity with **temporary** credentials, assumed by services, other accounts or federated users.
- **Policy**: a JSON document that defines allowed or denied actions on resources.
</details>

**Q3. What should you do with the root user?**
<details><summary>Answer</summary>

Enable MFA, delete/never create access keys, use a strong password, and don't use it for daily work. Create admin users (or use IAM Identity Center) instead. Use root only for the few tasks that require it (closing the account, changing the support plan, etc.).
</details>

**Q4. What is the principle of least privilege?**
<details><summary>Answer</summary>

Grant only the minimum permissions needed to perform a task, and nothing more. Start with minimal access, add when needed, and review regularly using Access Advisor / Access Analyzer.
</details>

### 🟡 Intermediate

**Q5. Explain the structure of an IAM policy.**
<details><summary>Answer</summary>

`Version` (always `2012-10-17`) and a list of `Statement`s. Each statement has `Effect` (Allow/Deny), `Action` (e.g. `s3:GetObject`), `Resource` (ARNs), and optional `Condition` (IP, MFA, tags, time...). Resource-based policies also have a `Principal`.
</details>

**Q6. How does AWS evaluate permissions when several policies apply?**
<details><summary>Answer</summary>

By default everything is denied (implicit deny). If any applicable policy has an **explicit Deny**, the request is denied. Otherwise, if there's an **explicit Allow**, it's allowed. Rule: **explicit Deny > explicit Allow > implicit Deny.** (In organizations, SCPs and permission boundaries also must allow the action.)
</details>

**Q7. What's the difference between an identity-based policy and a resource-based policy?**
<details><summary>Answer</summary>

Identity-based policies attach to users/groups/roles and define what **that identity** can do. Resource-based policies attach to a resource (S3 bucket policy, SQS queue policy, KMS key policy) and define **who** (`Principal`) can access it. Resource-based policies are commonly used for cross-account access.
</details>

**Q8. What's the difference between an IAM role's trust policy and permission policy?**
<details><summary>Answer</summary>

The **trust policy** defines who can assume the role (e.g. `ec2.amazonaws.com`, another AWS account, a SAML/OIDC provider). The **permission policy** defines what the role can do once assumed.
</details>

**Q9. What is STS?**
<details><summary>Answer</summary>

Security Token Service issues **temporary credentials** (access key, secret key, session token) with an expiry. Used by `AssumeRole`, federation, and IAM Identity Center. Temporary credentials are safer than long-term keys because they expire.
</details>

**Q10. What are SCPs and permission boundaries?**
<details><summary>Answer</summary>

- **SCP (Service Control Policy)**: an AWS Organizations policy that sets the **maximum** permissions for every identity in an account (even admins). It never grants permissions by itself.
- **Permission boundary**: sets the maximum permissions for a **single** user or role. Useful when you let developers create roles but want to cap what those roles can do.
</details>

### 🔴 Scenario

**Q11. An application on EC2 needs to read from an S3 bucket. How do you give it access?**
<details><summary>Answer</summary>

Create an IAM role with a policy allowing `s3:GetObject` (and `s3:ListBucket` if needed) on that specific bucket, trusted by `ec2.amazonaws.com`. Attach it to the instance via an **instance profile**. The SDK automatically gets temporary credentials from the instance metadata service. **Never** store access keys on the instance or in code.
</details>

**Q12. A developer accidentally pushed AWS access keys to a public GitHub repo. What do you do?**
<details><summary>Answer</summary>

1. **Immediately deactivate and delete** the keys in IAM (and create new ones only if really needed).
2. Check **CloudTrail** for any activity with those keys: new users, instances (crypto mining is common), data access.
3. Remove any resources or access the attacker created; review billing.
4. Remove the keys from Git history (not just the latest commit).
5. Prevent recurrence: use roles / IAM Identity Center instead of long-term keys, secret scanning (GitHub secret scanning, git-secrets), least privilege.
</details>

**Q13. How do you give users in account A access to resources in account B?**
<details><summary>Answer</summary>

Create a role in account B with the needed permissions and a trust policy that trusts account A (or specific roles in it). Grant users in account A permission to `sts:AssumeRole` on that role. Users assume the role and get temporary credentials for account B. For some services (S3, KMS, SQS) you can alternatively use a resource-based policy. In larger organizations, use IAM Identity Center with permission sets.
</details>

**Q14. A user says they get "Access Denied" even though their policy allows the action. How do you troubleshoot?**
<details><summary>Answer</summary>

Check for: an explicit **Deny** in any attached policy or group; an **SCP** in AWS Organizations; a **permission boundary**; a **resource-based policy** (e.g. bucket policy) that denies them; **conditions** not met (MFA, IP, tags, Region); wrong resource ARN in the policy; KMS key policy for encrypted data. Tools: **IAM Policy Simulator**, the decoded authorization message (`aws sts decode-authorization-message`), and **CloudTrail** for the exact failing call.
</details>

---

## Module 3 — Compute

### 🟢 Basic

**Q1. What is EC2? What is an AMI?**
<details><summary>Answer</summary>

EC2 provides resizable virtual servers (instances) in the cloud. An **AMI** (Amazon Machine Image) is the template used to launch an instance: OS, software and configuration. You can use AWS, Marketplace or your own custom AMIs. AMIs are regional but can be copied.
</details>

**Q2. What is a security group? How is it different from a Network ACL?**
<details><summary>Answer</summary>

A security group is an instance-level virtual firewall: **stateful**, **allow rules only**, all inbound denied by default. A **NACL** works at the **subnet** level, is **stateless** (return traffic must be explicitly allowed), supports **allow and deny** rules, and evaluates rules in number order. (NACLs are covered in the networking module.)
</details>

**Q3. What is the difference between stopping and terminating an instance?**
<details><summary>Answer</summary>

**Stop**: instance shuts down, you stop paying for compute but still pay for EBS; it can be started again (public IP changes, instance store data is lost). **Terminate**: instance is permanently deleted; the root EBS volume is deleted by default.
</details>

**Q4. What is AWS Lambda?**
<details><summary>Answer</summary>

A serverless compute service that runs your code in response to events (API calls, S3 uploads, queue messages, schedules). No servers to manage, scales automatically, and you pay per request and execution time. Max 15 minutes per run, up to 10 GB memory.
</details>

### 🟡 Intermediate

**Q5. Explain the types of Elastic Load Balancers.**
<details><summary>Answer</summary>

- **ALB** (layer 7): HTTP/HTTPS, routing by path/host/headers, targets can be instances, IPs, Lambda, containers. For web apps and microservices.
- **NLB** (layer 4): TCP/UDP/TLS, ultra-high performance, low latency, static IPs. For non-HTTP or extreme-scale traffic.
- **GWLB** (layer 3): routes traffic through third-party security appliances.
- Classic LB is legacy.
</details>

**Q6. How does an Auto Scaling Group work? What scaling policies are there?**
<details><summary>Answer</summary>

An ASG launches instances from a **launch template** and keeps the count between **min** and **max**, aiming for **desired**. It spreads instances across AZs, replaces unhealthy ones, and registers them with a load balancer. Policies: **target tracking** (e.g. keep CPU at 50%), **step/simple** (alarm-based), **scheduled**, and **predictive** scaling.
</details>

**Q7. What is user data? What is instance metadata?**
<details><summary>Answer</summary>

**User data** is a script passed at launch that runs once as root on first boot (install packages, configure the app). **Instance metadata** (`http://169.254.169.254`) lets the instance learn about itself: instance ID, IP, AZ, and the temporary credentials of its IAM role. Use **IMDSv2** (session-token based) for security.
</details>

**Q8. When would you use EC2 vs Lambda vs containers (ECS/Fargate)?**
<details><summary>Answer</summary>

- **EC2**: full OS control, long-running or specialized workloads (GPUs, custom software), steady high load.
- **Lambda**: short, event-driven tasks (<15 min), spiky traffic, glue between services, minimal ops.
- **Containers**: microservices, portability, consistent environments. Fargate if you don't want to manage servers; ECS on EC2 or EKS for more control or Kubernetes.
</details>

**Q9. What is a Lambda cold start, and how do you reduce it?**
<details><summary>Answer</summary>

The extra latency when Lambda must create a new execution environment (download code, start the runtime, run init code), typically after idle time or during a scale-up. Reduce it with: smaller packages and fewer dependencies, initializing clients outside the handler, faster runtimes, **Provisioned Concurrency**, or **SnapStart** (Java, Python, .NET).
</details>

**Q10. ECS vs EKS: what's the difference? What is Fargate?**
<details><summary>Answer</summary>

**ECS** is AWS's own container orchestrator: simpler, deeply integrated with AWS. **EKS** is managed Kubernetes: industry standard, portable across clouds, more complex. **Fargate** is a serverless launch type for both: you run containers without managing EC2 instances.
</details>

### 🔴 Scenario

**Q11. Design a highly available, scalable web application on AWS.**
<details><summary>Answer</summary>

- **Route 53** → **CloudFront** (optional, for caching) → **ALB** spanning at least 2 AZs.
- **Auto Scaling Group** of EC2 instances (or ECS Fargate services) in **private subnets** across 2–3 AZs, with target tracking scaling.
- **Database**: RDS Multi-AZ or Aurora (plus read replicas if read-heavy); ElastiCache for caching.
- Static assets in **S3**; sessions stored outside the instances (ElastiCache/DynamoDB) so instances are **stateless**.
- Security: security groups chained (ALB SG → app SG → DB SG), IAM roles, HTTPS with ACM, WAF.
- Monitoring with CloudWatch alarms; backups and IaC for repeatability.
</details>

**Q12. Your EC2 instance is running but the website isn't reachable. How do you troubleshoot?**
<details><summary>Answer</summary>

Work from outside in:
1. Does the instance have a **public IP** and is it in a **public subnet** (route table → Internet Gateway)?
2. **Security group** allows inbound 80/443 from your IP or `0.0.0.0/0`?
3. **NACL** allows inbound and the return (ephemeral) ports?
4. Instance status checks passing?
5. Is the web server actually running and listening (`systemctl status`, `curl localhost`, `ss -tlnp`)? Is the OS firewall blocking it?
6. If behind an ALB: target health checks, the instance SG allows traffic from the ALB SG, the health check path returns 200.
</details>

**Q13. Traffic spikes every morning at 9 AM and the app is slow for the first few minutes. How do you fix it?**
<details><summary>Answer</summary>

Reactive scaling is too slow because new instances take minutes to launch and warm up. Use **scheduled scaling** (raise desired capacity at 8:45) or **predictive scaling**. Also: use a **warm pool**, bake software into a custom AMI so boot is faster, tune health check grace periods, and add caching (CloudFront/ElastiCache) to reduce load.
</details>

**Q14. How would you reduce EC2 costs for a company?**
<details><summary>Answer</summary>

Rightsize using **Compute Optimizer** / CloudWatch metrics; buy **Savings Plans** or RIs for steady usage; use **Spot** for fault-tolerant workloads; switch to **Graviton** instances; auto-scale and shut down dev/test outside working hours; delete unused EBS volumes, snapshots and Elastic IPs; move suitable workloads to Lambda/Fargate.
</details>

**Q15. You lost the SSH key pair for a running Linux instance. How do you get access?**
<details><summary>Answer</summary>

Best: connect with **Session Manager** (if the SSM agent and role are set up) or **EC2 Instance Connect**, then add a new public key to `~/.ssh/authorized_keys`. Otherwise: stop the instance, detach the root volume, attach it to another instance, edit `authorized_keys` on it, reattach and start (or use the **EC2Rescue / AWSSupport-ResetAccess** automation).
</details>

---

## Module 4 — Storage

### 🟢 Basic

**Q1. What is the difference between S3, EBS and EFS?**
<details><summary>Answer</summary>

- **S3**: object storage accessed over HTTP/API, unlimited, for files, backups, media, static sites, data lakes.
- **EBS**: block storage, a disk attached to one EC2 instance in one AZ, for OS and databases.
- **EFS**: managed NFS file system shared by many Linux instances across AZs, auto-scaling.
</details>

**Q2. What are S3 storage classes?**
<details><summary>Answer</summary>

Standard (frequent access), Intelligent-Tiering (unknown patterns, auto-moves data), Standard-IA and One Zone-IA (infrequent, retrieval fee), Express One Zone (ultra-low latency), Glacier Instant Retrieval, Glacier Flexible Retrieval, and Glacier Deep Archive (cheapest, 12–48 h retrieval). Lower storage cost generally means higher retrieval cost or time and a minimum storage duration.
</details>

**Q3. What is S3 durability and availability?**
<details><summary>Answer</summary>

**Durability** (chance of not losing data) is 99.999999999% (11 nines) for all classes, since data is stored across multiple AZs (except One Zone classes). **Availability** (chance of being able to access it) is 99.99% for Standard and lower for IA classes.
</details>

**Q4. Is S3 bucket naming global? Is S3 regional?**
<details><summary>Answer</summary>

Bucket **names** are globally unique across all AWS accounts, but each bucket is created in, and its data stays in, a specific **Region**.
</details>

### 🟡 Intermediate

**Q5. How do you secure an S3 bucket?**
<details><summary>Answer</summary>

Keep **Block Public Access** on; control access with **IAM policies** and **bucket policies** (least privilege); keep ACLs disabled; enable **default encryption** (SSE-S3 or SSE-KMS); enforce HTTPS via a bucket policy (`aws:SecureTransport`); enable **versioning** and optionally **MFA Delete** / **Object Lock**; use **pre-signed URLs** for temporary sharing; use **VPC endpoints** for private access; log access with CloudTrail / server access logs; check with **Access Analyzer** and **Macie** (finds sensitive data).
</details>

**Q6. What is S3 versioning? What happens when you delete a versioned object?**
<details><summary>Answer</summary>

Versioning keeps every version of an object. Deleting it without a version ID adds a **delete marker**, which hides the object but keeps all versions. Removing the marker restores it. Deleting a specific version ID removes that version permanently. Once enabled, versioning can only be suspended.
</details>

**Q7. What are S3 lifecycle rules? Give an example.**
<details><summary>Answer</summary>

Rules that automatically transition objects to cheaper classes or expire them based on age. Example for logs: Standard → Standard-IA after 30 days → Glacier after 90 days → delete after 1 year. Also used to delete old non-current versions and incomplete multipart uploads.
</details>

**Q8. What are the S3 encryption options?**
<details><summary>Answer</summary>

Server-side: **SSE-S3** (AWS-managed keys, default), **SSE-KMS** (KMS keys, access control + CloudTrail audit), **DSSE-KMS** (dual-layer), **SSE-C** (customer-provided keys per request). Or **client-side** encryption before upload. In transit: HTTPS/TLS.
</details>

**Q9. What are EBS volume types and when do you use each?**
<details><summary>Answer</summary>

- **gp3**: general purpose SSD, default for most workloads; IOPS/throughput configurable separately from size.
- **io2 Block Express / io1**: provisioned IOPS SSD for critical, I/O-intensive databases; supports Multi-Attach.
- **st1**: throughput HDD for big sequential workloads (logs, big data).
- **sc1**: cold HDD for rarely accessed data, cheapest.
HDD types can't be boot volumes.
</details>

**Q10. What are EBS snapshots? Are they incremental?**
<details><summary>Answer</summary>

Point-in-time backups of EBS volumes stored in S3 (AWS-managed). Yes, incremental: only changed blocks after the first snapshot. Used to restore volumes, create AMIs, and copy data across AZs/Regions. Automate with Data Lifecycle Manager or AWS Backup.
</details>

**Q11. What is the difference between EBS and instance store?**
<details><summary>Answer</summary>

EBS is network-attached, persistent, survives stop/start and can be snapshotted. Instance store is a physical disk on the host: very high IOPS but **ephemeral** (data lost on stop, terminate or hardware failure). Use instance store only for caches, buffers or replicated data.
</details>

### 🔴 Scenario

**Q12. You need to move an EBS volume from one AZ to another (or another Region). How?**
<details><summary>Answer</summary>

Create a **snapshot** of the volume, then create a new volume from that snapshot in the target AZ. For another Region, **copy the snapshot** to that Region first, then create the volume there.
</details>

**Q13. Users must upload large files (several GB) directly to S3 from a web app without going through your servers. How?**
<details><summary>Answer</summary>

The backend generates a **pre-signed URL** (or pre-signed POST) with limited permissions and expiry; the browser uploads directly to S3. For large files, use **multipart upload** with pre-signed URLs per part. Optionally enable **Transfer Acceleration** for far-away users. Use an **S3 event notification** to trigger Lambda processing after upload.
</details>

**Q14. Someone accidentally deleted important files from S3. How do you recover, and how do you prevent it?**
<details><summary>Answer</summary>

Recover: if **versioning** is on, delete the delete markers (or restore previous versions). If replication or AWS Backup was configured, restore from there. Without versioning or backups, the data is gone. Prevent: enable versioning, **MFA Delete**, **Object Lock** for critical data, least-privilege IAM (deny `s3:DeleteObject` for most users), **cross-region replication**, AWS Backup for S3.
</details>

**Q15. Several EC2 instances in different AZs need to read and write the same files. What storage do you use?**
<details><summary>Answer</summary>

**EFS** (Linux, NFS) since EBS is single-AZ and usually single-instance. For Windows/SMB, use **FSx for Windows File Server**. If the app can use an API instead of a file system, **S3** is often cheaper and simpler.
</details>

**Q16. Your company stores 500 TB of logs in S3 Standard but rarely reads them. How do you cut costs?**
<details><summary>Answer</summary>

Analyze access with **S3 Storage Lens / Storage Class Analysis**. Add **lifecycle rules** to move older logs to Standard-IA, then Glacier Flexible Retrieval / **Deep Archive**, and expire logs beyond the retention requirement. Use **Intelligent-Tiering** if access is unpredictable. Compress logs and use columnar formats (Parquet) for analytics. Watch out for minimum storage durations and per-object transition costs with many tiny files.
</details>

**Q17. How do you migrate 200 TB of on-premises data to S3?**
<details><summary>Answer</summary>

Depends on bandwidth and timeline. Calculate transfer time first: 200 TB over a 1 Gbps link takes ~2–3 weeks at full speed. Options: **Snowball Edge** devices (offline, fast for large volumes), **DataSync** over the network or **Direct Connect** (online, incremental, verified), **Storage Gateway** for ongoing hybrid access. Often: Snowball for the bulk, then DataSync for the changes since.
</details>
