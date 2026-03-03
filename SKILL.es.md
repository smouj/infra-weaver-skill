name: "infrastructure-weaver"
description: "Orquestra y automatiza recursos en la nube en arquitecturas cohesivas y listas para producción utilizando herramientas de IaC y flujos de trabajo declarativos"
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

## Propósito

Infrastructure Weaver automatiza la creación, gestión y destrucción de arquitecturas en la nube complejas utilizando patrones de Infraestructura como Código (IaC). Orquesta múltiples herramientas (Terraform, Ansible, Kubernetes, CloudFormation) en flujos de trabajo unificados.

**Casos de uso reales:**
- Desplegar aplicaciones de 3 niveles en producción en AWS con VPC, subredes públicas/privadas, NAT gateways, RDS PostgreSQL, Auto Scaling Group, Application Load Balancer y CloudFront CDN
- Aprovisionar y configurar clústeres Kubernetes (EKS/GKE/AKS) con grupos de nodos, controladores de ingress (nginx/alb), service mesh (istio) y aislamiento de namespaces
- Implementar aislamiento multi-entorno (dev/staging/prod) con Terraform workspaces e inventarios dinámicos de Ansible
- Ejecutar despliegues blue-green con cero downtime usando Kubernetes Deployments y AWS CodeDeploy
- Configurar pipelines de recuperación ante desastres: replicación cross-region, copias de seguridad automatizadas de RDS, sincronización S3 cross-region y políticas de failover de Route53
- Construir infraestructura compliant: PCI-DSS, HIPAA, GDPR con módulos de seguridad (security groups, IAM policies, KMS encryption, CloudTrail)
- Migrar servidores on-prem legacy a la nube usando playbooks de migración de Ansible y mapeos de importación de Terraform generados
- Crear entornos de prueba efímeros que despliegan stacks completos en creación de PR y se autodestruyen después de 24h

## Alcance

La habilidad proporciona estos **comandos concretos**:

- `plan`: Genera plan de ejecución de Terraform y dry-run de Ansible. Flags: `--target MODULE`, `--var 'key=value'`, `--var-file FILE`, `--compact-warnings`, `--json`, `--color=always`
- `apply`: Aplica cambios de infraestructura. Flags: `--auto-approve`, `--target MODULE`, `--var 'key=value'`, `--var-file FILE`, `--compact-warnings`, `--skip-refresh`, `--parallelism=N`
- `destroy`: Destruye todos los recursos gestionados. Flags: `--auto-approve`, `--target MODULE`, `--force`, `--timeout=DURATION`
- `validate`: Ejecuta análisis estático en configs de Terraform, playbooks de Ansible y manifiestos de Kubernetes. Flags: `--strict`, `--json`, `--skip-child-modules`
- `inventory`: Genera inventario dinámico de Ansible desde outputs de Terraform o APIs de cloud provider. Flags: `--format json|yaml|ini`, `--refresh`, `--group-by TAG`
- `k8s-deploy`: Despliega manifiestos de Kubernetes. Flags: `--namespace NS`, `--context CTX`, `--server-side-apply`, `--wait`, `--timeout=DURATION`, `--prune`
- `output`: Consulta outputs del estado de Terraform. Flags: `--json`, `--raw`, `--state=PATH`, `--module MODULE`
- `import`: Importa recursos existentes al estado de Terraform. Flags: `--provider=PROVIDER`, `--var-file FILE`, `--dry-run`
- `test`: Ejecuta pruebas de integración de infraestructura usando terratest o inspec. Flags: `--suite SUITE`, `--parallel`, `--fail-fast`, `--test TIMEOUT`
- `module-publish`: Empaqueta y publica módulos de Terraform reutilizables a registro privado. Flags: `--source PATH`, `--version SEMVER`, `--registry URL`, `--namespace NS`
- `workspace-switch`: Cambia entre workspaces de Terraform y actualiza kubeconfig. Flags: `--workspace NAME`, `--context`, `--set-current`
- `cost-estimate`: Genera estimaciones de coste usando Infracost o terraform-cost-estimation. Flags: `--compare-with PLAN_FILE`, `--output format`, `--threshold VALUE`
- `security-scan`: Ejecuta escaneos con tfsec, checkov o trivy. Flags: `--severity LEVEL`, `--exit-code`, `--format json|sarif`, `--quiet`

## Proceso de Trabajo Detallado

Infrastructure Weaver sigue un pipeline estricto y reproducible:

**1. CARGA DE CONFIGURACIÓN**
```bash
# Cargar variables específicas de entorno
export AWS_PROFILE="production"
export TF_VAR_region="us-east-1"
export TF_VAR_environment="prod"
```

La habilidad lee la configuración en este orden:
- `infrastructure/backend.tf` para backend de estado remoto (S3+DynamoDB)
- `infrastructure/variables.tf` para variables declaradas
- `infrastructure/terraform.tfvars` para valores de variables
- `infrastructure/environments/${TF_VAR_environment}.tfvars` para overrides de entorno
- Variables de entorno prefijadas con `TF_VAR_`

**2. INICIALIZACIÓN DE TERRAFORM**
```bash
cd /tmp/flickclaw-work/infrastructure
terraform init \
  -backend-config="bucket=mycompany-terraform-state" \
  -backend-config="dynamodb_table=terraform-lock" \
  -backend-config="region=us-east-1" \
  -reconfigure \
  -input=false
```
Verifica proveedores requeridos (aws, random, kubernetes, helm, archive). Descarga plugins al directorio `.terraform`. Valida conectividad del backend.

**3. GENERACIÓN DE PLAN**
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
Captura plan como archivo binario `tfplan.binary` para apply idéntico garantizado.

**4. GENERACIÓN DE INVENTARIO DE ANSIBLE**
```bash
# Generar inventario desde outputs de Terraform
terraform output -json | jq -r '
  .ecs_cluster_endpoint.value // empty,
  ".ec2_public_ips[] as $ip | $ip" as $ip |
  "[webservers]\n$ip"'
> infrastructure/inventory/hosts.ini
```

O usando el comando inventory de la habilidad:
```bash
infrastructure-weaver inventory \
  --format ini \
  --refresh \
  --group-by "tag:Environment, tag:Role" \
  > infrastructure/inventory/hosts.ini
```

**5. GESTIÓN DE CONFIGURACIÓN**
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

**6. DESPLIEGUE DE KUBERNETES**
```bash
# Actualizar kubeconfig desde clúster EKS
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

**7. VALIDACIÓN POST-DESPLIEGUE**
```bash
# Verificar estado de pods
kubectl get pods -n prod -o wide --sort-by='.status.startTime'

# Verificar endpoints de servicios
kubectl get endpoints -n prod

# Verificar ingress Alb
kubectl get ingress -n prod -o jsonpath='{.items[*].status.loadBalancer.ingress[0].hostname}'

# Health check
curl -sSf https://$(terraform output -raw alb_dns_name)/health || exit 1
```

**8. ESCANEO DE COSTE Y SEGURIDAD**
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

**9. GESTIÓN DE ESTADO**
- Estado remoto almacenado en S3 con bloqueo DynamoDB
- Backups de estado creados automáticamente en cada apply: `terraform-state-backup-YYYY-MM-DD-HHMMSS.tfstate`
- Estado rotado cada 30 días, archivado a Glacier después de 90 días

## Reglas de Oro

1. **Estado remoto únicamente**: Nunca usar estado local. Todo estado debe almacenarse en S3 con bloqueo DynamoDB y encryption-at-rest. Fallar si falta bloque `backend`.
2. **Infraestructura inmutable**: Nunca modificar recursos directamente en consola cloud. Todos los cambios deben ir通过 Terraform apply. Si cambio manual detectado (`terraform plan` muestra diff con existente), levantar alerta.
3. **Aislamiento de workspace**: Cada entorno (dev/staging/prod) usa workspace de Terraform separado y key de estado S3 separada. No mezclar recursos entre workspaces.
4. **Gestión de secretos**: Nunca hardcodear secretos. Usar SSM Parameter Store, Secrets Manager o Vault. Ansible Vault para secretos de playbooks. Fallar scan si passwords en texto plano encontradas.
5. **Pinning de versiones de módulos**: Todas las fuentes de módulo deben tener constraints de versión explícitos (ej. `source = "terraform-aws-modules/vpc/aws" version = "5.1.2"`). No usar constraints `>=`, solo versiones exactas.
6. **Política de etiquetado**: Todos los recursos deben tener etiquetas obligatorias: `Environment`, `Owner`, `Project`, `ComplianceLevel`. Assert fallar en plan si faltan etiquetas.
7. **Compatibilidad hacia atrás**: Configuraciones de Terraform deben soportar al menos 2 minor version upgrades. Probar ruta de upgrade `terraform 0.13` a `1.5`.
8. **Despliegues cero downtime**: Usar rolling updates de Kubernetes con `maxSurge=25%` y `maxUnavailable=0`. Para EC2 ASG, usar lifecycle hooks y warm pools de ASG.
9. **Backup antes de destroy**: En `destroy`, automáticamente crear backup final de estado y exportar datos críticos (RDS snapshots, EBS volumes) antes de eliminación de recursos.
10. **Requisito DR**: Todos los stacks de producción deben tener replicación cross-region configurada. Validar via `infrastructure-weaver dr-validate --standby-region` antes de marcar entorno production-ready.

## Ejemplos

### Ejemplo 1: Desplegar aplicación web de 3 niveles en AWS

**Prompt del usuario:** "Deploy a production web app on AWS with auto-scaling, RDS, and CloudFront. Use us-east-1, prod environment, 3 AZs."

**Ejecución de la habilidad:**

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

### Ejemplo 2: Aprovisionar clúster EKS con ingress y service mesh

**Prompt del usuario:** "Create an EKS cluster with 3 managed node groups (general purpose, memory optimized, GPU), install nginx ingress, istio, and argocd."

**Ejecución de la habilidad:**

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

### Ejemplo 3: Configuración de DR multi-cloud

**Prompt del usuario:** "Configure DR: primary in us-east-1, standby in us-west-2. RDS cross-region read replica, S3 replication, Route53 failover."

**Ejecución de la habilidad:**

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

## Comandos de Rollback

**Rollback inmediato** (si apply falla o se detectan problemas dentro de 5 minutos):
```bash
# 1. Rollback de Terraform a estado anterior (si apply parcialmente tuvo éxito)
terraform state pull > terraform-state-backup-before-rollback.json
terraform apply tfplan-prev.binary  # usar archivo de plan anterior

# O destroy rápido si se necesita rollback completo
infrastructure-weaver destroy \
  --auto-approve \
  --force \
  --grace-period=30s \
  --target=module.vpc \
  --target=module.eks \
  --target=module.rds

# 2. Remover despliegues de Kubernetes
infrastructure-weaver k8s-deploy \
  --namespace prod \
  --prune \
  --cascade=foreground \
  --delete-emptydir-data

# 3. Restaurar datos desde snapshots (RDS/EBS/S3)
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier prod-db-rollback \
  --db-snapshot-identifier prod-db-snapshot-2024-01-15-1200 \
  --no-multi-az
```

**Recuperación de estado** (si archivo de estado corrupto o perdido):
```bash
# 1. Listar backups de estado disponibles en S3
aws s3 ls s3://company-terraform-state-prod/backups/ \
  --recursive \
  --profile company-prod

# 2. Restaurar último backup
aws s3 cp s3://company-terraform-state-prod/backups/terraform-state-backup-2024-01-15-1200.tfstate ./terraform.tfstate

# 3. Force unlock si lock stuck
terraform force-unlock $(terraform state pull | jq -r '.lockid')

# 4. Verificar integridad de estado
terraform validate
terraform plan -detailed-exitcode  # debe retornar 0 (sin cambios)
```

**Rollback de configuración** (revertir a versiones anteriores de módulos):
```bash
git -C infrastructure/ log --oneline -10
git -C infrastructure/ checkout <previous-commit-sha>

infrastructure-weaver init -reconfigure
infrastructure-weaver plan -detailed-exitcode

# Si plan muestra solo cambios de versión de módulo (sin cambios de recursos):
infrastructure-weaver apply -var="module_version=1.2.3" -auto-approve
```

**Rollback de configuración Ansible:**
```bash
# Listar versiones de playbook disponibles (git tags)
git -C infrastructure/ansible/ tag | grep -E 'playbook-v[0-9]+' | sort -V

# Rollback a versión específica
git -C infrastructure/ansible/ checkout tags/playbook-v1.4.0 -b rollback-branch

# Re-ejecutar configuración
ansible-playbook -i inventory/hosts.ini infrastructure/ansible/playbooks/configure-webservers.yml

# O usar tags de rollback de Ansible (si playbooks idempotentes)
ansible-playbook -i inventory/hosts.ini infrastructure/ansible/playbooks/rollback.yml \
  -e "rollback_to_version=1.4.0"
```

**Rollback de CloudFormation** (si se usan stacks de CloudFormation):
```bash
# 1. Listar eventos de stack para identificar fallo
aws cloudformation describe-stack-events \
  --stack-name prod-app-stack \
  --region us-east-1 \
  --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`]' \
  --output table

# 2. Eliminar stack (rollback automático si --disable-rollback no se设置 originalmente)
aws cloudformation delete-stack \
  --stack-name prod-app-stack \
  --region us-east-1

# 3. Esperar eliminación
aws cloudformation wait stack-delete-complete \
  --stack-name prod-app-stack \
  --region us-east-1

# 4. Re-desplegar con template actualizado
infrastructure-weaver apply --template-file=templates/fixed-stack.yaml
```

**Rollback de Kubernetes:**
```bash
# 1. Ver historial de rollout
kubectl rollout history deployment/webapp -n prod

# 2. Rollback a revisión anterior
kubectl rollout undo deployment/webapp -n prod

# 3. O rollback a revisión específica
kubectl rollout undo deployment/webapp -n prod --to-revision=3

# 4. Monitorear rollout
kubectl rollout status deployment/webapp -n prod --watch

# 5. Si pod falla, automáticamente restaurar revisiones anteriores de configMap/secret
kubectl rollout restart deployment/webapp -n prod
```

## Dependencias y Requisitos

**Dependencias del sistema** (verificar con `infrastructure-weaver check-prereqs`):
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

**Requisitos de AWS** (para proveedor AWS):
- IAM user/role con policies: `AdministratorAccess` (o custom con `*:*` para setup de infra)
- Bucket S3 para estado de Terraform (versioning + encryption enabled)
- Tabla DynamoDB para locking de estado
- Accceso a SSM Parameter Store o Secrets Manager

**Requisitos de Kubernetes**:
- kubectl configurado con contexto de clúster
- Permisos `kubectl apply` (cluster-admin para bootstrapping)
- Helm 3 instalado para despliegues de charts

**Estructura de repositorio** (layout esperado):
```
infrastructure/
├── backend.tf                    # config de estado remoto
├── main.tf                       # módulo root
├── variables.tf                  # declaraciones de variables
├── outputs.tf                    # declaraciones de outputs
├── terraform.tfvars              # valores de variables (dev)
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

## Pasos de Verificación

**Después de apply**, verificar automáticamente:

1. **Sanity check de outputs de Terraform**:
```bash
infrastructure-weaver output --json | jq '
  . as $o |
  keys[] as $k |
  "\($k): \($o[$k] // "null")"'
```
Assert todos los outputs críticos no-nulos.

2. **Conectividad VPC/Red**:
```bash
# Ping entre AZs
for az in $(terraform output -json availability_zones | jq -r '.[]'); do
  aws ec2 describe-subnets --filters Name=availability-zone,Values=$az --query 'Subnets[?MapPublicIpOnLaunch==`true`].SubnetId' --output text
done

# Validar route tables
aws ec2 describe-route-tables --filters Name=vpc-id,Values=$(terraform output vpc_id) --query 'RouteTables[*].Routes[?DestinationCidrBlock==`0.0.0.0/0`].GatewayId' --output text
```

3. **Salud de EC2/Compute**:
```bash
for ip in $(terraform output -json ec2_public_ips | jq -r '.[]'); do
  timeout 30 nc -z $ip 22 && echo "$ip: SSH OK" || echo "$ip: SSH FAIL"
done

# Verificar salud de Auto Scaling Group
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names $(terraform output -raw asg_name) --query 'AutoScalingGroups[0].Instances[*].HealthStatus'
```

4. **Salud de RDS/Aurora**:
```bash
aws rds describe-db-instances --db-instance-identifier $(terraform output -raw rds_id) --query 'DBInstances[0].DBInstanceStatus'
aws rds describe-db-log-files --db-instance-identifier $(terraform output -raw rds_id) --output table
```

5. **Salud de load balancer**:
```bash
ALB_DNS=$(terraform output -raw alb_dns_name)
ELB_INFO=$(aws elbv2 describe-load-balancers --names $ALB_DNS --query 'LoadBalancers[0]')
TARGET_GROUPS=$(echo $ELB_INFO | jq -r '.TargetGroups[].TargetGroupArn')
for tg in $TARGET_GROUPS; do
  aws elbv2 describe-target-health --target-group-arn $tg --query 'TargetHealthDescriptions[*].TargetHealth.State' --output text
done
```

6. **Salud de clúster Kubernetes**:
```bash
kubectl get nodes -o wide
kubectl cluster-info
kubectl get cs  # check componentstatus (deprecated pero útil)
kubectl get ns -o wide

# Validar deployments
kubectl get deployments -n ${TF_VAR_environment} -o json | jq -r '.items[] | select(.status.availableReplicas != .status.replicas) | .metadata.name'
```

7. **Salud de aplicación end-to-end**:
```bash
APP_URL=https://$(terraform output -raw alb_dns_name)
timeout 60 bash -c "while ! curl -sSf $APP_URL/health; do sleep 5; done"
curl -sSf $APP_URL/health | jq '.status'  # debe ser "ok"
```

8. **Reglas de security group**:
```bash
# Asegurar no 0.0.0.0/0 en puertos no-HTTP/HTTPS
aws ec2 describe-security-groups --group-ids $(terraform output -raw security_group_id) --query 'SecurityGroups[].IpPermissions[?IpProtocol!=`tcp` || (FromPort!=`80` && FromPort!=`443` && CidrIp==`0.0.0.0/0`)]'
```

9. **Check de anomalía de coste**:
```bash
infrastructure-weaver cost-estimate --format json | jq '.monthly_cost'  # debe ser < threshold
```

10. **Scan de compliance**:
```bash
infrastructure-weaver security-scan --severity high --format sarif | jq '.vulnerabilities[] | select(.severity=="high")'  # debe retornar vacío
```

## Solución de Problemas

**Problema:** `terraform init` falla con S3 backend access denied
**Resolución:**
```bash
# Verificar credenciales AWS
aws sts get-caller-identity --profile $AWS_PROFILE
# Checkear bucket policy
aws s3api get-bucket-policy --bucket company-terraform-state-prod --profile $AWS_PROFILE
# Asegurar IAM policy permite s3:GetObject, s3:PutObject, s3:ListBucket
```

**Problema:** `terraform plan` muestra "Error: Provider produced inconsistent result after apply"
**Resolución:**
```bash
# Indica issues de consistencia eventual en AWS. Agregar dependencias explícitas o usar refresh=false
terraform plan -refresh=false
# O agregar depends_on en módulos que interactúan con IAM roles
```

**Problema:** Ansible falla con "UNREACHABLE! => {"changed": false, "msg": "Failed to connect to the host via ssh: Permission denied (publickey,password)."}"
**Resolución:**
```bash
# Verificar key pair existe en AWS
aws ec2 describe-key-pairs --key-names prod-company-key --region $TF_VAR_region
# Checkear que instancia está corriendo y IP es correcta
terraform output -json ec2_public_ips | jq -r '.[]'
# Asegurar archivo de private key existe y permisos son 0600
chmod 0600 infrastructure/keys/${TF_VAR_environment}.pem
```

**Problema:** Deployments de Kubernetes stuck en "ContainerCreating"
**Resolución:**
```bash
# Ver eventos de pod para errores
kubectl describe pod <pod-name> -n prod
# Causas comunes: image pull secrets missing, recursos insuficientes, storage class unavailable
# Verificar capacidad de nodo
kubectl describe node <node-name> | grep -A5 "Allocated resources"
# Verificar estado de PVC si usa storage
kubectl get pvc -n prod
```

**Problema:** `infrastructure-weaver k8s-deploy` timeout después de 15 minutos
**Resolución:**
```bash
# Incrementar timeout
infrastructure-weaver k8s-deploy --timeout=30m --wait
# O checkear que deployments progresan
kubectl get deployments -n prod -o wide
kubectl get rs -n prod -o wide
# Ver eventos a nivel namespace
kubectl get events -n prod --sort-by='.lastTimestamp'
```

**Problema:** Security scan falla con "Resource 'aws_security_group' has not specified 'egress' block"
**Resolución:**
```bash
# Módulos de Terraform AWS modernos requieren egress rules explícitas. Agregar a módulo security group:
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

**Problema:** Cost estimate excede threshold por 300%
**Resolución:**
```bash
# Revisar breakdown detallado de costes
infrastructure-weaver cost-estimate --format json | jq '.resources[] | select(.monthly_cost > 100)'
# Problemas comunes:
# - Usar instancias on-demand en vez de spot o reserved
# - Overprovisioned instance counts
# - Instancias RDS/Aurora grandes
# - NAT gateways innecesarios ($32/mo cada una)
# Ajustar variables:
infrastructure-weaver apply -var="instance_count=2" -var="db_instance_class=db.t3.medium" -auto-approve
```

**Problema:** Terraform state drift detectado (cambios manuales en AWS)
**Resolución:**
```bash
# Identificar recursos con drift
terraform plan -detailed-exitcode  # exit code 2 indica drift

# Opción A: Reconciliar estado de Terraform (refresh)
terraform apply -refresh-only

# Opción B: Importar drifts al estado
terraform import module.asg.aws_autoscaling_group.main $(terraform output -raw asg_id)

# Opción C: taint recurso para forzar replacement (usar con precaución)
terraform taint module.ec2.aws_instance.web[0]
terraform apply -replace=module.ec2.aws_instance.web[0]
```

**Problema:** VPC peering o Transit Gateway attachment falla con "InvalidParameter: The CIDR '10.0.0.0/16' overlaps with"
**Resolución:**
```bash
# Verificar VPCs existentes por CIDRs superpuestos
aws ec2 describe-vpcs --query 'Vpcs[?CidrBlock==`10.0.0.0/16`].VpcId'  # si existe, cambiar CIDR
# Actualizar terraform.tfvars para usar CIDR no-superpuesto (ej. 10.10.0.0/16)
# Re-ejecutar terraform apply (puede requerir destroy de VPC primero si recursos creados)
```

**Problema:** `terraform apply` hangs en "module.vpc.aws_vpc.main: Still creating... [30s elapsed]"
**Resolución:**
```bash
# Rate limiting de AWS API o consistencia eventual. Agregar timeouts explícitos en provider:
provider "aws" {
  region = var.region
  default_tags {
    tags = var.default_tags
  }
  # Agregar retries y timeouts
  max_retries = 5
}
# O usar -parallelism=1 para serializar operaciones
terraform apply -parallelism=1
```

**Problema:** Creación de IAM role falla con "EntityTooLarge: Maximum allowable size for a policy document is 2048 characters"
**Resolución:**
```bash
# Inline policies hits size limit. Usar managed policies en su lugar:
resource "aws_iam_policy" "custom" {
  name   = "CustomPolicy"
  policy = data.aws_iam_policy_document.custom.json
}

resource "aws_iam_role_policy_attachment" "custom" {
  role       = aws_iam_role.custom_role.name
  policy_arn = aws_iam_policy.custom.arn
}
```

**Problema:** Creación de EKS cluster falla con "SubnetNotFound: The subnet ID 'subnet-xxx' does not exist"
**Resolución:**
```bash
# Esperar que módulo VPC complete antes de módulo EKS. Agregar dependencia explícita:
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "19.11.0"

  subnet_ids = module.vpc.private_subnets
  depends_on = [module.vpc]  # dependencia explícita
}
```

**Problema:** `infrastructure-weaver apply` exit con code 2 pero sin mensaje de error
**Resolución:**
```bash
# Logging verboso
TF_LOG=DEBUG terraform apply -auto-approve 2>&1 | tee debug.log
# O usar logging propio de la habilidad
infrastructure-weaver apply --log-level debug --log-file infrastructure-logs/apply-$(date +%s).log
```

**Problema:** RDS snapshot copy cross-regions falla con "DBSnapshotAlreadyExists"
**Resolución:**
```bash
# Snapshot ya copiado. Checkear:
aws rds describe-db-snapshots --db-snapshot-identifier prod-db-snapshot --region us-west-2
# Eliminar snapshot existente si necesario, o usar nombre único con timestamp
```

**Problema:** CloudFront distribution deployment tarda >30 minutos
**Comportamiento normal**: Despliegues CloudFront pueden tardar 15-45 minutos. Usar waiter:
```bash
infrastructure-weaver k8s-deploy --wait --timeout=60m
```

## Ejemplos de Prompt (Lo que los usuarios realmente escriben)

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
```