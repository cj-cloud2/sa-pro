# Solution:
## Final Org Structure Diagram

```
Root Account (MFA enforced, no IAM users or roles—use SSO only)
├── Security OU (Delegated: Audit Manager, GuardDuty)
│   ├── Logs-Central (Account 15: CloudWatch, S3 logs)
│   └── Backup-Central (Account 16: AWS Backup plans)
├── Prod OU (SCPs: Strict compliance)
│   ├── Prod-AP (Account 1: India banking—SCPs block public access)
│   └── Prod-EU (Account 2: EU payments—SCPs enforce GDPR regions)
├── Dev OU (SCPs: Cost controls)
│   └── Dev-US (Account 3: Testing—limited resources)
├── Sandbox OU (SCPs: High restrictions)
│   └── Sandbox (Account 4: Interns—auto-suspend after inactivity)
└── Infra OU (Delegated: Backup admin)
    ├── Shared-Services (Accounts 5-11, 13-14, 17-22: VPCs, EKS)
    └── Legacy-Migration (Account 12: Fixed cross-account IAM via Resource Access Manager)
```


## Detailed Justifications

### 1. Root Account Setup

Root holds no resources—delegates everything to OUs. Enforce MFA and use AWS SSO for access. Prevents single point of failure; aligns with AWS Well-Architected Framework (Security Pillar).

### 2. OU Design Logic

- **Security OU first under Root**: Centralized logging/monitoring inherits to all children. Delegate Audit Manager here for cross-account compliance scans.
- **Prod OU**: Isolates revenue-generating workloads. SCPs prevent accidents (e.g., public buckets).
- **Dev OU**: Cost-focused; separate from Prod to avoid quota exhaustion.
- **Sandbox OU**: Maximum blast radius containment—SCPs deny high-cost services.
- **Infra OU**: Shared resources like networking; delegated Backup admin allows Infra team to manage without org-wide perms.


### 3. SCP Examples

**Prod-EU SCP: Deny Cross-Region Replication (GDPR)**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": "s3:ReplicateObject",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "s3:DestinationRegion": "eu-west-1"
        }
      }
    }
  ]
}
```

Justification: Locks data to EU region; SCPs evaluate before IAM.

**Prod-AP SCP: Block Public S3**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyPublicS3",
      "Effect": "Deny",
      "Action": "s3:PutBucketPublicAccessBlock",
      "Resource": "*",
      "Condition": {
        "Bool": { "aws:PublicAccessBlockEnabled": false }
      }
    }
  ]
}
```

Justification: RBI/PCI-DSS compliance for India fintech; applies to buckets only.

**Dev SCP: Limit EC2 Size**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": "ec2:RunInstances",
      "Resource": "*",
      "Condition": {
        "StringNotLike": {
          "ec2:InstanceType": "t3.micro,t3.nano"
        }
      }
    }
  ]
}
```

Justification: Caps dev spend; encourages serverless.

**Sandbox SCP: Auto-Suspend (via Lambda + Config)**
Trigger AWS Config rule for inactive accounts → Lambda detaches/terminates. Justification: Cost governance; OUs enable inheritance.

### 4. Consolidated Billing \& Savings

All accounts under one payer → 20%+ discounts on EBS/S3. For \$500k/month: ~\$100k/year saved. Track via Cost Explorer tags inherited from OUs.

### 5. Delegated Administration

- Backup delegated to Infra OU: Team manages plans cross-account without elevating perms.
- Audit Manager to Security OU: Automated compliance reports.
Justification: Scales management; service trusts Organizations membership.


### 6. Account 12 Migration Fix

Use AWS RAM to share resources pre-migration. Post-move: Update IAM trust policies. Justification: Avoids downtime; RAM works across Organizations boundaries temporarily.

### 7. Migration Sequence (Step-by-Step)

1. Create OUs and invite accounts (no disruption).
2. Apply baseline SCPs (Security OU first).
3. Enable consolidated billing.
4. Delegate services.
5. Migrate/mass-invite remaining accounts.
6. Test SCPs in Dev/Sandbox.
7. Monitor with Config.



### 8. Additional SCP Examples

Here are 5 more production-ready SCPs tailored to the puzzle, drawn from 2026 best practices. Each includes JSON, when/where to apply, and why it fits TechNova's fintech setup. These build on the previous ones for comprehensive governance.

**1. Deny Leaving Organization (All OUs)**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyLeavingOrganization",
      "Effect": "Deny",
      "Action": ["organizations:LeaveOrganization"],
      "Resource": "*"
    }
  ]
}
```

Apply: Root and all child OUs. Justification: Prevents compromised accounts from escaping SCPs/billing—critical for audit compliance in regulated industries like fintech.[^1]

**2. Enforce EBS Encryption (Prod/Infra OUs)**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": [
        "ec2:CreateVolume",
        "ec2:CreateSnapshot"
      ],
      "Resource": "*",
      "Condition": {
        "Bool": {
          "ec2:Encrypted": "false"
        }
      }
    }
  ]
}
```

Apply: Prod OU. Justification: RBI mandates encrypted volumes; blocks unencrypted EBS at creation to avoid data exposure.

**3. Restrict Regions (Prod-EU OU)**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": "eu-west-1,eu-central-1"
        }
      }
    }
  ]
}
```

Apply: Prod-EU only. Justification: GDPR data residency; limits to EU partitions, overriding IAM allows.[^2]

**4. Deny High-Cost Services in Sandbox (e.g., No SageMaker)**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": [
        "sagemaker:*",
        "bedrock:*"
      ],
      "Resource": "*"
    }
  ]
}
```

Apply: Sandbox OU. Justification: Prevents runaway ML costs (2026 Bedrock updates); deny-list for experimental environments.

**5. Complete Lockdown for Suspended Accounts**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*"
    }
  ]
}
```

Apply: New "Suspended" OU under Root for compromised accounts. Justification: Zero-trust during investigations; allow cleanup via admin roles only.

## Student Notes Section

- **Key Takeaway**: OUs for hierarchy, SCPs for guardrails (not auth), delegations for scale.
- **Common Pitfalls**: SCPs should be applied to specific OU as per requirements. Not directly to Root
- **Resources**: AWS Orgs docs (2026 updates: AI policy generator in preview).
- **Time Saved**: Students reference this for certification (e.g., AWS Advanced Networking).

