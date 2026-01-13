### VPC Interface Endpoint for SNS Challenge: Private Publishing Without Internet

**The Scenario**
MegaCorp's **Order Processing VPC** (10.5.0.0/16) runs containerized microservices that generate business events. These must publish to a **central SNS topic** ("orders-processed") owned by the **EventBridge Platform Team** in a different account.

**Network Constraints:**

- **No Internet Gateway, no NAT Gateway** (compliance mandates zero internet).
- **Private subnets only** (10.5.1.0/24, 10.5.2.0/24 across 2 AZs).
- **EC2 instances** with IAM roles for `sns:Publish`.

**Current State:** Direct `sns.us-east-1.amazonaws.com` calls fail (no route to public internet). Workarounds like NAT = cost + attack surface. **Challenge:** Architect private SNS access using Interface VPC Endpoint. Map complete data flow, craft policies, troubleshoot AccessDenied (20-25 mins thinking time).

***

### Thought Process: How to Architect This Step-by-Step

1. **Reject Public Routing Fundamentals:** AWS services like SNS expose **public HTTPS endpoints** (sns.region.amazonaws.com). Private VPCs can't reach without IGW/NAT (egress costs ~\$0.045/GB + hourly NAT fees). **Ask:** How to keep traffic *inside AWS backbone*?
2. **Understand Endpoint Types:**
    - **Gateway Endpoints** (S3/DynamoDB): Free, route table magic.
    - **Interface Endpoints** (SNS, API Gateway): **PrivateLink technology** = ENIs in *your* subnets proxying to service. **Cost:** ~\$0.01/hr/ENI + \$0.01/GB processed.
3. **PrivateLink Deep Dive:** Creates **Elastic Network Interfaces** (dns entries too). Enable **Private DNS** → `sns.us-east-1.amazonaws.com` resolves to *private IPs* (10.x.x.x). No code changes needed.
4. **Double Authorization Challenge:**
    - **Endpoint Policy** (service-side): "Can endpoint reach this SNS topic?"
    - **IAM Policy** (identity-side): "Does caller have sns:Publish?"
    - **Topic Policy** (resource-side): "Does topic allow this principal?"
5. **Security Group Flow:** Endpoint ENI needs inbound 443 from VPC CIDR. EC2 outbound 443 to endpoint.
6. **Cross-Account Complexity:** Different accounts → Topic policy must allow endpoint principal + cross-account IAM.

**Time Check:** 5 mins flow mapping, 7 mins policy design, 7 mins troubleshooting layers, 5 mins cost/security analysis.

***

### The Solution: Complete Architecture + Deep Explanation

```
Order Processing VPC (10.5.0.0/16)  PrivateLink Magic  SNS Service (Platform Account)
+---------------------------+                        +---------------------+
| EC2 Microservice          |                        | SNS Topic           |
| IAM Role: sns:Publish     |                        | arn:aws:sns:us-east-1:
| 10.5.1.100 → sns:Publish  |                        | 99998888:orders-processed
+---------------------------+                        +---------------------+
         ↑ HTTPS 443
         | (Private DNS resolves to 10.5.1.50)
         ↓
+---------------------------+
| Interface Endpoint ENI    |  ← com.amazonaws.us-east-1.sns
| 10.5.1.50 (AZ1), 10.5.2.50|     (ENI per subnet)
|                           |     Policy: Allow sns:Publish
| SG: TCP 443 VPC CIDR      |     Private DNS: ENABLED
+---------------------------+
       ↑ Endpoint Policy + IAM Policy
```

**Detailed Breakdown (Layer by Layer):**

#### **1. Why Interface Endpoints Win (vs. NAT/IGW)**

Public SNS = internet routing. Interface Endpoint = **AWS private backbone** (no public IPs, encrypted TLS 1.2+, Regional HA). Costs: \$7.20/month per endpoint (2 subnets) + data (~\$0.01/GB) vs. NAT Gateway \$40+/month + egress.

#### **2. Step-by-Step Architecture Build**

```
Step 1: Create Endpoint (Order Processing Account)
aws ec2 create-vpc-endpoint --vpc-id vpc-1a2b3c4d 
  --service-name com.amazonaws.us-east-1.sns
  --subnet-ids subnet-1e2f subnet-2g3h --vpc-endpoint-type Interface
  --private-dns-enabled

Step 2: Security Group (Endpoint ENI)
Inbound: TCP 443 from 10.5.0.0/16

Step 3: Endpoint Policy (Service-Side Gate)
{
  "Statement": [{
    "Effect": "Allow",
    "Principal": "*",
    "Action": "sns:Publish",
    "Resource": "arn:aws:sns:us-east-1:99998888:orders-processed"
  }]
}
```


#### **3. EC2 IAM Role (Identity-Side)**

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "sns:Publish",
    "Resource": "arn:aws:sns:us-east-1:99998888:orders-processed"
  }]
}
```


#### **4. SNS Topic Policy (Resource-Side - Platform Account)**

```json
{
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::123456789012:root"  // Order VPC Account
    },
    "Action": "sns:Publish",
    "Resource": "arn:aws:sns:us-east-1:99998888:orders-processed"
  }]
}
```


#### **5. Complete Data Flow (EC2 → SNS Success)**

```
1. EC2 publishes: aws sns publish --topic-arn arn:aws:sns:us-east-1:99998888:orders-processed
2. Private DNS: sns.us-east-1.amazonaws.com → 10.5.1.50 (endpoint ENI)
3. Endpoint Policy: ✓ sns:Publish allowed
4. IAM Policy: ✓ EC2 role has sns:Publish
5. Topic Policy: ✓ Cross-account allowed
6. SNS processes → Subscribers receive
```


#### **6. Troubleshooting Matrix: AccessDenied Root Causes**

| Symptom | Endpoint Policy | IAM Role | Topic Policy | SG/NACL |
| :-- | :-- | :-- | :-- | :-- |
| Endpoint Available | Missing `sns:Publish` | No perms | Cross-acct deny | 443 blocked |
| **Most Common:** Endpoint ✓, **IAM missing** | ✓ | ❌ **FIX** | ✓ | ✓ |
| Logs: CloudTrail shows `AccessDenied` from endpoint ENI IP |  |  |  |  |

**Verification:**

```bash
nslookup sns.us-east-1.amazonaws.com  # → Private IP ✓
aws sns publish --topic-arn ...       # Success ✓
```

**Key Concepts Mastered:** Interface Endpoints + **triple policy layers** create zero-trust private service access. Essential for PCI/HIPAA workloads. Total time: ~23 mins.

