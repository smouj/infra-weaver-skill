name: "infrastructure-weaver"
description: "Orchestrates and automates cloud resources into cohesive, production-ready architectures using IaC tools and declarative workflows"
version: "2.1.0"
author: "OpenClaw Engineering Team"
tags:
  - "terraform"
  - "ansible"
  - "cloudformation"
  - "kubernetes"
  - "iac"
  - "orchestration"
  - "aws"
  - "gcp"
  - "azure"
  - "multi-cloud"
requires:
  - "terraform>=1.5.0"
  - "kubectl>=1.28.0"
  - "aws-cli>=2.15.0"
  - "ansible>=2.15.0"
  - "jq>=1.6"
  - "yq>=4.35.0"
  - "git>=2.40"
  - "python3>=3.11"
  - "gcloud-sdk>=456.0.0"  # optional for GCP
  - "azcli>=2.55.0"  # optional for Azure
conflicts:
  - "vps-ops"
  - "quick-deploy"
environment:
  - "AWS_PROFILE"
  - "AWS_REGION"
  - "TF_VAR_environment"
  - "TF_VAR_region"
  - "ANSIBLE_HOST_KEY_CHECKING"
  - "KUBECONFIG"
workdir: "./infrastructure"
logs: "./infrastructure-logs/$(date +%Y-%m-%d_%H-%M-%S).log"
outputs:
  - "terraform.tfstate"
  - "inventory.yml"
  - "k8s-manifests/"
  - "ansible-playbooks/"
```

# Infrastructure Weaver

## Purpose

Infrastructure Weaver automates the creation, management, and teardown of complex cloud architectures using Infrastructure-as-Code (IaC) patterns. It orchestrates multiple tools (Terraform, Ansible, Kubernetes, CloudFormation) into unified workflows.

**Real use cases:**
- Deploy production-grade 3-tier applications on AWS with VPC, public/private subnets, NAT gateways, RDS PostgreSQL, EC2 Auto Scaling Group, Application Load Balancer, and CloudFront CDN
- Provision and configure Kubernetes clusters (EKS/GKE/AKS) with node pools, ingress controllers (nginx/alb), service mesh (istio), and namespace isolation
- Implement multi-environment isolation (dev/staging/prod) with Terraform workspaces and Ansible dynamic inventories
- Execute blue-green deployments with zero-downtime using Kubernetes Deployments and AWS CodeDeploy
- Set up disaster recovery pipelines: cross-region replication, RDS automated backups, S3 cross-region sync, and Route53 failover policies
- Build compliant infrastructure: PCI-DSS, HIPAA, GDPR with security modules (security groups, IAM policies, KMS encryption, CloudTrail)
- Migrate legacy on-prem servers to cloud using Ansible migration playbooks and generated Terraform import mappings
- Create ephemeral testing environments that spin up full stacks on PR creation and auto-destroy after 24h

## Scope

The skill provides these **concrete commands**:

- `plan`: Generates Terraform execution plan and Ansible dry-run. Flags: `--target MODULE`, `--var 'key=value'`, `--var-file FILE`, `--compact-warnings`, `--json`, `--color=always`
- `apply`: Applies infrastructure changes. Flags: `--auto-approve`, `--target MODULE`, `--var 'key=value'`, `--var-file FILE`, `--compact-warnings`, `--skip-refresh`, `--parallelism=N`
- `destroy`: Tears down all managed resources. Flags: `--auto-approve`, `--target MODULE`, `--force`, `--timeout=DURATION`
- `validate`: Runs static analysis on Terraform configs, Ansible playbooks, and Kubernetes manifests. Flags: `--strict`, `--json`, `--skip-child-modules`
- `inventory`: Generates dynamic Ansible inventory from Terraform outputs or cloud provider APIs. Flags: `--format json|yaml|ini`, `--refresh`, `--group-by TAG`
- `k8s-deploy`: Deploys Kubernetes manifests. Flags: `--namespace NS`, `--context CTX`, `--server-side-apply`, `--wait`, `--timeout=DURATION`, `--prune`
- `output`: Queries Terraform state outputs. Flags: `--json`, `--raw`, `--state=PATH`, `--module MODULE`
- `import`: Imports existing resources into Terraform state. Flags: `--provider=PROVIDER`, `--var-file FILE`, `--dry-run`
- `test`: Executes infrastructure integration tests using terratest or inspec. Flags: `--suite SUITE`, `--parallel`, `--fail-fast`, `--test TIMEOUT`
- `module-publish`: Packages and publishes reusable Terraform modules to private registry. Flags: `--source PATH`, `--version SEMVER`, `--registry URL`, `--namespace NS`
- `workspace-switch`: Switches Terraform workspaces and updates kubeconfig. Flags: `--workspace NAME`, `--context`, `--set-current`
- `cost-estimate`: Generates cost estimates using Infracost or terraform-cost-estimation. Flags: `--compare-with PLAN_FILE`, `--output format`, `--threshold VALUE`
- `security-scan`: Runs tfsec, checkov, or trivy scans. Flags: `--severity LEVEL`, `--exit-code`, `--format json|sarif`, `--quiet`

## Detailed Work Process

Infrastructure Weaver follows a strict, reproducible pipeline:

**1. CONFIGURATION LOADING**
```bash
# Load environment-specific variables
export AWS_PROFILE="production"
export TF_VAR_region="us-east-1"
export TF_VAR_environment="prod"
```

The skill reads configuration in this order:
- `infrastructure/backend.tf` for remote state backend (S3+DynamoDB)
- `infrastructure/variables.tf` for declared variables
- `infrastructure/terraform.tfvars` for variable values
- `infrastructure/environments/${TF_VAR_environment}.tfvars` for environment overrides
- Environment variables prefixed with `TF_VAR_`

**2. TERRAFORM INITIALIZATION**
```bash
cd /tmp/flickclaw-work/infrastructure
terraform init \
  -backend-config="bucket=mycompany-terraform-state" \
  -backend-config="dynamodb_table=terraform-lock" \
  -backend-config="region=us-east-1" \
  -reconfigure \
  -input=false
```
Checks for required providers (aws, random, kubernetes, helm, archive). Downloads plugins to `.terraform` directory. Validates backend connectivity.

**3. PLAN GENERATION**
```bash
terraform plan \
  -var="environment=prod" \
  -var="region=us-east-1" \
  -var="instance_count=3" \
  -var="db_instance_class=db.t3.large" \
  -var="vpc_cidr=10.0.0.0/16" \
  -var="availability_zones=['us-east-1a','us-east-1b','us-east-1c']" \
  -var="ssh_key_name=prod-ec2-key" \
  -out=tfplan.binary \
  -compact-warnings \
  -input=false \
  -no-color
```
Captures plan as binary file `tfplan.binary` for guaranteed identical apply.

**4. ANsible INVENTORY GENERATION**
```bash
# Generate inventory from Terraform outputs
terraform output -json | jq -r '
  .ecs_cluster_endpoint.value // empty,
  ".ec2_public_ips[] as $ip | $ip" as $ip |
  "[webservers]\n$ip"'
> infrastructure/inventory/hosts.ini
```

Or using the skill's inventory command:
```bash
infrastructure-weaver inventory \
  --format ini \
  --refresh \
  --group-by "tag:Environment, tag:Role" \
  > infrastructure/inventory/hosts.ini
```

**5. CONFIGURATION MANAGEMENT**
```bash
ansible-playbook \
  -i infrastructure/inventory/hosts.ini \
  -e "env=prod region=us-east-1" \
  -e "@infrastructure/vars/${TF_VAR_environment}.yml" \
  --private-key infrastructure/keys/${TF_VAR_environment}.pem \
  --ssh-common-args="-o StrictHostKeyChecking=no" \
  --timeout=300 \
  infrastructure/ansible/playbooks/configure-webservers.yml
```

**6. KUBERNETES DEPLOYMENT**
```bash
# Update kubeconfig from EKS cluster
aws eks update-kubeconfig \
  --name $(terraform output -raw eks_cluster_name) \
  --region ${TF_VAR_region} \
  --alias ${TF_VAR_environment}

infrastructure-weaver k8s-deploy \
  --namespace ${TF_VAR_environment} \
  --namespace ${TF_VAR_environment} \
  --context ${TF_VAR_environment} \
  --wait \
  --timeout=10m \
  --prune \
  -f infrastructure/k8s/namespaces/ \
  -f infrastructure/k8s/configmaps/ \
  -f infrastructure/k8s/deployments/ \
  -f infrastructure/k8s/services/ \
  -f infrastructure/k8s/ingresses/
```

**7. POST-DEPLOYMENT VALIDATION**
```bash
# Check pod status
kubectl get pods -n prod -o wide --sort-by='.status.startTime'

# Check service endpoints
kubectl get endpoints -n prod

# Check Alb ingress
kubectl get ingress -n prod -o jsonpath='{.items[*].status.loadBalancer.ingress[0].hostname}'

# Health check
curl -sSf https://$(terraform output -raw alb_dns_name)/health || exit 1
```

**8. COST AND SECURITY SCANNING**
```bash
infrastructure-weaver security-scan \
  --severity high \
  --format sarif \
  --output infrastructure/security-scan-$(date +%Y%m%d).sarif \
  infrastructure/

infrastructure-weaver cost-estimate \
  --compare-with tfplan.binary \
  --output html \
  --threshold 1000 \
  > infrastructure/cost-report.html
```

**9. STATE MANAGEMENT**
- Remote state stored in S3 with DynamoDB locking
- State backups automatically created on every apply: `terraform-state-backup-YYYY-MM-DD-HHMMSS.tfstate`
- State rotated every 30 days, archived to Glacier after 90 days

## Golden Rules

1. **Remote state only**: Never use local state. All state must be stored in S3 with DynamoDB locking and encryption-at-rest. Fail if `backend` block missing.
2. **Immutable infrastructure**: Never modify resources directly in cloud console. All changes must go through Terraform apply. If manual change detected (`terraform plan` shows diff with existing), raise alert.
3. **Workspace isolation**: Each environment (dev/staging/prod) uses separate Terraform workspace and separate S3 state key. Do not mix resources across workspaces.
4. **Secrets management**: Never hardcode secrets. Use SSM Parameter Store, Secrets Manager, or Vault. Ansible Vault for playbook secrets. Fail scan if plaintext passwords found.
5. **Module version pinning**: All module sources must have explicit version constraints (e.g., `source = "terraform-aws-modules/vpc/aws" version = "5.1.2"`). No `>=` constraints, only exact versions.
6. **Tagging policy**: All resources must have mandatory tags: `Environment`, `Owner`, `Project`, `ComplianceLevel`. Assert failure on missing tags in plan.
7. **Backward compatibility**: Terraform configurations must support at least 2 minor version upgrades. Test `terraform 0.13` to `1.5` upgrade path.
8. **Zero-downtime deployments**: Use Kubernetes rolling updates with `maxSurge=25%` and `maxUnavailable=0`. For EC2 ASG, use lifecycle hooks and ASG warm pools.
9. **Backup before destroy**: On `destroy`, automatically create final state backup and export critical data (RDS snapshots, EBS volumes) before resource deletion.
10. **DR requirement**: All production stacks must have cross-region replication configured. Validate via `infrastructure-weaver dr-validate --standby-region` before marking environment production-ready.

## Examples

### Example 1: Deploy a 3-tier web application on AWS

**User prompt:** "Deploy a production web app on AWS with auto-scaling, RDS, and CloudFront. Use us-east-1, prod environment, 3 AZs."

**Skill execution:**

```bash
# 1. Setup environment
export TF_VAR_environment="prod"
export TF_VAR_region="us-east-1"
export AWS_PROFILE="company-prod"

# 2. Initialize with remote backend
infrastructure-weaver init \
  --backend-config="bucket=company-terraform-state-prod" \
  --backend-config="dynamodb_table=terraform-lock-prod" \
  --backend-config="region=us-east-1" \
  --backend-config="encrypt=true" \
  --reconfigure

# 3. Generate plan
infrastructure-weaver plan \
  --var="instance_count=4" \
  --var="db_instance_class=db.t3.xlarge" \
  --var="vpc_cidr=10.10.0.0/16" \
  --var="availability_zones=['us-east-1a','us-east-1b','us-east-1c']" \
  --var="ssh_key_name=prod-company-key" \
  --var="domain_name=app.company.com" \
  --var="ssl_arn=arn:aws:acm:us-east-1:123456789:certificate/xxxx-xxxx" \
  --var="enable_cloudfront=true" \
  --var="enable_monitoring=true" \
  --var="enable_waf=true" \
  --compact-warnings \
  --json > plan.json

# 4. Review plan output (plan.json contains resource counts, changes, costs)
cat plan.json | jq '.resource_changes[] | {address, change}'

# 5. Apply
infrastructure-weaver apply \
  --auto-approve \
  --compact-warnings \
  --parallelism=10 \
  tfplan.binary

# 6. Generate Ansible inventory
infrastructure-weaver inventory \
  --format ini \
  --refresh \
  --group-by "tag:Role" \
  > inventory/hosts.ini

# 7. Configure EC2 instances
ansible-playbook \
  -i inventory/hosts.ini \
  -e "env=prod" \
  --private-key infrastructure/keys/prod-company-key.pem \
  infrastructure/ansible/playbooks/webapp.yml

# 8. Deploy k8s if using EKS
infrastructure-weaver k8s-deploy \
  --namespace prod \
  --wait \
  --timeout=15m \
  -f infrastructure/k8s/

# 9. Health validation
URL=$(terraform output -raw alb_dns_name)
curl -sSf https://$URL/health || { echo "Health check failed"; exit 1; }

# 10. Cost and security scan
infrastructure-weaver security-scan --severity high --format json
infrastructure-weaver cost-estimate --threshold 5000

# Output:
# Apply complete! Resources: 42 added, 0 changed, 0 destroyed.
# Outputs:
# vpc_id = "vpc-0a1b2c3d4e5f6g7h8"
# alb_dns_name = "prod-app-123456.us-east-1.elb.amazonaws.com"
# cloudfront_domain = "d1234abcd.cloudfront.net"
# rds_endpoint = "prod-db.xxxxxxx.us-east-1.rds.amazonaws.com"
# eks_cluster_endpoint = "https://xxxx.g八字.us-east-1.eks.amazonaws.com"
```

### Example 2: Provision an EKS cluster with ingress and service mesh

**User prompt:** "Create an EKS cluster with 3 managed node groups (general purpose, memory optimized, GPU), install nginx ingress, istio, and argocd."

**Skill execution:**

```bash
# Configuration file: infrastructure/environments/eks-prod.tfvars
cluster_name = "company-eks-prod"
cluster_version = "1.28"
vpc_cidr = "10.20.0.0/16"
availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]
node_groups = {
  "general" = {
    instance_type = "m5.large"
    min_size      = 2
    max_size      = 6
    desired_size  = 3
  }
  "memory" = {
    instance_type = "r5.xlarge"
    min_size      = 2
    max_size      = 4
    desired_size  = 2
  }
  "gpu" = {
    instance_type = "g4dn.xlarge"
    min_size      = 0
    max_size      = 2
    desired_size  = 0
    gpu           = true
  }
}
enable_istio = true
enable_argocd = true
ingress_controller = "nginx"

# Build and apply
infrastructure-weaver apply \
  --var-file infrastructure/environments/eks-prod.tfvars \
  --auto-approve

# Post-apply: Generate kubeconfig
infrastructure-weaver k8s-deploy \
  --context company-eks-prod \
  --wait \
  --timeout=30m \
  -f infrastructure/k8s/istio/ \
  -f infrastructure/k8s/ingress-nginx/ \
  -f infrastructure/k8s/argocd/

# Verify
kubectl --context company-eks-prod get nodes -o wide
kubectl --context company-eks-prod get pods -n istio-system
kubectl --context company-eks-prod get svc -n ingress-nginx

# Output:
# EKS cluster created: company-eks-prod
# Node groups: 3 (general: 3 nodes, memory: 2 nodes, gpu: 0 nodes)
# Control plane endpoint: https://XXXX.g八字.us-east-1.eks.amazonaws.com
# Ingress controller: ingress-nginx (EXTERNAL-IP pending ~5min)
# Istio installed: control-plane revision default
# ArgoCD available at: http://company-argocd.xxx.elb.amazonaws.com
```

### Example 3: Multi-cloud disaster recovery setup

**User prompt:** "Configure DR: primary in us-east-1, standby in us-west-2. RDS cross-region read replica, S3 replication, Route53 failover."

**Skill execution:**

```bash
# Primary region (us-east-1)
infrastructure-weaver apply \
  --var="region=us-east-1" \
  --var="environment=prod-primary" \
  --var="enable_replication=true" \
  --var="replication_target_region=us-west-2"

# Standby region (us-west-2) - import from primary
infrastructure-weaver import \
  --provider=aws \
  --var="region=us-west-2" \
  --var="environment=prod-standby" \
  --var="replication_source_region=us-east-1"

# Configure Route53 failover
infrastructure-weaver k8s-deploy \
  --context prod-primary \
  -f infrastructure/k8s/route53-failover/

# Test DR failover (simulated)
infrastructure-weaver dr-test \
  --primary-region=us-east-1 \
  --standby-region=us-west-2 \
  --mode=sync

# Output:
# Primary stack (us-east-1): 38 resources
# Standby stack (us-west-2): 35 resources (async replication lag ~30s)
# Route53 health checks: 4 configured, 2 primary, 2 standby
# RDS cross-region replica: promoting in 60s (test)
# S3 replication: versioning enabled, replication rules: 6
# DR status: GREEN - failover RTO 300s, RPO 1min
```

## Rollback Commands

**Immediate rollback** (if apply fails or issues detected within 5 minutes):
```bash
# 1. Terraform rollback to previous state (if apply partially succeeded)
terraform state pull > terraform-state-backup-before-rollback.json
terraform apply tfplan-prev.binary  # use previous plan file

# OR quick destroy if complete rollback needed
infrastructure-weaver destroy \
  --auto-approve \
  --force \
  --grace-period=30s \
  --target=module.vpc \
  --target=module.eks \
  --target=module.rds

# 2. Remove Kubernetes deployments
infrastructure-weaver k8s-deploy \
  --namespace prod \
  --prune \
  --cascade=foreground \
  --delete-emptydir-data

# 3. Restore data from snapshots (RDS/EBS/S3)
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier prod-db-rollback \
  --db-snapshot-identifier prod-db-snapshot-2024-01-15-1200 \
  --no-multi-az
```

**State recovery** (if state file corrupted or lost):
```bash
# 1. List available state backups in S3
aws s3 ls s3://company-terraform-state-prod/backups/ \
  --recursive \
  --profile company-prod

# 2. Restore latest backup
aws s3 cp s3://company-terraform-state-prod/backups/terraform-state-backup-2024-01-15-1200.tfstate ./terraform.tfstate

# 3. Force unlock if lock stuck
terraform force-unlock $(terraform state pull | jq -r '.lockid')

# 4. Verify state integrity
terraform validate
terraform plan -detailed-exitcode  # should return 0 (no changes)
```

**Configuration rollback** (revert to previous module versions):
```bash
git -C infrastructure/ log --oneline -10
git -C infrastructure/ checkout <previous-commit-sha>

infrastructure-weaver init -reconfigure
infrastructure-weaver plan -detailed-exitcode

# If plan shows only module version changes (no resource changes):
infrastructure-weaver apply -var="module_version=1.2.3" -auto-approve
```

**Ansible configuration rollback:**
```bash
# List available playbook versions (git tags)
git -C infrastructure/ansible/ tag | grep -E 'playbook-v[0-9]+' | sort -V

# Rollback to specific version
git -C infrastructure/ansible/ checkout tags/playbook-v1.4.0 -b rollback-branch

# Re-run configuration
ansible-playbook -i inventory/hosts.ini infrastructure/ansible/playbooks/configure-webservers.yml

# Or use Ansible rollback tags (if playbooks idempotent)
ansible-playbook -i inventory/hosts.ini infrastructure/ansible/playbooks/rollback.yml \
  -e "rollback_to_version=1.4.0"
```

**CloudFormation rollback** (if using CloudFormation stacks):
```bash
# 1. List stack events to identify failure
aws cloudformation describe-stack-events \
  --stack-name prod-app-stack \
  --region us-east-1 \
  --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`]' \
  --output table

# 2. Delete stack (automatic rollback if --disable-rollback not set originally)
aws cloudformation delete-stack \
  --stack-name prod-app-stack \
  --region us-east-1

# 3. Wait for deletion
aws cloudformation wait stack-delete-complete \
  --stack-name prod-app-stack \
  --region us-east-1

# 4. Re-deploy with updated template
infrastructure-weaver apply --template-file=templates/fixed-stack.yaml
```

**Kubernetes rollback:**
```bash
# 1. Check rollout history
kubectl rollout history deployment/webapp -n prod

# 2. Rollback to previous revision
kubectl rollout undo deployment/webapp -n prod

# 3. Or rollback to specific revision
kubectl rollout undo deployment/webapp -n prod --to-revision=3

# 4. Monitor rollout
kubectl rollout status deployment/webapp -n prod --watch

# 5. If pod fails, automatically restore previous configMap/secret revisions
kubectl rollout restart deployment/webapp -n prod
```

## Dependencies & Requirements

**System dependencies** (verify with `infrastructure-weaver check-prereqs`):
```bash
terraform version  # >= 1.5.0
kubectl version --client  # >= 1.28.0
aws --version  # >= 2.15.0
ansible --version  # >= 2.15.0
jq --version  # >= 1.6
yq --version  # >= 4.35.0
gcloud version  # >= 456.0.0 (optional)
az version  # >= 2.55.0 (optional)
python3 --version  # >= 3.11
```

**AWS requirements** (for AWS provider):
- IAM user/role with policies: `AdministratorAccess` (or custom with `*:*` for infra setup)
- S3 bucket for Terraform state (versioning + encryption enabled)
- DynamoDB table for state locking
- SSM Parameter Store or Secrets Manager access

**Kubernetes requirements**:
- kubectl configured with cluster context
- `kubectl apply` permissions (cluster-admin for bootstrapping)
- Helm 3 installed for chart deployments

**Repository structure** (expected layout):
```
infrastructure/
├── backend.tf                    # remote state config
├── main.tf                       # root module
├── variables.tf                  # variable declarations
├── outputs.tf                    # output declarations
├── terraform.tfvars              # variable values (dev)
├── environments/
│   ├── dev.tfvars
│   ├── staging.tfvars
│   └── prod.tfvars
├── modules/
│   ├── vpc/
│   ├── eks/
│   ├── rds/
│   └── asg/
├── ansible/
│   ├── inventory/
│   ├── playbooks/
│   ├── group_vars/
│   └── host_vars/
├── k8s/
│   ├── namespaces/
│   ├── configmaps/
│   ├── deployments/
│   ├── services/
│   ├── ingress/
│   └── helm-charts/
├── scripts/
│   ├── pre-apply.sh
│   ├── post-apply.sh
│   └── healthcheck.sh
├── security/
│   ├── tfsec/
│   ├── checkov/
│   └── policies/
├── test/
│   ├── integration/
│   ├── acceptance/
│   └── fixtures/
└── docs/
    ├── architecture.md
    └── runbooks/
```

## Verification Steps

**After apply**, automatically verify:

1. **Terraform outputs sanity check**:
```bash
infrastructure-weaver output --json | jq '
  . as $o |
  keys[] as $k |
  "\($k): \($o[$k] // "null")"'
```
Assert all critical outputs non-null.

2. **VPC/Network connectivity**:
```bash
# Ping between AZs
for az in $(terraform output -json availability_zones | jq -r '.[]'); do
  aws ec2 describe-subnets --filters Name=availability-zone,Values=$az --query 'Subnets[?MapPublicIpOnLaunch==`true`].SubnetId' --output text
done

# Validate route tables
aws ec2 describe-route-tables --filters Name=vpc-id,Values=$(terraform output vpc_id) --query 'RouteTables[*].Routes[?DestinationCidrBlock==`0.0.0.0/0`].GatewayId' --output text
```

3. **EC2/Compute health**:
```bash
for ip in $(terraform output -json ec2_public_ips | jq -r '.[]'); do
  timeout 30 nc -z $ip 22 && echo "$ip: SSH OK" || echo "$ip: SSH FAIL"
done

# Check Auto Scaling Group health
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names $(terraform output -raw asg_name) --query 'AutoScalingGroups[0].Instances[*].HealthStatus'
```

4. **RDS/Aurora health**:
```bash
aws rds describe-db-instances --db-instance-identifier $(terraform output -raw rds_id) --query 'DBInstances[0].DBInstanceStatus'
aws rds describe-db-log-files --db-instance-identifier $(terraform output -raw rds_id) --output table
```

5. **Load balancer health**:
```bash
ALB_DNS=$(terraform output -raw alb_dns_name)
ELB_INFO=$(aws elbv2 describe-load-balancers --names $ALB_DNS --query 'LoadBalancers[0]')
TARGET_GROUPS=$(echo $ELB_INFO | jq -r '.TargetGroups[].TargetGroupArn')
for tg in $TARGET_GROUPS; do
  aws elbv2 describe-target-health --target-group-arn $tg --query 'TargetHealthDescriptions[*].TargetHealth.State' --output text
done
```

6. **Kubernetes cluster health**:
```bash
kubectl get nodes -o wide
kubectl cluster-info
kubectl get cs  # check componentstatus (deprecated but still useful)
kubectl get ns -o wide

# Validate deployments
kubectl get deployments -n ${TF_VAR_environment} -o json | jq -r '.items[] | select(.status.availableReplicas != .status.replicas) | .metadata.name'
```

7. **End-to-end application health**:
```bash
APP_URL=https://$(terraform output -raw alb_dns_name)
timeout 60 bash -c "while ! curl -sSf $APP_URL/health; do sleep 5; done"
curl -sSf $APP_URL/health | jq '.status'  # should be "ok"
```

8. **Security group rules**:
```bash
# Ensure no 0.0.0.0/0 on non-HTTP/HTTPS ports
aws ec2 describe-security-groups --group-ids $(terraform output -raw security_group_id) --query 'SecurityGroups[].IpPermissions[?IpProtocol!=`tcp` || (FromPort!=`80` && FromPort!=`443` && CidrIp==`0.0.0.0/0`)]'
```

9. **Cost anomaly check**:
```bash
infrastructure-weaver cost-estimate --format json | jq '.monthly_cost'  # should be < threshold
```

10. **Compliance scan**:
```bash
infrastructure-weaver security-scan --severity high --format sarif | jq '.vulnerabilities[] | select(.severity=="high")'  # should return empty
```

## Troubleshooting Common Issues

**Issue:** `terraform init` fails with S3 backend access denied
**Resolution:**
```bash
# Verify AWS credentials
aws sts get-caller-identity --profile $AWS_PROFILE
# Check S3 bucket policy
aws s3api get-bucket-policy --bucket company-terraform-state-prod --profile $AWS_PROFILE
# Ensure IAM policy allows s3:GetObject, s3:PutObject, s3:ListBucket
```

**Issue:** `terraform plan` shows "Error: Provider produced inconsistent result after apply"
**Resolution:**
```bash
# This indicates eventual consistency issues in AWS. Add explicit dependencies or use refresh=false
terraform plan -refresh=false
# Or add depends_on in modules that interact with IAM roles
```

**Issue:** Ansible fails with "UNREACHABLE! => {"changed": false, "msg": "Failed to connect to the host via ssh: Permission denied (publickey,password)."}"
**Resolution:**
```bash
# Verify SSH key pair exists in AWS
aws ec2 describe-key-pairs --key-names prod-company-key --region $TF_VAR_region
# Check instance is running and IP is correct
terraform output -json ec2_public_ips | jq -r '.[]'
# Ensure private key file exists and permissions are 0600
chmod 0600 infrastructure/keys/${TF_VAR_environment}.pem
```

**Issue:** Kubernetes deployments stuck in "ContainerCreating"
**Resolution:**
```bash
# Check pod events for errors
kubectl describe pod <pod-name> -n prod
# Common causes: image pull secrets missing, insufficient resources, storage class unavailable
# Check node capacity
kubectl describe node <node-name> | grep -A5 "Allocated resources"
# Check PVC status if using storage
kubectl get pvc -n prod
```

**Issue:** `infrastructure-weaver k8s-deploy` times out after 15 minutes
**Resolution:**
```bash
# Increase timeout
infrastructure-weaver k8s-deploy --timeout=30m --wait
# Or check deployments are progressing
kubectl get deployments -n prod -o wide
kubectl get rs -n prod -o wide
# Check events namespace-wide
kubectl get events -n prod --sort-by='.lastTimestamp'
```

**Issue:** Security scan fails with "Resource 'aws_security_group' has not specified 'egress' block"
**Resolution:**
```bash
# Modern Terraform AWS modules require explicit egress rules. Add to security group module:
module "security_group" {
  source  = "terraform-aws-modules/security-group/aws"
  version = "5.1.0"

  egress_with_cidr_block = [
    {
      from_port   = 0
      to_port     = 0
      protocol    = "-1"
      cidr_blocks = ["0.0.0.0/0"]
    }
  ]
}
```

**Issue:** Cost estimate exceeds threshold by 300%
**Resolution:**
```bash
# Review detailed cost breakdown
infrastructure-weaver cost-estimate --format json | jq '.resources[] | select(.monthly_cost > 100)'
# Common issues:
# - Using on-demand instances instead of spot or reserved
# - Overprovisioned instance counts
# - Large RDS/Aurora instances
# - Unnecessary NAT gateways ($32/mo each)
# Adjust variables:
infrastructure-weaver apply -var="instance_count=2" -var="db_instance_class=db.t3.medium" -auto-approve
```

**Issue:** Terraform state drift detected (manual changes in AWS)
**Resolution:**
```bash
# Identify drifted resources
terraform plan -detailed-exitcode  # exit code 2 indicates drift

# Option A: Reconcile Terraform state (refresh)
terraform apply -refresh-only

# Option B: Import drifts into state
terraform import module.asg.aws_autoscaling_group.main $(terraform output -raw asg_id)

# Option C: taint resource to force replacement (use with caution)
terraform taint module.ec2.aws_instance.web[0]
terraform apply -replace=module.ec2.aws_instance.web[0]
```

**Issue:** VPC peering or Transit Gateway attachment fails with "InvalidParameter: The CIDR '10.0.0.0/16' overlaps with"
**Resolution:**
```bash
# Check existing VPCs for overlapping CIDRs
aws ec2 describe-vpcs --query 'Vpcs[?CidrBlock==`10.0.0.0/16`].VpcId'  # if exists, change CIDR
# Update terraform.tfvars to use non-overlapping CIDR (e.g., 10.10.0.0/16)
# Re-run terraform apply (may require destroy of VPC first if resources created)
```

**Issue:** `terraform apply` hangs on "module.vpc.aws_vpc.main: Still creating... [30s elapsed]"
**Resolution:**
```bash
# AWS API rate limiting or eventual consistency. Add explicit timeouts in provider:
provider "aws" {
  region = var.region
  default_tags {
    tags = var.default_tags
  }
  # Add retries and timeouts
  max_retries = 5
}
# Or use -parallelism=1 to serialize operations
terraform apply -parallelism=1
```

**Issue:** IAM role creation fails with "EntityTooLarge: Maximum allowable size for a policy document is 2048 characters"
**Resolution:**
```bash
# Inline policies hitting size limit. Use managed policies instead:
resource "aws_iam_policy" "custom" {
  name   = "CustomPolicy"
  policy = data.aws_iam_policy_document.custom.json
}

resource "aws_iam_role_policy_attachment" "custom" {
  role       = aws_iam_role.custom_role.name
  policy_arn = aws_iam_policy.custom.arn
}
```

**Issue:** EKS cluster creation fails with "SubnetNotFound: The subnet ID 'subnet-xxx' does not exist"
**Resolution:**
```bash
# Wait for VPC module to complete before EKS module. Add explicit dependency:
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "19.11.0"

  subnet_ids = module.vpc.private_subnets
  depends_on = [module.vpc]  # explicit dependency
}
```

**Issue:** `infrastructure-weaver apply` exits with code 2 but no error message
**Resolution:**
```bash
# Verbose logging
TF_LOG=DEBUG terraform apply -auto-approve 2>&1 | tee debug.log
# Or use skill's own logging
infrastructure-weaver apply --log-level debug --log-file infrastructure-logs/apply-$(date +%s).log
```

**Issue:** RDS snapshot copy across regions fails with "DBSnapshotAlreadyExists"
**Resolution:**
```bash
# Snapshot already copied. Check:
aws rds describe-db-snapshots --db-snapshot-identifier prod-db-snapshot --region us-west-2
# Delete existing snapshot if needed, or use unique name with timestamp
```

**Issue:** CloudFront distribution deployment takes >30 minutes
**Normal behavior**: CloudFront deployments can take 15-45 minutes. Use waiter:
```bash
infrastructure-weaver k8s-deploy --wait --timeout=60m
```

## Prompt Examples (What users actually type)

```
"Create a complete AWS infrastructure for a Django app: VPC with public/private subnets across 3 AZs, 
ECS Fargate cluster with ALB, RDS PostgreSQL (multi-AZ), ElastiCache Redis, S3 bucket with CloudFront, 
WAF, CloudWatch alarms. Environment: staging, region: eu-west-1."

"Set up a Kubernetes cluster on GKE with: 2 node pools (n1-standard-2 for app, n1-highmem-4 for DB), 
ingress-nginx, cert-manager, external-dns, and link MetalLB for bare-metal fallback. Enable GKE Autopilot mode."

"Migrate an existing on-premises MySQL database to AWS RDS Aurora. Generate Terraform import commands 
for a manual migration plan. Include VPC peering between old and new environments."

"Implement blue-green deployment pipeline: Use CodePipeline, CodeBuild, ECR, ECS, ALB with weighted target groups. 
Auto-rollback on failed health checks."

"Create a multi-region active-active setup: Two identical stacks in us-east-1 and eu-central-1, 
RDS Global Database, S3 cross-region replication, Route 53 latency routing with health checks."

"Spin up a ephemeral QA environment from Terraform workspaces. Should auto-destroy after 24 hours. 
Include Slack notifications on creation and destruction."

"Add PCI-DSS compliance: Encrypt all storage (EBS, RDS, S3), enable CloudTrail across all regions, 
enforce MFA for privileged users, VPC flow logs, GuardDuty enabled."
```