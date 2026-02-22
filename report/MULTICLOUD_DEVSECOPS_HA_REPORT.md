# Multi-Cloud, DevSecOps & High Availability Report  
## Pizza App Project — Detailed

**Report focus:** Multi-Cloud strategy, DevSecOps practices, and High Availability (HA) implementations in the Pizza App project.  
**Date:** February 22, 2025

---

## Table of Contents

1. [Multi-Cloud Implementation](#1-multi-cloud-implementation)  
2. [DevSecOps Practices](#2-devsecops-practices)  
3. [High Availability (HA)](#3-high-availability-ha)  
4. [Summary Tables](#4-summary-tables)  
5. [Conclusion](#5-conclusion)  
6. [Image Placeholders Index](#6-image-placeholders-index)

---

## 1. Multi-Cloud Implementation

### 1.1 Overview

The project deploys the **same MERN application** (MongoDB, Express, React, Node.js) on **two public clouds** in parallel. Both environments serve the same front-end and back-end; only the underlying infrastructure and cloud APIs differ.

| Cloud   | Region / Location | Primary components |
|---------|-------------------|---------------------|
| **AWS** | `us-east-1`       | VPC, 2 Availability Zones, Application Load Balancer (ALB), Auto Scaling Group (ASG), EC2 (custom AMI from Packer) |
| **Azure** | East US        | Resource group `mern-app-rg`, VNet, subnet, NSG, VM Scale Set (VMSS) `mern-vmss`, Standard Load Balancer `mern-app-lb`, static public IP |

> **📷 IMAGE PLACEHOLDER — Multi-cloud high-level architecture**  
> *Insert image: Diagram showing Pizza App deployed on both AWS and Azure with shared components (Terraform, Packer, Vault, GitHub Actions). Suggested filename: `img-01-multicloud-architecture.png`*

### 1.2 Unified Infrastructure as Code (Terraform)

**Single codebase, two providers**

- One Terraform root module uses both **AWS** (`hashicorp/aws` ~> 5.0) and **Azure** (`hashicorp/azurerm` ~> 3.0) providers.
- Same `terraform init`, `terraform plan`, and `terraform apply` provision and update resources on **both** clouds.
- No separate Terraform projects per cloud; all resources are defined under `IaC_Terraform/`.

**Remote state (shared and locked)**

- **Backend:** AWS S3.
- **Bucket:** `company-tf-state-backend`.
- **State key:** `multi-cloud-mern/terraform.tfstate`.
- **Region:** `us-east-1`.
- **Locking:** DynamoDB table `terraform-state-lock` to prevent concurrent applies and state corruption.
- **Encryption:** State at rest is encrypted (S3 backend `encrypt = true`).

**Exact Terraform backend block (from `providers.tf`):**

```hcl
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
    azurerm = { source = "hashicorp/azurerm", version = "~> 3.0" }
  }
  backend "s3" {
    bucket         = "company-tf-state-backend"
    key            = "multi-cloud-mern/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}
```

**Terraform file layout**

| File | Description |
|------|-------------|
| `providers.tf` | AWS (us-east-1), Azure (features {}), Vault; S3 backend + DynamoDB lock |
| `data.tf` | Vault KV v2 data source for secrets |
| `AWS_network.tf` | VPC, subnets (2 AZs), IGW, route tables, security group |
| `AWS_compute.tf` | Key pair, launch template, user-data, ALB, target group, listener, ASG |
| `Azure_network.tf` | Resource group, VNet, subnet, NSG |
| `Azure_compute.tf` | Linux VMSS; AWS ASG (same file) |
| `Azure_LB.tf` | Public IP, load balancer, backend pool, health probe, LB rule |
| `outputs.tf` | AWS ALB DNS, AWS instance IP, Azure RG, VMSS name, Azure LB public IP |

> **📷 IMAGE PLACEHOLDER — Terraform resource map**  
> *Insert image: Diagram or table of Terraform resources per cloud (VPC/ASG/ALB on AWS; RG/VNet/VMSS/LB on Azure). Suggested filename: `img-02-terraform-resources.png`*

### 1.3 Consistent Base Images (Packer)

**One Packer config, two builders**

- **Config file:** `IaC_Packer/build.pkr.hcl`.
- **Required plugins:** `amazon` (~> 1.2), `azure` (~> 2.0), `ansible` (~> 1.1).

**AWS builder (`amazon-ebs.ubuntu_base`)**

| Setting | Value |
|---------|--------|
| Region | `us-east-1` |
| Instance type | `t2.micro` |
| SSH username | `ubuntu` |
| AMI name | `mern-base-image-{{timestamp}}` |
| VPC ID | Set in `build.pkr.hcl` (e.g. `vpc-028581e5eb7ed274f`) |
| Subnet ID | Set in `build.pkr.hcl` (e.g. `subnet-047768671da863997`) |
| Source AMI | Canonical Ubuntu 22.04 (`ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*`), EBS, HVM, owner `099720109477` |

**Azure builder (`azure-arm.ubuntu_base`)**

| Setting | Value |
|---------|--------|
| OS type | Linux |
| Build resource group | `packer-build-rg` |
| VM size | `Standard_B1s` |
| Source image | Canonical `0001-com-ubuntu-server-jammy`, sku `22_04-lts` |
| Managed image resource group | `mern-app-rg` |
| Managed image name | `mern-base-image-{{timestamp}}` |
| Auth | `use_azure_cli_auth = true` (uses `az login`) |

**Shared provisioning: Ansible playbook**

- **Playbook:** `ansible/deploy.yml`.
- **User:** SSH as `ubuntu`; Ansible runs with `become: yes`.
- **Host key checking:** Disabled for ephemeral Packer instances (`ANSIBLE_HOST_KEY_CHECKING=False`).

**Ansible tasks (full list)**

1. **Create 2GB swap** — `fallocate -l 2G /swapfile` (idempotent with `creates: /swapfile`) to avoid OOM during npm build.
2. **Set swap permissions and activate** — `chmod 600`, `mkswap`, `swapon`, append to `/etc/fstab`.
3. **Update apt and upgrade** — `apt update_cache`, `upgrade: dist`.
4. **Install system packages** — `git`, `curl`, `nginx`.
5. **Add NodeSource repo** — Run `curl -fsSL https://deb.nodesource.com/setup_20.x | bash -` (creates `/etc/apt/sources.list.d/nodesource.list`).
6. **Install Node.js** — `apt name: nodejs`.
7. **Install PM2 globally** — `npm name: pm2, global: yes`.
8. **Create app directory** — `/var/www/mern-app/frontend/build` (so Nginx does not fail on first boot).
9. **Copy Nginx config** — `./nginx.conf` → `/etc/nginx/sites-available/default`.
10. **Restart Nginx** — `service nginx restarted`.
11. **Remove curl** — `apt state: absent` for `curl` (cleanup).

Result: **identical runtime environment** (Ubuntu 22.04, Node 20, Nginx, PM2, app dirs) on both AWS and Azure base images.

> **📷 IMAGE PLACEHOLDER — Packer build flow**  
> *Insert image: Flow diagram — Ubuntu 22.04 → Packer AWS + Azure builders → Ansible deploy.yml → MERN base image for both clouds. Suggested filename: `img-03-packer-flow.png`*

### 1.4 Single CI/CD Pipeline (GitHub Actions)

**Workflow file:** `Front-end/.github/workflows/deploy.yaml`  
**Name:** Multi-Cloud Instance Refresh CI/CD

**Trigger**

- **On:** `push` to branch `master`.

**Permissions (OIDC)**

- `id-token: write` — Required for GitHub to request OIDC tokens for AWS and Azure.
- `contents: read` — Read repository code.

**Jobs (run in parallel)**

| Job | Name | Runner | Purpose |
|-----|------|--------|---------|
| `refresh-aws` | Refresh AWS Auto Scaling Group | `ubuntu-latest` | Start instance refresh on ASG `mern-asg` |
| `refresh-azure` | Refresh Azure VMSS | `ubuntu-latest` | Upgrade all VMSS instances for `mern-vmss` |

**AWS job steps**

1. **Configure AWS Credentials via OIDC** — Action: `aws-actions/configure-aws-credentials@v4`. Uses `secrets.AWS_ROLE_ARN`, region `us-east-1`.
2. **Start AWS Instance Refresh** — CLI: `aws autoscaling start-instance-refresh --auto-scaling-group-name mern-asg --preferences '{"InstanceWarmup": 60, "MinHealthyPercentage": 50}'`.

**Azure job steps**

1. **Log into Azure via OIDC** — Action: `azure/login@v2`. Uses `secrets.AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`.
2. **Upgrade Azure VM Scale Set** — CLI: `az vmss update-instances --resource-group mern-app-rg --name mern-vmss --instance-ids "*"`.

**Required GitHub secrets**

- **AWS:** `AWS_ROLE_ARN` (IAM role ARN for OIDC).
- **Azure:** `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`.

**Full workflow YAML (reference):**

```yaml
name: Multi-Cloud Instance Refresh CI/CD
on:
  push:
    branches: [master]
permissions:
  id-token: write
  contents: read
jobs:
  refresh-aws:
    name: Refresh AWS Auto Scaling Group
    runs-on: ubuntu-latest
    steps:
      - name: Configure AWS Credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-east-1
      - name: Start AWS Instance Refresh
        run: |
          aws autoscaling start-instance-refresh \
            --auto-scaling-group-name mern-asg \
            --preferences '{"InstanceWarmup": 60, "MinHealthyPercentage": 50}'
  refresh-azure:
    name: Refresh Azure VMSS
    runs-on: ubuntu-latest
    steps:
      - name: Log into Azure via OIDC
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      - name: Upgrade Azure VM Scale Set
        run: |
          az vmss update-instances \
            --resource-group mern-app-rg \
            --name mern-vmss \
            --instance-ids "*"
```

> **📷 IMAGE PLACEHOLDER — CI/CD pipeline diagram**  
> *Insert image: GitHub Actions workflow — push to master → refresh-aws + refresh-azure (parallel), OIDC to AWS and Azure. Suggested filename: `img-04-cicd-pipeline.png`*

### 1.5 Secrets and configuration (single store for both clouds)

- **HashiCorp Vault** holds secrets used by Terraform and by EC2 user-data (e.g. clone repos, set `MONGO_URI`).
- **One KV v2 path** supports both clouds: no cloud-specific secret duplication in code.
- **Vault provider in Terraform:** Address and namespace are set in `providers.tf` (e.g. HCP Vault: `https://...hashicorp.cloud:8200`, namespace `admin`). Token via `VAULT_TOKEN` environment variable.

### 1.6 Multi-cloud benefits in this project

- **Vendor flexibility:** Run on AWS, Azure, or both from one codebase.
- **Resilience:** One cloud can fail or be deprecated without rewriting the app; each cloud is independent.
- **Consistency:** Same app, same base image build process, same deployment workflow.
- **Learning/reference:** Clear pattern for multi-cloud MERN deployments and DevOps.

---

## 2. DevSecOps Practices

### 2.1 Secrets management

**Principle:** No secrets in source code or long-lived credentials in the pipeline.

| Practice | Implementation |
|----------|----------------|
| **No secrets in code** | GitHub token and MongoDB URI are read from **HashiCorp Vault** KV v2 path `secret/sample-secret` (keys: `github_token`, `MONGO_URI`). |
| **Terraform + Vault** | `data.tf` declares `vault_kv_secret_v2`; Terraform injects these values into EC2 user-data at apply time. |
| **Backend .env** | Local dev uses `.env` (from `.env.example`); `.env` is not committed. Production instances get Vault-injected values in user-data. |

**Vault data source (`data.tf`):**

```hcl
data "vault_kv_secret_v2" "vault_secret" {
  mount = "secret"
  name  = "sample-secret"
}
```

**Usage in EC2 user-data (from `AWS_compute.tf`):**

- Clone private repos: `https://${data.vault_kv_secret_v2.vault_secret.data["github_token"]}@github.com/.../Back-end.git` (and Front-end).
- Backend `.env`: `echo "MONGO_URI=${...vault_secret.data["MONGO_URI"]...}" >> /var/www/mern-app/backend/.env`.

**Vault provider block (`providers.tf`):**

```hcl
provider "vault" {
  address          = "https://mern-multi-cloud-app-public-vault-b6e468d1.6721a3f6.z1.hashicorp.cloud:8200"
  namespace        = "admin"
  skip_child_token = true
  # VAULT_TOKEN env var used for auth
}
```

> **📷 IMAGE PLACEHOLDER — Secrets flow (Vault → Terraform → instances)**  
> *Insert image: Diagram showing Vault KV v2 → Terraform data source → user-data (clone + .env). Suggested filename: `img-05-vault-secrets-flow.png`*

### 2.2 Secure CI/CD (OIDC)

- **No long-lived cloud credentials** stored in the repository or as GitHub Actions secrets (no raw AWS access keys or Azure passwords).
- **AWS:** GitHub Actions uses **OIDC** with `aws-actions/configure-aws-credentials@v4` and `secrets.AWS_ROLE_ARN`. GitHub’s OIDC provider issues a short-lived token; AWS STS validates it and returns temporary credentials.
- **Azure:** `azure/login@v2` with `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID` (federated credential / OIDC).
- **Minimal permissions:** Workflow needs only `id-token: write` and `contents: read`; no broad repo or secret access.

### 2.3 Infrastructure security

**State protection**

- Terraform state stored in **S3** with **encryption** enabled.
- **DynamoDB** table `terraform-state-lock` prevents concurrent applies and state corruption.

**Network controls — AWS**

- **Security group:** `mern-web-sg` (VPC: `aws_vpc.main_vpc`).
- **Ingress:** HTTP 80, HTTPS 443, SSH 22 from `0.0.0.0/0`. Comments recommend restricting SSH to trusted IPs (e.g. GitHub Actions) in production.
- **Egress:** All traffic allowed (`0.0.0.0/0`, protocol `-1`).

**Network controls — Azure**

- **NSG:** `mern-nsg` in `mern-app-rg`.
- **Rules:** Allow HTTP (80), HTTPS (443), SSH (22); priority 100, 101, 110; source/destination `*`. Same recommendation to restrict SSH in production.

**Application layer**

- **CORS** enabled on the Express backend (`app.use(cors())`) so only allowed origins can call the API, reducing misuse from arbitrary domains.

### 2.4 DevSecOps summary table

| Area | What is done |
|------|----------------------|
| **Secrets** | Vault KV v2; no secrets in source or long-lived in CI |
| **CI/CD auth** | OIDC for AWS and Azure; no static keys |
| **State** | S3 + encryption + DynamoDB lock |
| **Network** | SG (AWS) and NSG (Azure) limit to HTTP/HTTPS/SSH; SSH restriction recommended for prod |
| **App** | CORS; env-based config (Vault-injected in prod) |

> **📷 IMAGE PLACEHOLDER — DevSecOps overview**  
> *Insert image: Diagram covering Vault, OIDC, encrypted state, network controls, CORS. Suggested filename: `img-06-devsecops-overview.png`*

---

## 3. High Availability (HA)

### 3.1 AWS High Availability

**Multiple Availability Zones**

- **VPC:** `main_vpc` (CIDR `10.0.0.0/16`), DNS hostnames enabled.
- **AZs:** Two availability zones in `us-east-1` via `data.aws_availability_zones.available` (names[0] and names[1]).
- **Subnets:**
  - `public_subnet`: CIDR `10.0.1.0/24`, AZ = `names[0]`, map_public_ip_on_launch = true.
  - `public_subnet_2`: CIDR `10.0.2.0/24`, AZ = `names[1]`, map_public_ip_on_launch = true.
- **ALB** is attached to **both** subnets, so it can survive the loss of one AZ.

**Load balancing**

- **ALB:** `mern-app-alb`, internet-facing, type `application`, in both subnets, security group `mern-web-sg`.
- **Target group:** `mern-app-tg`, port 80, HTTP, VPC `main_vpc`.
- **Health check (target group):**
  - Path: `/`
  - Healthy threshold: 2
  - Unhealthy threshold: 5
  - Timeout: 5 s
  - Interval: 30 s
  - Matcher: 200
- **Listener:** HTTP port 80, default action forward to `mern-app-tg`.

**Auto Scaling Group**

- **Name:** `mern-asg`.
- **Subnets:** `vpc_zone_identifier = [public_subnet.id, public_subnet_2.id]` — instances spread across **two AZs**.
- **Capacity:** min_size = 2, desired_capacity = 2, max_size = 2 → **always two instances**.
- **Target group:** `target_group_arns = [aws_lb_target_group.app_tg.arn]` — new instances register with the ALB.
- **Health check:** type = `ELB`, grace period = 300 s. ASG uses ALB target group health; unhealthy instances are replaced.
- **Launch template:** `app_lt` (custom AMI `mern-base-image-*`, t2.micro, user-data for clone, npm install, build, PM2, Nginx).

**Result:** Two EC2 instances in two AZs behind one ALB; loss of one instance or one AZ still leaves the app served by the other instance(s).

> **📷 IMAGE PLACEHOLDER — AWS HA architecture**  
> *Insert image: Diagram — Internet → ALB (in 2 AZs) → Target Group → 2 EC2 instances (one per AZ), with health checks. Suggested filename: `img-07-aws-ha-architecture.png`*

### 3.2 Azure High Availability

**VM Scale Set**

- **Name:** `mern-vmss`.
- **Resource group:** `mern-app-rg`.
- **Location:** East US (from resource group).
- **SKU:** `Standard_B2ms`; **instances: 2**.
- **Admin:** `adminuser`; SSH key from file (path in Terraform; for CI/CD should be parameterized or from secret).
- **Image:** Canonical Ubuntu 22.04 LTS (`0001-com-ubuntu-server-jammy`, 22_04-lts, latest).
- **OS disk:** Standard_LRS, ReadWrite caching.
- **Network:** NIC `vmss-nic`, NSG `mern-nsg`, subnet `subnet`, public IP per instance (`vmss-pip`) for management/Ansible.

**Load balancing**

- **Public IP:** `mern-lb-pip`, Static, Standard SKU (required for Standard LB with VMSS).
- **Load balancer:** `mern-app-lb`, Standard SKU, frontend `PublicIPAddress`.
- **Backend pool:** `BackEndAddressPool` attached to `mern-app-lb`.
- **Health probe:** `http-probe`, HTTP, port 80, request_path `/`. Only healthy instances receive traffic.
- **Rule:** `http-rule`, TCP frontend 80 → backend 80, backend pool and probe linked.

**Result:** Two VMs behind one Standard Load Balancer; health probe ensures traffic only to healthy instances.

> **📷 IMAGE PLACEHOLDER — Azure HA architecture**  
> *Insert image: Diagram — Internet → Public IP → Standard LB → Backend pool → 2 VMSS instances, with health probe. Suggested filename: `img-08-azure-ha-architecture.png`*

### 3.3 HA comparison (detailed)

| Aspect | AWS | Azure |
|--------|-----|--------|
| **Multi-AZ / spread** | 2 subnets in 2 AZs; ASG + ALB in both | Single region (East US); VMSS in one subnet (can add zones in Terraform) |
| **Instance count** | 2 (min = desired = max = 2) | 2 (`instances = 2`) |
| **Load balancer** | Application Load Balancer, public | Standard Load Balancer, static public IP |
| **Health check** | ALB target group: path `/`, interval 30 s, healthy 2, unhealthy 5, timeout 5 s, matcher 200 | LB probe: HTTP port 80, path `/` |
| **Auto-recovery** | ASG replaces unhealthy (ELB health); grace 300 s | LB stops forwarding to unhealthy; VMSS can add auto-repair/scale policies |
| **Instance type** | t2.micro (launch template) | Standard_B2ms |

### 3.4 Terraform outputs (access and verification)

After `terraform apply`, outputs expose:

- **AWS:** `aws_alb_dns_name` (URL for the app via ALB), `aws_public_ip` (one ASG instance IP).
- **Azure:** `azure_lb_public_ip`, `azure_resource_group`, `azure_vmss_name` (for CLI/scripts).

### 3.5 Gaps and recommendations for higher HA

- **AWS:** Consider increasing `max_size` (e.g. 4) and adding a scaling policy (e.g. CPU-based) for load and replacement.
- **Azure:** Consider **availability zones** for VMSS (e.g. `zone_balance = true`, zones `["1","2","3"]`) for AZ-level resilience.
- **Both:** Add a dedicated **health endpoint** (e.g. `GET /api/health`) that checks DB connectivity; point ALB/probe to it for more meaningful health checks.
- **Database:** Use MongoDB replica sets or Atlas multi-region for DB-level HA; app already uses `MONGO_URI` from env/Vault.

> **📷 IMAGE PLACEHOLDER — HA comparison (AWS vs Azure)**  
> *Insert image: Side-by-side or combined diagram of AWS 2-AZ + ALB + ASG vs Azure VMSS + LB. Suggested filename: `img-09-ha-comparison.png`*

---

## 4. Summary Tables

### 4.1 Multi-cloud

| Item | Detail |
|------|--------|
| **Clouds** | AWS (us-east-1), Azure (East US) |
| **IaC** | Single Terraform codebase; S3 state + DynamoDB lock |
| **Base images** | Packer: one config, two builders (AMI + Azure Managed Image); same Ansible playbook |
| **CI/CD** | One GitHub Actions workflow; two jobs (AWS refresh, Azure VMSS update) on push to master |
| **Secrets** | Single Vault path for both clouds |

### 4.2 DevSecOps

| Item | Detail |
|------|--------|
| **Secrets** | Vault KV v2; Terraform injects into user-data; no secrets in code |
| **CI/CD auth** | OIDC for AWS and Azure; no long-lived keys |
| **State** | S3 encrypted + DynamoDB lock |
| **Network** | SG (AWS) and NSG (Azure): HTTP, HTTPS, SSH; restrict SSH in prod |
| **App** | CORS; env-based config |

### 4.3 High availability

| Item | AWS | Azure |
|------|-----|--------|
| **Instances** | 2 (ASG) | 2 (VMSS) |
| **Spread** | 2 AZs | 1 region (can add zones) |
| **Load balancer** | ALB | Standard LB |
| **Health check** | ALB TG: `/`, 30 s | LB probe: `/`, HTTP 80 |
| **Recovery** | ASG + ELB health, 300 s grace | LB + probe; VMSS can add policies |

---

## 5. Conclusion

The Pizza App project implements **multi-cloud** by running the same MERN stack on **AWS** and **Azure** with a single Terraform codebase, shared Packer base images, and one CI/CD workflow that refreshes both environments. **DevSecOps** is addressed through **Vault** for secrets, **OIDC** in GitHub Actions for AWS and Azure, **encrypted and locked** Terraform state, and **network security** (with recommendations to harden SSH). **High availability** is achieved via **multiple instances**, **load balancers**, and **health checks** on both clouds, with **AWS** explicitly using **two Availability Zones** for AZ-level resilience. Together, these choices make the project a concrete reference for multi-cloud, DevSecOps, and HA in a single codebase.

---

## 6. Image Placeholders Index

Use this list to add or replace images in the report. Suggested filenames assume an `images/` folder in the report directory.

| # | Placeholder location | Suggested filename | Description |
|---|----------------------|--------------------|-------------|
| 1 | §1.1 Overview | `img-01-multicloud-architecture.png` | High-level multi-cloud architecture (AWS + Azure, Terraform, Packer, Vault, GitHub Actions) |
| 2 | §1.2 Terraform | `img-02-terraform-resources.png` | Terraform resource map per cloud |
| 3 | §1.3 Packer | `img-03-packer-flow.png` | Packer build flow: Ubuntu → AWS + Azure builders → Ansible → base images |
| 4 | §1.4 CI/CD | `img-04-cicd-pipeline.png` | GitHub Actions: push → refresh-aws + refresh-azure (OIDC) |
| 5 | §2.1 Secrets | `img-05-vault-secrets-flow.png` | Vault → Terraform → user-data (clone + .env) |
| 6 | §2.4 DevSecOps summary | `img-06-devsecops-overview.png` | DevSecOps overview (Vault, OIDC, state, network, CORS) |
| 7 | §3.1 AWS HA | `img-07-aws-ha-architecture.png` | AWS: ALB, 2 AZs, target group, 2 EC2 instances, health checks |
| 8 | §3.2 Azure HA | `img-08-azure-ha-architecture.png` | Azure: Public IP, LB, backend pool, 2 VMSS instances, probe |
| 9 | §3.5 HA recommendations | `img-09-ha-comparison.png` | Side-by-side AWS vs Azure HA comparison |

**Adding images in Markdown:**  
Replace each placeholder block with, for example:

```markdown
![Multi-cloud high-level architecture](images/img-01-multicloud-architecture.png)
```

Ensure the `images/` directory exists and place the image files there, or adjust paths to match your layout.

---

*End of Multi-Cloud, DevSecOps & High Availability Report — Detailed*
