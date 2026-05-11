> All questions are **scenario-based, senior/advanced level** — covering real-world situations, debugging, architecture decisions, and trade-offs.

---

## Table of Contents

- [Terraform](#terraform)
  - [Q1 — Codebase Structure for Multi-Account Multi-Service](#q1-codebase-structure-for-multi-account-multi-service)
  - [Q2 — Terraform Apply Crashes Midway — Recovery](#q2-terraform-apply-crashes-midway--recovery)
  - [Q3 — Prevent Destroy-Recreate of EKS Node Group](#q3-prevent-destroy-recreate-of-eks-node-group)
  - [Q4 — Governance Model After Accidental Production Destroy](#q4-governance-model-after-accidental-production-destroy)
  - [Q5 — Importing 200+ Console-Created Resources](#q5-importing-200-console-created-resources)
  - [Q6 — Safe RDS PostgreSQL Major Version Upgrade](#q6-safe-rds-postgresql-major-version-upgrade)
  - [Q7 — Handling Breaking Module Changes Across 30 Environments](#q7-handling-breaking-module-changes-across-30-environments)
  - [Q8 — Optimizing Slow Terraform Plan/Apply](#q8-optimizing-slow-terraform-planapply)
  - [Q9 — State Locking and Concurrent Access](#q9-state-locking-and-concurrent-access)
  - [Q10 — Multi-Account Multi-Region Provider Configuration](#q10-multi-account-multi-region-provider-configuration)
  - [Q11 — Terraform taint vs replace — Forcing Recreation](#q11-terraform-taint-vs-replace--forcing-resource-recreation)
  - [Q12 — Secrets Management — Handling Sensitive Variables](#q12-terraform-secrets-management--handling-sensitive-variables)
  - [Q13 — Data Sources vs Resources](#q13-terraform-data-sources-vs-resource--when-to-use-which)
  - [Q14 — Workspaces vs Directory-Based Separation](#q14-terraform-workspaces-vs-directory-based-environment-separation)
  - [Q15 — Moved Blocks — Refactoring Without Destroying](#q15-terraform-moved-blocks--refactoring-without-destroying)
  - [Q16 — Dynamic Blocks and Complex Variables](#q16-terraform-dynamic-blocks-and-complex-variable-structures)
  - [Q17 — State Manipulation Commands](#q17-terraform-state-manipulation-commands)
  - [Q18 — Conditional Logic and Expressions](#q18-terraform-conditional-logic-and-expressions)
  - [Q19 — depends_on — Explicit vs Implicit Dependencies](#q19-terraform-depends_on--explicit-vs-implicit-dependencies)
  - [Q20 — Providers — Pinning, Mirrors, and Custom Providers](#q20-terraform-providers--custom-providers-version-pinning-and-mirror-registries)
- [Docker](#docker) *(coming soon)*
- [Kubernetes](#kubernetes) *(coming soon)*
- [EKS](#eks) *(coming soon)*
- [AWS](#aws) *(coming soon)*
- [Prometheus](#prometheus) *(coming soon)*
- [Grafana](#grafana) *(coming soon)*

---

# Terraform

## Q1. Codebase Structure for Multi-Account Multi-Service

### Question

> You're managing infrastructure for 15 microservices across 3 AWS accounts (dev, staging, prod) using Terraform. Your team is growing from 3 to 10 engineers. How would you structure your Terraform codebase — mono-repo vs multi-repo, module design, state management, and workspace strategy? Walk me through your approach and justify your choices.

### Real-Life Scenario

You join a company where 3 engineers were managing everything in one big `main.tf`. Now the team is scaling to 10 people, and everyone is stepping on each other's toes — merge conflicts, accidental changes to prod, and no code reuse.

### Answer

#### A) Mono-Repo vs Multi-Repo — Go with Mono-Repo

For 15 microservices across 3 accounts, choose **mono-repo** because:

- Shared modules are easier to version and test
- Single PR shows cross-cutting changes (e.g., updating a VPC module affects all services)
- Easier CI/CD pipeline setup
- Code reviews are centralized

Multi-repo makes sense only when teams are fully independent (e.g., different business units).

#### B) Folder Structure

```
infrastructure/
├── modules/                        # Reusable modules
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   ├── eks-cluster/
│   ├── rds/
│   ├── s3-bucket/
│   ├── iam-roles/
│   └── security-groups/
│
├── environments/                   # Environment-specific deployments
│   ├── dev/
│   │   ├── account.hcl             # Account ID, region
│   │   ├── networking/
│   │   │   ├── main.tf             # Calls module "vpc"
│   │   │   ├── backend.tf
│   │   │   └── terraform.tfvars
│   │   ├── microservice-order/
│   │   ├── microservice-payment/
│   │   └── shared-infra/           # RDS, ElastiCache, etc.
│   ├── staging/
│   │   └── ... (same structure)
│   └── prod/
│       └── ... (same structure)
│
├── policies/                       # Sentinel / OPA policies
│   ├── restrict-instance-types.sentinel
│   └── require-tags.sentinel
│
└── scripts/
    ├── import-resources.sh
    └── drift-detection.sh
```

#### C) State Management

Each microservice per environment gets its **own state file**. Never share state across services.

```hcl
# environments/prod/microservice-order/backend.tf
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state-prod"
    key            = "prod/microservice-order/terraform.tfstate"
    region         = "ap-south-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"     # Prevents concurrent applies
  }
}
```

**Why separate state files?**

- If one state file corrupts, only one service is affected
- Different teams can run `terraform apply` simultaneously without locking each other
- Blast radius is limited

#### D) DynamoDB Locking (Critical with 10 engineers)

```hcl
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

When engineer A runs `terraform apply`, engineer B trying to apply the same workspace gets:

```
Error: Error locking state: ConditionalCheckFailedException:
The conditional request failed. Lock Info: ID: xxx, Operation: OperationTypeApply
```

#### E) Workspaces — Avoid Default Workspaces, Use Folder-Based Separation

Prefer folder-per-environment over `terraform workspace` because:

- Workspaces share the same backend config — risky
- Folder separation is explicit and visible in PRs
- Each folder has its own `.tfvars` — no room for mistakes

#### F) Module Versioning with Tags

```hcl
module "vpc" {
  source  = "git::https://github.com/mycompany/terraform-modules.git//vpc?ref=v2.1.0"
  # or if mono-repo:
  source  = "../../modules/vpc"
}
```

**Real-world tip:** Use **Terragrunt** if you find yourself repeating backend configs and provider blocks across 45 folders (15 services × 3 envs):

```hcl
# terragrunt.hcl (DRY backend config)
remote_state {
  backend = "s3"
  generate = { path = "backend.tf", if_exists = "overwrite" }
  config = {
    bucket         = "mycompany-tf-state-${local.account_name}"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = "ap-south-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

---

## Q2. Terraform Apply Crashes Midway — Recovery

### Question

> During a `terraform apply`, the process crashes midway — 4 out of 10 resources were created. The state file is now partially updated. Your team is panicking because it's a production VPC with subnets and route tables. How do you recover? What if the state file got corrupted?

### Real-Life Scenario

You ran `terraform apply` to create a VPC with 3 subnets, 3 route tables, a NAT gateway, and an internet gateway. Your laptop lost internet at resource #4. Now you have a VPC and 3 subnets created in AWS, but the route tables and gateways are missing. The state file only knows about the VPC and 2 subnets.

### Answer

#### Step 1: DON'T PANIC. Do NOT re-run `terraform apply` immediately.

First, check what Terraform thinks it knows:

```bash
terraform state list
```

Output might show:

```
aws_vpc.main
aws_subnet.private[0]
aws_subnet.private[1]
```

But you know AWS actually has `aws_subnet.private[2]` too.

#### Step 2: Check what actually exists in AWS

```bash
aws ec2 describe-subnets --filters "Name=vpc-id,Values=vpc-0abc123" \
  --query "Subnets[*].[SubnetId,CidrBlock,Tags]" --output table
```

#### Step 3: Import the missing resources into state

```bash
terraform import 'aws_subnet.private[2]' subnet-0xyz789

# Verify
terraform state show 'aws_subnet.private[2]'
```

#### Step 4: Run plan to verify alignment

```bash
terraform plan
```

If the plan shows `0 to add, 0 to change, 0 to destroy` — you're clean. If it shows remaining resources to create (route tables, gateways), that's expected — those never got created.

#### Step 5: Apply the remaining resources

```bash
terraform apply
```

#### What if the state file is CORRUPTED (unreadable JSON)?

```bash
# Step 1: Check for backup (Terraform auto-creates terraform.tfstate.backup)
ls -la terraform.tfstate.backup

# Step 2: If using S3 backend with versioning
aws s3api list-object-versions \
  --bucket mycompany-terraform-state-prod \
  --prefix prod/networking/terraform.tfstate \
  --query "Versions[0:5].[VersionId,LastModified,Size]" --output table

# Step 3: Restore a previous version
aws s3api get-object \
  --bucket mycompany-terraform-state-prod \
  --key prod/networking/terraform.tfstate \
  --version-id "abc123previousversion" \
  restored-terraform.tfstate

# Step 4: Push restored state back
terraform force-unlock LOCK_ID
terraform state push restored-terraform.tfstate
```

#### Prevention — ALWAYS enable versioning on your state bucket

```hcl
resource "aws_s3_bucket_versioning" "state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_public_access_block" "state" {
  bucket                  = aws_s3_bucket.terraform_state.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

#### Nuclear Option — State totally gone, no backup

```bash
# Re-import EVERY resource
terraform import aws_vpc.main vpc-0abc123
terraform import 'aws_subnet.private[0]' subnet-0aaa
terraform import 'aws_subnet.private[1]' subnet-0bbb
terraform import 'aws_subnet.private[2]' subnet-0ccc
terraform import aws_nat_gateway.main nat-0ddd

# Then run plan and fix diffs until clean
terraform plan
```

---

## Q3. Prevent Destroy-Recreate of EKS Node Group

### Question

> You have a Terraform module that provisions an EKS cluster. A developer makes a change to the `node_group` configuration, and `terraform plan` shows it wants to destroy and recreate the entire node group, which would cause downtime. How do you handle this? What Terraform lifecycle features and strategies would you use?

### Real-Life Scenario

A developer changed the `ami_type` from `AL2_x86_64` to `AL2023_x86_64` in the EKS node group module. `terraform plan` output:

```
# module.eks.aws_eks_node_group.workers must be replaced
-/+ resource "aws_eks_node_group" "workers" {
      ~ ami_type = "AL2_x86_64" -> "AL2023_x86_64"  # forces replacement
    }

Plan: 1 to add, 0 to change, 1 to destroy.
```

All pods on those nodes get killed. In production. During business hours.

### Answer

#### Strategy 1: `create_before_destroy` lifecycle

```hcl
resource "aws_eks_node_group" "workers" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "workers-${var.ami_type}"   # Dynamic name!
  node_role_arn   = aws_iam_role.node.arn
  subnet_ids      = var.private_subnet_ids
  ami_type        = var.ami_type

  scaling_config {
    desired_size = 3
    max_size     = 5
    min_size     = 1
  }

  lifecycle {
    create_before_destroy = true    # New node group comes up FIRST
  }
}
```

What happens:

1. Terraform creates the new node group with AL2023
2. New nodes join the cluster
3. Once healthy, Terraform destroys the old node group
4. Kubernetes reschedules pods to new nodes

#### Strategy 2: `ignore_changes` for externally managed attributes

```hcl
resource "aws_eks_node_group" "workers" {
  # ...

  lifecycle {
    ignore_changes = [
      scaling_config[0].desired_size,   # Cluster Autoscaler manages this
      ami_type,                          # Managed via rolling update
    ]
  }
}
```

#### Strategy 3: Blue/Green Node Groups (safest for production)

```hcl
variable "active_node_group" {
  default = "blue"
}

resource "aws_eks_node_group" "blue" {
  count           = var.active_node_group == "blue" || var.active_node_group == "both" ? 1 : 0
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "workers-blue"
  ami_type        = "AL2_x86_64"
  # ...
}

resource "aws_eks_node_group" "green" {
  count           = var.active_node_group == "green" || var.active_node_group == "both" ? 1 : 0
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "workers-green"
  ami_type        = "AL2023_x86_64"
  # ...
}
```

Deployment steps:

```bash
# Step 1: Bring up green alongside blue
terraform apply -var="active_node_group=both"

# Step 2: Cordon and drain blue nodes
kubectl cordon -l eks.amazonaws.com/nodegroup=workers-blue
kubectl drain -l eks.amazonaws.com/nodegroup=workers-blue \
  --ignore-daemonsets --delete-emptydir-data

# Step 3: Verify pods are running on green
kubectl get pods -o wide

# Step 4: Remove blue
terraform apply -var="active_node_group=green"
```

#### Strategy 4: `moved` block (Terraform 1.1+)

```hcl
moved {
  from = aws_eks_node_group.workers
  to   = aws_eks_node_group.workers_v2
}
```

Tells Terraform "this isn't a destroy+create, it's a rename" — no downtime.

#### Troubleshooting: Identify which attribute forces replacement

```bash
# The plan output shows it:
# ~ ami_type = "AL2_x86_64" -> "AL2023_x86_64"  # forces replacement

# Use JSON output for scripting:
terraform plan -json | jq '.resource_changes[] | select(.change.actions | contains(["delete"]))'
```

---

## Q4. Governance Model After Accidental Production Destroy

### Question

> Your organization uses Terraform Cloud/Enterprise. A junior engineer accidentally runs `terraform destroy` on the production workspace. What guardrails, policies (Sentinel/OPA), and operational practices should have been in place to prevent this? Design a governance model.

### Real-Life Scenario

Monday morning. Slack explodes. A junior engineer ran `terraform destroy` on the `prod-networking` workspace in Terraform Cloud. The production VPC, subnets, NAT gateways, and all route tables are gone. 200+ services are down.

### Answer

**This should NEVER be possible. Here's a 7-layer governance model:**

#### Layer 1: Terraform Cloud Workspace Settings

```
Workspace: prod-networking
├── Execution Mode:          Remote (never local for prod)
├── Apply Method:            Manual Approve (NO auto-apply)
├── Destruction Protection:  ENABLED  ← #1 control
└── VCS-Driven Workflow:     PR-only (no CLI triggers for prod)
```

In Terraform Cloud UI → Workspace → Settings → Destruction and Deletion → Enable **"Prevent workspace destruction"**.

#### Layer 2: Sentinel Policies (Terraform Cloud/Enterprise)

```python
# policy: restrict-destroy-prod.sentinel
import "tfplan/v2" as tfplan
import "strings"

param workspace_name

main = rule {
    not strings.has_prefix(workspace_name, "prod-") or
    length([for r in tfplan.resource_changes as rc
        if rc.change.actions contains "delete"
    ]) == 0
}

# Enforcement level: hard-mandatory (cannot override)
```

#### Layer 3: OPA (Open Policy Agent) — for non-TFC setups

```rego
# policy/deny_prod_destroy.rego
package terraform

deny[msg] {
    input.resource_changes[_].change.actions[_] == "delete"
    contains(input.configuration.root_module.variables.environment.default, "prod")
    msg := "DENIED: Cannot destroy resources in production environment"
}
```

```bash
terraform plan -out=plan.tfplan
terraform show -json plan.tfplan > plan.json
opa eval --data policy/ --input plan.json "data.terraform.deny"
```

#### Layer 4: RBAC — Role-Based Access Control

| Team | dev-* | staging-* | prod-* |
|------|-------|-----------|--------|
| Junior Engineers | Write | Plan-only | **Read-only** |
| Platform Engineers | Admin | Write (no destroy) | **Plan-only** |
| Senior Architects | Admin | Admin | Admin (destroy needs 2 approvals) |

#### Layer 5: CI/CD Pipeline Controls

```yaml
# .github/workflows/terraform-prod.yml
name: Terraform Production
on:
  pull_request:
    branches: [main]
    paths: ['environments/prod/**']

jobs:
  terraform:
    runs-on: ubuntu-latest
    environment: production          # Requires manual approval
    steps:
      - uses: actions/checkout@v4

      - name: Terraform Plan
        run: terraform plan -out=plan.tfplan

      - name: Check for Destroys
        run: |
          DESTROYS=$(terraform show -json plan.tfplan | \
            jq '[.resource_changes[] | select(.change.actions | contains(["delete"]))] | length')
          if [ "$DESTROYS" -gt "0" ]; then
            echo "::error::Plan contains $DESTROYS resource deletions!"
            exit 1
          fi

      - name: Apply (only after 2 approvals)
        if: github.event.review.state == 'approved'
        run: terraform apply plan.tfplan
```

#### Layer 6: AWS-Level Safeguards (Defense in Depth)

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"

  lifecycle {
    prevent_destroy = true    # Terraform will ERROR if destroy is attempted
  }
}

resource "aws_db_instance" "production" {
  # ...
  deletion_protection = true   # AWS-level protection

  lifecycle {
    prevent_destroy = true     # Terraform-level protection
  }
}
```

Error when someone tries to destroy:

```
Error: Instance cannot be destroyed
  on main.tf line 15:
  Resource aws_vpc.main has lifecycle.prevent_destroy set,
  but the plan calls for this resource to be destroyed.
```

#### Layer 7: Break-Glass Procedure (for legitimate destroys)

```
1. Create a JIRA ticket with business justification
2. Architecture review approval (2 senior architects)
3. Change Advisory Board (CAB) approval
4. Schedule maintenance window
5. Temporarily remove prevent_destroy via approved PR
6. Run: terraform destroy -target=specific_resource (NEVER full destroy)
7. Verify and restore prevent_destroy
```

---

## Q5. Importing 200+ Console-Created Resources

### Question

> You're importing 200+ existing AWS resources (VPCs, subnets, security groups, EC2 instances, RDS) into Terraform that were originally created via the AWS Console. Walk me through your migration strategy — tooling, state import, code generation, validation, and how you'd handle drift detection going forward.

### Real-Life Scenario

The company has been running on AWS for 3 years. Everything was clicked through the Console. Now management wants "Infrastructure as Code." You're staring at 50 security groups, 30 subnets, 15 EC2 instances, 5 RDS databases, 10 S3 buckets, and dozens of IAM roles — all unmanaged.

### Answer

#### Phase 1: Discovery — What exists?

```bash
# VPCs
aws ec2 describe-vpcs \
  --query "Vpcs[*].[VpcId,CidrBlock,Tags[?Key=='Name'].Value|[0]]" --output table

# Subnets
aws ec2 describe-subnets \
  --query "Subnets[*].[SubnetId,VpcId,CidrBlock,AvailabilityZone]" --output table

# Security Groups
aws ec2 describe-security-groups \
  --query "SecurityGroups[*].[GroupId,GroupName,VpcId]" --output table

# EC2 Instances
aws ec2 describe-instances \
  --query "Reservations[*].Instances[*].[InstanceId,InstanceType,State.Name,Tags[?Key=='Name'].Value|[0]]" \
  --output table

# RDS
aws rds describe-db-instances \
  --query "DBInstances[*].[DBInstanceIdentifier,Engine,DBInstanceClass]" --output table
```

#### Phase 2: Auto-Generate Terraform Code

```bash
# Option A: Terraformer (by Google)
brew install terraformer

terraformer import aws \
  --resources=vpc,subnet,sg,igw,nat,route_table,eip \
  --regions=ap-south-1 \
  --profile=production

# Generates:
# generated/aws/vpc/
#   ├── vpc.tf
#   ├── outputs.tf
#   ├── provider.tf
#   └── terraform.tfstate

# Option B: aws2tf
git clone https://github.com/aws-samples/aws2tf.git
cd aws2tf
./aws2tf.sh -t vpc -i vpc-0abc123 -r ap-south-1
```

#### Phase 3: Manual Import (for critical resources)

```bash
# Step 1: Write the Terraform code skeleton FIRST
cat > vpc.tf << 'EOF'
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags = {
    Name = "production-vpc"
  }
}
EOF

# Step 2: Import the resource into state
terraform import aws_vpc.main vpc-0abc123def456

# Step 3: See what Terraform knows — copy attributes to your .tf
terraform state show aws_vpc.main

# Step 4: Run plan and fix diffs until clean
terraform plan
# Repeat until: "No changes. Your infrastructure matches the configuration."
```

#### Phase 4: Bulk Import Script

```bash
#!/bin/bash
# scripts/import-all.sh
set -e

echo "=== Importing VPCs ==="
terraform import 'aws_vpc.main' vpc-0abc123

echo "=== Importing Subnets ==="
terraform import 'aws_subnet.private["ap-south-1a"]' subnet-0aaa111
terraform import 'aws_subnet.private["ap-south-1b"]' subnet-0bbb222
terraform import 'aws_subnet.public["ap-south-1a"]' subnet-0ccc333

echo "=== Importing Security Groups ==="
terraform import 'aws_security_group.web' sg-0ddd444
terraform import 'aws_security_group.app' sg-0eee555
terraform import 'aws_security_group.db' sg-0fff666

echo "=== Importing RDS ==="
terraform import 'aws_db_instance.production' mydb-production

echo "=== Importing EC2 (dynamic) ==="
for i in $(aws ec2 describe-instances \
  --filters "Name=tag:Environment,Values=production" \
  --query "Reservations[*].Instances[*].InstanceId" \
  --output text); do
  echo "Importing $i"
  terraform import "aws_instance.app[\"$i\"]" "$i"
done

echo "=== Verifying ==="
terraform plan -detailed-exitcode
```

#### Phase 5: Modern Approach — Terraform 1.5+ Import Blocks

```hcl
# imports.tf — Declarative imports, no CLI needed!
import {
  to = aws_vpc.main
  id = "vpc-0abc123"
}

import {
  to = aws_subnet.private["ap-south-1a"]
  id = "subnet-0aaa111"
}

import {
  to = aws_security_group.web
  id = "sg-0ddd444"
}
```

```bash
# Auto-generate the .tf code from imports:
terraform plan -generate-config-out=generated_resources.tf

# Review, clean up, then:
terraform apply
```

#### Phase 6: Drift Detection (Ongoing)

```yaml
# .github/workflows/drift-detection.yml
name: Drift Detection
on:
  schedule:
    - cron: '0 6 * * *'   # Daily at 6 AM

jobs:
  detect-drift:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        workspace: [prod-networking, prod-eks, prod-rds]
    steps:
      - uses: actions/checkout@v4
      - name: Terraform Plan
        run: |
          cd environments/prod/${{ matrix.workspace }}
          terraform init
          terraform plan -detailed-exitcode 2>&1 | tee plan.txt
          EXIT_CODE=$?
          if [ $EXIT_CODE -eq 2 ]; then
            echo "DRIFT DETECTED in ${{ matrix.workspace }}"
            curl -X POST "$SLACK_WEBHOOK" \
              -d '{"text":"⚠️ Drift detected in ${{ matrix.workspace }}"}'
          fi
```

#### Phase 7: Migration Priority Order

| Week | Layer | Why This Order |
|------|-------|----------------|
| 1-2 | Networking (VPC, Subnets, Route Tables, NAT, IGW) | Foundational — everything depends on them |
| 3 | Security (Security Groups, NACLs, IAM Roles) | Must be stable before touching compute |
| 4-5 | Data (RDS, ElastiCache, S3) | Use `prevent_destroy = true` |
| 6-7 | Compute (EC2, ECS, EKS, Lambda) | Most dynamic — import last |
| 8 | DNS & CDN (Route53, CloudFront), Monitoring (CloudWatch) | Least risky |

#### Troubleshooting Common Import Issues

```bash
# Problem: "Resource already managed by Terraform"
terraform state list | grep vpc
terraform state rm aws_vpc.main
terraform import aws_vpc.main vpc-0abc123

# Problem: "Error importing: Not Found"
# Check region and resource ID
aws ec2 describe-vpcs --vpc-ids vpc-0abc123 --region ap-south-1

# Problem: "Plan shows changes after import"
# Your .tf code doesn't exactly match reality
terraform state show aws_vpc.main
# Copy EVERY attribute into your .tf file
# Common culprits: missing tags, wrong boolean values, default values
```

---

## Q6. Safe RDS PostgreSQL Major Version Upgrade

### Question

> You're using Terraform to manage an RDS PostgreSQL instance in production. The business needs you to upgrade from PostgreSQL 14 to PostgreSQL 16. `terraform plan` shows an in-place update, but you're worried about data loss and downtime. How do you plan and execute this upgrade safely using Terraform? What if the upgrade fails midway?

### Real-Life Scenario

Your production RDS PostgreSQL 14 instance has 500GB of data, serves 20 microservices, and handles 5,000 queries/second. Management wants PostgreSQL 16 for performance improvements and new features (logical replication improvements, `pg_stat_io`). You can't afford more than 15 minutes of downtime.

### Answer

#### Step 1: Pre-Upgrade Checks

```bash
# Check current version and instance details
aws rds describe-db-instances \
  --db-instance-identifier prod-main-db \
  --query "DBInstances[0].[EngineVersion,DBInstanceClass,MultiAZ,StorageType]"

# Check available upgrade paths (you CANNOT skip major versions)
aws rds describe-db-engine-versions \
  --engine postgres \
  --engine-version 14.11 \
  --query "DBEngineVersions[0].ValidUpgradeTarget[*].[EngineVersion]" --output table

# Output might show: 14.12, 15.x, 16.x — PostgreSQL allows 14 → 16 directly
```

```bash
# Check for breaking changes / incompatible parameters
aws rds describe-db-parameters \
  --db-parameter-group-name prod-pg14-params \
  --query "Parameters[?IsModifiable=='true'].[ParameterName,ParameterValue]" --output table
```

#### Step 2: Create a Manual Snapshot BEFORE Anything

```bash
aws rds create-db-snapshot \
  --db-instance-identifier prod-main-db \
  --db-snapshot-identifier prod-main-db-pre-pg16-upgrade-$(date +%Y%m%d)

# Wait for snapshot to complete
aws rds wait db-snapshot-available \
  --db-snapshot-identifier prod-main-db-pre-pg16-upgrade-20260511
```

#### Step 3: Test the Upgrade on a Clone First

```hcl
# test-upgrade/main.tf — Create a clone from snapshot
resource "aws_db_instance" "upgrade_test" {
  identifier          = "prod-db-upgrade-test"
  snapshot_identifier = "prod-main-db-pre-pg16-upgrade-20260511"
  instance_class      = "db.r6g.xlarge"    # Same as prod
  engine              = "postgres"
  engine_version      = "16.3"             # Target version
  parameter_group_name = aws_db_parameter_group.pg16_test.name
  skip_final_snapshot  = true

  tags = {
    Purpose = "upgrade-test"
    DeleteAfter = "2026-05-15"
  }
}
```

```bash
terraform apply
# Monitor the upgrade process
# Test your application against the clone
# Run your test suite against it
# Check for deprecated functions, extension compatibility
```

#### Step 4: Create a PostgreSQL 16 Parameter Group

```hcl
resource "aws_db_parameter_group" "pg16" {
  name   = "prod-pg16-params"
  family = "postgres16"

  parameter {
    name  = "shared_preload_libraries"
    value = "pg_stat_statements,pg_cron"
  }

  parameter {
    name  = "max_connections"
    value = "500"
  }

  parameter {
    name  = "work_mem"
    value = "256000"    # 256MB
  }

  # ... copy all custom parameters from pg14 group
}
```

#### Step 5: Execute the Upgrade via Terraform

```hcl
resource "aws_db_instance" "production" {
  identifier          = "prod-main-db"
  engine              = "postgres"
  engine_version      = "16.3"              # Changed from "14.11"
  instance_class      = "db.r6g.xlarge"
  parameter_group_name = aws_db_parameter_group.pg16.name   # New PG16 group
  multi_az            = true
  deletion_protection = true
  allow_major_version_upgrade = true        # REQUIRED for major upgrades

  # Upgrade during maintenance window
  apply_immediately   = false               # Wait for maintenance window
  # OR set to true and schedule manually during off-peak

  lifecycle {
    prevent_destroy = true
  }
}
```

```bash
terraform plan
# Output:
# ~ engine_version      = "14.11" -> "16.3"
# ~ parameter_group_name = "prod-pg14-params" -> "prod-pg16-params"
# ~ allow_major_version_upgrade = false -> true

# Apply during maintenance window
terraform apply
```

#### Step 6: Monitor the Upgrade

```bash
# Watch RDS events during upgrade
aws rds describe-events \
  --source-identifier prod-main-db \
  --source-type db-instance \
  --duration 60 \
  --query "Events[*].[Date,Message]" --output table

# Check instance status
watch -n 10 "aws rds describe-db-instances \
  --db-instance-identifier prod-main-db \
  --query 'DBInstances[0].[DBInstanceStatus,EngineVersion]'"
```

**Expected downtime:** 10–30 minutes for Multi-AZ (failover happens first, then upgrade on old primary).

#### What if the Upgrade Fails Midway?

```bash
# RDS automatically rolls back on failure — your data is safe
# Check the event log:
aws rds describe-events --source-identifier prod-main-db

# If stuck in "upgrading" state for too long:
# 1. Check RDS console → Logs → postgresql.log.upgrade
# 2. Common failures:
#    - Incompatible extensions (PostGIS version mismatch)
#    - Custom functions using removed features
#    - Parameter group incompatibilities

# Restore from pre-upgrade snapshot if needed:
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier prod-main-db-restored \
  --db-snapshot-identifier prod-main-db-pre-pg16-upgrade-20260511

# Then update your Terraform to point to the restored instance
# and import it
```

#### Rollback Strategy in Terraform

```hcl
# If you need to rollback, revert your .tf code:
resource "aws_db_instance" "production" {
  engine_version       = "14.11"    # Revert
  parameter_group_name = aws_db_parameter_group.pg14.name  # Revert
  # ...
}
```

> **WARNING:** RDS does NOT support downgrading major versions in-place. You MUST restore from the pre-upgrade snapshot. That's why Step 2 is critical.

---

## Q7. Handling Breaking Module Changes Across 30 Environments

### Question

> Your team uses Terraform modules from a private registry. A module update introduces a breaking change — the `subnet_ids` variable changed from `list(string)` to `map(object)`. 30 environments reference this module. How do you handle this breaking change without disrupting any running infrastructure? What versioning and deprecation strategy would you follow?

### Real-Life Scenario

Your VPC module v2.x has been stable for a year. All 30 environments (15 services × 2 non-prod envs + critical prod ones) use it. Now you need to redesign the subnet configuration for multi-AZ support. The variable changes from:

```hcl
# v2.x (old)
variable "subnet_ids" {
  type = list(string)
}
```

to:

```hcl
# v3.x (new)
variable "subnets" {
  type = map(object({
    cidr_block        = string
    availability_zone = string
    public            = bool
  }))
}
```

### Answer

#### Step 1: Semantic Versioning — NEVER break on minor/patch

```
v2.5.0  ← current stable (all 30 envs use this)
v2.6.0  ← add new variable alongside old one (backward compatible)
v2.7.0  ← deprecation warnings
v3.0.0  ← remove old variable (breaking change)
```

#### Step 2: Create v2.6.0 — Support BOTH old and new interfaces

```hcl
# modules/vpc/variables.tf

# OLD interface (deprecated but still works)
variable "subnet_ids" {
  type        = list(string)
  default     = []
  description = "DEPRECATED: Use 'subnets' variable instead. Will be removed in v3.0.0."
}

# NEW interface
variable "subnets" {
  type = map(object({
    cidr_block        = string
    availability_zone = string
    public            = bool
  }))
  default     = {}
  description = "Subnet configuration map. Replaces subnet_ids."
}
```

```hcl
# modules/vpc/main.tf

locals {
  # If old variable is used, convert to new format
  use_legacy = length(var.subnet_ids) > 0 && length(var.subnets) == 0

  effective_subnets = local.use_legacy ? {
    for idx, id in var.subnet_ids : "subnet-${idx}" => {
      cidr_block        = ""          # looked up from data source
      availability_zone = ""
      public            = false
      id                = id
    }
  } : var.subnets
}

# Deprecation warning
resource "null_resource" "deprecation_warning" {
  count = local.use_legacy ? 1 : 0

  triggers = {
    warning = "WARNING: 'subnet_ids' is deprecated. Migrate to 'subnets' before v3.0.0"
  }

  provisioner "local-exec" {
    command = "echo 'WARNING: subnet_ids is deprecated in VPC module. Migrate to subnets map.'"
  }
}
```

#### Step 3: Pin Versions in Every Environment

```hcl
# environments/prod/networking/main.tf

module "vpc" {
  source  = "app.terraform.io/mycompany/vpc/aws"
  version = "~> 2.5"     # Uses 2.5.x, won't auto-upgrade to 3.x

  # Old way (still works on 2.6.0)
  subnet_ids = ["subnet-aaa", "subnet-bbb"]
}
```

#### Step 4: Migrate Environments Gradually

```hcl
# Migration order: dev → staging → prod (one service at a time)

# BEFORE (v2.5)
module "vpc" {
  source  = "app.terraform.io/mycompany/vpc/aws"
  version = "2.5.0"
  subnet_ids = ["subnet-aaa", "subnet-bbb", "subnet-ccc"]
}

# AFTER (v2.6 with new interface)
module "vpc" {
  source  = "app.terraform.io/mycompany/vpc/aws"
  version = "2.6.0"

  subnets = {
    "private-1a" = {
      cidr_block        = "10.0.1.0/24"
      availability_zone = "ap-south-1a"
      public            = false
    }
    "private-1b" = {
      cidr_block        = "10.0.2.0/24"
      availability_zone = "ap-south-1b"
      public            = false
    }
    "public-1a" = {
      cidr_block        = "10.0.101.0/24"
      availability_zone = "ap-south-1a"
      public            = true
    }
  }
}
```

```bash
# After changing, ALWAYS verify no infra changes:
terraform plan
# Expected: "No changes" or only tag updates — NEVER destroy/recreate
```

#### Step 5: Release v3.0.0 — Remove old variable

```hcl
# modules/vpc/variables.tf (v3.0.0)
# subnet_ids variable is GONE

variable "subnets" {
  type = map(object({
    cidr_block        = string
    availability_zone = string
    public            = bool
  }))
  # No default — required
}
```

#### Step 6: Add Validation and Custom Error Messages

```hcl
# v3.0.0 — helpful error if someone still uses old interface
variable "subnet_ids" {
  type    = list(string)
  default = []
  validation {
    condition     = length(var.subnet_ids) == 0
    error_message = "subnet_ids was removed in v3.0.0. Use 'subnets' map instead. See migration guide: https://wiki.mycompany.com/vpc-module-v3"
  }
}
```

#### CI/CD: Auto-Detect Outdated Module Versions

```bash
# Script to find all environments still on old versions
grep -r "version.*=.*\"2\." environments/ --include="*.tf" -l

# Output:
# environments/prod/microservice-payment/main.tf
# environments/staging/microservice-order/main.tf
```

---

## Q8. Optimizing Slow Terraform Plan/Apply

### Question

> You have a Terraform configuration that creates 50 IAM roles with complex policies. `terraform plan` takes 12 minutes and `terraform apply` takes 25 minutes. Engineers are complaining. What strategies would you use to optimize Terraform performance? Walk me through state splitting, parallelism tuning, targeted applies, and provider caching.

### Real-Life Scenario

Your monolithic Terraform config manages everything: VPC, EKS, RDS, 50 IAM roles, 30 security groups, S3 buckets, CloudWatch alarms, Lambda functions — all in one state. `terraform plan` refreshes ALL 300+ resources every time, even when you only changed one IAM policy.

### Answer

#### Strategy 1: State Splitting (Biggest Impact)

Split the monolith into logical components:

```
# BEFORE: One giant state (300+ resources, 12-min plan)
environments/prod/
└── main.tf          ← everything in here

# AFTER: Separate states per concern
environments/prod/
├── networking/       ← VPC, subnets, NAT (rarely changes)
│   ├── main.tf
│   └── backend.tf   ← separate state file
├── iam/              ← 50 IAM roles
│   ├── main.tf
│   └── backend.tf   ← separate state file
├── eks/              ← EKS cluster
│   ├── main.tf
│   └── backend.tf
├── data-stores/      ← RDS, ElastiCache
│   ├── main.tf
│   └── backend.tf
└── monitoring/       ← CloudWatch, alarms
    ├── main.tf
    └── backend.tf
```

Cross-reference using `terraform_remote_state` or `data` sources:

```hcl
# environments/prod/eks/main.tf
data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = "mycompany-terraform-state-prod"
    key    = "prod/networking/terraform.tfstate"
    region = "ap-south-1"
  }
}

resource "aws_eks_cluster" "main" {
  vpc_config {
    subnet_ids = data.terraform_remote_state.networking.outputs.private_subnet_ids
  }
}
```

**Result:** Plan for IAM-only changes now takes **45 seconds** instead of 12 minutes.

#### Strategy 2: Parallelism Tuning

```bash
# Default parallelism is 10 (10 API calls at a time)
# Increase for large configs with independent resources
terraform apply -parallelism=30

# For plans
terraform plan -parallelism=30

# Beware: AWS API rate limits — if you see throttling:
# "ThrottlingException: Rate exceeded"
# Reduce to 15-20
```

#### Strategy 3: Targeted Applies

```bash
# Only plan/apply a specific resource:
terraform plan -target=aws_iam_role.lambda_executor
terraform apply -target=aws_iam_role.lambda_executor

# Target a module:
terraform plan -target=module.iam_roles

# Multiple targets:
terraform apply \
  -target=aws_iam_role.lambda_executor \
  -target=aws_iam_policy.lambda_policy
```

> **Warning:** `-target` is for emergencies/debugging. Don't use it as a workflow — it skips dependency validation.

#### Strategy 4: Skip Refresh When Safe

```bash
# Skip the "Refreshing state" phase (risky if someone changed AWS manually)
terraform plan -refresh=false

# This skips checking 300+ resources against AWS API
# Plan drops from 12 minutes to 2 minutes
# ONLY use when you're confident no manual changes were made
```

#### Strategy 5: Provider Plugin Caching

```bash
# Terraform downloads providers on every init by default
# Set up a local plugin cache:

# Linux/Mac:
export TF_PLUGIN_CACHE_DIR="$HOME/.terraform.d/plugin-cache"
mkdir -p $TF_PLUGIN_CACHE_DIR

# Windows (PowerShell):
$env:TF_PLUGIN_CACHE_DIR = "$env:APPDATA\terraform.d\plugin-cache"

# Or in .terraformrc:
plugin_cache_dir = "$HOME/.terraform.d/plugin-cache"
```

#### Strategy 6: Use `count` and `for_each` Efficiently

```hcl
# SLOW: 50 separate resource blocks
resource "aws_iam_role" "role_1" { ... }
resource "aws_iam_role" "role_2" { ... }
# ... 48 more

# FAST: Single block with for_each
resource "aws_iam_role" "roles" {
  for_each = var.role_configs   # Map of 50 roles

  name               = each.key
  assume_role_policy = each.value.assume_policy
  tags               = each.value.tags
}
```

#### Strategy 7: Use `-compact-warnings` and Reduce Output

```bash
terraform plan -compact-warnings 2>&1 | tail -20
```

#### Performance Comparison After Optimization

| Scenario | Before | After |
|----------|--------|-------|
| Full plan (300 resources) | 12 min | N/A (split now) |
| IAM changes only | 12 min | 45 sec |
| EKS node group change | 12 min | 1.5 min |
| Network change | 12 min | 30 sec |
| `-refresh=false` | 12 min | 2 min |
| `-parallelism=30` | 12 min | 7 min |

---

## Q9. State Locking and Concurrent Access

### Question

> Two engineers are working on the same Terraform workspace. Engineer A adds a new subnet, Engineer B modifies a security group. Both run `terraform apply` almost simultaneously. What happens? How does Terraform handle state locking, and what do you do when a lock gets stuck? Describe a real scenario where you'd need to use `terraform force-unlock`.

### Real-Life Scenario

It's Friday afternoon. Engineer A starts `terraform apply` to add a new private subnet. The apply is expected to take 3 minutes. 30 seconds later, Engineer B (who doesn't know A is running) tries to apply a security group change. B gets a lock error. A's laptop then crashes mid-apply, and the lock is now stuck in DynamoDB — nobody can apply anything to production networking.

### Answer

#### What Happens When Two Engineers Apply Simultaneously

```
Timeline:
  T+0s:   Engineer A runs terraform apply → Acquires DynamoDB lock ✅
  T+30s:  Engineer B runs terraform apply → BLOCKED ❌

  Engineer B sees:
  Error: Error acquiring the state lock
  Error message: ConditionalCheckFailedException: The conditional request failed
  Lock Info:
    ID:        a1b2c3d4-e5f6-7890-abcd-ef1234567890
    Path:      mycompany-terraform-state-prod/prod/networking/terraform.tfstate
    Operation: OperationTypeApply
    Who:       engineer-a@laptop
    Version:   1.7.0
    Created:   2026-05-11 14:32:00 UTC
    Info:

  Terraform acquires a state lock to protect the state from being
  written by multiple users at the same time.
```

#### How DynamoDB Locking Works

```
1. terraform apply starts
2. Terraform writes a LOCK record to DynamoDB:
   {
     "LockID": "mycompany-terraform-state-prod/prod/networking/terraform.tfstate",
     "Info": "{\"ID\":\"a1b2c3d4...\",\"Operation\":\"OperationTypeApply\",\"Who\":\"engineer-a@laptop\",...}"
   }
3. Uses DynamoDB Conditional Write (PutItem with ConditionExpression)
   - If LockID doesn't exist → lock acquired ✅
   - If LockID exists → lock denied ❌
4. After apply completes, Terraform DELETES the lock record
```

#### Stuck Lock — Engineer A's Laptop Crashes

```bash
# Engineer A's laptop crashes at T+90s. The lock is never released.
# Engineer B (or anyone) sees:

Error: Error acquiring the state lock
Lock Info:
  ID:        a1b2c3d4-e5f6-7890-abcd-ef1234567890
  Operation: OperationTypeApply
  Who:       engineer-a@laptop
  Created:   2026-05-11 14:32:00 UTC

# The lock will stay FOREVER until manually removed
```

#### Step-by-Step Recovery

```bash
# Step 1: Verify the lock is actually stuck (wait a few minutes first)
# Check if Engineer A is still running:
# - Slack them, check their laptop status
# - If they confirm they're NOT running, proceed

# Step 2: Check DynamoDB directly
aws dynamodb get-item \
  --table-name terraform-locks \
  --key '{"LockID":{"S":"mycompany-terraform-state-prod/prod/networking/terraform.tfstate"}}' \
  --output json

# Step 3: Check state file for corruption
aws s3 cp s3://mycompany-terraform-state-prod/prod/networking/terraform.tfstate - | python -m json.tool > /dev/null
# If this succeeds, state is valid JSON

# Step 4: Force unlock using the Lock ID from the error message
terraform force-unlock a1b2c3d4-e5f6-7890-abcd-ef1234567890

# You'll see:
# Do you really want to force-unlock?
#   Terraform will remove the lock on the remote state.
#   This will allow local Terraform commands to modify this state...
# Enter a value: yes
# Terraform state has been successfully unlocked!

# Step 5: Verify state is correct
terraform plan
# Check the plan carefully — some resources might be partially created
```

#### When You Need `force-unlock` — Real Scenarios

| Scenario | What Happened | Action |
|----------|---------------|--------|
| Laptop crash | Engineer's laptop died mid-apply | `force-unlock` + verify state + import missing resources |
| Network timeout | VPN dropped during apply | `force-unlock` + re-run plan to check |
| CI/CD runner killed | GitHub Actions runner timed out | `force-unlock` in next pipeline run |
| Ctrl+C during apply | Engineer killed the process | Sometimes Terraform auto-unlocks; if not, `force-unlock` |
| OOM kill | Terraform process killed by OS (huge state file) | `force-unlock` + split state |

#### Preventing This — Operational Practices

```hcl
# 1. Always use remote state with locking
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state-prod"
    key            = "prod/networking/terraform.tfstate"
    region         = "ap-south-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"   # NEVER skip this
  }
}
```

```yaml
# 2. Run applies ONLY through CI/CD (not laptops)
# .github/workflows/terraform.yml
concurrency:
  group: terraform-prod-networking    # Only one apply at a time
  cancel-in-progress: false           # Don't cancel running applies!
```

```bash
# 3. Use -lock-timeout instead of immediate failure
terraform apply -lock-timeout=5m
# Waits up to 5 minutes for the lock instead of failing immediately
# Perfect when you know a colleague's apply will finish soon
```

```bash
# 4. Add lock timeout in CI/CD
terraform apply -lock-timeout=10m -auto-approve
```

---

## Q10. Multi-Account Multi-Region Provider Configuration

### Question

> You need to manage resources across 3 AWS accounts (dev, staging, prod) and 2 regions (ap-south-1, us-east-1) from a single Terraform configuration. How do you configure multiple providers with assume-role, and how do you structure your code to avoid duplication? Show the provider configuration and cross-account resource creation.

### Real-Life Scenario

You have a central "management" AWS account where your CI/CD runs. From this account, you need to:
- Create a VPC in the dev account (ap-south-1)
- Create an S3 bucket in the prod account (us-east-1) for DR
- Create IAM roles in all 3 accounts
- Set up VPC peering between dev and prod accounts

### Answer

#### Step 1: IAM Roles for Cross-Account Access

In EACH target account (dev, staging, prod), create an assume-role:

```hcl
# In dev account (111111111111):
resource "aws_iam_role" "terraform_access" {
  name = "TerraformCrossAccountRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        AWS = "arn:aws:iam::000000000000:root"  # Management account
      }
      Action = "sts:AssumeRole"
      Condition = {
        StringEquals = {
          "sts:ExternalId" = "terraform-cross-account-dev"
        }
      }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "terraform_admin" {
  role       = aws_iam_role.terraform_access.name
  policy_arn = "arn:aws:iam::aws:policy/AdministratorAccess"
  # In production, use a custom least-privilege policy
}
```

#### Step 2: Provider Configuration with Aliases

```hcl
# providers.tf

# Default provider (management account, primary region)
provider "aws" {
  region = "ap-south-1"
}

# ─── Dev Account ───────────────────────────────────
provider "aws" {
  alias  = "dev_mumbai"
  region = "ap-south-1"

  assume_role {
    role_arn     = "arn:aws:iam::111111111111:role/TerraformCrossAccountRole"
    session_name = "terraform-dev"
    external_id  = "terraform-cross-account-dev"
  }
}

provider "aws" {
  alias  = "dev_virginia"
  region = "us-east-1"

  assume_role {
    role_arn     = "arn:aws:iam::111111111111:role/TerraformCrossAccountRole"
    session_name = "terraform-dev"
    external_id  = "terraform-cross-account-dev"
  }
}

# ─── Staging Account ──────────────────────────────
provider "aws" {
  alias  = "staging_mumbai"
  region = "ap-south-1"

  assume_role {
    role_arn     = "arn:aws:iam::222222222222:role/TerraformCrossAccountRole"
    session_name = "terraform-staging"
    external_id  = "terraform-cross-account-staging"
  }
}

# ─── Prod Account ─────────────────────────────────
provider "aws" {
  alias  = "prod_mumbai"
  region = "ap-south-1"

  assume_role {
    role_arn     = "arn:aws:iam::333333333333:role/TerraformCrossAccountRole"
    session_name = "terraform-prod"
    external_id  = "terraform-cross-account-prod"
  }
}

provider "aws" {
  alias  = "prod_virginia"
  region = "us-east-1"

  assume_role {
    role_arn     = "arn:aws:iam::333333333333:role/TerraformCrossAccountRole"
    session_name = "terraform-prod"
    external_id  = "terraform-cross-account-prod"
  }
}
```

#### Step 3: Use Providers in Resources

```hcl
# VPC in dev account (Mumbai)
resource "aws_vpc" "dev" {
  provider   = aws.dev_mumbai
  cidr_block = "10.1.0.0/16"
  tags       = { Name = "dev-vpc", Environment = "dev" }
}

# S3 bucket in prod account (Virginia — for DR)
resource "aws_s3_bucket" "prod_dr_backup" {
  provider = aws.prod_virginia
  bucket   = "mycompany-prod-dr-backup-us-east-1"
  tags     = { Name = "prod-dr-backup", Environment = "prod" }
}

# IAM role in staging account
resource "aws_iam_role" "staging_app_role" {
  provider = aws.staging_mumbai
  name     = "application-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}
```

#### Step 4: Cross-Account VPC Peering

```hcl
# Requester side (dev account)
resource "aws_vpc_peering_connection" "dev_to_prod" {
  provider      = aws.dev_mumbai
  vpc_id        = aws_vpc.dev.id
  peer_vpc_id   = aws_vpc.prod.id
  peer_owner_id = "333333333333"    # Prod account ID
  peer_region   = "ap-south-1"
  auto_accept   = false

  tags = { Name = "dev-to-prod-peering" }
}

# Accepter side (prod account)
resource "aws_vpc_peering_connection_accepter" "prod_accept" {
  provider                  = aws.prod_mumbai
  vpc_peering_connection_id = aws_vpc_peering_connection.dev_to_prod.id
  auto_accept               = true

  tags = { Name = "dev-to-prod-peering" }
}

# Route in dev VPC to prod
resource "aws_route" "dev_to_prod" {
  provider                  = aws.dev_mumbai
  route_table_id            = aws_route_table.dev_private.id
  destination_cidr_block    = "10.3.0.0/16"    # Prod CIDR
  vpc_peering_connection_id = aws_vpc_peering_connection.dev_to_prod.id
}

# Route in prod VPC to dev
resource "aws_route" "prod_to_dev" {
  provider                  = aws.prod_mumbai
  route_table_id            = aws_route_table.prod_private.id
  destination_cidr_block    = "10.1.0.0/16"    # Dev CIDR
  vpc_peering_connection_id = aws_vpc_peering_connection.dev_to_prod.id
}
```

#### Step 5: DRY Approach Using Modules with Provider Passthrough

```hcl
# modules/vpc/main.tf
terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
    }
  }
}

resource "aws_vpc" "this" {
  cidr_block           = var.cidr_block
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags                 = merge(var.tags, { Name = var.name })
}
```

```hcl
# Root module — pass providers to the module
module "dev_vpc" {
  source = "./modules/vpc"
  providers = {
    aws = aws.dev_mumbai       # This module uses dev account
  }
  name       = "dev-vpc"
  cidr_block = "10.1.0.0/16"
  tags       = { Environment = "dev" }
}

module "prod_vpc" {
  source = "./modules/vpc"
  providers = {
    aws = aws.prod_mumbai      # This module uses prod account
  }
  name       = "prod-vpc"
  cidr_block = "10.3.0.0/16"
  tags       = { Environment = "prod" }
}
```

#### Step 6: Dynamic Provider Configuration Using Variables

```hcl
# variables.tf
variable "accounts" {
  type = map(object({
    account_id  = string
    role_name   = string
    external_id = string
    regions     = list(string)
  }))

  default = {
    dev = {
      account_id  = "111111111111"
      role_name   = "TerraformCrossAccountRole"
      external_id = "terraform-cross-account-dev"
      regions     = ["ap-south-1"]
    }
    staging = {
      account_id  = "222222222222"
      role_name   = "TerraformCrossAccountRole"
      external_id = "terraform-cross-account-staging"
      regions     = ["ap-south-1"]
    }
    prod = {
      account_id  = "333333333333"
      role_name   = "TerraformCrossAccountRole"
      external_id = "terraform-cross-account-prod"
      regions     = ["ap-south-1", "us-east-1"]
    }
  }
}
```

#### Troubleshooting Common Issues

```bash
# Problem: "AccessDenied when calling AssumeRole"
# Check trust policy in target account:
aws sts assume-role \
  --role-arn "arn:aws:iam::111111111111:role/TerraformCrossAccountRole" \
  --role-session-name test \
  --external-id terraform-cross-account-dev

# Problem: "Provider configuration not present"
# Forgot to pass provider to module:
module "vpc" {
  source = "./modules/vpc"
  providers = {
    aws = aws.dev_mumbai     # ← MUST include this
  }
}

# Problem: "Conflicting provider version constraints"
# Each module specifies its own provider requirements
# Use required_providers block in modules to set constraints
```

---

## Q11. Terraform `taint` vs `replace` — Forcing Resource Recreation

### Question

> A production EC2 instance is behaving strangely — the application is running but the instance's user-data script didn't execute properly during the last apply. You need to force Terraform to destroy and recreate this specific instance without affecting other resources. What are your options? Explain `taint`, `-replace`, and when to use each.

### Real-Life Scenario

You provisioned 5 EC2 instances using `for_each`. Instance `app-server-3` has a corrupted user-data — the CloudWatch agent never installed. The application works but you have no monitoring on that one instance. You need to recreate just this one instance without touching the other 4.

### Answer

#### Option 1: `terraform taint` (Legacy — Deprecated in Terraform 1.5+)

```bash
# Mark a specific resource as tainted
terraform taint 'aws_instance.app["app-server-3"]'

# Check what Terraform will do
terraform plan
# Output:
# -/+ resource "aws_instance" "app["app-server-3"]" (tainted, will be replaced)

# Apply — only app-server-3 gets recreated
terraform apply
```

**Problem with `taint`:**
- It modifies the state file immediately (even before apply)
- If you change your mind, you need to `untaint`
- It's a separate step — error-prone in CI/CD

```bash
# Undo a taint
terraform untaint 'aws_instance.app["app-server-3"]'
```

#### Option 2: `-replace` Flag (Modern — Terraform 1.5+, Recommended)

```bash
# Plan with replacement — does NOT modify state
terraform plan -replace='aws_instance.app["app-server-3"]'

# Output:
# -/+ resource "aws_instance" "app["app-server-3"]" (will be replaced)
#     4 other instances: no changes

# Apply with replacement
terraform apply -replace='aws_instance.app["app-server-3"]'
```

**Why `-replace` is better:**
- Does NOT modify state until apply runs
- Atomic — plan and apply in one flow
- Can be used in CI/CD safely
- Can replace multiple resources in one command:

```bash
terraform apply \
  -replace='aws_instance.app["app-server-3"]' \
  -replace='aws_instance.app["app-server-5"]'
```

#### Option 3: `create_before_destroy` with `-replace`

```hcl
resource "aws_instance" "app" {
  for_each      = var.app_servers
  ami           = var.ami_id
  instance_type = var.instance_type

  lifecycle {
    create_before_destroy = true   # New instance comes up before old one dies
  }
}
```

```bash
terraform apply -replace='aws_instance.app["app-server-3"]'
# Step 1: New app-server-3 created and healthy
# Step 2: Old app-server-3 terminated
# Result: Minimal downtime
```

#### Real-World: When to Force-Recreate

| Scenario | Use `-replace`? | Alternative |
|----------|----------------|-------------|
| User-data didn't run | Yes | SSM Run Command to run script manually |
| Instance in degraded state | Yes | AWS EC2 stop/start (keeps instance ID) |
| AMI updated | No — change `ami` in `.tf` | Terraform will auto-replace |
| EBS volume issue | `-replace` the volume | Snapshot + restore |
| Security patch needed | No — use SSM Patch Manager | Immutable infra: new AMI + replace |

#### Troubleshooting

```bash
# Problem: "Resource not found in state"
terraform state list | grep app-server
# Check the exact resource address — quotes and brackets matter

# Problem: "Cannot replace a resource that doesn't exist"
# The resource was already destroyed or never created
terraform plan  # Check what exists

# Problem: "Replacement would cause downtime"
# Use create_before_destroy lifecycle + target group health checks
# ELB will drain connections from old instance first
```

---

## Q12. Terraform Secrets Management — Handling Sensitive Variables

### Question

> Your Terraform configuration needs database passwords, API keys, and SSL certificates. A junior engineer committed a `terraform.tfvars` file with plaintext passwords to Git. How do you remediate this, and what is the proper way to manage secrets in Terraform? Compare at least 4 approaches.

### Real-Life Scenario

You run `git log --all --full-history -- terraform.tfvars` and discover that `db_password = "Pr0duction!P@ss"` has been in the repo for 6 months. Even if you delete the file now, the password is in Git history. Meanwhile, the state file also contains the password in plaintext.

### Answer

#### Immediate Remediation

```bash
# Step 1: Rotate the compromised password IMMEDIATELY
aws rds modify-db-instance \
  --db-instance-identifier prod-main-db \
  --master-user-password "NewSecureP@ssw0rd$(openssl rand -hex 8)" \
  --apply-immediately

# Step 2: Remove the file from Git history (BFG is faster than git filter-branch)
# Install BFG: https://rtyley.github.io/bfg-repo-cleaner/
java -jar bfg.jar --delete-files terraform.tfvars
git reflog expire --expire=now --all
git gc --prune=now --aggressive
git push --force    # ⚠️ Coordinate with team first

# Step 3: Add to .gitignore
echo "*.tfvars" >> .gitignore
echo "*.auto.tfvars" >> .gitignore
git add .gitignore
git commit -m "Prevent tfvars from being committed"

# Step 4: Invalidate any API keys that were exposed
# Rotate ALL secrets that were in that file
```

#### Approach 1: AWS Secrets Manager + Data Source (Recommended)

```hcl
# Store the secret in AWS Secrets Manager first:
# aws secretsmanager create-secret --name prod/db/password --secret-string "MyS3cur3P@ss"

data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "prod/db/password"
}

resource "aws_db_instance" "production" {
  identifier     = "prod-main-db"
  engine         = "postgres"
  instance_class = "db.r6g.xlarge"
  username       = "admin"
  password       = data.aws_secretsmanager_secret_version.db_password.secret_string

  lifecycle {
    ignore_changes = [password]   # Don't show password in plan output
  }
}
```

**Advantages:** Secrets are centralized, audited (CloudTrail), and rotatable.

#### Approach 2: HashiCorp Vault Provider

```hcl
provider "vault" {
  address = "https://vault.mycompany.com:8200"
  # Auth via VAULT_TOKEN env var or AWS IAM auth
}

data "vault_generic_secret" "db" {
  path = "secret/data/production/database"
}

resource "aws_db_instance" "production" {
  username = data.vault_generic_secret.db.data["username"]
  password = data.vault_generic_secret.db.data["password"]
}
```

#### Approach 3: Environment Variables

```bash
# Set secrets as env vars (CI/CD pipeline or local)
export TF_VAR_db_password="MyS3cur3P@ss"
export TF_VAR_api_key="sk-live-abc123"

# Terraform automatically reads TF_VAR_* variables
terraform apply
```

```hcl
# variables.tf
variable "db_password" {
  type      = string
  sensitive = true    # Hides from plan output (Terraform 0.14+)
}

resource "aws_db_instance" "production" {
  password = var.db_password
}
```

```bash
# Plan output shows:
# ~ password = (sensitive value)
# Never shows the actual password
```

#### Approach 4: SOPS (Mozilla) — Encrypted Files in Git

```bash
# Install SOPS
brew install sops

# Create encrypted tfvars
sops --encrypt --kms "arn:aws:kms:ap-south-1:123456789:key/abc-123" \
  secrets.tfvars.json > secrets.enc.tfvars.json

# Decrypt at runtime
sops --decrypt secrets.enc.tfvars.json > secrets.tfvars.json
terraform apply -var-file=secrets.tfvars.json
rm secrets.tfvars.json   # Clean up plaintext immediately
```

#### State File — The Hidden Danger

```bash
# Even with sensitive variables, the STATE FILE contains plaintext!
terraform state show aws_db_instance.production
# Output includes: password = "MyS3cur3P@ss"

# MITIGATIONS:
# 1. Encrypt state at rest (S3 SSE-KMS)
# 2. Restrict state file access via IAM
# 3. Use Terraform Cloud (state is encrypted and access-controlled)
```

```hcl
# S3 backend with KMS encryption
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "ap-south-1"
    encrypt        = true
    kms_key_id     = "arn:aws:kms:ap-south-1:123456789:key/abc-123"
    dynamodb_table = "terraform-locks"
  }
}
```

#### Comparison Matrix

| Approach | Security Level | Complexity | Rotation | Audit Trail | Cost |
|----------|---------------|------------|----------|-------------|------|
| AWS Secrets Manager | High | Low | Auto-rotation | CloudTrail | ~$0.40/secret/month |
| HashiCorp Vault | Very High | High | Auto-rotation | Built-in | Self-hosted or $$$  |
| Environment Variables | Medium | Low | Manual | None | Free |
| SOPS | High | Medium | Manual | Git history | Free (KMS cost) |
| Plaintext in tfvars | **NONE** | Low | Manual | None | Free (but costly when breached) |

---

## Q13. Terraform `data` Sources vs `resource` — When to Use Which

### Question

> A new engineer keeps creating `resource` blocks for things that already exist (like the default VPC, existing IAM policies, or Route53 hosted zones). Explain the difference between `data` sources and `resource` blocks. When should you use each? Show a scenario where using the wrong one causes a disaster.

### Real-Life Scenario

An engineer writes:

```hcl
resource "aws_iam_policy" "admin" {
  name = "AdministratorAccess"
  # ...
}
```

`terraform apply` fails: "EntityAlreadyExists: A policy named AdministratorAccess already exists." They then try `terraform import`, and now Terraform manages AWS's built-in AdministratorAccess policy. Later, someone runs `terraform destroy` and it **deletes the AdministratorAccess policy from the entire AWS account**.

### Answer

#### The Fundamental Difference

```
resource = "I want Terraform to CREATE and MANAGE this"
data     = "I want Terraform to READ an existing thing (that I don't manage)"
```

#### When to Use `data`

```hcl
# 1. AWS-managed policies (you didn't create these)
data "aws_iam_policy" "admin" {
  name = "AdministratorAccess"
}
# Use: data.aws_iam_policy.admin.arn

# 2. Existing VPC (created by another team or another Terraform config)
data "aws_vpc" "existing" {
  filter {
    name   = "tag:Name"
    values = ["production-vpc"]
  }
}
# Use: data.aws_vpc.existing.id

# 3. Current AWS account info
data "aws_caller_identity" "current" {}
# Use: data.aws_caller_identity.current.account_id

# 4. AMI lookup (find the latest Amazon Linux 2023)
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}
# Use: data.aws_ami.amazon_linux.id

# 5. Availability zones
data "aws_availability_zones" "available" {
  state = "available"
}
# Use: data.aws_availability_zones.available.names

# 6. Route53 zone (already exists, you just need the zone ID)
data "aws_route53_zone" "main" {
  name = "mycompany.com"
}
# Use: data.aws_route53_zone.main.zone_id
```

#### When to Use `resource`

```hcl
# 1. Something YOU create and want Terraform to manage its lifecycle
resource "aws_iam_role" "app_role" {
  name = "my-app-role"
  # Terraform creates it, updates it, and can destroy it
}

# 2. New infrastructure
resource "aws_vpc" "new" {
  cidr_block = "10.0.0.0/16"
}

# 3. Custom IAM policies (not AWS-managed)
resource "aws_iam_policy" "custom_s3_read" {
  name = "custom-s3-read-only"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["s3:GetObject", "s3:ListBucket"]
      Resource = ["arn:aws:s3:::my-bucket", "arn:aws:s3:::my-bucket/*"]
    }]
  })
}
```

#### The Disaster Scenario — Using Resource Instead of Data

```hcl
# ❌ WRONG — This imports and manages AWS's built-in policy
resource "aws_iam_policy" "admin" {
  name = "AdministratorAccess"
}
# If someone runs terraform destroy → ALL admin access in the account is gone!

# ✅ CORRECT — Just read the existing policy
data "aws_iam_policy" "admin" {
  name = "AdministratorAccess"
}

resource "aws_iam_role_policy_attachment" "admin_attach" {
  role       = aws_iam_role.admin_role.name
  policy_arn = data.aws_iam_policy.admin.arn   # Read-only reference
}
```

#### Common Pattern — Data Source for Cross-State References

```hcl
# Team A manages networking (separate state)
# Team B needs the VPC ID and subnet IDs

# Option 1: terraform_remote_state
data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = "mycompany-terraform-state"
    key    = "prod/networking/terraform.tfstate"
    region = "ap-south-1"
  }
}

resource "aws_instance" "app" {
  subnet_id = data.terraform_remote_state.networking.outputs.private_subnet_ids[0]
}

# Option 2: AWS data sources (no state coupling)
data "aws_subnets" "private" {
  filter {
    name   = "tag:Type"
    values = ["private"]
  }
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.existing.id]
  }
}
```

#### Troubleshooting

```bash
# Problem: "No matching data source found"
# Your filter/name doesn't match anything in AWS
aws ec2 describe-vpcs --filters "Name=tag:Name,Values=production-vpc"
# Verify the resource exists and the filter is correct

# Problem: "Multiple data sources matched"
# Add more specific filters or use most_recent = true (for AMIs)
data "aws_ami" "latest" {
  most_recent = true        # ← Takes the newest match
  owners      = ["amazon"]
  # ...
}

# Problem: Data source returns empty/wrong values
terraform console
> data.aws_vpc.existing
# Inspect all attributes interactively
```

---

## Q14. Terraform Workspaces vs Directory-Based Environment Separation

### Question

> Your team debates whether to use Terraform workspaces (`terraform workspace new prod`) or directory-based separation (`environments/prod/`, `environments/dev/`) for managing multiple environments. You need to present both approaches with pros, cons, and real-world failure cases. Which do you recommend for a team managing 15 microservices across 3 accounts?

### Real-Life Scenario

The current setup uses workspaces. An engineer runs `terraform workspace select prod` on their laptop, forgets they're in prod, makes a "test change," and applies it. Production goes down. Meanwhile, another team member runs `terraform plan` without checking which workspace they're in — the plan output doesn't clearly show they're looking at production.

### Answer

#### Terraform Workspaces — How They Work

```bash
# Create workspaces
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod

# Switch workspace
terraform workspace select prod

# List workspaces
terraform workspace list
#   default
#   dev
#   staging
# * prod          ← current

# Check current workspace in code
terraform.workspace   # Returns "prod"
```

```hcl
# Using workspace name in configurations
resource "aws_instance" "app" {
  instance_type = terraform.workspace == "prod" ? "m5.xlarge" : "t3.medium"
  tags = {
    Environment = terraform.workspace
  }
}

# Backend — each workspace gets a separate state file automatically
terraform {
  backend "s3" {
    bucket = "mycompany-terraform-state"
    key    = "app/terraform.tfstate"     # Workspaces add prefix automatically
    region = "ap-south-1"
  }
}
# State paths become:
#   env:/dev/app/terraform.tfstate
#   env:/staging/app/terraform.tfstate
#   env:/prod/app/terraform.tfstate
```

#### Directory-Based Separation — How It Works

```
environments/
├── dev/
│   ├── main.tf
│   ├── backend.tf          # key = "dev/app/terraform.tfstate"
│   ├── terraform.tfvars    # instance_type = "t3.medium"
│   └── provider.tf         # assume_role → dev account
├── staging/
│   ├── main.tf
│   ├── backend.tf          # key = "staging/app/terraform.tfstate"
│   ├── terraform.tfvars
│   └── provider.tf
└── prod/
    ├── main.tf
    ├── backend.tf          # key = "prod/app/terraform.tfstate"
    ├── terraform.tfvars    # instance_type = "m5.xlarge"
    └── provider.tf         # assume_role → prod account
```

```hcl
# environments/prod/terraform.tfvars
environment    = "prod"
instance_type  = "m5.xlarge"
instance_count = 5
multi_az       = true

# environments/dev/terraform.tfvars
environment    = "dev"
instance_type  = "t3.medium"
instance_count = 1
multi_az       = false
```

#### Comparison

| Feature | Workspaces | Directory-Based |
|---------|-----------|----------------|
| State isolation | Automatic (same backend, prefixed keys) | Manual (different backend keys) |
| Provider per env | Same provider config | Different provider per dir (different accounts) |
| Variable per env | Conditional logic (`terraform.workspace == "prod"`) | Separate `.tfvars` files — explicit |
| Risk of wrong env | **HIGH** — one wrong `workspace select` | **LOW** — you `cd` into the right folder |
| Code duplication | None — single codebase | Some — `main.tf` repeated (use modules to reduce) |
| PR visibility | Hard to see which env is affected | Clear — PR touches `environments/prod/` |
| Multi-account | Difficult — same provider block | Natural — each dir has its own provider |
| CI/CD | Complex — must pass workspace name | Simple — trigger per directory path |

#### The Workspace Disaster — Why I Recommend Directories

```bash
# The accident that happens with workspaces:

# Engineer thinks they're in dev:
terraform workspace select dev
# ... makes changes ...
terraform apply   # ✅ dev updated

# Next day, forgets to switch back:
# (they're still in prod from yesterday)
terraform apply   # ❌ APPLIED TO PROD!

# With directories, this CAN'T happen:
cd environments/dev
terraform apply   # Only dev files here, dev backend, dev provider
# There's no way to accidentally hit prod
```

#### My Recommendation for 15 Microservices, 3 Accounts

**Use directory-based separation + shared modules:**

```
infrastructure/
├── modules/
│   ├── microservice/         # Shared module
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│
├── environments/
│   ├── dev/
│   │   └── microservice-order/
│   │       ├── main.tf       # module "app" { source = "../../../modules/microservice" }
│   │       ├── backend.tf
│   │       └── terraform.tfvars
│   ├── staging/
│   │   └── microservice-order/
│   └── prod/
│       └── microservice-order/
```

```hcl
# environments/prod/microservice-order/main.tf
module "app" {
  source = "../../../modules/microservice"

  environment    = "prod"
  instance_type  = "m5.xlarge"
  instance_count = 5
  # ...
}
```

**Zero code duplication** in `main.tf` — all logic lives in the module. Each environment directory only has:
1. Module call with env-specific variables
2. Backend config (unique state path)
3. Provider config (unique account)

---

## Q15. Terraform `moved` Blocks — Refactoring Without Destroying

### Question

> You need to refactor your Terraform codebase — renaming resources, moving resources into modules, and changing `count` to `for_each`. Each of these would normally cause Terraform to destroy and recreate resources. How do you use `moved` blocks to refactor safely? Show examples for each scenario.

### Real-Life Scenario

Your initial Terraform code was written by a junior engineer 2 years ago. Resources have names like `aws_instance.server1`, `aws_instance.server2`. You need to refactor to `aws_instance.app["web"]`, `aws_instance.app["api"]` using `for_each`. Without `moved` blocks, Terraform would destroy both instances and create new ones.

### Answer

#### Scenario 1: Renaming a Resource

```hcl
# BEFORE
resource "aws_instance" "server1" {
  ami           = "ami-abc123"
  instance_type = "m5.xlarge"
}

# AFTER — renamed to something meaningful
resource "aws_instance" "web_server" {
  ami           = "ami-abc123"
  instance_type = "m5.xlarge"
}

# Tell Terraform: "web_server IS server1, don't recreate it"
moved {
  from = aws_instance.server1
  to   = aws_instance.web_server
}
```

```bash
terraform plan
# Output:
# aws_instance.server1 has moved to aws_instance.web_server
# No changes. Your infrastructure matches the configuration.
```

#### Scenario 2: Moving Resources INTO a Module

```hcl
# BEFORE — flat resources in root
resource "aws_vpc" "main" { ... }
resource "aws_subnet" "private" { ... }
resource "aws_nat_gateway" "main" { ... }

# AFTER — moved into a module
module "networking" {
  source     = "./modules/networking"
  cidr_block = "10.0.0.0/16"
}

# Tell Terraform about the moves
moved {
  from = aws_vpc.main
  to   = module.networking.aws_vpc.main
}
moved {
  from = aws_subnet.private
  to   = module.networking.aws_subnet.private
}
moved {
  from = aws_nat_gateway.main
  to   = module.networking.aws_nat_gateway.main
}
```

#### Scenario 3: Converting `count` to `for_each`

```hcl
# BEFORE — using count (fragile, index-based)
resource "aws_instance" "app" {
  count         = 3
  ami           = "ami-abc123"
  instance_type = "m5.large"
  tags = {
    Name = "app-${count.index}"
  }
}
# State addresses: aws_instance.app[0], aws_instance.app[1], aws_instance.app[2]

# AFTER — using for_each (stable, key-based)
variable "app_servers" {
  default = {
    "web"  = { type = "m5.large" }
    "api"  = { type = "m5.large" }
    "worker" = { type = "m5.large" }
  }
}

resource "aws_instance" "app" {
  for_each      = var.app_servers
  ami           = "ami-abc123"
  instance_type = each.value.type
  tags = {
    Name = "app-${each.key}"
  }
}

# Map old index-based addresses to new key-based addresses
moved {
  from = aws_instance.app[0]
  to   = aws_instance.app["web"]
}
moved {
  from = aws_instance.app[1]
  to   = aws_instance.app["api"]
}
moved {
  from = aws_instance.app[2]
  to   = aws_instance.app["worker"]
}
```

```bash
terraform plan
# Output:
# aws_instance.app[0] has moved to aws_instance.app["web"]
# aws_instance.app[1] has moved to aws_instance.app["api"]
# aws_instance.app[2] has moved to aws_instance.app["worker"]
# No changes.
```

#### Scenario 4: Moving Between Modules

```hcl
# Resource was in module.old_infra, now moving to module.new_infra
moved {
  from = module.old_infra.aws_s3_bucket.data
  to   = module.new_infra.aws_s3_bucket.data
}
```

#### Scenario 5: Renaming a Module Itself

```hcl
# BEFORE
module "database" {
  source = "./modules/rds"
}

# AFTER
module "primary_database" {
  source = "./modules/rds"
}

moved {
  from = module.database
  to   = module.primary_database
}
# ALL resources inside the module are moved automatically
```

#### When to Remove `moved` Blocks

```hcl
# moved blocks are safe to keep forever, but you can remove them after:
# 1. All environments (dev, staging, prod) have been applied
# 2. The state has been updated in all workspaces
# 3. You're confident no one is running old code

# Good practice: add a comment with removal date
moved {
  from = aws_instance.server1
  to   = aws_instance.web_server
  # Safe to remove after 2026-06-01 (all envs migrated)
}
```

#### Troubleshooting

```bash
# Problem: "Moved object still exists"
# You have both the old AND new resource defined
# Remove the old resource block, keep only the new one + moved block

# Problem: "Cannot move to an address that already exists"
# The target resource already exists in state
terraform state list | grep web_server
# Remove the conflicting resource from state first:
terraform state rm aws_instance.web_server

# Problem: "Unsupported: moved blocks between different resource types"
# You can't move aws_instance to aws_launch_template
# moved only works within the same resource type
```

---

## Q16. Terraform Dynamic Blocks and Complex Variable Structures

### Question

> You need to create 20 security groups, each with different numbers of ingress and egress rules. Some have 2 rules, some have 15. A junior engineer created 20 separate `resource` blocks with hardcoded rules. How would you refactor this using `dynamic` blocks, `for_each`, and complex variable types?

### Real-Life Scenario

Your security group Terraform file is 800 lines long. Every time someone needs a new rule, they copy-paste an entire block and modify IPs. This leads to merge conflicts, inconsistencies, and audit nightmares. You need a data-driven approach where security groups and their rules are defined in a single variable.

### Answer

#### Step 1: Design the Variable Structure

```hcl
# variables.tf
variable "security_groups" {
  type = map(object({
    description = string
    vpc_id      = string
    ingress_rules = list(object({
      description     = string
      from_port       = number
      to_port         = number
      protocol        = string
      cidr_blocks     = optional(list(string), [])
      security_groups = optional(list(string), [])
    }))
    egress_rules = list(object({
      description = string
      from_port   = number
      to_port     = number
      protocol    = string
      cidr_blocks = list(string)
    }))
    tags = optional(map(string), {})
  }))
}
```

#### Step 2: Define Rules as Data

```hcl
# terraform.tfvars
security_groups = {
  "web-alb" = {
    description = "ALB Security Group"
    vpc_id      = "vpc-0abc123"
    ingress_rules = [
      {
        description = "HTTPS from internet"
        from_port   = 443
        to_port     = 443
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
      },
      {
        description = "HTTP from internet"
        from_port   = 80
        to_port     = 80
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
      }
    ]
    egress_rules = [
      {
        description = "All outbound"
        from_port   = 0
        to_port     = 0
        protocol    = "-1"
        cidr_blocks = ["0.0.0.0/0"]
      }
    ]
    tags = { Tier = "public" }
  }

  "app-server" = {
    description = "Application Server SG"
    vpc_id      = "vpc-0abc123"
    ingress_rules = [
      {
        description     = "From ALB on port 8080"
        from_port       = 8080
        to_port         = 8080
        protocol        = "tcp"
        cidr_blocks     = ["10.0.0.0/16"]
      },
      {
        description = "SSH from bastion"
        from_port   = 22
        to_port     = 22
        protocol    = "tcp"
        cidr_blocks = ["10.0.1.0/24"]
      }
    ]
    egress_rules = [
      {
        description = "All outbound"
        from_port   = 0
        to_port     = 0
        protocol    = "-1"
        cidr_blocks = ["0.0.0.0/0"]
      }
    ]
    tags = { Tier = "private" }
  }
}
```

#### Step 3: Use `dynamic` Blocks with `for_each`

```hcl
# main.tf
resource "aws_security_group" "this" {
  for_each    = var.security_groups
  name        = each.key
  description = each.value.description
  vpc_id      = each.value.vpc_id

  dynamic "ingress" {
    for_each = each.value.ingress_rules
    content {
      description     = ingress.value.description
      from_port       = ingress.value.from_port
      to_port         = ingress.value.to_port
      protocol        = ingress.value.protocol
      cidr_blocks     = ingress.value.cidr_blocks
      security_groups = ingress.value.security_groups
    }
  }

  dynamic "egress" {
    for_each = each.value.egress_rules
    content {
      description = egress.value.description
      from_port   = egress.value.from_port
      to_port     = egress.value.to_port
      protocol    = egress.value.protocol
      cidr_blocks = egress.value.cidr_blocks
    }
  }

  tags = merge(
    { Name = each.key, ManagedBy = "terraform" },
    each.value.tags
  )
}
```

#### Result

```bash
terraform plan
# Output:
# + aws_security_group.this["web-alb"]     — 2 ingress, 1 egress
# + aws_security_group.this["app-server"]  — 2 ingress, 1 egress
# ... 18 more security groups

# Adding a new rule = add to the variable, NOT touch the resource code
```

#### Nested Dynamic Blocks (Advanced — e.g., S3 Lifecycle Rules)

```hcl
variable "s3_buckets" {
  type = map(object({
    lifecycle_rules = list(object({
      id      = string
      enabled = bool
      prefix  = string
      transitions = list(object({
        days          = number
        storage_class = string
      }))
      expiration_days = optional(number)
    }))
  }))
}

resource "aws_s3_bucket_lifecycle_configuration" "this" {
  for_each = var.s3_buckets
  bucket   = aws_s3_bucket.this[each.key].id

  dynamic "rule" {
    for_each = each.value.lifecycle_rules
    content {
      id     = rule.value.id
      status = rule.value.enabled ? "Enabled" : "Disabled"

      filter {
        prefix = rule.value.prefix
      }

      dynamic "transition" {
        for_each = rule.value.transitions
        content {
          days          = transition.value.days
          storage_class = transition.value.storage_class
        }
      }

      dynamic "expiration" {
        for_each = rule.value.expiration_days != null ? [rule.value.expiration_days] : []
        content {
          days = expiration.value
        }
      }
    }
  }
}
```

#### Troubleshooting

```bash
# Problem: "dynamic block iterator 'ingress' not expected here"
# Wrong nesting or typo in the dynamic block name
# The dynamic block name must match the resource's nested block name

# Problem: "Inappropriate value for attribute: list of string required"
# Type mismatch — check your variable type definition
# Use terraform console to debug:
terraform console
> var.security_groups["web-alb"].ingress_rules[0].cidr_blocks

# Problem: "Invalid for_each argument — must be a map or set of strings"
# for_each on a resource needs a map or set, not a list
# Convert list to map: { for idx, rule in var.rules : idx => rule }
```

---

## Q17. Terraform State Manipulation Commands

### Question

> Your production Terraform state has become messy — resources were imported incorrectly, some resources exist in state but not in AWS (phantom resources), and you need to move resources between state files. Walk me through `terraform state mv`, `terraform state rm`, `terraform state pull/push`, and when to use each. Include a real disaster recovery scenario.

### Real-Life Scenario

During a team reorganization, the networking team's resources (VPC, subnets) need to be moved from the monolithic `app-infra` state file to a new `networking` state file. Meanwhile, 3 EC2 instances in the state were manually terminated via the Console — they no longer exist in AWS but Terraform still thinks they're there.

### Answer

#### Command Reference

| Command | What It Does | When to Use |
|---------|-------------|-------------|
| `terraform state list` | List all resources in state | Always start here |
| `terraform state show <resource>` | Show details of one resource | Inspect before modifying |
| `terraform state mv` | Move/rename resource in state | Refactoring, splitting states |
| `terraform state rm` | Remove resource from state (NOT from AWS) | Orphaned resources, unmanage |
| `terraform state pull` | Download remote state to stdout | Backup, inspection |
| `terraform state push` | Upload state to remote backend | Restore from backup |
| `terraform state replace-provider` | Change provider in state | Provider migration |

#### Scenario 1: Remove Phantom Resources (exist in state but NOT in AWS)

```bash
# Step 1: Identify the problem
terraform plan
# Output:
# Error: reading EC2 Instance (i-0abc123): couldn't find resource
# Error: reading EC2 Instance (i-0def456): couldn't find resource
# Error: reading EC2 Instance (i-0ghi789): couldn't find resource

# Step 2: Verify they're really gone from AWS
aws ec2 describe-instances --instance-ids i-0abc123 i-0def456 i-0ghi789
# "InvalidInstanceID.NotFound"

# Step 3: Remove from state (does NOT try to delete from AWS)
terraform state rm 'aws_instance.app["web-1"]'
terraform state rm 'aws_instance.app["web-2"]'
terraform state rm 'aws_instance.app["web-3"]'

# Step 4: Verify clean plan
terraform plan
# "No changes. Your infrastructure matches the configuration."

# Step 5: Also remove the resource blocks from your .tf files
# or Terraform will try to CREATE new ones
```

#### Scenario 2: Move Resources Between State Files

```bash
# Moving VPC resources from "app-infra" state to new "networking" state

# Step 1: List what needs to move
cd environments/prod/app-infra
terraform state list | grep -E "vpc|subnet|nat|igw|route_table"
# Output:
# aws_vpc.main
# aws_subnet.private["ap-south-1a"]
# aws_subnet.private["ap-south-1b"]
# aws_subnet.public["ap-south-1a"]
# aws_nat_gateway.main
# aws_internet_gateway.main
# aws_route_table.private
# aws_route_table.public

# Step 2: Pull current state as backup
terraform state pull > backup-app-infra-$(date +%Y%m%d).tfstate

# Step 3: Move resources to the new state
# First, initialize the new networking config
cd environments/prod/networking
terraform init

# Step 4: Use terraform state mv with -state-out (move between states)
cd environments/prod/app-infra

terraform state mv \
  -state-out=../networking/terraform.tfstate \
  aws_vpc.main aws_vpc.main

terraform state mv \
  -state-out=../networking/terraform.tfstate \
  'aws_subnet.private["ap-south-1a"]' 'aws_subnet.private["ap-south-1a"]'

terraform state mv \
  -state-out=../networking/terraform.tfstate \
  'aws_subnet.private["ap-south-1b"]' 'aws_subnet.private["ap-south-1b"]'

terraform state mv \
  -state-out=../networking/terraform.tfstate \
  aws_nat_gateway.main aws_nat_gateway.main

# Step 5: Push the new state to remote backend
cd environments/prod/networking
terraform state push terraform.tfstate

# Step 6: Verify BOTH states
cd environments/prod/app-infra
terraform plan    # Should not try to recreate VPC resources

cd environments/prod/networking
terraform plan    # Should show "No changes"
```

#### Scenario 3: Rename a Resource in State

```bash
# Rename without moved blocks (older Terraform)
terraform state mv aws_instance.server1 aws_instance.web_server

# Rename a module
terraform state mv module.old_name module.new_name

# Rename a resource inside a module
terraform state mv \
  module.app.aws_instance.main \
  module.app.aws_instance.primary
```

#### Scenario 4: Disaster Recovery — Restore Corrupted State

```bash
# Step 1: Pull the last known good state from S3 versioning
aws s3api list-object-versions \
  --bucket mycompany-terraform-state \
  --prefix prod/app/terraform.tfstate \
  --query "Versions[0:5].[VersionId,LastModified,Size]" --output table

# Step 2: Download a specific version
aws s3api get-object \
  --bucket mycompany-terraform-state \
  --key prod/app/terraform.tfstate \
  --version-id "v3rsion1d_good" \
  good-state.tfstate

# Step 3: Verify it's valid JSON
python -m json.tool good-state.tfstate > /dev/null && echo "Valid" || echo "Corrupted"

# Step 4: Force-unlock if locked
terraform force-unlock LOCK-ID-HERE

# Step 5: Push the good state
terraform state push good-state.tfstate

# Step 6: Run plan to check alignment
terraform plan
```

#### Scenario 5: Unmanage a Resource (stop managing without deleting)

```bash
# You want Terraform to "forget" about an RDS instance
# (maybe another team will manage it manually or with a different tool)

# Step 1: Remove from state
terraform state rm aws_db_instance.legacy_db
# "Successfully removed 1 resource instance(s)."

# Step 2: Remove from .tf code
# Delete the resource block from your .tf files

# Step 3: The RDS instance is still running in AWS — untouched
# Terraform just doesn't know about it anymore

# WARNING: If you DON'T remove the resource block from .tf,
# Terraform will try to CREATE a new RDS instance with the same config
```

---

## Q18. Terraform Conditional Logic and Expressions

### Question

> You need to create resources conditionally — for example, NAT Gateways only in prod (not dev), CloudWatch alarms only when monitoring is enabled, and different instance types per environment. How do you implement conditional resource creation, conditional attributes, and complex conditional logic in Terraform?

### Real-Life Scenario

Your dev environment costs $5,000/month because it has the same NAT Gateways, Multi-AZ RDS, and CloudFront distribution as production. The finance team demands cost reduction. You need to conditionally skip expensive resources in dev while keeping the same codebase.

### Answer

#### Pattern 1: Conditional Resource Creation with `count`

```hcl
variable "environment" {
  type = string   # "dev", "staging", "prod"
}

# NAT Gateway — only in staging and prod (costs ~$32/month + data transfer)
resource "aws_nat_gateway" "main" {
  count         = var.environment != "dev" ? 1 : 0
  allocation_id = aws_eip.nat[0].id
  subnet_id     = aws_subnet.public[0].id
}

resource "aws_eip" "nat" {
  count  = var.environment != "dev" ? 1 : 0
  domain = "vpc"
}

# Reference conditionally created resources:
resource "aws_route" "private_nat" {
  count                  = var.environment != "dev" ? 1 : 0
  route_table_id         = aws_route_table.private.id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id         = aws_nat_gateway.main[0].id   # [0] because count was used
}
```

#### Pattern 2: Conditional Resource with `for_each` and Empty Map

```hcl
# CloudWatch Alarms — only when monitoring is enabled
variable "enable_monitoring" {
  type    = bool
  default = true
}

variable "alarms" {
  type = map(object({
    metric_name = string
    threshold   = number
  }))
  default = {
    "high-cpu" = { metric_name = "CPUUtilization", threshold = 80 }
    "high-mem" = { metric_name = "MemoryUtilization", threshold = 85 }
  }
}

resource "aws_cloudwatch_metric_alarm" "this" {
  for_each = var.enable_monitoring ? var.alarms : {}   # Empty map = no resources

  alarm_name  = "${var.environment}-${each.key}"
  metric_name = each.value.metric_name
  threshold   = each.value.threshold
  # ...
}
```

#### Pattern 3: Conditional Attributes with Ternary

```hcl
resource "aws_db_instance" "main" {
  identifier     = "${var.environment}-db"
  engine         = "postgres"
  engine_version = "16.3"

  # Prod: m5.xlarge, Dev: t3.micro
  instance_class = var.environment == "prod" ? "db.r6g.xlarge" : "db.t3.micro"

  # Prod: Multi-AZ, Dev: Single-AZ
  multi_az = var.environment == "prod" ? true : false

  # Prod: 1000 GB, Dev: 20 GB
  allocated_storage = var.environment == "prod" ? 1000 : 20

  # Prod: gp3 with 3000 IOPS, Dev: gp2
  storage_type = var.environment == "prod" ? "gp3" : "gp2"
  iops         = var.environment == "prod" ? 3000 : null   # null = use default

  # Deletion protection only in prod
  deletion_protection = var.environment == "prod"

  # Backup retention: prod = 35 days, dev = 1 day
  backup_retention_period = var.environment == "prod" ? 35 : 1
}
```

#### Pattern 4: Locals for Complex Conditional Logic

```hcl
locals {
  # Environment-specific configuration map
  env_config = {
    dev = {
      instance_type    = "t3.medium"
      min_size         = 1
      max_size         = 2
      enable_nat       = false
      enable_waf       = false
      enable_cloudfront = false
      rds_multi_az     = false
    }
    staging = {
      instance_type    = "m5.large"
      min_size         = 2
      max_size         = 4
      enable_nat       = true
      enable_waf       = true
      enable_cloudfront = false
      rds_multi_az     = false
    }
    prod = {
      instance_type    = "m5.xlarge"
      min_size         = 3
      max_size         = 10
      enable_nat       = true
      enable_waf       = true
      enable_cloudfront = true
      rds_multi_az     = true
    }
  }

  # Select current environment config
  config = local.env_config[var.environment]
}

# Usage — clean and readable
resource "aws_autoscaling_group" "app" {
  min_size         = local.config.min_size
  max_size         = local.config.max_size
  # ...
}

resource "aws_nat_gateway" "main" {
  count = local.config.enable_nat ? 1 : 0
  # ...
}
```

#### Pattern 5: Conditional Dynamic Blocks

```hcl
resource "aws_launch_template" "app" {
  name          = "${var.environment}-app"
  image_id      = data.aws_ami.latest.id
  instance_type = local.config.instance_type

  # Add detailed monitoring block only in prod
  dynamic "monitoring" {
    for_each = var.environment == "prod" ? [1] : []
    content {
      enabled = true
    }
  }

  # Add placement group only in prod
  dynamic "placement" {
    for_each = var.environment == "prod" ? [1] : []
    content {
      group_name = aws_placement_group.app[0].name
    }
  }
}
```

#### Cost Comparison After Optimization

| Resource | Dev (Before) | Dev (After) | Monthly Savings |
|----------|-------------|-------------|-----------------|
| NAT Gateway | $32 + data | $0 (removed) | $32+ |
| RDS Multi-AZ | $400 | $15 (t3.micro, single-AZ) | $385 |
| CloudFront | $50 | $0 (removed) | $50 |
| EC2 (m5.xlarge×3) | $410 | $30 (t3.medium×1) | $380 |
| WAF | $10 | $0 (removed) | $10 |
| **Total** | **~$900** | **~$45** | **~$855/month** |

---

## Q19. Terraform `depends_on` — Explicit vs Implicit Dependencies

### Question

> Terraform usually handles dependencies automatically. But sometimes `terraform apply` fails because resources are created in the wrong order. When do you need explicit `depends_on`, and when does it cause more problems than it solves? Show a real scenario where missing `depends_on` causes a failure and one where using it incorrectly causes slowness.

### Real-Life Scenario

You're creating an EKS cluster, and `terraform apply` fails because the IAM role policy attachment hasn't propagated yet when EKS tries to use the role. The error is "InvalidParameterException: The role does not have sufficient permissions." Adding `depends_on` fixes it — but then you start adding `depends_on` everywhere "just to be safe," and your apply time doubles because resources that could be parallel are now serial.

### Answer

#### Implicit Dependencies (Terraform Handles Automatically)

```hcl
# Terraform KNOWS the subnet needs the VPC first
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "private" {
  vpc_id     = aws_vpc.main.id      # ← Terraform sees this reference
  cidr_block = "10.0.1.0/24"        #    and builds the dependency graph
}

# Order: VPC → Subnet (automatic, no depends_on needed)
```

```bash
# Visualize the dependency graph
terraform graph | dot -Tpng > graph.png
# Or use:
terraform graph -type=plan
```

#### When You NEED `depends_on`

**Case 1: IAM Eventually Consistent (Most Common)**

```hcl
resource "aws_iam_role" "eks" {
  name = "eks-cluster-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "eks.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "eks_cluster" {
  role       = aws_iam_role.eks.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
}

resource "aws_eks_cluster" "main" {
  name     = "production"
  role_arn = aws_iam_role.eks.arn

  # Without depends_on, Terraform creates the cluster immediately after the role
  # But the policy attachment might not have propagated yet!
  # Error: "The role does not have sufficient permissions"

  depends_on = [
    aws_iam_role_policy_attachment.eks_cluster    # Wait for policy to attach
  ]

  vpc_config {
    subnet_ids = var.subnet_ids
  }
}
```

**Case 2: S3 Bucket Policy Before CloudFront Access**

```hcl
resource "aws_s3_bucket_policy" "cdn" {
  bucket = aws_s3_bucket.static.id
  policy = data.aws_iam_policy_document.cdn.json
}

resource "aws_cloudfront_distribution" "main" {
  # CloudFront tries to access S3, but policy might not be ready
  depends_on = [aws_s3_bucket_policy.cdn]

  origin {
    domain_name = aws_s3_bucket.static.bucket_regional_domain_name
    origin_id   = "s3-static"
  }
  # ...
}
```

**Case 3: VPC Endpoints Before Lambda in VPC**

```hcl
resource "aws_vpc_endpoint" "s3" {
  vpc_id       = aws_vpc.main.id
  service_name = "com.amazonaws.ap-south-1.s3"
}

resource "aws_lambda_function" "app" {
  function_name = "my-app"
  role          = aws_iam_role.lambda.arn
  handler       = "index.handler"
  runtime       = "nodejs18.x"

  vpc_config {
    subnet_ids         = aws_subnet.private[*].id
    security_group_ids = [aws_security_group.lambda.id]
  }

  # Lambda needs the VPC endpoint to access S3
  depends_on = [aws_vpc_endpoint.s3]
}
```

#### When `depends_on` is WRONG (Causes Problems)

```hcl
# ❌ WRONG — Unnecessary depends_on slows everything down
resource "aws_instance" "web" {
  ami           = "ami-abc123"
  instance_type = "m5.large"
  subnet_id     = aws_subnet.private[0].id

  depends_on = [
    aws_vpc.main,               # Already implicit via subnet
    aws_subnet.private,         # Already implicit via subnet_id reference
    aws_security_group.web,     # Add this to vpc_security_group_ids instead
    aws_iam_role.app,           # Unrelated — forces serial execution
    aws_s3_bucket.logs,         # Unrelated — forces serial execution
  ]
}

# ✅ CORRECT — Let Terraform figure out dependencies
resource "aws_instance" "web" {
  ami                    = "ami-abc123"
  instance_type          = "m5.large"
  subnet_id              = aws_subnet.private[0].id
  vpc_security_group_ids = [aws_security_group.web.id]   # Implicit dependency
  iam_instance_profile   = aws_iam_instance_profile.app.name  # Implicit dependency
}
```

#### Module-Level `depends_on`

```hcl
# Sometimes you need a whole module to wait for another
module "eks" {
  source = "./modules/eks"
  # ...

  depends_on = [
    module.vpc,                    # All VPC resources must exist first
    aws_iam_role_policy_attachment.eks_policies   # IAM must propagate
  ]
}
```

#### Debugging Dependency Issues

```bash
# See what Terraform thinks the dependencies are
terraform graph -type=plan | grep -A2 "aws_eks_cluster"

# Verbose logging during apply
TF_LOG=DEBUG terraform apply 2>&1 | grep -i "depend"

# If a resource fails due to missing dependency:
# 1. Check if the error is about permissions → IAM propagation → add depends_on
# 2. Check if the error is "not found" → resource reference missing → use attribute reference
# 3. Check if the error is timing → use depends_on as last resort
```

---

## Q20. Terraform Providers — Custom Providers, Version Pinning, and Mirror Registries

### Question

> Your organization has strict security requirements: no direct internet access from CI/CD runners, all Terraform providers must be pre-approved and scanned, and you need to use a custom provider for an internal tool. How do you set up a provider mirror/cache, pin provider versions, and create a custom provider? What happens when a provider version has a breaking change?

### Real-Life Scenario

Your CI/CD pipeline suddenly fails one Monday because HashiCorp released AWS provider v6.0.0 with breaking changes, and your config didn't pin the version. Meanwhile, your air-gapped production environment can't download providers from the internet at all.

### Answer

#### Step 1: Version Pinning (ALWAYS Do This)

```hcl
# versions.tf
terraform {
  required_version = ">= 1.5.0, < 2.0.0"   # Pin Terraform itself

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.40"    # Allows 5.40.x, 5.41.x but NOT 6.0.0
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.27"
    }
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.12"
    }
  }
}
```

**Version constraint syntax:**

```
"= 5.40.0"     Exact version only
"~> 5.40"       >= 5.40.0, < 6.0.0 (recommended for major protection)
"~> 5.40.0"     >= 5.40.0, < 5.41.0 (very strict)
">= 5.40"       5.40 or higher (dangerous — allows 6.0!)
">= 5.40, < 6"  Same as ~> 5.40
```

#### Step 2: Lock File (`.terraform.lock.hcl`)

```bash
# Terraform auto-generates this file
terraform init

# The lock file pins exact versions AND checksums:
cat .terraform.lock.hcl
```

```hcl
# .terraform.lock.hcl (COMMIT THIS TO GIT)
provider "registry.terraform.io/hashicorp/aws" {
  version     = "5.40.0"
  constraints = "~> 5.40"
  hashes = [
    "h1:abc123...",     # Hash of the provider binary
    "zh:def456...",
  ]
}
```

```bash
# Update lock file when upgrading providers
terraform init -upgrade

# Verify lock file hasn't been tampered with
terraform providers lock \
  -platform=linux_amd64 \
  -platform=darwin_amd64 \
  -platform=windows_amd64
```

#### Step 3: Provider Mirror for Air-Gapped Environments

```bash
# Option A: Filesystem Mirror
# Download providers on a machine WITH internet
terraform providers mirror /opt/terraform/providers

# Directory structure created:
/opt/terraform/providers/
└── registry.terraform.io/
    └── hashicorp/
        ├── aws/
        │   └── 5.40.0/
        │       ├── terraform-provider-aws_5.40.0_linux_amd64.zip
        │       └── terraform-provider-aws_5.40.0_darwin_amd64.zip
        └── kubernetes/
            └── 2.27.0/
                └── ...

# Copy this directory to air-gapped servers via USB/S3/internal network
```

```hcl
# CLI config on air-gapped servers: ~/.terraformrc
provider_installation {
  filesystem_mirror {
    path    = "/opt/terraform/providers"
    include = ["registry.terraform.io/*/*"]
  }

  # Fallback to direct (only if allowed)
  # direct {
  #   exclude = []
  # }
}
```

```bash
# Option B: Network Mirror (internal HTTP server)
# Host a mirror on your internal Artifactory/Nexus

# ~/.terraformrc
provider_installation {
  network_mirror {
    url = "https://artifactory.mycompany.com/terraform-providers/"
  }
}
```

#### Step 4: Artifactory/Nexus as Provider Registry

```bash
# JFrog Artifactory setup:
# 1. Create a "terraform" remote repository pointing to registry.terraform.io
# 2. Create a virtual repository combining remote + local
# 3. Configure Terraform to use it:

# ~/.terraformrc
provider_installation {
  network_mirror {
    url = "https://artifactory.mycompany.com/api/terraform/providers/v1/"
  }
}

# Approval workflow:
# 1. Security team scans new provider version
# 2. If approved, promote to "approved" repo
# 3. CI/CD only uses "approved" repo
```

#### Step 5: Custom Provider (for internal tools)

```go
// internal/provider/provider.go (Go code — Terraform SDK v2)
package provider

import (
    "github.com/hashicorp/terraform-plugin-sdk/v2/helper/schema"
)

func Provider() *schema.Provider {
    return &schema.Provider{
        Schema: map[string]*schema.Schema{
            "api_endpoint": {
                Type:        schema.TypeString,
                Required:    true,
                DefaultFunc: schema.EnvDefaultFunc("MYCOMPANY_API_URL", nil),
            },
        },
        ResourcesMap: map[string]*schema.Resource{
            "mycompany_database":    resourceDatabase(),
            "mycompany_application": resourceApplication(),
        },
    }
}
```

```bash
# Build and install custom provider
go build -o terraform-provider-mycompany
mkdir -p ~/.terraform.d/plugins/mycompany.com/internal/mycompany/1.0.0/linux_amd64/
cp terraform-provider-mycompany ~/.terraform.d/plugins/mycompany.com/internal/mycompany/1.0.0/linux_amd64/
```

```hcl
# Use in Terraform
terraform {
  required_providers {
    mycompany = {
      source  = "mycompany.com/internal/mycompany"
      version = "1.0.0"
    }
  }
}

provider "mycompany" {
  api_endpoint = "https://internal-api.mycompany.com"
}

resource "mycompany_database" "app" {
  name   = "production-db"
  engine = "postgres"
}
```

#### Handling Provider Breaking Changes

```bash
# Monday morning: CI/CD fails after provider update

# Step 1: Check what changed
terraform version
# Terraform v1.7.0
# + provider registry.terraform.io/hashicorp/aws v6.0.0   ← Breaking!

# Step 2: Roll back by pinning version in versions.tf
# version = "~> 5.40"    ← This would have prevented the issue

# Step 3: Delete lock file and reinit
rm .terraform.lock.hcl
rm -rf .terraform/
terraform init

# Step 4: Read the upgrade guide
# https://registry.terraform.io/providers/hashicorp/aws/latest/docs/guides/version-6-upgrade

# Step 5: Make changes required by v6.0, test in dev first
# Common breaking changes:
# - Removed deprecated attributes
# - Changed attribute types
# - Renamed resources
# - Changed default values
```

#### Troubleshooting

```bash
# Problem: "Could not retrieve the list of available versions"
# CI/CD can't reach registry.terraform.io
# Solution: Set up a network mirror or filesystem mirror

# Problem: "Inconsistent dependency lock file"
# Lock file doesn't match required version constraints
terraform init -upgrade

# Problem: "Failed to query available provider packages"
# Wrong source address in required_providers
terraform providers
# Shows which providers are required and from where

# Problem: "Provider version not available on your platform"
# You're on ARM Mac but provider only has amd64 builds
terraform providers lock -platform=darwin_arm64
```

---

*More topics coming: Docker, Kubernetes, EKS, AWS, Prometheus, Grafana*
