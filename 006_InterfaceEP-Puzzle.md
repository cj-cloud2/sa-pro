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
