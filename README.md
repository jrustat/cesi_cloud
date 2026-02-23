# AWS Web Application Infrastructure

Infrastructure Terraform pour déployer une application web avec haute disponibilité sur AWS.

## Architecture

```
                    ┌─────────────┐
                    │   Internet  │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │     ALB     │
                    │  (public)   │
                    └──────┬──────┘
                           │
       ┌───────────────────┼───────────────────┐
       │                   │                   │
┌──────▼──────┐     ┌──────▼──────┐
│  Public     │     │  Public     │
│  Subnet 1   │     │  Subnet 2   │
│  (AZ-a)     │     │  (AZ-b)     │
└──────┬──────┘     └──────┬──────┘
       │                   │
┌──────▼──────┐     ┌──────▼──────┐
│  NAT GW     │     │  (NAT GW)   │
└──────┬──────┘     └──────┬──────┘
       │                   │
┌──────▼──────┐     ┌──────▼──────┐
│  Private    │     │  Private    │
│  Subnet 1   │     │  Subnet 2   │
│  EC2 (httpd)│     │  EC2 (httpd)│
└─────────────┘     └─────────────┘
```

## Ressources déployées

- **VPC** avec DNS support
- **2 Subnets publics** (pour ALB et NAT Gateway)
- **2 Subnets privés** (pour les instances EC2)
- **Internet Gateway**
- **NAT Gateway** (configurable: single ou par AZ)
- **Route Tables** (publiques et privées)
- **Application Load Balancer** (internet-facing)
- **Target Group** avec health checks
- **2 Instances EC2** avec serveur httpd
- **Security Groups** (ALB et EC2)

## Prérequis

- Terraform >= 1.0.0
- AWS CLI configuré
- Bucket S3 pour le state backend

## Structure des fichiers

```
.
├── main.tf                    # Ressources principales
├── providers.tf               # Configuration provider AWS
├── variables.tf               # Définition des variables
├── outputs.tf                 # Outputs
├── terraform.tfvars.example   # Exemple de variables
├── environments/
│   ├── dev.tfvars            # Configuration dev
│   ├── staging.tfvars        # Configuration staging
│   └── prod.tfvars           # Configuration production
└── README.md
```

## Utilisation

### 1. Initialisation

```bash
terraform init
```

### 2. Déploiement par environnement

**Développement:**
```bash
terraform plan -var-file="environments/dev.tfvars"
terraform apply -var-file="environments/dev.tfvars"
```

**Staging:**
```bash
terraform plan -var-file="environments/staging.tfvars"
terraform apply -var-file="environments/staging.tfvars"
```

**Production:**
```bash
terraform plan -var-file="environments/prod.tfvars"
terraform apply -var-file="environments/prod.tfvars"
```

### 3. Accès à l'application

Après le déploiement, l'URL de l'application est affichée dans les outputs:

```bash
terraform output alb_url
```

## Variables configurables

| Variable | Description | Default |
|----------|-------------|---------|
| `environment` | Nom de l'environnement | `dev` |
| `project_name` | Nom du projet | `webapp` |
| `aws_region` | Région AWS | `us-east-1` |
| `vpc_cidr` | CIDR du VPC | `10.0.0.0/16` |
| `instance_type` | Type d'instance EC2 | `t2.micro` |
| `instance_count` | Nombre d'instances | `2` |
| `enable_nat_gateway` | Activer NAT Gateway | `true` |
| `single_nat_gateway` | NAT unique (économie) | `true` |

Voir [variables.tf](variables.tf) pour la liste complète.

## Pipeline CI/CD

Pour intégrer dans un pipeline, utilisez les commandes suivantes:

```yaml
# Exemple GitHub Actions
- name: Terraform Init
  run: terraform init

- name: Terraform Plan
  run: terraform plan -var-file="environments/${{ env.ENVIRONMENT }}.tfvars" -out=tfplan

- name: Terraform Apply
  run: terraform apply -auto-approve tfplan
```

## Sécurité

- Les instances EC2 sont dans des subnets **privés**
- Seul l'ALB est accessible depuis Internet
- Les EC2 n'acceptent le trafic HTTP que depuis l'ALB
- SSH désactivé par défaut (configurable via `ssh_allowed_cidr_blocks`)
- Egress ouvert pour les mises à jour système

## Coûts

En environnement dev avec les paramètres par défaut:
- NAT Gateway: ~$0.045/heure + transfert
- ALB: ~$0.0225/heure + LCU
- EC2 t2.micro: Eligible Free Tier

Pour réduire les coûts en dev/staging:
- `single_nat_gateway = true` (1 NAT au lieu de 2)
- `instance_type = "t2.micro"`

## Destruction

```bash
terraform destroy -var-file="environments/dev.tfvars"
```
