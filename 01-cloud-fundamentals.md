# 01 — Cloud Fundamentals

## 1. What is cloud computing?

Cloud computing means **renting computing resources (servers, storage, databases, networking) over the internet** instead of buying and running your own hardware.

You pay only for what you use, the way you pay for electricity.

### Traditional (on-premises) vs Cloud

| On-premises | Cloud |
|-------------|-------|
| Buy servers upfront (CapEx) | Pay as you go (OpEx) |
| Takes weeks to get new hardware | New server in minutes |
| You guess capacity in advance | Scale up and down on demand |
| You maintain the data center | AWS maintains the hardware |
| Usually one location | Deploy globally in minutes |

### The 6 advantages of cloud (from AWS)
1. Trade upfront expense for variable expense
2. Benefit from massive economies of scale (lower prices)
3. Stop guessing capacity
4. Increase speed and agility
5. Stop spending money running data centers
6. Go global in minutes

---

## 2. Cloud service models

| Model | What you manage | What the provider manages | AWS example |
|-------|-----------------|---------------------------|-------------|
| **IaaS** (Infrastructure as a Service) | OS, runtime, app, data | Hardware, network, virtualization | EC2 |
| **PaaS** (Platform as a Service) | Your app and data | Everything underneath | Elastic Beanstalk |
| **SaaS** (Software as a Service) | Just using it | Everything | Gmail, Amazon WorkMail |

**Analogy (pizza):**
- On-prem = make pizza at home from scratch
- IaaS = buy a frozen pizza and bake it yourself
- PaaS = pizza delivery
- SaaS = eat at a restaurant

### Deployment models
- **Public cloud**: AWS, Azure, GCP
- **Private cloud**: your own cloud infrastructure, used only by your organization
- **Hybrid cloud**: on-prem + public cloud connected together

---

## 3. AWS Global Infrastructure

### Regions
- A **Region** is a geographic area, e.g. `us-east-1` (N. Virginia) or `ap-south-1` (Mumbai)
- Each Region is **fully independent**
- Most services are **region-scoped**: resources created in one region don't appear in another

**How to choose a Region:**
1. **Compliance**: some data must legally stay in a certain country
2. **Latency**: pick a region close to your users
3. **Service availability**: not every service exists in every region
4. **Pricing**: prices vary by region

### Availability Zones (AZs)
- Each Region has **multiple AZs** (usually 3, minimum 2), e.g. `ap-south-1a`, `ap-south-1b`
- An AZ is **one or more data centers** with their own power, cooling and networking
- AZs are physically separated, so one disaster doesn't take them all down
- AZs are connected with high-speed, low-latency links

> 💡 **Key idea:** To be highly available, run your app in **at least 2 AZs**.

### Edge Locations (Points of Presence)
- 400+ locations worldwide
- Used by **CloudFront** (CDN) and **Route 53** (DNS) to deliver content to users with low latency

### Global vs Regional services
- **Global:** IAM, Route 53, CloudFront, WAF
- **Regional:** most other services (EC2, S3 buckets, RDS, Lambda...)

```
AWS Global
 └── Region (e.g. ap-south-1)
      ├── AZ ap-south-1a  → data center(s)
      ├── AZ ap-south-1b  → data center(s)
      └── AZ ap-south-1c  → data center(s)
 └── Edge Locations (worldwide, for caching/DNS)
```

---

## 4. Shared Responsibility Model

Security is split between AWS and you.

| AWS: security **OF** the cloud | You: security **IN** the cloud |
|--------------------------------|--------------------------------|
| Physical data centers | Your data (encryption, backups) |
| Hardware, servers | IAM users, roles, permissions |
| Network infrastructure | OS patching on EC2 |
| Virtualization layer | Security groups / firewall rules |
| Managed service infrastructure | Application code |

**Your share shrinks as services get more managed:**
- **EC2:** you patch the OS, configure the firewall, manage the app
- **RDS:** AWS patches the DB engine; you manage users, access and encryption settings
- **Lambda / S3:** AWS handles almost all infrastructure; you manage code, data and permissions

---

## 5. AWS Pricing Basics

### Three things you pay for
1. **Compute**: time your servers run
2. **Storage**: data stored
3. **Data transfer OUT**: data leaving AWS (data coming IN is usually free)

### EC2 purchasing options
| Option | Best for | Discount |
|--------|----------|----------|
| **On-Demand** | Short-term, unpredictable workloads | None (most expensive) |
| **Reserved Instances** | Steady usage, 1 or 3 year commitment | Up to ~72% |
| **Savings Plans** | Commit to $/hour spend, flexible | Up to ~72% |
| **Spot Instances** | Fault-tolerant jobs that can be interrupted | Up to ~90% |
| **Dedicated Hosts** | Licensing or compliance needs | Most expensive |

### Free Tier
- **Always free:** e.g. Lambda 1M requests/month, DynamoDB 25 GB
- **12 months free** (older accounts) / **credits-based** (newer accounts): e.g. limited EC2 t2/t3.micro hours, S3 storage
- **Trials:** short-term free trials of specific services

> ⚠️ Free Tier rules change over time. Always check the current details at https://aws.amazon.com/free/

### Cost tools
- **AWS Budgets**: set a budget and get email alerts
- **Billing Dashboard / Cost Explorer**: see where money goes
- **Pricing Calculator**: estimate costs before building (https://calculator.aws)

---

## 6. 🛠️ Hands-on: Set up your AWS account safely

Do these in order:

- [ ] Create an AWS account at https://aws.amazon.com
- [ ] **Enable MFA on the root account** (Account menu → Security credentials)
- [ ] Create a **budget** with an alert (Billing → Budgets → "Zero spend budget" or a monthly $5 budget)
- [ ] Turn on **Free Tier usage alerts** (Billing → Billing preferences)
- [ ] Create an **admin IAM user** (or use IAM Identity Center) for daily use, and **stop using root** (covered properly in 02-iam.md)
- [ ] Pick your default **Region** (top right of the console), e.g. `ap-south-1` (Mumbai) if you're in India
- [ ] Explore the console: open EC2 and notice the region selector and AZs

---

## 7. ✅ Quick self-check

1. What's the difference between a Region and an Availability Zone?
2. Why should you deploy across at least 2 AZs?
3. Who is responsible for patching the OS on an EC2 instance? And for RDS?
4. Which EC2 pricing option is cheapest, and what's the catch?
5. Name 2 global AWS services.
6. Is data transfer into AWS charged?

<details>
<summary>Answers</summary>

1. A Region is a geographic area; an AZ is an isolated data center group inside a Region.
2. If one AZ fails, your app keeps running in the other (high availability).
3. EC2: you. RDS: AWS.
4. Spot: up to 90% off, but AWS can take the instance back with a 2-minute warning.
5. IAM, Route 53, CloudFront (any two).
6. Usually no; data transfer OUT is what's charged.

</details>

---

## 📝 My notes

### Summary: what we learned
- **Cloud = renting** servers, storage and databases over the internet. Pay as you go instead of buying hardware upfront.
- **Service models:** IaaS (EC2, you manage the OS) → PaaS (Elastic Beanstalk, you manage only the app) → SaaS (you just use it).
- **Global infrastructure:**
  - **Region** = a geographic area (e.g. `ap-south-1` Mumbai). Pick one based on compliance, latency, service availability and price.
  - **Availability Zone (AZ)** = isolated data center(s) inside a Region. Use **at least 2 AZs** for high availability.
  - **Edge Locations** = CDN/DNS points used by CloudFront and Route 53.
  - Most services are **regional**. IAM, Route 53 and CloudFront are **global**.
- **Shared Responsibility:** AWS secures the cloud itself (hardware, data centers). **You** secure what you put in it (data, IAM, OS patches on EC2, firewall rules). The more managed the service, the less you manage.
- **Pricing:** you pay for compute, storage and **data going out** (data coming in is usually free).
  - EC2 options: On-Demand, Reserved / Savings Plans (~72% off with a commitment), **Spot** (~90% off but can be interrupted).
- **Account safety first:** MFA on root, a budget alert, and an admin user for daily work.

<!-- Add your own notes, doubts and learnings here -->
