### Puzzle: Multi-Account Migration Riddle

**Puzzle Briefing for Students**
You're consulting for TechNova Inc., a Nagpur-based fintech with 22 standalone AWS accounts created haphazardly over 5 years. They're hitting limits on service quotas, inconsistent billing, and audit failures. Your task: Migrate into AWS Organizations with OUs, apply SCPs, and enable delegated admins. Design the structure to support 500+ users across India, EU, and US regions while minimizing blast radius from errors.

**Current Messy Setup (Diagram)**

```
Root
├── Account1: Prod-AP (India banking app, high spend)
├── Account2: Prod-EU (EU payments, GDPR strict)
├── Account3: Dev-US (testing, public S3 buckets)
├── Account4: Sandbox (interns, unrestricted)
├── Accounts 5-22: Scattered infra/logs/shared services
```

No OUs, no SCPs, billing fragmented.

**Requirements \& Clues (8 Total - Solve in Sequence)**

1. Create top-level OUs: Prod, Dev, Infra, Sandbox. Assign accounts logically.
2. Prod OU: Block public S3 ACLs and unencrypted EBS via SCP. (Clue: Use `Deny` statement on `s3:PutBucketPublicAccessBlock`.)
3. Dev OU: Allow all but limit EC2 instance types to t3.micro for cost control.
4. Enable consolidated billing—calculate potential savings (hint: 20% volume discount on \$500k/month).
5. Delegate Backup admin to Infra OU owner. (Clue: Which service? Enable via Organizations console.)
6. Sandbox OU: Auto-suspend accounts after 7 days inactivity using custom policy. (Twist: Integrates with Config rules.)
7. Migrate Accounts 5-22: Group into Infra (logs/monitoring), but Account 12 has legacy IAM needing cross-account roles.
8. **Gotcha**: EU account needs regional guardrails (no cross-region replication). Justify with diagram.

**Student Worksheet (Fill-in-the-Blank)**

- Draw new Org structure:

```
Root
├── OU: ________
│   ├── Account: ________ (SCPs applied: ________)
│   └── ...
```

- Short answers:

1. SCP JSON snippet for Prod S3 block: ________________
2. Delegated service for Infra: ________________ (Why this one?)
3. Migration step sequence (1-5): ________________

**Partial Hints (Reveal After 5 Mins)**

- "Remember SCPs don't affect IAM—use for guardrails only."
- "Check AWS docs: Delegated admin prioritizes OUs."
- "Savings calc: Consolidated = pay-as-you-go + discounts."

