# 03 — Compute (EC2, Load Balancing, Auto Scaling, Lambda, Containers)

AWS gives you three main ways to run code:

| Option | You manage | Good for | Service |
|--------|-----------|----------|---------|
| **Virtual machines** | OS + everything above it | Full control, legacy apps, long-running servers | EC2 |
| **Containers** | Container images (+ cluster, unless Fargate) | Microservices, portable apps | ECS, EKS, Fargate |
| **Serverless functions** | Just your code | Event-driven, short tasks, spiky traffic | Lambda |

---

## Part A — EC2 (Elastic Compute Cloud)

EC2 = renting virtual servers (called **instances**) in the cloud.

### 1. What you choose when launching an instance

```
Launch EC2 instance
 ├── AMI               → which OS/software image
 ├── Instance type     → how much CPU / RAM / network
 ├── Key pair          → SSH login key
 ├── Network           → VPC, subnet, public IP or not
 ├── Security group    → firewall rules
 ├── Storage           → EBS volume(s)
 ├── IAM role          → permissions for code on the instance (from lesson 02!)
 └── User data         → script that runs on first boot
```

### 2. AMI (Amazon Machine Image)
- A template for the instance: OS + pre-installed software + configuration
- Types:
  - **AWS-provided**: Amazon Linux 2023, Ubuntu, Windows Server, etc.
  - **Marketplace**: third-party images (some paid)
  - **Your own**: build a server the way you like, then create an AMI from it to launch identical copies
- AMIs are **regional**, but you can copy them to other regions

### 3. Instance types
Name format: **`m7g.large`**
```
m      7      g        .large
│      │      │          └── size (nano, micro, small, medium, large, xlarge, 2xlarge ...)
│      │      └── extra attributes (g = Graviton/ARM, i = Intel, a = AMD, n = networking, d = local disk)
│      └── generation (higher = newer, usually cheaper per performance)
└── family
```

| Family | Optimized for | Example use |
|--------|---------------|-------------|
| **t** (t3, t4g) | General purpose, **burstable** CPU | Dev/test, small websites. Free Tier is usually t2/t3.micro |
| **m** (m7g, m7i) | General purpose, balanced | App servers, backends |
| **c** (c7g) | Compute | Batch processing, gaming servers, video encoding |
| **r / x** (r7g) | Memory | Large databases, in-memory caches |
| **i / d** | Storage (fast local disk) | High I/O databases, data warehousing |
| **p / g / inf / trn** | Accelerated (GPU / ML chips) | ML training and inference, graphics |

> 💡 **Graviton (`g`)** instances use AWS's ARM chips: often ~20–40% better price/performance if your software runs on ARM.

### 4. Security Groups (instance firewall)
- Virtual firewall attached to instances
- **Only ALLOW rules**, no deny rules
- **Stateful**: if inbound traffic is allowed, the response goes out automatically
- Default: **all inbound blocked**, **all outbound allowed**
- Rules specify: protocol, port, source (IP range **or another security group**)

Example for a web server:
| Type | Port | Source | Why |
|------|------|--------|-----|
| SSH | 22 | **My IP only** | Admin login |
| HTTP | 80 | 0.0.0.0/0 | Public website |
| HTTPS | 443 | 0.0.0.0/0 | Public website |

> ⚠️ Never open SSH (22) or RDP (3389) to `0.0.0.0/0` (the whole internet).
> 💡 A source can be another security group: e.g. "DB allows port 5432 only from the App security group". This is very common.

**Common ports to remember:** 22 SSH · 80 HTTP · 443 HTTPS · 3389 RDP · 3306 MySQL · 5432 PostgreSQL

### 5. Connecting to an instance
| Method | Notes |
|--------|-------|
| **SSH with key pair** | `ssh -i mykey.pem ec2-user@<public-ip>` (user is `ubuntu` on Ubuntu). Needs port 22 open |
| **EC2 Instance Connect** | Browser-based SSH from the console |
| **Session Manager (SSM)** | ⭐ Best practice: no open ports, no SSH keys, access controlled by IAM, sessions logged. Needs the instance role `AmazonSSMManagedInstanceCore` |
| **RDP** | For Windows instances |

### 6. Storage for EC2 (intro — more in lesson 04)
- **EBS volume**: network drive attached to the instance; persists independently. Root volume is deleted on termination by default
- **Instance store**: physical disk on the host; very fast but **data is lost when the instance stops**
- **Snapshots**: backups of EBS volumes, stored in S3 behind the scenes

### 7. User data (bootstrapping)
A script that runs **once, as root, on first boot**. Example to install a web server on Amazon Linux:
```bash
#!/bin/bash
dnf update -y
dnf install -y httpd
systemctl enable --now httpd
echo "<h1>Hello from $(hostname -f)</h1>" > /var/www/html/index.html
```

### 8. Instance lifecycle
```
pending → running ⇄ stopping → stopped
              │
              └→ shutting-down → terminated (gone forever)
```
- **Stopped**: you don't pay for compute, but **you still pay for EBS storage**
- **Terminated**: instance is deleted
- **Hibernate**: saves RAM to disk so it resumes quickly
- Public IP **changes** when you stop/start. Use an **Elastic IP** for a fixed one (charged; all public IPv4 addresses cost money now)

### 9. Pricing options (recap from lesson 01)
On-Demand · Reserved Instances · Savings Plans · **Spot** (up to 90% off, can be interrupted) · Dedicated Hosts

### 10. Placement groups (good to know)
- **Cluster**: instances close together in one AZ → lowest latency (HPC)
- **Spread**: each instance on different hardware → max fault isolation
- **Partition**: groups of instances on separate racks → big distributed systems (Kafka, Cassandra, HDFS)

---

## Part B — Elastic Load Balancing (ELB)

A load balancer **spreads incoming traffic across multiple targets** (instances, containers, IPs, Lambda) and stops sending traffic to unhealthy ones.

```
            Users
              │
      ┌───────▼────────┐
      │ Load Balancer  │   (one DNS name, spans multiple AZs)
      └──┬──────────┬──┘
         │          │
   ┌─────▼──┐   ┌───▼────┐
   │ EC2 #1 │   │ EC2 #2 │
   │ AZ  a  │   │ AZ  b  │
   └────────┘   └────────┘
```

### Types
| Type | Layer | Use for |
|------|-------|---------|
| **Application LB (ALB)** | 7 (HTTP/HTTPS) | Web apps, microservices. Routes by **path** (`/api/*`), **host** (`api.example.com`), headers, query strings |
| **Network LB (NLB)** | 4 (TCP/UDP/TLS) | Extreme performance, millions of req/sec, **static IPs**, non-HTTP protocols |
| **Gateway LB (GWLB)** | 3 (IP) | Routing traffic through third-party firewalls / inspection appliances |
| Classic LB | old | Legacy, don't use for new work |

### Key concepts
- **Listener**: port/protocol the LB accepts (e.g. HTTPS:443)
- **Target group**: the set of targets traffic is sent to
- **Health checks**: LB pings each target (e.g. `GET /health`); unhealthy ones get no traffic
- **SSL/TLS termination**: LB handles HTTPS using a free certificate from **ACM (AWS Certificate Manager)**
- **Sticky sessions**: send a user to the same target each time (use only if needed)
- Best practice: instance security group allows traffic **only from the LB's security group**

---

## Part C — Auto Scaling Groups (ASG)

An ASG **automatically adds or removes EC2 instances** based on demand, and replaces unhealthy ones.

### Settings
- **Launch template**: what to launch (AMI, instance type, security group, user data, IAM role)
- **Min / Desired / Max** capacity, e.g. min 2, desired 2, max 6
- **Subnets in multiple AZs**: ASG spreads instances across them
- Attach to an **ALB target group** so new instances get traffic automatically

### Scaling policies
| Policy | How it works | Example |
|--------|-------------|---------|
| **Target tracking** ⭐ | Keep a metric at a target | Keep average CPU at 50% |
| **Step / simple** | Add/remove N instances when an alarm fires | +2 instances if CPU > 80% |
| **Scheduled** | Change capacity at set times | Scale up at 9 AM weekdays |
| **Predictive** | ML forecasts load from history | Daily traffic patterns |

- **Cooldown**: wait period after a scaling action so it doesn't overreact
- **Self-healing**: if an instance fails its health check, the ASG terminates it and launches a new one

> 💡 **ALB + ASG across 2+ AZs** is the classic highly available, scalable web architecture.

---

## Part D — Lambda (serverless)

Lambda runs your code **without you managing any servers**. You upload a function; AWS runs it when an event happens.

### Key facts
- Languages: Python, Node.js, Java, Go, .NET, Ruby, or anything via a custom runtime / container image
- **Pay per request + execution time** (GB-seconds). Nothing running = nothing billed
- Scales automatically from 0 to thousands of concurrent executions
- **Max timeout: 15 minutes** (not for long-running jobs)
- Memory: 128 MB – 10 GB (CPU scales up with memory)
- Permissions come from an **execution role** (IAM role again!)

### Common triggers
| Trigger | Example |
|---------|---------|
| API Gateway / Function URL | Build a REST API |
| S3 | Create a thumbnail when an image is uploaded |
| DynamoDB Streams | React to database changes |
| SQS / SNS / EventBridge | Process messages and events |
| EventBridge Scheduler | Run a cron job every night |

### Example (Python)
```python
def lambda_handler(event, context):
    name = event.get("name", "world")
    return {"statusCode": 200, "body": f"Hello, {name}!"}
```

### Things to know
- **Cold start**: the first call after idle is slower while AWS sets up the environment (reduced with Provisioned Concurrency or SnapStart)
- Store config in **environment variables**; secrets in **Secrets Manager / Parameter Store**
- **Layers**: share libraries between functions
- Logs go automatically to **CloudWatch Logs**

### EC2 vs Lambda — when to use which
| Use EC2 when | Use Lambda when |
|-------------|-----------------|
| Long-running processes (>15 min) | Short, event-driven tasks |
| Need full OS control | Don't want to manage servers |
| Steady, constant high load | Spiky or unpredictable traffic |
| Special software or GPUs | Glue code between AWS services |

---

## Part E — Containers

A **container** packages your app with everything it needs (runtime, libraries) so it runs the same anywhere. Usually built with **Docker**.

| Service | What it is |
|---------|-----------|
| **ECR** (Elastic Container Registry) | Private storage for your Docker images (like Docker Hub, inside AWS) |
| **ECS** (Elastic Container Service) | AWS's own container orchestrator: simpler, tightly integrated with AWS |
| **EKS** (Elastic Kubernetes Service) | Managed **Kubernetes**: industry standard, portable, more complex |
| **Fargate** | **Serverless compute for containers**: run ECS/EKS tasks without managing EC2 instances |

### ECS vocabulary
- **Task definition**: blueprint for a container (image, CPU/RAM, ports, env vars, IAM task role)
- **Task**: a running copy of a task definition
- **Service**: keeps N tasks running, replaces failed ones, connects to a load balancer
- **Cluster**: logical group of tasks/services
- **Launch type**: **Fargate** (serverless) or **EC2** (you manage the instances)

> 💡 **Beginner path:** ECS + Fargate. Learn EKS only if you need Kubernetes.

### Other compute services (good to know)
- **Elastic Beanstalk**: upload your code, AWS provisions EC2 + ALB + ASG for you (PaaS)
- **App Runner**: run a container or source repo as a web service with almost no configuration
- **Lightsail**: simple, fixed-price VPS for small projects
- **AWS Batch**: run large batch jobs

---

## 🛠️ Hands-on

> ⚠️ **Cost safety:** use Free Tier-eligible types (`t2.micro` / `t3.micro`), and **terminate everything** at the end of each lab. Load balancers and NAT gateways are **not** free: delete them the same day.

### Lab 1 — Launch your first web server
- [ ] EC2 → Launch instance → name `web-1`
- [ ] AMI: **Amazon Linux 2023** · Type: **t2.micro / t3.micro** (Free Tier eligible)
- [ ] Create a key pair (download the `.pem` file; keep it safe)
- [ ] Security group: allow **HTTP (80) from anywhere**, **SSH (22) from My IP**
- [ ] Advanced details → User data → paste the user data script from section A7
- [ ] Launch, wait ~1 minute, open `http://<public-ip>` in your browser → you should see "Hello from ..."
- [ ] Connect with **EC2 Instance Connect** from the console and run `curl localhost`

### Lab 2 — Connect with SSH from your PC
- [ ] In PowerShell: `ssh -i C:\path\to\mykey.pem ec2-user@<public-ip>`
- [ ] If you get a permissions error on the key: right-click the `.pem` → Properties → Security → leave only your user with read access
- [ ] Stop the instance, start it again, and notice the **public IP changed**

### Lab 3 — Load balancer + Auto Scaling
- [ ] Create a **launch template** from your Lab 1 settings (same AMI, type, SG, user data)
- [ ] Create a **target group** (HTTP:80, health check path `/`)
- [ ] Create an **ALB** across at least 2 AZs, listener HTTP:80 → your target group
- [ ] Create an **ASG** from the launch template: min 2 / desired 2 / max 4, same AZs, attach to the target group
- [ ] Open the ALB's DNS name and refresh several times: the hostname should change (different instances)
- [ ] **Terminate one instance manually** → watch the ASG launch a replacement (self-healing)
- [ ] 🧹 **Clean up:** delete the ASG (it terminates its instances), then the ALB, target group and launch template

### Lab 4 — Your first Lambda function
- [ ] Lambda → Create function → Author from scratch → name `hello`, runtime **Python 3.x**
- [ ] Paste the example code from section D, click **Deploy**
- [ ] Test tab → create a test event `{"name": "Aayush"}` → **Test**
- [ ] Check the output and open **Monitor → View CloudWatch logs**
- [ ] Configuration → **Function URL** → create (auth: NONE) → open the URL in the browser
- [ ] 🧹 Delete the function URL (or the function) when done

### Lab 5 (optional) — Run a container on ECS Fargate
- [ ] ECS → Create cluster (Fargate)
- [ ] Create a task definition using the public image `public.ecr.aws/nginx/nginx:latest`, port 80
- [ ] Run a task in a public subnet with a public IP and a security group allowing port 80
- [ ] Open the task's public IP → nginx welcome page
- [ ] 🧹 Stop the task and delete the cluster

---

## ✅ Quick self-check

1. What does `c7g.xlarge` tell you about an instance?
2. Are security groups stateful or stateless? Can they have deny rules?
3. You stop an EC2 instance. What do you still pay for?
4. What's the best way to access an EC2 instance without opening port 22?
5. ALB vs NLB: which would you use for path-based routing? Which for a static IP?
6. What three numbers do you set on an Auto Scaling Group?
7. What's the maximum runtime of a Lambda function?
8. How does a Lambda function get permission to write to DynamoDB?
9. What is Fargate?
10. An instance in an ASG becomes unhealthy. What happens?

<details>
<summary>Answers</summary>

1. Compute-optimized family (`c`), 7th generation, Graviton/ARM processor (`g`), size xlarge.
2. Stateful. No, security groups only have allow rules.
3. The attached EBS volumes (and any Elastic IP / public IPv4 address).
4. **Session Manager** (AWS Systems Manager), using an IAM role on the instance.
5. ALB for path-based routing (layer 7). NLB for a static IP (layer 4).
6. Minimum, desired and maximum capacity.
7. 15 minutes.
8. Through its **execution role**, an IAM role with DynamoDB permissions.
9. Serverless compute for containers: you run ECS/EKS tasks without managing the underlying EC2 instances.
10. The ASG terminates it and launches a replacement to keep the desired capacity.

</details>

---

## 📝 My notes

### Summary: what we learned
- **Three ways to run code:** EC2 (virtual machines, full control) → containers (ECS/EKS/Fargate) → Lambda (serverless, just code).
- **EC2:**
  - Launch = AMI + instance type + key pair + network + security group + EBS storage + IAM role + user data.
  - Instance types: `m7g.large` = family (m) + generation (7) + attributes (g = Graviton/ARM) + size. t = burstable, m = general, c = compute, r = memory, p/g = GPU.
  - **Security groups**: stateful, **allow rules only**, all inbound blocked by default. Never open SSH (22) to `0.0.0.0/0`. A rule's source can be another security group.
  - Connect via SSH, EC2 Instance Connect, or ⭐ **Session Manager** (no open ports, no keys).
  - **User data** = a script that runs once as root on first boot.
  - A stopped instance still costs money for its **EBS storage**. The public IP changes on stop/start (an Elastic IP stays fixed).
- **Load balancers (ELB):**
  - **ALB** (layer 7, HTTP): routes by path or host. **NLB** (layer 4, TCP/UDP): fastest, static IPs. **GWLB**: for firewall appliances.
  - Listener → target group → health checks. Free HTTPS certificates from ACM.
- **Auto Scaling Groups:**
  - Launch template + **min / desired / max** + subnets in multiple AZs + attach to an ALB target group.
  - Scaling policies: target tracking ⭐ (e.g. keep CPU at 50%), step, scheduled, predictive.
  - **Self-healing**: unhealthy instances are replaced automatically.
  - **ALB + ASG across 2+ AZs** = the classic highly available web architecture.
- **Lambda:**
  - Runs on events (API Gateway, S3, SQS, EventBridge, schedules). Pay per request + duration.
  - **15 min max**, 128 MB–10 GB memory, permissions through an **execution role**, logs in CloudWatch.
  - Cold starts happen on the first call after idle.
- **Containers:** ECR (image registry), **ECS** (AWS's own orchestrator, simpler), **EKS** (Kubernetes), **Fargate** (serverless containers). Beginner path: **ECS + Fargate**.
- **Others:** Elastic Beanstalk (PaaS), App Runner, Lightsail, Batch.
- ⚠️ Always delete the ALB, ASG and instances after labs.

<!-- Add your own notes, doubts and learnings here -->
