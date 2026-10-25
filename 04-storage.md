# 04 — Storage (S3, Glacier, EBS, EFS)

AWS has three kinds of storage:

| Type | How you use it | Analogy | AWS service |
|------|---------------|---------|-------------|
| **Object storage** | Store/retrieve whole files over HTTP (API) | Google Drive / Dropbox | **S3**, S3 Glacier |
| **Block storage** | A disk attached to **one** server | Hard drive / SSD in your PC | **EBS**, instance store |
| **File storage** | A shared network drive mounted by **many** servers | Shared office network folder | **EFS**, FSx |

---

## Part A — S3 (Simple Storage Service)

S3 stores **any amount of data as objects** in **buckets**. It's one of the oldest and most-used AWS services: backups, images, videos, logs, static websites, data lakes.

### 1. Core concepts
- **Bucket**: a container for objects
  - Name must be **globally unique** (across all AWS accounts), lowercase, 3–63 chars, no underscores
  - Created **in a Region** (but the S3 console shows all buckets globally)
- **Object**: a file + metadata
  - Max object size: **5 TB** (use **multipart upload** for anything over ~100 MB; required over 5 GB)
  - Identified by its **key**: the full "path", e.g. `photos/2026/trip.jpg`
- **No real folders.** "Folders" are just key prefixes (`photos/2026/`) that the console displays like folders
- Unlimited storage, you pay per GB stored + requests + data transfer out
- **Durability: 99.999999999% (11 nines)**: you essentially never lose data
- **Strong read-after-write consistency**: after a write, reads immediately return the new data

Object URL format:
```
https://<bucket-name>.s3.<region>.amazonaws.com/<key>
https://my-learning-bucket.s3.ap-south-1.amazonaws.com/photos/trip.jpg
```

### 2. Storage classes

| Class | Use for | Retrieval | Min. storage duration |
|-------|---------|-----------|-----------------------|
| **S3 Standard** | Frequently accessed data | Instant | None |
| **S3 Intelligent-Tiering** ⭐ | Unknown/changing access patterns. Moves objects between tiers automatically | Instant (for default tiers) | None |
| **S3 Standard-IA** (Infrequent Access) | Accessed rarely, needed fast (backups) | Instant, per-GB retrieval fee | 30 days |
| **S3 One Zone-IA** | Re-creatable, infrequent data. Stored in **1 AZ only** | Instant, retrieval fee | 30 days |
| **S3 Express One Zone** | Ultra-low latency, single AZ (ML, analytics) | Single-digit ms | None |
| **Glacier Instant Retrieval** | Archives accessed ~once a quarter | Milliseconds | 90 days |
| **Glacier Flexible Retrieval** | Archives, rarely accessed | Minutes to 12 hours | 90 days |
| **Glacier Deep Archive** | Long-term archives (compliance, 7–10 years) | 12–48 hours. **Cheapest** | 180 days |

> 💡 Cheaper storage = more expensive / slower retrieval. If you don't know the access pattern, use **Intelligent-Tiering**.

### 3. Versioning
- Keeps **every version** of an object when you overwrite or delete it
- Deleting a versioned object just adds a **delete marker**; older versions can be restored
- Protects against accidental deletes and overwrites
- Enabled at the bucket level. Once enabled, it can only be **suspended**, not turned off
- You pay for every stored version, so pair it with lifecycle rules

### 4. Lifecycle rules
Automatically move or delete objects as they age. Example:
```
Day 0     → S3 Standard
Day 30    → Standard-IA
Day 90    → Glacier Flexible Retrieval
Day 365   → Delete
```
Also useful: delete old non-current versions, and clean up incomplete multipart uploads.

### 5. Security ⭐

**By default, everything in S3 is private.** Ways to control access:

| Mechanism | What it does |
|-----------|-------------|
| **IAM policies** | What *your* users/roles can do in S3 (lesson 02) |
| **Bucket policy** | Resource-based JSON policy on the bucket: who (`Principal`) can do what. Used for public access, cross-account access, forcing HTTPS |
| **Block Public Access** | Account/bucket-level safety switch. **On by default; keep it on** unless you really need public data |
| ACLs | Old mechanism, **disabled by default** ("Bucket owner enforced"). Don't use |
| **Pre-signed URLs** | Temporary link (e.g. valid 1 hour) that lets anyone download/upload one specific object without AWS credentials |

Example bucket policy, **public read** for a static website:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "PublicRead",
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-website-bucket/*"
  }]
}
```

### 6. Encryption
- **At rest** (on disk):
  - **SSE-S3**: AWS-managed keys. **Enabled by default on all new objects**
  - **SSE-KMS**: keys in KMS. You control the key, and every use is logged in CloudTrail
  - **DSSE-KMS**: double encryption with KMS
  - **SSE-C**: you provide your own key with each request
  - **Client-side**: you encrypt before uploading
- **In transit**: HTTPS. You can force it with a bucket policy condition `aws:SecureTransport = false → Deny`

### 7. Other S3 features
| Feature | What it does |
|---------|-------------|
| **Static website hosting** | Serve HTML/CSS/JS straight from a bucket (put CloudFront in front for HTTPS + speed) |
| **Replication** | Copy objects automatically to another bucket: **CRR** (cross-region) or **SRR** (same-region). Needs versioning on both buckets |
| **Event notifications** | Trigger Lambda, SQS, SNS or EventBridge when objects are created/deleted |
| **Object Lock** | **WORM** (write once, read many): objects can't be deleted or changed for a set period. For compliance |
| **Transfer Acceleration** | Faster long-distance uploads via edge locations |
| **Multipart upload** | Upload large files in parallel parts |
| **Access logs / Storage Lens** | See who accessed what, and analyze usage and cost |
| **S3 Select / Athena** | Run SQL queries on data stored in S3 (Athena covered later) |

---

## Part B — EBS (Elastic Block Store)

EBS is a **network-attached disk** for EC2 instances.

### Key facts
- An EBS volume lives in **one AZ** and can only attach to instances **in that same AZ**
- Normally attached to **one instance at a time** (io1/io2 support **Multi-Attach** in special cases)
- Data **persists** independently of the instance: you can detach a volume and attach it to another instance
- Root volume is **deleted on termination by default**; extra volumes are **kept** by default (configurable with "Delete on termination")
- Billed for **provisioned size**, not used space (a 100 GB volume with 5 GB of data costs 100 GB)
- You can **increase size or change type** while the volume is in use

### Volume types
| Type | Kind | Use for |
|------|------|---------|
| **gp3** ⭐ | General purpose SSD | Default choice: boot volumes, most apps. Set IOPS/throughput independently of size |
| gp2 | General purpose SSD (older) | Legacy: IOPS tied to size. Prefer gp3 |
| **io2 Block Express / io1** | Provisioned IOPS SSD | Critical, high-performance databases |
| **st1** | Throughput HDD | Big sequential data: logs, data warehouses. Can't be a boot volume |
| **sc1** | Cold HDD | Rarely accessed data, lowest cost. Can't be a boot volume |

### Snapshots
- **Point-in-time backup** of a volume, stored in S3 (managed by AWS, you don't see the bucket)
- **Incremental**: only changed blocks are saved after the first snapshot
- Use a snapshot to:
  - Restore a volume
  - **Move a volume to another AZ** (snapshot → create a new volume in a different AZ)
  - Copy to another Region (disaster recovery)
  - Create an AMI
- Automate with **Data Lifecycle Manager** or **AWS Backup**
- **Recycle Bin**: recover accidentally deleted snapshots
- **Snapshot Archive**: ~75% cheaper storage for snapshots you rarely restore

### Encryption
- Encrypts data at rest, in transit between instance and volume, and snapshots, using **KMS**
- Almost no performance impact. You can turn on **"encryption by default"** for your account per Region ⭐
- To encrypt an unencrypted volume: snapshot → copy the snapshot with encryption → create a volume from it

### EBS vs Instance Store
| | EBS | Instance store |
|--|-----|---------------|
| Location | Network drive | Physical disk on the host |
| Persistence | Survives stop/start | **Lost on stop, terminate or hardware failure** |
| Speed | Fast | Very fast (very high IOPS) |
| Use for | Almost everything | Caches, buffers, temporary data |

---

## Part C — EFS and other file storage

### EFS (Elastic File System)
- A managed **NFS** shared file system for **Linux** instances
- **Many instances across multiple AZs** can mount it at the same time
- **Grows and shrinks automatically**. No size to provision; pay for what you store
- More expensive per GB than EBS, but shared
- Storage classes: Standard, Infrequent Access, Archive (with lifecycle rules, like S3)
- Access controlled by security groups (port **2049** NFS) and IAM
- Use for: shared content for web servers, home directories, shared ML datasets, containers that need shared storage

### FSx (good to know)
Managed third-party file systems:
- **FSx for Windows File Server**: SMB, Active Directory, for Windows apps
- **FSx for Lustre**: high-performance computing and ML, integrates with S3
- **FSx for NetApp ONTAP** / **OpenZFS**: lift-and-shift enterprise NAS workloads

### EBS vs EFS vs S3
| | EBS | EFS | S3 |
|--|-----|-----|-----|
| Type | Block | File (NFS) | Object |
| Access from | 1 instance, same AZ | Many instances, many AZs | Anywhere, over HTTP (API) |
| Size | You provision | Auto-scaling | Unlimited |
| Pricing | Provisioned GB | GB used | GB used + requests |
| Typical use | OS disk, databases | Shared files | Backups, media, static sites, data lakes |

---

## Part D — Moving data & backups (good to know)

| Service | What it does |
|---------|-------------|
| **AWS Backup** | Central place to schedule and manage backups for EBS, EFS, RDS, DynamoDB, S3 and more |
| **DataSync** | Copy large amounts of data online between on-prem and AWS, or between AWS storage services |
| **Storage Gateway** | Connect on-prem servers to cloud storage (hybrid): file, volume and tape gateways |
| **Snow Family** (Snowball) | Physical devices AWS ships to you to move huge data offline |
| **Transfer Family** | Managed SFTP/FTPS/FTP servers backed by S3 or EFS |

---

## 🛠️ Hands-on

> ⚠️ **Cost safety:** S3 and EBS have Free Tier allowances, but **delete everything** after each lab. Stopped instances still pay for EBS, and snapshots cost money too.

### Lab 1 — S3 basics
- [ ] S3 → Create bucket → name like `aayush-aws-learning-<random>`, Region `ap-south-1`
- [ ] Keep **Block Public Access ON**, keep default encryption (SSE-S3)
- [ ] Upload a few files, including one inside a "folder" (e.g. `notes/test.txt`)
- [ ] Open an object → click its **Object URL** → you get **Access Denied** (it's private)
- [ ] Click **Open** instead → it works, because the console makes a **pre-signed URL** for you

### Lab 2 — S3 with the CLI
```powershell
aws s3 ls                                               # list buckets
aws s3 cp .\README.md s3://<your-bucket>/docs/         # upload
aws s3 ls s3://<your-bucket>/ --recursive               # list objects
aws s3 sync . s3://<your-bucket>/aws-notes/             # sync this folder
aws s3 presign s3://<your-bucket>/docs/README.md --expires-in 300   # 5-minute link
aws s3 rm s3://<your-bucket>/docs/README.md             # delete
```
- [ ] Open the pre-signed link in a browser. Wait 5 minutes and try again: it should have expired

### Lab 3 — Versioning & lifecycle
- [ ] Bucket → Properties → **enable Versioning**
- [ ] Upload `test.txt`, edit it locally, upload again → turn on **Show versions** → see both versions
- [ ] Delete the object → see the **delete marker** → delete the marker → the file is back
- [ ] Management → **Create lifecycle rule**: move to Standard-IA after 30 days, delete non-current versions after 30 days

### Lab 4 — Host a static website
- [ ] Create a new bucket `aayush-site-<random>`
- [ ] Upload an `index.html`:
  ```html
  <!DOCTYPE html>
  <html><body><h1>My first S3 website 🚀</h1></body></html>
  ```
- [ ] Properties → **Static website hosting** → Enable → index document `index.html`
- [ ] Permissions → turn **off** Block Public Access *for this bucket only*
- [ ] Add the public-read **bucket policy** from section A5 (change the bucket name)
- [ ] Open the **website endpoint** URL from Properties → your page loads
- [ ] (In the networking lesson we'll put CloudFront in front of it for HTTPS)

### Lab 5 — EBS volume and snapshot
- [ ] Launch a t2/t3.micro instance (Amazon Linux) and note its **AZ**
- [ ] EC2 → Volumes → Create volume: **gp3, 1 GiB, same AZ** → Attach to the instance
- [ ] Connect (Instance Connect) and run:
  ```bash
  lsblk                                    # find the new disk, e.g. nvme1n1 or xvdf
  sudo mkfs -t xfs /dev/nvme1n1            # format it (use the name from lsblk)
  sudo mkdir /data && sudo mount /dev/nvme1n1 /data
  echo "hello ebs" | sudo tee /data/hello.txt
  ```
- [ ] Create a **snapshot** of the volume
- [ ] Create a new volume **from the snapshot in a different AZ**: this is how you move data across AZs
- [ ] 🧹 Detach and delete both volumes, delete the snapshot, terminate the instance

### Lab 6 (optional) — EFS shared between two instances
- [ ] Create an EFS file system (default settings) in your default VPC
- [ ] Launch 2 instances in **different AZs**; make sure the EFS security group allows **NFS 2049** from their security group
- [ ] On both instances, mount it (the EFS console's **Attach** button shows the exact command)
- [ ] Create a file on instance 1 → it appears on instance 2
- [ ] 🧹 Terminate the instances and delete the file system

### 🧹 Final clean-up
- [ ] Empty and delete all lab buckets (a bucket must be empty before it can be deleted. With versioning, delete all versions too)

---

## ✅ Quick self-check

1. What's the difference between object, block and file storage?
2. Do S3 folders really exist?
3. What's the maximum size of one S3 object?
4. Which storage class would you use if you don't know how often data will be accessed?
5. Which is the cheapest storage class, and what's the catch?
6. A user deletes an object in a versioned bucket. What actually happens?
7. How do you give someone temporary access to one private object?
8. Can an EBS volume in `ap-south-1a` attach to an instance in `ap-south-1b`? How would you move it?
9. What happens to instance store data when you stop the instance?
10. You need a file system shared by 10 Linux instances across 3 AZs. Which service?
11. Which EBS volume type is the default choice for most workloads?
12. Are new S3 objects encrypted by default?

<details>
<summary>Answers</summary>

1. Object: whole files accessed over HTTP (S3). Block: a raw disk attached to one server (EBS). File: a shared file system mounted by many servers (EFS).
2. No. They're just key prefixes like `photos/2026/` that the console shows as folders.
3. 5 TB (use multipart upload for large files).
4. S3 Intelligent-Tiering.
5. Glacier Deep Archive. Retrieval takes 12–48 hours, and there's a 180-day minimum storage charge.
6. A delete marker is added. Older versions still exist, and removing the marker restores the object.
7. A pre-signed URL.
8. No, EBS is locked to one AZ. Take a snapshot, then create a new volume from it in the other AZ.
9. It's lost.
10. EFS.
11. gp3.
12. Yes, with SSE-S3.

</details>

---

## 📝 My notes

### Summary: what we learned
- **Three storage types:** object (**S3**, over HTTP), block (**EBS**, a disk for one instance), file (**EFS**, shared by many instances).
- **S3:**
  - **Buckets** (globally unique names, created in a Region) hold **objects** (max 5 TB) identified by **keys**. Folders are just key prefixes.
  - 11 nines durability, strong consistency, unlimited storage.
  - **Storage classes:** Standard → Intelligent-Tiering ⭐ → Standard-IA / One Zone-IA → Glacier Instant / Flexible / **Deep Archive** (cheapest, 12–48 h retrieval). Cheaper storage = slower or pricier retrieval + minimum storage days.
  - **Versioning** keeps every version; a delete just adds a delete marker. **Lifecycle rules** move or delete objects as they age.
  - **Security:** private by default. **Block Public Access** stays on. Control access with IAM policies + **bucket policies** (with `Principal`). ACLs are legacy. **Pre-signed URLs** give temporary access.
  - **Encryption:** SSE-S3 is on by default. SSE-KMS gives you control + audit logs. Force HTTPS with a bucket policy.
  - Other features: static website hosting, replication (CRR/SRR, needs versioning), event notifications (trigger Lambda), Object Lock (WORM), multipart upload.
- **EBS:**
  - A network disk **locked to one AZ**, usually attached to one instance. Billed for **provisioned** size.
  - Types: **gp3** ⭐ default, io2 for heavy databases, st1/sc1 HDD (can't boot).
  - **Snapshots** are incremental backups, used to restore, **move across AZs/Regions**, or create AMIs.
  - Turn on encryption by default (KMS).
  - **Instance store** = very fast but **temporary** (lost on stop).
- **EFS:** managed NFS for Linux, **shared across many instances and AZs**, auto-grows, port 2049. **FSx** for Windows (SMB), Lustre (HPC/ML), NetApp, OpenZFS.
- **Data movement & backup:** AWS Backup, DataSync, Storage Gateway (hybrid), Snowball (offline), Transfer Family (SFTP).
- ⚠️ Empty and delete buckets, volumes and snapshots after labs.

<!-- Add your own notes, doubts and learnings here -->
