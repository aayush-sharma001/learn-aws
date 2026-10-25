# 02 — IAM (Identity and Access Management)

## 1. What is IAM?

IAM controls **who** can do **what** on **which** AWS resources.

- **Who** = authentication ("Are you really Aayush?")
- **What / which** = authorization ("Is Aayush allowed to delete this S3 bucket?")

Key facts:
- IAM is a **global** service (not tied to a Region)
- IAM is **free**
- By default, a new identity can do **nothing**. Everything is denied until you allow it.

---

## 2. The building blocks

```
AWS Account
 ├── Root user          (the email you signed up with — god mode)
 ├── IAM Users          (people or apps with long-term credentials)
 ├── IAM Groups         (collections of users)
 ├── IAM Roles          (temporary permissions anyone/anything can "assume")
 └── IAM Policies       (JSON documents that define permissions)
```

### Root user
- Created when you open the account; has **full, unrestricted access**
- Can do a few things no one else can (close the account, change the support plan)
- ✅ Enable MFA, ✅ delete root access keys, ❌ **never use it for daily work**

### IAM Users
- A person (or, in older setups, an application) that needs access
- Can have:
  - **Console password**: to log in to the web console
  - **Access keys** (Access Key ID + Secret Access Key): for the CLI, SDKs and API calls
- Users have **long-term credentials**, which are a security risk if leaked

### IAM Groups
- A collection of users, e.g. `Developers`, `Admins`, `ReadOnly`
- Attach policies to the **group**; every user in it gets those permissions
- A user can be in multiple groups
- Groups **cannot** contain other groups

### IAM Roles ⭐ (very important)
- An identity with permissions but **no long-term credentials**
- Someone or something **assumes** the role and gets **temporary credentials**
- Used for:
  - **AWS services**: e.g. an EC2 instance that needs to read from S3
  - **Cross-account access**: a user in account A working in account B
  - **Federation / SSO**: logging in with Google, Okta or IAM Identity Center
- **Trust policy** = *who* may assume the role
- **Permission policy** = *what* the role can do

> 💡 **Rule of thumb:** if code runs *inside AWS* (EC2, Lambda, ECS), give it a **role**, never hard-coded access keys.

### IAM Policies
- JSON documents that list what is allowed or denied
- Attached to users, groups or roles

---

## 3. Anatomy of a policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowReadMyBucket",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-learning-bucket",
        "arn:aws:s3:::my-learning-bucket/*"
      ],
      "Condition": {
        "IpAddress": { "aws:SourceIp": "203.0.113.0/24" }
      }
    }
  ]
}
```

| Field | Meaning |
|-------|---------|
| `Version` | Policy language version. Always `"2012-10-17"` |
| `Statement` | List of permission rules |
| `Sid` | Optional label for the statement |
| `Effect` | `Allow` or `Deny` |
| `Action` | API actions, format `service:Action` (e.g. `ec2:StartInstances`). Wildcards work: `s3:*`, `s3:Get*` |
| `Resource` | Which resources, written as **ARNs** |
| `Condition` | Optional: only applies when conditions match (IP, time, MFA, tags...) |

### ARN (Amazon Resource Name)
Every AWS resource has a unique ID called an ARN:
```
arn:partition:service:region:account-id:resource
arn:aws:ec2:ap-south-1:123456789012:instance/i-0abc123
arn:aws:s3:::my-bucket                  ← S3 is global-ish, so region/account are blank
arn:aws:iam::123456789012:user/aayush   ← IAM is global, so region is blank
```

### Types of policies
| Type | Description |
|------|-------------|
| **AWS managed** | Pre-built by AWS, e.g. `AdministratorAccess`, `ReadOnlyAccess`, `AmazonS3ReadOnlyAccess` |
| **Customer managed** | Ones you write; reusable across users/groups/roles |
| **Inline** | Embedded directly in one user/group/role; deleted with it (use sparingly) |
| **Resource-based** | Attached to a *resource* instead of an identity, e.g. an **S3 bucket policy**. Has a `Principal` field saying *who* is allowed |

---

## 4. How AWS evaluates permissions

When a request comes in, AWS checks all applicable policies:

```
1. Is there an explicit DENY anywhere?  → ❌ DENIED (deny always wins)
2. Is there an explicit ALLOW?          → ✅ ALLOWED
3. Neither?                             → ❌ DENIED (implicit deny, the default)
```

> 💡 **Explicit Deny > Explicit Allow > Implicit Deny**

Example: a user's group allows `s3:*`, but a separate policy denies `s3:DeleteBucket`. The user can do everything in S3 **except** delete buckets.

---

## 5. Security best practices

1. **Lock away the root user**: MFA on, no access keys, don't use it day to day
2. **Least privilege**: give only the permissions needed, nothing more. Start small, add when required
3. **Use groups** to assign permissions to people, not individual users
4. **Use roles** for applications and AWS services
5. **Enable MFA** for all human users
6. **Prefer IAM Identity Center (SSO)** for humans in real organizations: temporary credentials instead of long-term passwords and keys
7. **Rotate access keys** and delete unused ones; **never commit keys to Git**
8. **Strong password policy** (length, complexity, rotation)
9. **Review regularly** with the tools below

### Auditing tools
| Tool | What it shows |
|------|---------------|
| **IAM Credential Report** | Account-level CSV of all users and the status of their passwords, keys and MFA |
| **IAM Access Advisor** | Per user/role: which services they actually used and when. Remove what's unused |
| **IAM Access Analyzer** | Finds resources shared outside your account; can generate least-privilege policies from activity |

---

## 6. Other IAM-related concepts (good to know)

- **IAM Identity Center** (formerly AWS SSO): one login for many AWS accounts; the recommended way for humans to access AWS today
- **AWS Organizations**: manage multiple AWS accounts centrally (covered more later)
- **SCPs (Service Control Policies)**: guardrails at the Organization level that cap what *any* identity in an account can do, even admins
- **Permission boundaries**: set the *maximum* permissions a user/role can ever have
- **STS (Security Token Service)**: the service that issues temporary credentials when a role is assumed

---

## 7. 🛠️ Hands-on

### Lab 1 — Stop using root
- [ ] Log in as root → IAM → confirm **MFA is enabled** on root
- [ ] Create a group `Admins` and attach the AWS managed policy `AdministratorAccess`
- [ ] Create a user for yourself (e.g. `aayush-admin`) with console access, and add it to `Admins`
- [ ] Set a **sign-in alias** for your account (IAM Dashboard → Account Alias) so the login URL is easy to remember
- [ ] Log out of root, log in as the new user, and **enable MFA** on it
- [ ] From now on, use this user for everything

### Lab 2 — See least privilege in action
- [ ] Create a group `ReadOnly` with the policy `ReadOnlyAccess`
- [ ] Create a user `test-readonly` in that group
- [ ] Log in as `test-readonly` (use a private/incognito window) and try to launch an EC2 instance or create an S3 bucket: you should get **Access Denied**
- [ ] Delete the test user when done

### Lab 3 — Write your own policy
- [ ] IAM → Policies → Create policy → JSON tab
- [ ] Write a policy that allows **only** `s3:ListAllMyBuckets`
- [ ] Attach it to a test user and verify they can list buckets but not open them
- [ ] Try the **IAM Policy Simulator** (https://policysim.aws.amazon.com) to test policies without logging in as the user

### Lab 4 — Set up the AWS CLI
- [ ] Install the AWS CLI v2: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html
- [ ] Create an access key for your admin user (IAM → Users → Security credentials → Create access key → "CLI")
- [ ] Run `aws configure` and enter the key, secret, default region (e.g. `ap-south-1`) and output format `json`
- [ ] Test it: `aws sts get-caller-identity` should print your user's ARN
- [ ] Try `aws iam list-users`

> ⚠️ The keys are saved in `C:\Users\<you>\.aws\credentials`. Never share this file or paste the secret anywhere public.
> Better option once you're comfortable: `aws configure sso` with IAM Identity Center, which uses temporary credentials.

### Lab 5 — Audit
- [ ] Download the **Credential Report** (IAM → Credential report)
- [ ] Open **Access Advisor** for your test user and see which services were used

---

## 8. ✅ Quick self-check

1. What's the difference between a user and a role?
2. An EC2 instance needs to write to S3. How should you give it permission?
3. A policy allows `s3:*` and another denies `s3:DeleteObject`. Can the user delete objects?
4. Can an IAM group contain another group?
5. What does a trust policy define?
6. What is the principle of least privilege?
7. What's the difference between an identity-based policy and a resource-based policy?
8. Is IAM regional or global?

<details>
<summary>Answers</summary>

1. A user has long-term credentials (password/access keys) and represents one person or app. A role has no long-term credentials; it is assumed temporarily by users, services or other accounts.
2. Create an IAM role with S3 write permissions and attach it to the instance (via an instance profile). Never store access keys on the instance.
3. No. An explicit deny always wins.
4. No. Groups contain only users.
5. Who is allowed to assume the role.
6. Give each identity only the minimum permissions it needs to do its job.
7. Identity-based policies attach to users/groups/roles and say what *they* can do. Resource-based policies attach to a resource (like an S3 bucket) and say *who* can access it, using a `Principal`.
8. Global.

</details>

---

## 📝 My notes

### Summary: what we learned
- **IAM** controls **who** can do **what** on **which** resources. It's **global** and **free**. By default, everything is **denied**.
- **Identities:**
  - **Root user**: full access. MFA on, no access keys, never use it day to day.
  - **Users**: people with long-term credentials (password and/or access keys).
  - **Groups**: attach permissions to a group, not to individual users. Groups can't contain other groups.
  - **Roles** ⭐: temporary credentials, assumed by AWS services, other accounts or SSO users. Code running in AWS should **always use a role, never access keys**.
    - Trust policy = **who** can assume the role. Permission policy = **what** it can do.
- **Policies** are JSON: `Effect` (Allow/Deny) + `Action` (`service:Action`) + `Resource` (an **ARN**) + an optional `Condition`.
  - Policy types: AWS managed, customer managed, inline, and resource-based (e.g. an S3 bucket policy, which has a `Principal`).
- **How access is decided:** **explicit Deny > explicit Allow > implicit Deny**.
- **Best practices:** least privilege, MFA for everyone, IAM Identity Center (SSO) for people, rotate keys and never commit them to Git.
- **Audit tools:** Credential Report (all users and their credential status), Access Advisor (which services were actually used), Access Analyzer (resources shared outside the account).
- **Good to know:** Organizations and SCPs (account-wide guardrails), permission boundaries (maximum permissions), STS (issues temporary credentials).
- **CLI:** `aws configure`, then `aws sts get-caller-identity` to check who you're logged in as.

<!-- Add your own notes, doubts and learnings here -->
