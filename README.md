# aws-iac — Project 4

A three-tier web app on AWS (VPC → ALB → Auto Scaling EC2 running Flask behind nginx → private RDS MySQL), defined entirely in CloudFormation. Every push to `main` deploys it through CodePipeline, with a manual approval step.

## Repository layout

| File | What it is | How it gets to AWS |
|---|---|---|
| `pipeline.yaml` | The CI/CD pipeline itself: CodePipeline, CodeBuild project, IAM roles, artifact bucket, optional SNS approval topic | **You upload this to CloudFormation once, manually** |
| `template.yaml` | The application infrastructure (VPC, subnets, ALB, ASG, RDS, Secrets Manager, IAM) | **Never upload by hand.** The pipeline deploys it on every push to `main` |
| `buildspec.yml` | Instructions for the CodeBuild "Build" stage: lint and validate `template.yaml` | Read by CodeBuild from the repo |

> **Upload order:** push all three files to GitHub first, **then** create the `pipeline.yaml` stack. That's the only file you ever upload to CloudFormation. The pipeline takes care of `template.yaml` from there.

## Pipeline flow

```
GitHub push (main)
   │
   ▼
Source ──► Build (CodeBuild) ──► Deploy
           cfn-lint                1. CreateChangeSet   (stack: project4-V2)
           validate-template       2. ManualApproval    (review the change set)
                                   3. ExecuteChangeSet
```

- **Source:** GitHub through the existing CodeConnections connection (`Anurag-personalGit/aws-iac`, branch `main`).
- **Trigger:** any push to `main`, including merged pull requests. Merge-commit, squash and rebase merges all push to `main`, so they start the pipeline. Pull requests that are opened, updated, or closed without merging do **not** start it.
- **Build:** runs `buildspec.yml`. The pipeline stops here if `template.yaml` has a lint error or fails `validate-template`, before AWS gets touched.
- **Deploy:** creates a change set against `project4-V2` using a dedicated CloudFormation service role. It waits for approval, then executes. If the stage fails, the pipeline rolls back to the last successful execution.

## What's implemented

### Pipeline (`pipeline.yaml`)
- CodePipeline **V2**, QUEUED execution mode, with a push trigger on `main`
- Encrypted S3 artifact bucket that blocks public access, enforces TLS and expires artifacts after 30 days
- **Least-privilege roles:**
  - The CodeBuild role can only write logs, read/write artifacts and call `ValidateTemplate`
  - The CodePipeline role can only use the connection, start the build, manage change sets on `project4-V2` and pass the deploy role
  - The CloudFormation deploy role covers the services the app template uses. Its IAM permissions only apply to roles and instance profiles named `project4-V2-*`
- **No secrets in the pipeline:** parameter overrides send only `EnvironmentName` and `KeyPairName`
- **Optional email:** set `ApprovalEmail` to get an email when an approval is waiting

### Application (`template.yaml`)
- **Networking:** VPC `10.0.0.0/16` with 2 public and 2 private subnets across `ap-south-1a`/`1b`, an IGW and route tables
- **Security groups:** ALB (80 from the internet) → EC2 (80 from the ALB only, 22 from `SSHLocation`) → RDS (3306 from EC2 only)
- **Compute:** launch template on Amazon Linux 2023. cfn-init installs nginx + Python, deploys the Flask app as a systemd service and puts nginx in front of it as a reverse proxy
- **Auto Scaling:** min 2 / max 4 instances behind the ALB, with ELB health checks
  - Instances send `cfn-signal`, and updates roll **one instance at a time**
  - If an instance fails to bootstrap, the deployment rolls back
- **Database:** RDS MySQL `db.t3.micro` in the private subnets
- **Credentials:**
  - Secrets Manager generates the RDS master password, so nobody types or commits it
  - `SecretTargetAttachment` writes the real host and port into the same secret (`project4-V2/database`)
  - The app reads the secret at runtime through its EC2 instance role
- **Outputs:** ALB URL, secret ARN, VPC ID (exported as `Project4-VPC-ID`)

## First-time setup

### Prerequisites
- AWS CLI v2 configured for account `355947670295`, region `ap-south-1`
- A GitHub connection in **Available** status (Developer Tools → Settings → Connections)
- The EC2 key pair `project4KeyPair` exists, if you want SSH access. Set `KeyPairName` to empty to rely on SSM Session Manager only

### Step 1: push the code
Commit `template.yaml`, `buildspec.yml`, `pipeline.yaml` and this README to `main`.

### Step 2: create the pipeline stack (upload `pipeline.yaml`)

**Console:**
1. CloudFormation → **Create stack** → *With new resources* → **Upload a template file** → choose `pipeline.yaml`
2. Stack name: `project4-pipeline`
3. Review the parameters. The defaults already point at the right connection, repo, branch and target stack. Optionally fill in `ApprovalEmail`
4. Tick **"I acknowledge that AWS CloudFormation might create IAM resources with custom names"** → **Submit**

**CLI:**
```bash
aws cloudformation deploy \
  --stack-name project4-pipeline \
  --template-file pipeline.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --region ap-south-1
  # optional: --parameter-overrides ApprovalEmail=you@example.com
```

When the stack is created, the pipeline starts by itself on the latest commit in `main`.

### Step 3: approve the first deployment
1. Open the pipeline. Wait for **ManualApproval** to become active
2. In CloudFormation → `project4-V2` → **Change sets**, open `project4-V2-pipeline-changeset` and check:
   - **No resource shows `Replacement: True`** (VPC, subnets, ALB and RDS in particular)
   - Expected changes on the first run: Secret *Modify*, RDS *Modify* (password), LaunchTemplate *Modify*, ASG *Modify* (rolling update), SecretTargetAttachment *Add*
3. Approve in the pipeline

> **Expect a short outage on the first deploy.** The RDS master password is regenerated, so the app returns errors for a few minutes until the secret attachment writes the host back. The ALB health check hits `/`, which queries the DB, so the ASG may replace instances during that window.

### Step 4: get the app URL
```bash
aws cloudformation describe-stacks --stack-name project4-V2 --region ap-south-1 \
  --query "Stacks[0].Outputs[?OutputKey=='ALBDNSName'].OutputValue" --output text
```

## Migrating from the old console-created pipeline

Before this, the pipeline came from the console starter template: the stack `CodePipelineStarterTemplate-DeployToCloudFormation-…` and the pipeline `DeployToCloudFormationService`. To switch over without both pipelines deploying the same stack:

1. **Before** Step 1 above, block the old pipeline's deploy stage:
   ```bash
   aws codepipeline disable-stage-transition \
     --pipeline-name DeployToCloudFormationService \
     --stage-name Deploy --transition-type Inbound \
     --reason "Migrating to project4-pipeline" --region ap-south-1
   ```
2. Follow Steps 1–4.
3. **After the new pipeline has deployed `project4-V2` successfully once**, retire the old stack:
   - Empty its artifact bucket (`codepipelinestartertempla-codepipelineartifactsbuc-…`)
   - Delete the stack `CodePipelineStarterTemplate-DeployToCloudFormation-…`

   Don't delete it any earlier. `project4-V2` keeps using that stack's CloudFormation role until the new pipeline executes a change set with its own role.

## Day-to-day workflow

1. Edit `template.yaml` and push to `main`
2. The pipeline lints, validates and creates a change set
3. Review the change set, then approve
4. Changes roll out. EC2 instances are replaced one at a time when the launch template changes

**If you change only the `AWS::CloudFormation::Init` section** (packages, files, commands), bump the `# bootstrap-rev:` number in the launch template's UserData. Metadata-only changes don't create a new launch template version, so instances would never pick them up.

**Run the same checks locally before pushing:**
```bash
pip install cfn-lint
cfn-lint --non-zero-exit-code error template.yaml pipeline.yaml
aws cloudformation validate-template --template-body file://template.yaml
```

## Changing the pipeline itself

The pipeline doesn't redeploy itself. After editing `pipeline.yaml`, push it and then update the stack yourself, either with the `aws cloudformation deploy` command from Step 2 or through **Update stack → Replace existing template** in the console.

## Known limitations / to do
- **Fixed AZs and resource names:** AZs are hardcoded (`ap-south-1a`/`1b`), and several resources have fixed names (`Project4-ALB`, `Project4-TG`, `project4-db`, …). Only one copy of this stack can exist per account/region. Changing either replaces resources.
- **SSH open by default:** `SSHLocation` defaults to `0.0.0.0/0`. Restrict it, or use SSM Session Manager (the instance role already has `AmazonSSMManagedInstanceCore`).
- **Health check depends on the DB:** the ALB health check uses `/`, which queries the database. A dedicated `/health` route would stop DB hiccups from cycling instances.
- **Manual DB setup:** the database `project4app` and the `users` table aren't created by CloudFormation. Create them once by hand on a fresh deployment.
- **No HTTPS:** the ALB only serves HTTP (port 80) so far.
