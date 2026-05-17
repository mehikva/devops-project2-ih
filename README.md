# 🍔 Burger Builder — Production 3-Tier Azure Architecture

![Azure](https://img.shields.io/badge/Azure-Cloud-0078D4?logo=microsoftazure&logoColor=white)
![Terraform](https://img.shields.io/badge/Terraform-1.5+-7B42BC?logo=terraform&logoColor=white)
![Ansible](https://img.shields.io/badge/Ansible-2.14+-EE0000?logo=ansible&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-CI%2FCD-2088FF?logo=githubactions&logoColor=white)
![Java](https://img.shields.io/badge/Java-21-ED8B00?logo=openjdk&logoColor=white)
![React](https://img.shields.io/badge/React-19-61DAFB?logo=react&logoColor=black)

A full-stack web application deployed on a secure, scalable 3-tier Azure architecture — fully automated via **Terraform**, **Ansible**, and **GitHub Actions**.

---

## 📑 Table of Contents

- [Architecture Overview](#-architecture-overview)
- [Live URLs](#-live-urls)
- [Tech Stack](#-tech-stack)
- [Repository Structure](#-repository-structure)
- [Prerequisites](#-prerequisites)
- [Provisioning with Terraform](#-provisioning-with-terraform)
- [Configuration with Ansible](#-configuration-with-ansible)
- [Deploying with GitHub Actions](#-deploying-with-github-actions)
- [Validation & Testing](#-validation--testing)
- [Monitoring & Alerts](#-monitoring--alerts)
- [Security Model](#-security-model)
- [Network Layout](#-network-layout)
- [Demo Script](#-demo-script)

---

## 🏗️ Architecture Overview

```
Internet
   │
   ▼
Application Gateway (WAF v2)  ◄── Single public entry point
   │
   ├──► Frontend VM (Nginx + React)
   │
   └──► Backend VM (Java Spring Boot)
              │
              ▼
        Azure SQL Database
        (Private Endpoint only)
```

All resources live inside a **private VNet**. No VM has a public IP. SQL is only reachable via Private Endpoint. The Application Gateway is the sole public-facing entry point.

![Architecture Diagram](docs/architecture-diagram.png)

---

## 🌐 Live URLs

| Service        | URL                                          |
|----------------|----------------------------------------------|
| Frontend       | https://40.67.225.150.nip.io                         |
| API Health     | https://40.67.225.150.nip.io/api/health              |
| API Ingredients| https://40.67.225.150.nip.io/api/ingredients         |

---

## 📦 Tech Stack

| Layer          | Technology                                      |
|----------------|-------------------------------------------------|
| Frontend       | React 19, TypeScript, Vite                      |
| Backend        | Java 21, Spring Boot 3.2, Maven                 |
| Database       | Azure SQL Database                              |
| Infrastructure | Terraform + Azure RM Provider                   |
| Configuration  | Ansible                                         |
| CI/CD          | GitHub Actions                                  |
| Networking     | Azure VNet, NSGs, Private Endpoints             |
| Gateway        | Azure Application Gateway WAF v2                |
| Monitoring     | Azure App Insights + Log Analytics              |
| Secret Storage | GitHub Actions Secrets                          |

---

## 🗂️ Repository Structure

```
devops-project2-ih/
├── frontend/                           # React + TypeScript + Vite
├── backend/                            # Spring Boot Java REST API
├── infra/terraform/
│   ├── environments/dev/               # Dev environment entry point
│   │   ├── main.tf                     # Provider, backend, module calls
│   │   ├── variables.tf                # Variable declarations
│   │   └── terraform.tfvars            # Variable values (gitignored)
│   └── modules/
│       ├── network/                    # VNet, subnets, NSGs
│       ├── compute/                    # Frontend + backend VMs
│       ├── database/                   # Azure SQL + Private Endpoint
│       ├── appgateway/                 # App Gateway WAF v2
│       └── monitoring/                 # App Insights, Log Analytics, Alerts
├── config/ansible/
│   ├── inventories/dev/hosts.yml       # VM inventory
│   ├── roles/common/                   # Base VM setup
│   ├── roles/frontend/                 # Nginx + React deployment
│   ├── roles/backend/                  # Java + Spring Boot deployment
│   └── site.yml                        # Main playbook
├── .github/workflows/
│   ├── infra.yml                       # Terraform pipeline
│   ├── frontend.yml                    # Frontend CI/CD
│   └── backend.yml                     # Backend CI/CD
├── docs/
│   ├── architecture-diagram.png
│   └── runbook.md
└── README.md
```

---

## 📋 Prerequisites

Before you begin, make sure the following are installed and configured:

- **Azure CLI** — `az login` working
- **Terraform** >= 1.5
- **Ansible** >= 2.14
- **Node.js** 20+
- **Java** 21 + Maven
- **Git** + GitHub account
- **Azure subscription** with Contributor access

---

## 🔧 Provisioning with Terraform

### 1. Create Remote State Storage

```bash
az group create --name rg-tfstate --location northeurope

az storage account create \
  --name tfstatefidail \
  --resource-group rg-tfstate \
  --sku Standard_LRS \
  --location northeurope

az storage container create \
  --name tfstate \
  --account-name tfstatefidail \
  --auth-mode login
```

### 2. Generate SSH Key Pair

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/burgerbuilder -N ""
```

### 3. Configure Variables

```bash
cp infra/terraform/environments/dev/terraform.tfvars.example \
   infra/terraform/environments/dev/terraform.tfvars
```

Edit `terraform.tfvars` with your values:

```hcl
resource_group_name = "rg-burgerbuilder-dev"
location            = "northeurope"
admin_username      = "azureuser"
ssh_public_key      = "ssh-rsa AAAA..."
vm_size             = "Standard_D2ads_v7"
sql_server_name     = "sql-burgerbuilder-yourname"
sql_database_name   = "burgerbuilderdb"
sql_admin_username  = "sqladmin"
sql_admin_password  = "YourPassword123!"
alert_email         = "your@email.com"
```

### 4. Apply Infrastructure

```bash
cd infra/terraform/environments/dev
terraform init
terraform plan
terraform apply
```

This creates: VNet, 5 subnets, NSGs, 2 VMs, Azure SQL, Private Endpoint, Private DNS, App Gateway (WAF v2), App Insights, Log Analytics, and 3 alert rules.

### 5. Terraform Workspaces (Multi-Environment)

```bash
terraform workspace new prod
terraform workspace select prod
terraform apply -var-file="../../environments/prod/terraform.tfvars"
```

---

## ⚙️ Configuration with Ansible

### 1. Update Inventory

Edit `config/ansible/inventories/dev/hosts.yml` with your VM IPs:

```yaml
all:
  children:
    frontend:
      hosts:
        vm-frontend:
          ansible_host: <frontend-private-ip>
          ansible_user: azureuser
          ansible_ssh_private_key_file: ~/.ssh/burgerbuilder
    backend:
      hosts:
        vm-backend:
          ansible_host: <backend-private-ip>
          ansible_user: azureuser
          ansible_ssh_private_key_file: ~/.ssh/burgerbuilder
```

### 2. Run Playbook

```bash
# Configure all VMs
ansible-playbook config/ansible/site.yml \
  -i config/ansible/inventories/dev/hosts.yml \
  --private-key ~/.ssh/burgerbuilder

# Configure only frontend
ansible-playbook config/ansible/site.yml \
  -i config/ansible/inventories/dev/hosts.yml \
  --limit frontend \
  --private-key ~/.ssh/burgerbuilder \
  --extra-vars "frontend_build_path=frontend/dist/"

# Configure only backend
ansible-playbook config/ansible/site.yml \
  -i config/ansible/inventories/dev/hosts.yml \
  --limit backend \
  --private-key ~/.ssh/burgerbuilder \
  --extra-vars "backend_jar_path=backend/target/burger-builder-backend-1.0.0.jar"
```

---

## 🚀 Deploying with GitHub Actions

### 1. Required GitHub Secrets

Go to: **GitHub repo → Settings → Secrets and variables → Actions**

| Secret                | Description                              |
|-----------------------|------------------------------------------|
| `ARM_CLIENT_ID`       | Azure Service Principal Client ID        |
| `ARM_CLIENT_SECRET`   | Azure Service Principal Secret           |
| `ARM_TENANT_ID`       | Azure Tenant ID                          |
| `ARM_SUBSCRIPTION_ID` | Azure Subscription ID                    |
| `AZURE_CREDENTIALS`   | Full JSON credentials block              |
| `SSH_PRIVATE_KEY`     | Contents of `~/.ssh/burgerbuilder`       |
| `SSH_PUBLIC_KEY`      | Contents of `~/.ssh/burgerbuilder.pub`   |
| `SQL_ADMIN_PASSWORD`  | SQL Server admin password                |
| `APPGW_PUBLIC_IP`     | App Gateway public IP address            |

### 2. Create Service Principal

```bash
az ad sp create-for-rbac \
  --name "sp-burgerbuilder-github" \
  --role contributor \
  --scopes /subscriptions/<subscription-id>/resourceGroups/<rg-name>
```

### 3. Trigger Pipelines

Pipelines trigger automatically on push to `main` for their respective paths, or you can run them manually:

| Pipeline                  | Path in GitHub Actions                              |
|---------------------------|-----------------------------------------------------|
| Infrastructure (Terraform)| Actions → Infrastructure → Run workflow             |
| Frontend (Build + Deploy) | Actions → Frontend CI/CD → Run workflow             |
| Backend (Build + Deploy)  | Actions → Backend CI/CD → Run workflow              |

---

## ✅ Validation & Testing

### Basic Health Check

```bash
curl http://<appgw-ip>/
curl http://<appgw-ip>/api/health
curl http://<appgw-ip>/api/ingredients
```

### End-to-End SQL Flow

```bash
# 1. Add item to cart
curl -X POST http://<appgw-ip>/api/cart/items \
  -H "Content-Type: application/json" \
  -d '{"ingredientId":1,"quantity":1,"sessionId":"test123"}'

# 2. Create order (use cart item ID from above response)
curl -X POST http://<appgw-ip>/api/orders \
  -H "Content-Type: application/json" \
  -d '{"sessionId":"test123","cartItemIds":[1],"totalPrice":1.50,"customerName":"Test User"}'

# 3. Read order back from SQL
curl http://<appgw-ip>/api/orders/history
```

### Verify Security

```bash
# VMs should have NO public IPs
az vm list-ip-addresses \
  --resource-group <rg-name> \
  --query "[].{vm:virtualMachine.name, publicIP:virtualMachine.network.publicIpAddresses[0].ipAddress}" \
  --output table

# SQL public access should be disabled
az sql server show \
  --name <sql-server-name> \
  --resource-group <rg-name> \
  --query publicNetworkAccess
```

### Sample Log Analytics (Kusto) Queries

```kusto
// App Gateway requests — last 1 hour
AzureDiagnostics
| where ResourceType == "APPLICATIONGATEWAYS"
| where TimeGenerated > ago(1h)
| summarize count() by httpStatus_d

// Backend exceptions — last 1 hour
AppExceptions
| where TimeGenerated > ago(1h)
| order by TimeGenerated desc
```

---

## 📊 Monitoring & Alerts

### Alert Rules

| Alert                        | Trigger Condition              | Severity     |
|------------------------------|-------------------------------|--------------|
| App Gateway Backend Health   | UnhealthyHostCount > 0 (5 min) | 2 — Warning  |
| VM CPU High                  | CPU > 70% (5 min)              | 2 — Warning  |
| SQL DTU High                 | DTU > 80% (5 min)              | 2 — Warning  |

All alerts send email notifications to the configured `alert_email`.

### Application Insights

- **Frontend** instrumented via `appi-frontend`
- **Backend** instrumented via `appi-backend`
- Both connected to a shared **Log Analytics workspace**

---

## 🔐 Security Model

| Control                  | Detail                                                             |
|--------------------------|--------------------------------------------------------------------|
| VM Public IPs            | None — VMs are only reachable from the App Gateway or ops subnet   |
| SQL Access               | Public network access disabled — Private Endpoint only             |
| NSGs                     | Restrict inbound traffic per tier                                  |
| SSH Access               | Locked to ops subnet (`snet-ops`) only                            |
| WAF                      | App Gateway WAF v2 in Detection mode                               |
| Secrets                  | Stored in GitHub Actions secrets — never in code                   |

---

## 🗺️ Network Layout

| Subnet         | CIDR          | Purpose                    |
|----------------|---------------|----------------------------|
| `snet-appgw`   | 10.0.0.0/24   | Application Gateway        |
| `snet-frontend`| 10.0.1.0/24   | Frontend VM                |
| `snet-backend` | 10.0.2.0/24   | Backend VM                 |
| `snet-data`    | 10.0.3.0/24   | SQL Private Endpoint       |
| `snet-ops`     | 10.0.4.0/24   | Bastion / GitHub Runner    |

---

## 🎬 Demo Script (3–5 min walkthrough)

1. **Architecture diagram** — explain 3 tiers, private networking, and the single public entry point
2. **Open browser** at `http://40.67.225.150` — show Burger Builder UI loading
3. **Ingredients loading** — prove frontend → App Gateway → backend → SQL flow end-to-end
4. **curl commands** — create cart item, create order, read order history from SQL
5. **GitHub Actions** — show 3 green pipelines (infra, frontend, backend)
6. **Azure Portal** — show resource group with all expected services
7. **Monitoring** — App Insights, Log Analytics workspace, 3 alert rules
8. **Terraform state** — remote state stored in Azure Storage
9. **NSGs** — confirm no public access to VMs
10. **SQL** — confirm public network access is disabled

---

## 🤝 Contributing

Pull requests are welcome. For major changes, please open an issue first to discuss what you would like to change.

---

## 📄 License

This project is for educational purposes. Forked from [saurabhd2106/devops-project2-ih](https://github.com/saurabhd2106/devops-project2-ih).
