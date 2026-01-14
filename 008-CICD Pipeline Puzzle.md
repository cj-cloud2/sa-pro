### CI/CD Pipeline Challenge: From CodeCommit Chaos to Automated Gold (15 mins)

**The Scenario**
DevTeam at MegaCorp builds a **Node.js e-commerce API** (`shop-api`). Current pain points:

- **5 Developers** pushing to shared GitHub repo → merge conflicts, no review process.
- **Manual builds** on dev laptops → inconsistent Node versions, failed deploys.
- **Prod deployments** via SCP to EC2 fleet → downtime, rollback hell during Black Friday.
- **Environments:** Dev (EC2 Blue/Green), Staging (ECS), Prod (ECS Fargate).
- **Requirements:** Git commit → auto-build → auto-test → auto-deploy (all envs), approvals for Prod, rollback safety.

**Current State:** 2-hour manual cycles, 30% deploy failures. **Challenge:** Architect full CI/CD using CodeCommit, CodeBuild, CodeDeploy, CodePipeline. Include `buildspec.yml`, `appspec.yml`, pipeline stages, and failure modes. Design for 100+ commits/day (20-25 mins deep analysis time).

***

### Thought Process: How to Architect Step-by-Step

1. **Source Control Diagnosis:** GitHub = external risk, no native IAM. **CodeCommit** = fully-managed Git (encrypted, IAM auth, triggers). Migrate repo → branch protection → pull requests.
2. **Build Phase Fundamentals:** No servers to patch (CodeBuild = ephemeral containers). Need `buildspec.yml` for Node: install deps, lint, test (Jest), package artifacts (zip/ECR). Parallel builds scale automatically.
3. **Deploy Phase Strategy:**
    - **Dev:** CodeDeploy Blue/Green (zero-downtime EC2).
    - **Staging/Prod:** CodeDeploy ECS (traffic shifting).
    - `appspec.yml` + hooks (BeforeInstall, AfterInstall, ValidateService).
4. **Pipeline Orchestration:** CodePipeline = **workflow engine**. Stages: Source → Build → Test → Approve → DeployDev → DeployStaging → ManualApprove → DeployProd. Artifacts pass between stages (S3).
5. **Observability \& Safety:** CloudWatch alarms trigger rollbacks. Manual approval gates Prod. Branch logic (dev/main/prod branches → env mapping).
6. **Cost/Scale Check:** CodeBuild: \$0.005/min compute. CodePipeline: \$1/active pipeline/month. Scales to 10k builds/day.

**Time Allocation:** 5 mins source/build, 7 mins deploy strategies, 7 mins pipeline design, 6 mins files + failure modes.

***




***
### The Solution: Complete Pipeline + Deep Implementation

```
Developer Laptops ──git push──> CodeCommit Repo (main/dev/prod branches)
                           │
                           ▼ CloudWatch Events Trigger (auto-detect commits)
                    +───────────────────────+
                    | AWS CodePipeline      |
                    | Stages:               |
                    | 1.Source(CodeCommit)  |
                    | 2.Build(CodeBuild)    | 
                    | 3.Test(Phase in Build)|
                    | 4.Approve(Dev)        |
                    | 5.DeployDev(Blue/Green)|
                    | 6.DeployStaging(ECS)  |
                    | 7.ManualApprove(Prod) |
                    | 8.DeployProd(Fargate) |
                    +───────────────────────+
                           │ Artifacts (S3/ECR)
                           ▼
       +───────────────+  +───────────────────+
       | CodeBuild     |  | CodeDeploy        |
       | - Node 18     |  | - EC2 Blue/Green  |
       | - npm install |  | - ECS Traffic     |
       | - npm test    |  | - Fargate Zero    |
       | - zip/ECR     |  |   Downtime        |
       +───────────────+  +───────────────────+
                           │
                           ▼
                  Dev/EC2 ─ Staging ECS ─ Prod Fargate
```


#### **1. Why This Stack Wins (vs. Jenkins/EC2 Self-Managed)**

```
Manual:    2hr cycles, 30% failure → $120k/year dev time waste
AWS CI/CD: 3min cycles, 99% success → Scales to 100+ devs, $2k/month
```

No servers, IAM integration, built-in Blue/Green, audit trails.

#### **2. Core Files: The Pipeline Blueprint**

**A. `buildspec.yml` (CodeBuild Instructions)**

```yaml
version: 0.2
phases:
  install:
    runtime-versions:
      nodejs: 18     # Consistent across builds
    commands:
      - echo "Installing dependencies..."
      - npm ci      # Clean install vs npm install
  
  pre_build:
    commands:
      - echo "Linting..."
      - npm run lint
      - echo "Running tests..."
      - npm test    # Jest unit/integration
  
  build:
    commands:
      - echo "Building app..."
      - npm run build
      - echo "Packaging..."
      - zip -r app.zip . -x "*.git*"    # Dev/Staging
      - docker build -t shop-api .       # ECS/ECR
      - docker tag shop-api:latest 123456.dkr.ecr.us-east-1.amazonaws.com/shop-api:$CODEBUILD_BUILD_NUMBER
      - docker push 123456.dkr.ecr...    # Prod ECR artifact
  
  post_build:
    commands:
      - echo "Tests passed, artifacts ready"
artifacts:
  files: [app.zip, buildspec.yml, appspec.yml, scripts/**]
secondary-artifacts:
  docker:
    files: [Dockerfile, **/*]
reports:
  jest-reports:
    files:
      - 'coverage/**'
    base-directory: './test-results'
```

**B. `appspec.yml` (CodeDeploy Deployment Rules)**

```yaml
version: 0.2
os: linux
files:
  - source: /
    destination: /var/www/shop-api
hooks:
  BeforeInstall:
    - location: scripts/stop_server.sh
      timeout: 300
      runas: ec2-user
  AfterInstall:
    - location: scripts/npm_install.sh
      timeout: 600
  ValidateService:
    - location: scripts/health_check.sh
      timeout: 300
      onFailure: AbortDeployment  # Rollback if health check fails
```


#### **3. CodePipeline Stage Configuration**

```
Stage 1: Source (CodeCommit)
├── Repository: shop-api
├── Branch: main → DeployProd, dev → DeployDev
└── DetectChanges: CloudWatch Events (5min poll)

Stage 2: Build (CodeBuild)
├── Environment: Node 18, 3GB RAM, Ubuntu Standard:5.0
├── Timeout: 20min
└── Parallelism: Auto-scale

Stage 3: Test (CodeBuild Phase)
├── JUnit coverage >90% → Pass
└── Security: CodeGuru Reviewer (optional)

Stage 4: Manual Approval (Dev → Staging)
Stage 7: Manual Approval (QA → Prod)

Stage 5/8: Deploy (CodeDeploy)
├── Dev: EC2 Blue/Green (Auto-rollback on 5xx errors)
├── Staging: ECS Canary (10% traffic → 100%)
├── Prod: Fargate Linear (25% every 5min → 100%)
```


#### **4. Complete Developer Workflow**

```
1. git checkout -b feature/payment-api
2. git commit -m "Add Stripe integration"
3. git push origin feature/payment-api  # Triggers PR pipeline
4. Merge to dev → Auto-deploys Dev env
5. QA tests → Merge to main → Prod deploy (after approval)
6. Monitor: CloudWatch alarms → Auto-rollback
```


#### **5. Failure Modes + Self-Healing**

| Failure | Symptom | Auto-Fix | Manual Fix |
| :-- | :-- | :-- | :-- |
| Build Fail | npm test fails | Retry 2x | Fix tests |
| Deploy Fail | Health check fails | Blue/Green rollback | appspec hook |
| Prod Alarm | Latency >2s | Traffic shift back | Post-mortem |
| Pipeline Fail | IAM perms | CloudWatch → Slack | Role trust |

**Verification Commands:**

```bash
# Pipeline status
aws codepipeline get-pipeline-state --name shop-api-pipeline

# Build logs
aws codebuild batch-get-builds --ids shop-api-build-123

# Deployment status
aws deploy list-deployments --application-name shop-api-app
```


#### **6. Cost Breakdown (100 devs, 1k builds/month)**

```
CodeCommit: $1/repo/month
CodeBuild: 1k builds × 5min × $0.005/min = $250/month
CodePipeline: $1/pipeline/month
CodeDeploy: Free (pay for compute)
Total: ~$300/month vs. $10k+ Jenkins cluster
```

**Key Concepts Mastered:** Full CI/CD automation eliminates human error, scales infinitely, Blue/Green eliminates downtime. Developers focus on code, not ops. 
