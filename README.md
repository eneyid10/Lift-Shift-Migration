# Lift-And-Shift-Migration
AWS to Azure Lift &amp; Shift Migration

# ☁️ Cloud-to-Cloud Migration Lab: AWS EC2 → Azure via Azure Migrate

![Lab Status](https://img.shields.io/badge/Status-Complete-brightgreen)
![Difficulty](https://img.shields.io/badge/Difficulty-Intermediate-yellow)
![Duration](https://img.shields.io/badge/Duration-4--6%20Hours-blue)
![Tools](https://img.shields.io/badge/Tools-Terraform%20%7C%20AWS%20CLI%20%7C%20Azure%20CLI-informational)

---

## 📋 Overview

This lab demonstrates an **end-to-end cloud-to-cloud migration** of a Windows Server 2022 EC2 instance from AWS into Azure using **Azure Migrate** — Microsoft's native migration orchestration platform. The lab covers all four phases of the migration lifecycle: discovery, assessment, replication, and cutover.

Cloud migrations are one of the highest-value workloads in cloud engineering. Organizations move workloads across clouds for cost optimization, compliance consolidation, or post-acquisition integration. This lab replicates that real-world process using production-grade tooling and infrastructure-as-code.

> **Interview-ready summary:** *"I performed an end-to-end cloud migration from AWS to Azure using Azure Migrate — covering discovery, assessment, agentless replication, and cutover — and I can walk through every step and explain the decisions made at each stage."*

---

## 🏗️ Architecture

### High-Level Migration Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           MIGRATION ARCHITECTURE                            │
└─────────────────────────────────────────────────────────────────────────────┘

  AWS ACCOUNT                              AZURE SUBSCRIPTION
  ─────────────────────────────────        ──────────────────────────────────────
  ┌───────────────────────────────┐        ┌──────────────────────────────────────┐
  │  VPC: 10.0.0.0/16             │        │  rg-migrate-source-[yourname]        │
  │  ┌─────────────────────────┐  │        │  ┌────────────────────────────────┐  │
  │  │  Subnet: 10.0.1.0/24    │  │        │  │  VNet: vnet-migrate            │  │
  │  │  ┌─────────────────┐    │  │        │  │  (10.1.0.0/16)                 │  │
  │  │  │  EC2 Instance   │    │  │        │  │  ┌──────────────────────────┐  │  │
  │  │  │  Windows Server │◄── ┼─ ┼────┐   │  │  │  Subnet: snet-migrate    │  │  │
  │  │  │  2022 (source)  │    │  │    │   │  │  │  (10.1.1.0/24)           │  │  │
  │  │  │  t3.medium      │    │  │    │   │  │  │                          │  │  │
  │  │  │  30GB gp3 disk  │    │  │    │   │  │  │  ┌────────────────────┐  │  │  │
  │  │  └─────────────────┘    │  │    │   │  │  │  │ Discovery Appliance│  │  │  │
  │  │                         │  │    │   │  │  │  │ vm-mig-appl-[name] │  │  │  │
  │  │  Security Group:        │  │    │   │  │  │  │ Standard_A4_v2     │  │  │  │
  │  │  • Port 443 (HTTPS)     │  │    │   │  │  │  │ 4 vCPU / 8GB RAM   │  │  │  │
  │  │  • Port 3389 (RDP)      │  │    │   │  │  │  └────────────────────┘  │  │  │
  │  └─────────────────────────┘  │    │   │  │  │                          │  │  │
  │                               │    │   │  │  │  ┌────────────────────┐  │  │  │
  │  IAM Service Account          │    └───┼──┼──┼──►│ Replication Appl. │  │  │  │
  │  svc-azure-migrate-[name]     │        │  │  │  │ vm-mig-repl-[name] │  │  │  │
  │  Permissions:                 │        │  │  │  │ Standard_A4_v2     │  │  │  │
  │  • ec2:Describe*              │        │  │  │  └────────────────────┘  │  │  │
  │  • ec2:CreateSnapshot         │        │  │  └──────────────────────────┘  │  │
  │  • ec2:DeleteSnapshot         │        │  │                                │  │
  └───────────────────────────────┘        │  │  Storage Account (cache)       │  │
                                           │  │  stmigrate[yourname]           │  │
                                           │  │  Standard LRS / StorageV2      │  │
                                           │  │                                │  │
                                           │  │  Recovery Services Vault       │  │
                                           │  │  rsv-migrate-[yourname]        │  │
                                           │  │                                │  │
                                           │  │  Log Analytics Workspace       │  │
                                           │  │  law-migrate-[yourname]        │  │
                                           │  └────────────────────────────────┘  │
                                           │                                      │
                                           │  rg-migrate-target-[yourname]        │
                                           │  ┌─────────────────────────────────┐ │
                                           │  │  Azure Migrate Project          │ │
                                           │  │  migrate-project-[yourname]     │ │
                                           │  │                                 │ │
                                           │  │  ┌─────────────────────────┐    │ │
                                           │  │  │  Migrated VM (Target)   │    │ │
                                           │  │  │  Windows Server 2022    │    │ │
                                           │  │  │  Standard_B2s           │    │ │
                                           │  │  │  (post-cutover)         │    │ │
                                           │  │  └─────────────────────────┘    │ │
                                           │  └─────────────────────────────────┘ │
                                           └──────────────────────────────────────┘
```

---

### Migration Phase Flow

```
┌───────────────────────────────────────────────────────────────────────────────┐
│                          MIGRATION PHASE SEQUENCE                             │
└───────────────────────────────────────────────────────────────────────────────┘

  PART 1           PART 2           PART 3            PART 4         PARTS 5–6
  AWS Source       Azure Target     Appliance         Assessment     Replication
  Infra            Infra            Setup             & Readiness    & Cutover
  ─────────        ────────         ────────          ──────────     ─────────────

  ┌──────────┐    ┌──────────┐    ┌──────────┐      ┌──────────┐   ┌──────────┐
  │ Terraform│    │ Terraform│    │  Portal  │      │  Portal  │   │  Portal  │
  │  apply   │───►│  apply   │───►│  Deploy  │─────►│ Assess   │──►│ Replicate│
  └──────────┘    └──────────┘    │ Register │      │ VM       │   │ Monitor  │
       │               │          │ Configure│      │ Readiness│   └────┬─────┘
       ▼               ▼          └──────────┘      └──────────┘        │
  ┌──────────┐    ┌──────────┐         │                                 ▼
  │ EC2 Win  │    │ VNet     │         │ AWS Credentials            ┌──────────┐
  │ Server   │    │ RG x2    │         │ Discovery                  │ Test     │
  │ IAM User │    │ RSV      │         │ Source: EC2                │ Migration│
  │ SG Rules │    │ Storage  │         │ public IP                  │ Verify   │
  │ Key Pair │    │ Appliance│         │                            │ Cutover  │
  └──────────┘    │ VM       │         ▼                            └────┬─────┘
                  └──────────┘   ┌─────────--─┐                           │
                                 │Discovery   │                            ▼
                                 │5–15 min    │                      ┌──────────┐
                                 │EC2 appears │                      │ Migrated │
                                 │in Migrate  │                      │ VM live  │
                                 │project     │                      │ in Azure │
                                 └─────────--─┘                      └──────────┘
```

---

### Key Concept: What "Agentless" Actually Means

```
  ┌─────────────────────────────────────────────────────────────────────────────┐
  │  AGENTLESS = Nothing installed on the SOURCE EC2 instance                   │
  │                                                                             │
  │  However, AWS migrations still require TWO appliance VMs in Azure:          │
  │                                                                             │
  │  ┌─────────────────────────┐    ┌─────────────────────────────────────┐     │
  │  │  Discovery Appliance    │    │  Replication Appliance              │     │
  │  │  (Assessment phase)     │    │  (Configuration Server - ASR)       │     │
  │  │                         │    │                                     │     │
  │  │  • Reads EC2 metadata   │    │  • Handles disk-level replication   │     │
  │  │  • Calls AWS APIs       │    │  • Uses Azure Site Recovery (ASR)   │     │
  │  │  • Sends inventory to   │    │  • Required for ALL AWS migrations  │     │
  │  │    Migrate project      │    │  • Even without source agents       │     │
  │  └─────────────────────────┘    └─────────────────────────────────────┘     │
  │                                                                             │
  │  Note: VMware agentless migrations do NOT require a replication appliance.  │
  │  AWS migrations always do — regardless of whether agents are on the source. │
  └─────────────────────────────────────────────────────────────────────────────┘
```

---

## 🧰 Tech Stack

| Layer | Tool / Service |
|---|---|
| Infrastructure as Code | Terraform (AWS + Azure providers) |
| Source Cloud | AWS EC2 (Windows Server 2022, t3.medium) |
| Target Cloud | Microsoft Azure |
| Migration Orchestration | Azure Migrate |
| Replication Engine | Azure Site Recovery (under the hood) |
| Identity (AWS) | IAM User + Access Key (least-privilege) |
| Networking (AWS) | VPC, Subnet, IGW, Route Table, Security Group |
| Networking (Azure) | VNet, Subnet, NSG |
| Replication Buffer | Azure Storage Account (Standard LRS) |
| Replication State | Recovery Services Vault |
| Discovery Telemetry | Log Analytics Workspace |
| CLI Tools | AWS CLI, Azure CLI |

---

## 💰 Estimated Lab Cost

> ⚠️ **Destroy all resources immediately after completing the lab to stop charges.**

| Resource | Estimated Cost |
|---|---|
| EC2 t3.medium (Windows) | ~$0.08/hr (~$0.50 for 6 hrs) |
| Azure Migrate appliance VM (Standard_A4_v2) | ~$0.40/hr |
| Replication appliance VM (Standard_A4_v2) | ~$0.40/hr |
| Azure Storage (replication cache, ~30GB) | ~$0.60/day |
| Azure target VM (Standard_B2s, post-cutover) | ~$0.05/hr |
| **Total for a full-day lab** | **~$5–8** |

---

## 📁 Repository Structure

```
aws-to-azure-migrate/
├── aws-side/
│   ├── main.tf              # VPC, EC2, SG, IAM role, IAM user/key
│   ├── variables.tf         # Region, name, AMI, instance type, password
│   ├── outputs.tf           # EC2 IDs, IPs, access keys
│   └── terraform.tfvars     # Your variable values (gitignored)
│
└── azure-side/
    ├── main.tf              # RGs, VNet, RSV, Storage, Appliance VMs, NSGs
    ├── variables.tf         # Location, name, passwords
    ├── outputs.tf           # Resource names, subnet IDs, vault name
    └── terraform.tfvars     # Your variable values (gitignored)
```

> **Security reminder:** `terraform.tfvars` contains passwords and access keys. Add it to `.gitignore` before pushing to any repository.

```gitignore
# .gitignore
*.tfvars
*.tfvars.json
.terraform/
.terraform.lock.hcl
terraform.tfstate
terraform.tfstate.backup
```

---

## ✅ Prerequisites

### Accounts Required
- AWS account with programmatic access enabled
- Azure subscription (free tier works for the appliance; paid required for the Windows VM)

### IAM Setup (AWS)
Create an IAM user named `terraform-migrate-lab` with:
- `AmazonEC2FullAccess`
- `AmazonVPCFullAccess`

Save the Access Key ID and Secret Access Key — you will configure these in `aws configure`.

### CLI Tools

**macOS:**
```bash
brew tap hashicorp/tap && brew install hashicorp/tap/terraform
brew install awscli
brew install azure-cli
```

**Windows (PowerShell):**
- Download [AWS CLI](https://aws.amazon.com/cli/)
- Download [Azure CLI](https://aka.ms/installazurecliwindows)
- Download [Terraform](https://developer.hashicorp.com/terraform/downloads)

### Configure CLIs
```bash
# AWS
aws configure
# Enter: Access Key ID, Secret Access Key, us-east-1, json

# Azure
az login
az account set --subscription "Azure subscription 1"

# Verify both
aws sts get-caller-identity
az account show
```

---

## 🚀 Lab Walkthrough

### Part 1 — Deploy AWS Source Environment

**What this builds:** VPC, public subnet, internet gateway, route table, security group, Windows Server 2022 EC2 instance, IAM role + policy for Azure Migrate access, and a dedicated IAM service account with access keys.

```bash
cd aws-to-azure-migrate/aws-side
terraform init
terraform plan
terraform apply
```

Expected: **10 resources added** in ~3–5 minutes. Wait an additional 5 minutes after apply for Windows to finish initializing before connecting via RDP.

**Capture these outputs — needed in Part 3:**
```bash
terraform output ec2_public_ip
terraform output ec2_instance_id
terraform output migrate_access_key_id
terraform output -raw migrate_secret_access_key
```

**Verify:** RDP to the EC2 public IP using `Administrator` and your configured password. A successful Windows Server desktop confirms the source is ready.

---

### Part 2 — Deploy Azure Target Environment

**What this builds:** Two resource groups (source/staging and target), VNet with subnet (10.1.0.0/16 — intentionally different from AWS to avoid CIDR overlap), Storage Account for replication cache, Recovery Services Vault, Log Analytics Workspace, and target NSG.

```bash
cd aws-to-azure-migrate/azure-side
terraform init
terraform plan
terraform apply
```

Expected: **9 resources added** in ~3–4 minutes.

> **Note:** The Azure Migrate project itself cannot be created via Terraform — the `azurerm` provider does not support `azurerm_migrate_project`. You will create it manually in the portal in Part 3.

---

### Part 3 — Deploy & Configure the Azure Migrate Appliance

**This part is done in the Azure Portal** — the appliance requires interactive registration that cannot be automated via Terraform.

#### 3A — Create the Migrate Project (Portal)

1. Search **Azure Migrate** → **Servers, databases and web apps**
2. Click **Discover** → Select: *Yes, with another cloud provider (AWS, GCP, etc.)*
3. Target: **Azure VM** → Select your project name: `migrate-project-[yourname]`
4. Name your appliance: `appliance-migrate-[yourname]`
5. Click **Generate key** — **copy and save this key immediately**
6. Download the appliance installer (~1.5GB .zip)

#### 3B — Deploy the Appliance VM via Terraform

Add the appliance VM resources to `azure-side/main.tf` (see lab document for full HCL) and re-run:

```bash
terraform plan
terraform apply
```

VM spec: `Standard_A4_v2` (4 vCPU / 8GB RAM — minimum required by the appliance).

#### 3C — Install & Register the Appliance

1. RDP into the appliance VM using the public IP from `terraform output appliance_public_ip`
2. Extract the downloaded installer zip → run `AzureMigrateInstaller.ps1` as Administrator
3. Answer the installer prompts:

| Prompt | Answer |
|---|---|
| Execution Policy Change | Y |
| Select scenario | 3 (Physical or other — AWS, GCP, Xen) |
| Select cloud | 1 (Azure Public) |
| Select connectivity | 1 (Public endpoint) |
| Continue with deployment | Y |

4. In the configuration manager that opens: paste your **project key**, sign in to Azure, confirm **Successfully registered**

#### 3D — Add Credentials and Start Discovery

In the appliance configuration manager, add two credential entries:

| Entry | Friendly Name | Username | Password |
|---|---|---|---|
| AWS IAM service account | `awsmigratesvc` | Access Key ID (from `terraform output`) | Secret Access Key (from `terraform output -raw`) |
| Windows admin (WinRM) | `ec2winadmin` | `Administrator` | Your EC2 admin password |

Add the EC2 **public IP** as the discovery source (not private IP — no VPN exists between clouds). Click **Validate** → wait for *Validation successful* → click **Start discovery**.

Discovery takes **5–15 minutes**.

---

### Part 4 — Assess the EC2 Instance

1. Azure Migrate → **Assess** → Assessment type: **Azure VM**
2. Assessment settings:
   - Target location: East US
   - Storage type: Automatic
   - Sizing criteria: Performance-based
3. Create a new group: `aws-ec2-group` → add your EC2 instance → **Create assessment**

Assessment takes **2–5 minutes**.

**What to look for in results:**

| Field | Expected Value |
|---|---|
| Azure readiness | ✅ Ready for Azure |
| Recommended VM size | Standard_B2s (for lightly loaded t3.medium) |
| Monthly cost estimate | Varies by region |

> **What "Ready for Azure" means:** Azure Migrate validated the OS version, boot type, disk count, and network config and found no migration blockers. This is the highest-value output of the assessment phase in real enterprise migrations.

---

### Part 5 — Deploy the Replication Appliance and Replicate

The replication appliance is a **separate VM from the discovery appliance** — it handles continuous disk-level replication using Azure Site Recovery.

#### 5A — Deploy Replication Appliance VM (Terraform)

Add the replication VM resources to `azure-side/main.tf` (see lab document for full HCL) — **must use Windows Server 2022**; 2019 fails during installer execution.

```bash
terraform plan
terraform apply
```

#### 5B — Install the Replication Appliance

1. RDP into the replication VM (`terraform output replication_appliance_public_ip`)
2. In the portal (inside RDP session): Azure Migrate → **Migration and modernization** → **Discover** → create resources
3. Download and run `DRInstaller.ps1` as Administrator
4. Register with your Recovery Services Vault (`rsv-migrate-[yourname]`)

#### 5C — Enable Replication

1. Azure Migrate → **Migration and modernization** → **Replicate**
2. Source: AWS, On-premises appliance: your registered appliance
3. Select your EC2 instance
4. Target settings:
   - Resource group: `rg-migrate-target-[yourname]`
   - Replication storage: `stmigrate[yourname]`
   - VNet: `vnet-migrate-[yourname]` / Subnet: `snet-migrate`
5. Set OS type: **Windows** → click **Replicate**

Initial replication (full disk copy) takes **20–60 minutes** for a 30GB Windows disk.

#### 5D — Monitor Replication

Azure Migrate → **Replicating machines** → watch status progress:

```
Initial replication in progress  →  [20–60 min]  →  Protected ✅
```

Wait for **Protected** status before proceeding to cutover.

---

### Part 6 — Test Migration and Cutover

#### 6A — Test Migration (Recommended)

A test migration spins up a temporary VM from the replicated disk in an isolated network. Always run this before committing to cutover in production.

1. Replicating machines → click your instance → **Test migration**
2. Select `vnet-migrate-[yourname]` for the test network → **Test migration**
3. RDP into the test VM (appears in target resource group in ~5–10 minutes)
4. Verify: Windows desktop loads, hostname and OS version match the source
5. Return to Azure Migrate → **Clean up test migration**

#### 6B — Cutover

1. Replicating machines → click your instance → **Migrate**
2. Shut down machines: **No** (acceptable for lab; use Yes in production)
3. Click **Migrate** → Azure finalizes replication and creates the target VM (~5–10 minutes)

#### 6C — Verify the Migrated VM

```bash
# macOS
az network public-ip create \
  --resource-group rg-migrate-target-charles \
  --name pip-migrated-vm \
  --sku Standard

az network public-ip show \
  --resource-group rg-migrate-target-charles \
  --name pip-migrated-vm \
  --query ipAddress -o tsv
```

```powershell
# Windows PowerShell
az network public-ip create `
  --resource-group rg-migrate-target-charles `
  --name pip-migrated-vm `
  --sku Standard

az network public-ip show `
  --resource-group rg-migrate-target-charles `
  --name pip-migrated-vm `
  --query ipAddress -o tsv
```

RDP using `Administrator` and your original AWS admin password. Confirm hostname and OS version match the source EC2 instance.

---

## ✅ Verification Checklist

```
[ ] EC2 instance visible and running in AWS console
[ ] Azure Migrate project created in portal with appliance registered
[ ] EC2 instance appears in discovered machines list
[ ] Assessment shows "Ready for Azure" status
[ ] Replication completed — status shows "Protected"
[ ] Test migration succeeded and cleaned up
[ ] Cutover triggered — migrated VM appears in target resource group
[ ] RDP to migrated VM succeeds using original AWS credentials
[ ] Migrated VM hostname and OS version match the source EC2 instance
```

---

## 🔧 Troubleshooting

| Error | Cause | Resolution |
|---|---|---|
| EC2 instance not discovered | AWS credentials entered incorrectly in appliance | Re-enter the access key and secret in the appliance configuration manager |
| Discovery shows 0 machines | Region mismatch | Verify the region in the appliance matches where the EC2 instance runs |
| Replication stuck at 0% | Storage account not accessible | Verify `stmigrate[yourname]` is in the same subscription and region as the Migrate project |
| Assessment shows "Not ready" | Unsupported Windows version | Must be Windows Server 2012 R2 or later — 2022 should always pass |
| RDP to migrated VM fails | NSG not attached to migrated VM's NIC | Attach `nsg-migrate-target-[yourname]` to the VM's NIC in the portal |
| Cutover VM has wrong IP | DHCP assigns a new Azure IP | Expected behavior — IP changes when moving between clouds. Update any DNS records. |
| `terraform error: invalid value for name (cannot begin with sg-)` | AWS reserves the `sg-` prefix for system IDs | Rename the SG from `sg-migrate-source-` to `migrate-source-sg-` |
| `terraform error: collecting instance settings: empty result` | AMI ID invalid for selected region | Run: `aws ec2 describe-images --region us-east-1 --owners amazon --filters "Name=name,Values=Windows_Server-2022-English-Full-Base-*" "Name=state,Values=available" --query "sort_by(Images, &CreationDate)[-1].ImageId" --output text` |
| `provider does not support resource type azurerm_migrate_project` | AzureRM provider doesn't support Migrate projects | Replace with `null_resource` + create the project manually in portal. Add `null` provider to `required_providers` and re-run `terraform init`. |
| RSV `terraform destroy` fails with "vault is not empty" | Replication items still registered | Portal → RSV → delete all Backup items and Replication items manually, then retry `terraform destroy` |

---

## 🧹 Teardown

Destroy in this exact order to avoid dependency errors.

### Step 1 — Stop Replication (if active)
Azure Migrate → **Replicating machines** → select machine → **Stop replication** → wait for confirmation.

### Step 2 — Destroy AWS Resources
```bash
# macOS
cd ~/aws-to-azure-migrate/aws-side
terraform destroy

# Windows PowerShell
cd "$HOME\aws-to-azure-migrate\aws-side"
terraform destroy
```

### Step 3 — Destroy Azure Resources
```bash
# macOS
cd ~/aws-to-azure-migrate/azure-side
terraform destroy

# Windows PowerShell
cd "$HOME\aws-to-azure-migrate\azure-side"
terraform destroy
```

### Step 4 — Delete Target Resource Group (migrated VM not tracked by Terraform)
```bash
# macOS
az group delete --name rg-migrate-target-charles --yes

# Windows PowerShell
az group delete --name rg-migrate-target-charles --yes
```

> If `terraform destroy` fails on the Recovery Services Vault, go to the portal → RSV → manually delete all **Backup items** and **Replication items**, then retry.

---

## 🎓 Skills Demonstrated

| Domain | Skills |
|---|---|
| Cloud Architecture | Cross-cloud networking, CIDR design, IAM least-privilege |
| Infrastructure as Code | Terraform (multi-root), AWS + AzureRM providers, sensitive outputs, null_resource patterns |
| AWS | EC2, VPC, Subnets, IGW, Route Tables, Security Groups, IAM Roles, IAM Policies, Access Keys |
| Azure | Azure Migrate, Azure Site Recovery, Recovery Services Vault, Storage Accounts, VNet, NSG, Log Analytics |
| Migration Methodology | Discovery → Assessment → Replication → Test → Cutover lifecycle |
| Security | IAM least-privilege scoping, secrets handling in Terraform, NSG rules |
| Troubleshooting | AMI region validation, provider limitations, vault teardown sequencing |

---

## 🔗 References

- [Azure Migrate Documentation](https://learn.microsoft.com/en-us/azure/migrate/)
- [Azure Site Recovery — Physical/AWS Server Replication](https://learn.microsoft.com/en-us/azure/site-recovery/physical-azure-disaster-recovery)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [Terraform AzureRM Provider](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs)
- [AWS EC2 Windows AMI Locator](https://docs.aws.amazon.com/AWSEC2/latest/WindowsGuide/finding-an-ami.html)

---

## 👤 Author

**Eneyi** — Cloud Engineer | AWS Solutions Architect | IaC
GitHub: [@eneyid10](https://github.com/eneyid10)

