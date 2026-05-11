# Nagarro — AWS Cloud Architect Interview Preparation

> **Profile:** AWS Cloud Architect (10+ YoE) | **Topics:** Terraform, Docker, Kubernetes, EKS, AWS, Prometheus, Grafana
>
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

*More topics coming: Docker, Kubernetes, EKS, AWS, Prometheus, Grafana*
