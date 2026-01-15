# Puzzle 1: AWS KMS and AWS Secrets Manager

## The Puzzle (for your students)

You are the lead architect for a fintech startup running entirely on AWS. You have **three microservices** in production:

1. **Payments API** (Amazon ECS on Fargate, behind an ALB)
2. **Analytics Service** (AWS Lambda functions processing events from an SQS queue)
3. **Admin Console** (React SPA hosted on S3 + CloudFront, with backend on Amazon API Gateway + Lambda)

Your company must now comply with **strict security and compliance requirements**:

- All **database credentials and API keys** must be stored securely.
- **Automatic rotation** of secrets is required where possible.
- **Fine‑grained IAM access control** is required (least privilege).
- **Auditability** of who/what accessed secrets and keys is important.
- Some keys must be **customer‑managed**, not AWS‑managed.

You have the following sensitive items:

- RDS PostgreSQL master password for the Payments database
- Third‑party payment gateway API key
- OAuth client secret for the Admin Console
- Internal service‑to‑service “shared secret” used only between Payments API and Analytics Service
- A requirement to **encrypt transaction reports** (files) stored in S3 and allow **read‑only decryption** from a different AWS account used by external auditors

You can use:

- **AWS KMS** (with AWS‑managed and customer‑managed keys)
- **AWS Secrets Manager**

You must design **where and how** to store and use these secrets/keys.

***

### Task 1 – Choosing Between KMS and Secrets Manager

For each of the following items, decide whether to use:

- **A. AWS KMS directly (CMK / customer‑managed KMS key)**
- **B. AWS Secrets Manager (with KMS under the hood)**

Items:

1. RDS PostgreSQL master password
2. Third‑party payment gateway API key
3. OAuth client secret for the Admin Console
4. Internal service‑to‑service shared secret between Payments API and Analytics Service
5. KMS key to encrypt/decrypt transaction report files in S3 (with cross‑account read‑only decryption for auditors)

For each item, think about:

- Do you need **automatic secret rotation** or just encryption?
- Is it a **“value” secret** (like a password) or a **“key‑encryption key / data key”**?
- Is **cross‑account access** to encrypted data required?
- Do you need **per‑call retrieval of the secret value**, or just encryption/decryption APIs?

Provide a **short justification** for each choice.

**Guiding hints inside the puzzle:**

- Hint 1: *Remember that Secrets Manager stores and versions “values” (passwords, API keys, JSON blobs) and uses KMS in the background.*
- Hint 2: *KMS is great for encrypting data and managing keys, including cross‑account use, but does not itself store arbitrary secret strings for application retrieval.*
- Hint 3: *Automatic rotation logic differs between RDS integration and external APIs; think about where AWS gives you “built‑in” rotation flows.*

***

### Task 2 – Lambda \& ECS Access Pattern

You decide:

- Payments API (ECS) needs access to:
    - RDS password
    - Payment gateway API key
- Analytics Service (Lambda) needs access to:
    - RDS read‑only password (different user)
    - Internal shared secret

Design **how these workloads retrieve secrets at runtime**:

1. Where will the secrets be **stored**?
2. How will ECS tasks and Lambda functions **authenticate** to retrieve them (what AWS feature is used)?
3. How do you enforce **least privilege** so that:
    - Payments API **cannot** see the OAuth client secret.
    - Analytics Service **cannot** see the payment gateway API key.

**Guiding hints:**

- Hint 4: *Consider using task/role identities instead of embedding any AWS credentials in code.*
- Hint 5: *IAM policies can restrict access by secret ARN and by KMS key.*
- Hint 6: *Lambda and ECS both support role‑based access via AWS Security Token Service (STS) under the hood.*

***

### Task 3 – Rotation Strategy

You must define a rotation strategy for:

1. RDS PostgreSQL master password.
2. Payment gateway API key (rotation supported manually via 3rd‑party dashboard).
3. OAuth client secret for Admin Console.

For each, answer:

- Should rotation be:
    - **Managed via Secrets Manager built‑in rotation integration**
    - **Managed via a custom Lambda rotation function**
    - **Managed manually outside AWS and just updated in AWS**
- How often would you rotate it (directionally: frequent / moderate / rare) and why?

**Guiding hints:**

- Hint 7: *Secrets Manager has native integration for some AWS services (like RDS), but not for every external provider.*
- Hint 8: *Rotation must be carefully coordinated: “update secret in AWS” vs “update credential at the provider” vs “update application configuration.”*
- Hint 9: *Consider the operational blast radius if something goes wrong during rotation.*

***

### Task 4 – Encrypting \& Sharing S3 Data with Auditors

The compliance team wants all **transaction reports** stored in S3:

- Encrypted at rest.
- Accessible for **decryption** only by:
    - Internal analytics role (same account).
    - External auditors in a **separate AWS account**, read‑only.

You must:

1. Decide **what type of KMS key** to use (AWS‑managed vs customer‑managed) and why.
2. Describe at a high level how to configure **cross‑account decryption** using KMS:
    - Where is the KMS key created?
    - What needs to be configured so auditors can decrypt but not modify the key?
3. Should these encrypted files’ **object metadata** or **content** ever contain “raw secrets”? Why or why not?

**Guiding hints:**

- Hint 10: *KMS key policies and IAM policies both affect whether another account can use a key.*
- Hint 11: *Using AWS‑managed keys limits your control over key policy and cross‑account usage.*
- Hint 12: *Secrets Manager is not a replacement for S3‑level encryption for large files, but can work together with KMS.*

***

## Suggested Solution (with detailed teaching)

### Task 1 – Choosing Between KMS and Secrets Manager

First, clarify the conceptual roles:

- **AWS KMS**:
    - Manages **cryptographic keys**.
    - Provides APIs: `Encrypt`, `Decrypt`, `GenerateDataKey`, `Sign`, `Verify`.
    - Ideal for **encryption of data**, including S3, EBS, RDS, DynamoDB and custom application data.
    - Is **not** a general secret store. It does not keep arbitrary secret values for retrieval like “give me the current password.”
- **AWS Secrets Manager**:
    - Manages **secret values** (passwords, API keys, tokens) as **versioned records**.
    - Uses a **KMS key in the background** to encrypt those secret values at rest.
    - Provides APIs: `GetSecretValue`, `PutSecretValue`, with built‑in **rotation** support (often via Lambda).
    - Focus is on **secure storage + retrieval + rotation** of secret strings/binary.

With that in mind:

#### 1. RDS PostgreSQL master password

**Best choice: AWS Secrets Manager (B).**

Reasoning:

- This is a **classic “value secret”**: a password that the application needs to retrieve and use when connecting to the database.
- Secrets Manager has **built‑in integration** for RDS. It can:
    - Change the password in the database.
    - Update the stored secret value.
    - Coordinate the rotation workflow safely.
- KMS alone would only encrypt/decrypt the password string, but would not provide:
    - Storage.
    - Versioning.
    - Rotation orchestration.

Concept to teach: *Use Secrets Manager for storing and rotating connection credentials, not bare KMS.*

#### 2. Third‑party payment gateway API key

**Best choice: AWS Secrets Manager (B).**

Reasoning:

- Again, this is a **value secret** that the Payments API needs to read at runtime.
- You likely want **automatic or semi‑automatic rotation**, which is possible via:
    - A **custom Lambda rotation function** in Secrets Manager that:
        - Calls the 3rd‑party API/provider to generate a new key.
        - Stores the new key in Secrets Manager.
- KMS alone cannot integrate with the 3rd‑party provider and cannot manage the lifecycle of the API key.

Concept to teach: *Secrets Manager is ideal for non‑AWS secrets as well, as long as you provide rotation logic via Lambda.*

#### 3. OAuth client secret for the Admin Console

**Best choice: AWS Secrets Manager (B).**

Reasoning:

- This is also a **value secret** used by OAuth flows (e.g., for an identity provider).
- Security best practice:
    - Store it centrally in Secrets Manager.
    - Access only from the backend Lambda/API that needs it.
- Rotation can be:
    - Custom rotation (via Lambda) if your IdP supports programmatic rotation.
    - Or manual rotation with Secrets Manager still used as the secure store.

Concept to teach: *Do not put OAuth secrets in environment variables or config files; store them in Secrets Manager with controlled access.*

#### 4. Internal service‑to‑service shared secret between Payments API and Analytics Service

**Best choice: AWS Secrets Manager (B), but with a conceptual twist.**

Reasoning:

- This is again a **shared “value secret”** (e.g., an HMAC key or token).
- It must be **read by two services** (ECS and Lambda).
- Using Secrets Manager:
    - One secret, controlled by IAM.
    - Both roles (ECS task role and Lambda role) can be granted access to the same secret.
- KMS alone would not store that secret conveniently; it would just encrypt/decrypt it.

Concept to teach: *Service‑to‑service secrets are still secrets; use Secrets Manager as the source of truth with careful IAM.*

#### 5. KMS key to encrypt/decrypt transaction report files in S3 (with cross‑account auditors)

**Best choice: AWS KMS directly (A).**

Reasoning:

- Here, you are not storing one small secret value; you are encrypting **files/objects in S3**.
- You need:
    - S3 server‑side encryption with **KMS customer‑managed keys (SSE‑KMS)**.
    - Ability to configure **cross‑account access** for auditors.
- This is exactly a **KMS use case**:
    - Define a **customer‑managed CMK** in the producer account.
    - Configure **key policy and IAM** to allow the auditors’ account to call `Decrypt` on that key (under controlled conditions).

Concept to teach: *Use KMS when the primary problem is “encrypt data using keys,” not “store a secret string to be retrieved.”*

***

### Task 2 – Lambda \& ECS Access Pattern

Objective: Show **how secrets are retrieved without hardcoding credentials**, using **IAM roles** and **least privilege**.

#### 1. Where are the secrets stored?

From Task 1:

- RDS master password → **Secrets Manager**
- Payment gateway API key → **Secrets Manager**
- RDS read‑only password → **Secrets Manager** (another secret)
- Internal shared secret → **Secrets Manager**

So you will have multiple **Secrets Manager secrets**, each with its own ARN, for example:

- `arn:aws:secretsmanager:region:account-id:secret:rds-master-password-...`
- `arn:aws:secretsmanager:region:account-id:secret:rds-ro-password-...`
- `arn:aws:secretsmanager:region:account-id:secret:payment-gateway-api-key-...`
- `arn:aws:secretsmanager:region:account-id:secret:internal-shared-secret-...`
- `arn:aws:secretsmanager:region:account-id:secret:oauth-client-secret-...` (for Admin Console backend)


#### 2. How do ECS tasks and Lambda functions authenticate?

Key concept: **IAM roles for compute services**.

- **For ECS on Fargate**:
    - Define an **ECS Task Role** (e.g., `PaymentsApiTaskRole`).
    - Assign an IAM policy to this role granting `secretsmanager:GetSecretValue` on:
        - RDS master password secret.
        - Payment gateway API key secret.
        - Internal shared secret.
    - Configure the ECS task definition to use this task role.
    - When the container starts, the container’s SDK (e.g., AWS SDK for Java/Python) uses the **task role’s temporary credentials** (provided by STS) to call Secrets Manager.
- **For Lambda**:
    - Define a **Lambda execution role** (e.g., `AnalyticsLambdaRole`).
    - Assign an IAM policy granting `secretsmanager:GetSecretValue` on:
        - RDS read‑only password secret.
        - Internal shared secret.
    - Lambda runtime automatically obtains temporary credentials for this role and uses them for AWS API calls.

Teaching point: *Neither application ever needs long‑lived AWS access keys. Authentication is handled transparently by the compute service using IAM roles.*

#### 3. Enforcing least privilege

Goal:

- Payments API **cannot** see OAuth client secret.
- Analytics Service **cannot** see the payment gateway API key.

Implementation:

- Configure **separate secrets for each purpose**, not a single “mega secret”.
- Granular IAM policies:
    - For `PaymentsApiTaskRole`:
        - Allow `secretsmanager:GetSecretValue` **only** on:
            - `rds-master-password-secret-arn`
            - `payment-gateway-api-key-secret-arn`
            - `internal-shared-secret-arn`
    - For `AnalyticsLambdaRole`:
        - Allow `secretsmanager:GetSecretValue` **only** on:
            - `rds-ro-password-secret-arn`
            - `internal-shared-secret-arn`
    - For the Admin Console backend role:
        - Allow `secretsmanager:GetSecretValue` only on:
            - `oauth-client-secret-arn`

Concept to teach:

- **Least privilege**:
    - Limit permissions by **resource ARN**, not by wildcard `*`.
    - Avoid giving roles access to **all** secrets.
- **Separation of concerns**:
    - Each microservice gets only what it strictly needs.

***

### Task 3 – Rotation Strategy

Teaching objective: Show the difference between **built‑in rotation**, **custom rotation**, and **manual rotation**, and the importance of **coordination**.

#### 1. RDS PostgreSQL master password

Recommended: **Secrets Manager built‑in rotation.**

Explanation:

- Secrets Manager provides **native integration with RDS**:
    - It can connect to the RDS instance.
    - Update the master password.
    - Store the new password as a new version of the secret.
    - Test the new credentials before promoting them.
- Rotation frequency:
    - **Moderate**, e.g., every 30–60 days.
- Why not extremely frequent?
    - Changing the master password is sensitive.
    - If rotation fails, it could break admin access or other dependent tools.
    - Needs careful monitoring.

Concept: *Use built‑in integrations where possible. They encapsulate complex rotation workflows and reduce risk of misconfiguration.*

#### 2. Payment gateway API key (3rd‑party provider)

Recommended: **Custom Lambda rotation with Secrets Manager** (if API supports it); otherwise manual.

Two options:

1. If the provider has an API to create/rotate keys:
    - Implement a **Lambda rotation function** attached to the secret in Secrets Manager.
    - The rotation function:
        - Calls the provider API to generate a new key.
        - Stores the new key in Secrets Manager as `AWSCURRENT`.
        - Optionally tests the new key (e.g., minor test transaction).
    - Rotation frequency:
        - **Moderate/frequent**, e.g., every 30 days or per security policy.
2. If provider supports only manual key generation:
    - Rotation is **manual**:
        - Operator logs into provider portal, generates new key.
        - Operator updates Secrets Manager with the new key and marks it current.
    - Still, applications read from Secrets Manager, so **no code changes** for applications.

Concept: *Secrets Manager does not magically rotate 3rd‑party secrets; you must implement or coordinate the rotation process.*

#### 3. OAuth client secret for the Admin Console

Recommended:

- **Custom Lambda rotation**, if your identity provider supports an API for updating client secrets.
- Otherwise, **manual rotation** using Secrets Manager as storage.

Considerations:

- OAuth client secret often requires:
    - Updating it in the **IdP (e.g., Cognito, Okta, Auth0, corporate IdP)**.
    - Ensuring that all relying parties use the **new secret**.
- Rotation frequency:
    - Typically **less frequent**, e.g., every 90 days, because:
        - Change involves coordination with identity systems.
        - Mis‑rotation can break authentication for all users.

Concept: *More “central” and critical secrets tend to rotate less frequently but under stricter change control processes.*

***

### Task 4 – Encrypting \& Sharing S3 Data with Auditors

#### 1. What type of KMS key?

Use a **customer‑managed KMS key (CMK)**, not an AWS‑managed key.

Reasons:

- Need for **cross‑account** decryption by auditors:
    - Requires explicit control over **key policies**.
- Customer‑managed key allows:
    - Custom key policy to grant external account’s role `Decrypt` permissions.
    - Control over key rotation, disabling, and tagging.
- AWS‑managed keys:
    - Automatically created and managed by AWS for some services.
    - Not suitable when you need **fine‑grained, explicit policy control**, especially cross‑account.

Concept: *For complex enterprise scenarios, choose customer‑managed KMS keys so you can define key policies and sharing precisely.*

#### 2. Configuring cross‑account decryption

High‑level steps:

1. **Create the CMK** in the **producer account** (where S3 bucket and transaction files live).
2. Configure the **KMS key policy** to:
    - Allow the producer account’s roles (e.g., analytics role) to use `Encrypt`, `Decrypt`, etc., as necessary.
    - Allow the auditors’ account (specific IAM role ARN from that account) to perform:
        - `kms:Decrypt`
        - Possibly `kms:DescribeKey`
    - Do **not** give auditors permissions to:
        - `kms:ScheduleKeyDeletion`
        - `kms:DisableKey`
        - `kms:CreateGrant` (unless very specifically needed)
3. In the S3 bucket:
    - Use **SSE‑KMS** with that CMK for object encryption.
    - Configure **bucket policies** and **S3 object ACLs** (or IAM) to:
        - Allow auditors’ role to **read** (GetObject) the encrypted objects.
        - They will then call `Decrypt` on the KMS key to decode the data.
4. Auditors’ flow:
    - Assume their read‑only role in their account.
    - Use S3 `GetObject` to retrieve the encrypted report.
    - Use the KMS `Decrypt` API (on the producer account’s key) to decrypt the object (or rely on transparent decryption if using S3 APIs with the right permissions).

Concept: *True cross‑account encryption sharing with KMS always involves key policy + IAM + sometimes grants. The key lives in one account, not duplicated.*

#### 3. Should metadata/content ever contain raw secrets?

Best practice: **No.**

Explanation:

- Transaction reports are business data, not secret‑store replacements.
- Even though the data is encrypted with KMS, **embedding raw secrets inside reports** would:
    - Expand the attack surface (more locations containing secrets).
    - Complicate rotation (every report would contain stale secrets).
    - Make data exfiltration events more damaging.
- Secrets (API keys, passwords, tokens) should remain:
    - In **Secrets Manager** as dedicated secrets.
    - Encrypted and controlled through IAM.
- The purpose of S3 encryption is to secure data at rest, not to store high‑sensitivity credentials inside payloads.

Concept: *Data encryption ≠ secret management. Keep secrets in a dedicated service and keep data as clean of credentials as possible.*

***

## How This Teaches KMS vs Secrets Manager From Scratch

- **Role separation:**
    - KMS is for **keys and cryptographic operations**.
    - Secrets Manager is for **secret values, retrieval, and rotation**.
- **Runtime access:**
    - Applications should use **IAM roles** to call Secrets Manager/KMS.
    - No hardcoded AWS credentials.
- **Least privilege IAM:**
    - Limit secrets by ARN.
    - Limit keys by key policy and IAM policy.
- **Rotation models:**
    - Built‑in rotation for AWS services (like RDS).
    - Custom Lambda rotation for 3rd‑party or special cases.
    - Manual rotation but Secrets Manager as the single source of truth.
- **Cross‑account encryption:**
    - Customer‑managed CMKs.
    - Key policy design.
    - Bucket/IAM permissions for access and decryption.

This puzzle forces students to think in **concepts and trade‑offs**, not click‑by‑click configuration, and should comfortably take **10–15 minutes** to solve and discuss.

