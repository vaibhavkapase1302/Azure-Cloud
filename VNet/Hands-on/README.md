# Azure VNet and VM Setup Guide with Bastion

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Architecture Overview](#architecture-overview)
3. [Setup Instructions](#setup-instructions)
4. [VNet Components Documentation](#vnet-components-documentation)
5. [Connecting to VMs](#connecting-to-vms)
6. [Cost Management](#cost-management)
7. [AWS to Azure Comparison](#aws-to-azure-comparison)
8. [Troubleshooting](#troubleshooting)

---

## Prerequisites

### Requirements
- Azure CLI installed (`az --version` to verify)
- Active Azure subscription
- Appropriate permissions to create resources
- Terminal/PowerShell access

### Login to Azure
```bash
# Login to Azure
az login

# Verify your subscription
az account show

# Set specific subscription if needed
az account set --subscription "fscloud-sbx-001"
```

---

## Architecture Overview

### What We're Building

```
Internet
    |
    v
[Azure Bastion: bastion-demo-learn]
    Public IP: xxx.xxx.xxx.xxx (only Bastion has public IP)
    |
    v
[VNet: vnet-demo-learn (10.0.0.0/16)]
    |
    +-- [AzureBastionSubnet: 10.0.2.0/26]
    |       └── Bastion Host (managed service)
    |
    +-- [subnet-vms: 10.0.1.0/24]
            └── [vk-demo-vm-01] (Private IP: 10.0.1.4)
            └── [vk-demo-vm-02] (Private IP: 10.0.1.5) [Optional]
```

### Key Security Features
- **No Public IPs on VMs**: All VMs have private IPs only
- **Bastion Access**: Secure RDP/SSH access without exposing VMs to internet
- **Network Security Groups**: Control inbound/outbound traffic
- **Private Subnet**: VMs isolated in private subnet

---

## Setup Instructions

### Step 1: Define Variables

```bash
# Resource Group and Location
RESOURCE_GROUP="rg-vk-learn-01"
LOCATION="eastus"  # Change to your preferred region

# VM Configuration
VM_NAME="vk-demo-vm-01"
VM_SIZE="Standard_B2s"
IMAGE="Canonical:ubuntu-24_04-lts:server:latest"
ADMIN_USERNAME="azureuser"

# Network Configuration
VNET_NAME="vnet-demo-learn"
VNET_PREFIX="10.0.0.0/16"
SUBNET_VM_NAME="subnet-vms"
SUBNET_VM_PREFIX="10.0.1.0/24"
SUBNET_BASTION_NAME="AzureBastionSubnet"  # Must be exactly this name
SUBNET_BASTION_PREFIX="10.0.2.0/26"  # Must be /26 or larger

# Bastion Configuration
BASTION_NAME="bastion-demo-learn"
BASTION_PUBLIC_IP="pip-bastion-demo"
NSG_NAME="nsg-demo-vms"
NIC_NAME="${VM_NAME}-nic"
```

### Step 2: Create Resource Group

```bash
# Create resource group (if it doesn't exist)
az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION
```

### Step 3: Create Virtual Network

```bash
# Create VNet
az network vnet create \
  --resource-group $RESOURCE_GROUP \
  --name $VNET_NAME \
  --address-prefix $VNET_PREFIX \
  --location $LOCATION

# Create VM Subnet
az network vnet subnet create \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --name $SUBNET_VM_NAME \
  --address-prefix $SUBNET_VM_PREFIX

# Create Bastion Subnet (name MUST be AzureBastionSubnet)
az network vnet subnet create \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --name $SUBNET_BASTION_NAME \
  --address-prefix $SUBNET_BASTION_PREFIX
```

### Step 4: Create Network Security Group

```bash
# Create NSG for VMs
az network nsg create \
  --resource-group $RESOURCE_GROUP \
  --name $NSG_NAME \
  --location $LOCATION

# Optional: Add outbound internet access rule
az network nsg rule create \
  --resource-group $RESOURCE_GROUP \
  --nsg-name $NSG_NAME \
  --name AllowInternetOutbound \
  --priority 100 \
  --direction Outbound \
  --access Allow \
  --protocol '*' \
  --source-address-prefixes '*' \
  --source-port-ranges '*' \
  --destination-address-prefixes Internet \
  --destination-port-ranges '*'
```

### Step 5: Create Network Interface

```bash
# Create NIC without public IP
az network nic create \
  --resource-group $RESOURCE_GROUP \
  --name $NIC_NAME \
  --vnet-name $VNET_NAME \
  --subnet $SUBNET_VM_NAME \
  --network-security-group $NSG_NAME \
  --location $LOCATION
```

### Step 6: Create Virtual Machine

```bash
# Create VM with SSH key authentication (recommended)
az vm create \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --location $LOCATION \
  --nics $NIC_NAME \
  --image $IMAGE \
  --size $VM_SIZE \
  --admin-username $ADMIN_USERNAME \
  --generate-ssh-keys \
  --public-ip-address "" \
  --verbose

# Alternative: Create VM with password authentication
# az vm create \
#   --resource-group $RESOURCE_GROUP \
#   --name $VM_NAME \
#   --location $LOCATION \
#   --nics $NIC_NAME \
#   --image $IMAGE \
#   --size $VM_SIZE \
#   --admin-username $ADMIN_USERNAME \
#   --admin-password 'YourSecurePassword123!' \
#   --authentication-type password \
#   --public-ip-address "" \
#   --verbose
```

**Note:** SSH keys are stored at:
- Private key: `~/.ssh/id_rsa`
- Public key: `~/.ssh/id_rsa.pub`

### Step 7: Verify VM Creation

```bash
# Check VM status
az vm show \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --query "{Name:name, ProvisioningState:provisioningState, Location:location}" \
  --output table

# Get VM private IP
az vm show \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --show-details \
  --query privateIps \
  --output tsv

# Check VM power state
az vm get-instance-view \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --query "instanceView.statuses[?starts_with(code, 'PowerState/')].displayStatus" \
  --output tsv
```

### Step 8: Create Azure Bastion

```bash
# Create Public IP for Bastion (only Bastion needs public IP)
az network public-ip create \
  --resource-group $RESOURCE_GROUP \
  --name $BASTION_PUBLIC_IP \
  --sku Standard \
  --allocation-method Static \
  --location $LOCATION

# Create Bastion (takes 5-10 minutes)
az network bastion create \
  --resource-group $RESOURCE_GROUP \
  --name $BASTION_NAME \
  --public-ip-address $BASTION_PUBLIC_IP \
  --vnet-name $VNET_NAME \
  --location $LOCATION \
  --sku Basic
```

**Note:** Bastion creation takes 8-12 minutes on average.

### Step 9: Verify Bastion Deployment

```bash
# Check Bastion status
az network bastion show \
  --resource-group $RESOURCE_GROUP \
  --name $BASTION_NAME \
  --query "{Name:name, ProvisioningState:provisioningState}" \
  --output table

# List all resources
az resource list \
  --resource-group $RESOURCE_GROUP \
  --output table
```

### Step 10: Verify Complete Setup

```bash
# Check VNet and subnets
az network vnet subnet list \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --output table

# Verify NIC configuration
az network nic show \
  --resource-group $RESOURCE_GROUP \
  --name $NIC_NAME \
  --query "{Name:name, PrivateIP:ipConfigurations[0].privateIPAddress, PublicIP:ipConfigurations[0].publicIPAddress}" \
  --output table
```

---

## VNet Components Documentation

### 1. Virtual Network (VNet)

**What it is:** Azure Virtual Network is the fundamental building block for your private network in Azure. Similar to AWS VPC.

**Key Properties:**
- Address space: `10.0.0.0/16` (65,536 IPs)
- Location: Must match resource location
- Can contain multiple subnets

**Commands:**
```bash
# List VNets
az network vnet list --output table

# Show VNet details
az network vnet show \
  --resource-group $RESOURCE_GROUP \
  --name $VNET_NAME

# Update VNet address space
az network vnet update \
  --resource-group $RESOURCE_GROUP \
  --name $VNET_NAME \
  --address-prefixes 10.0.0.0/16 10.1.0.0/16
```

### 2. Subnets

**What they are:** Logical subdivisions of a VNet. Each subnet has its own address range within the VNet.

**Our Subnets:**

#### a. VM Subnet (subnet-vms)
- **Address range:** `10.0.1.0/24` (256 IPs, ~251 usable)
- **Purpose:** Hosts virtual machines
- **Associated NSG:** `nsg-demo-vms`

#### b. Azure Bastion Subnet (AzureBastionSubnet)
- **Address range:** `10.0.2.0/26` (64 IPs, ~59 usable)
- **Purpose:** Reserved for Azure Bastion service
- **Requirements:** 
  - Name MUST be exactly "AzureBastionSubnet"
  - Minimum size: /26 or larger
  - No NSG required (Azure manages it)

**Commands:**
```bash
# List subnets
az network vnet subnet list \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --output table

# Show subnet details
az network vnet subnet show \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --name $SUBNET_VM_NAME

# Update subnet
az network vnet subnet update \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --name $SUBNET_VM_NAME \
  --network-security-group $NSG_NAME
```

### 3. Network Security Group (NSG)

**What it is:** Acts as a virtual firewall to control inbound and outbound traffic. Similar to AWS Security Groups.

**Key Features:**
- Contains security rules with priority (100-4096, lower = higher priority)
- Can be associated with subnets or individual NICs
- Default rules allow VNet-to-VNet traffic and outbound internet

**Our NSG Rules:**
```bash
# List NSG rules
az network nsg rule list \
  --resource-group $RESOURCE_GROUP \
  --nsg-name $NSG_NAME \
  --output table

# Create custom rule (example: allow HTTP)
az network nsg rule create \
  --resource-group $RESOURCE_GROUP \
  --nsg-name $NSG_NAME \
  --name AllowHTTP \
  --priority 200 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes '*' \
  --source-port-ranges '*' \
  --destination-address-prefixes '*' \
  --destination-port-ranges 80

# Delete rule
az network nsg rule delete \
  --resource-group $RESOURCE_GROUP \
  --nsg-name $NSG_NAME \
  --name AllowHTTP
```

**Common Rule Examples:**
```bash
# Allow SSH from specific IP
az network nsg rule create \
  --resource-group $RESOURCE_GROUP \
  --nsg-name $NSG_NAME \
  --name AllowSSHFromOffice \
  --priority 150 \
  --source-address-prefixes 203.0.113.0/24 \
  --destination-port-ranges 22 \
  --protocol Tcp \
  --access Allow

# Allow HTTPS
az network nsg rule create \
  --resource-group $RESOURCE_GROUP \
  --nsg-name $NSG_NAME \
  --name AllowHTTPS \
  --priority 210 \
  --destination-port-ranges 443 \
  --protocol Tcp \
  --access Allow

# Deny all other inbound traffic
az network nsg rule create \
  --resource-group $RESOURCE_GROUP \
  --nsg-name $NSG_NAME \
  --name DenyAllInbound \
  --priority 4000 \
  --direction Inbound \
  --access Deny \
  --protocol '*' \
  --source-address-prefixes '*' \
  --destination-address-prefixes '*'
```

### 4. Network Interface Card (NIC)

**What it is:** Virtual network interface attached to a VM. Provides network connectivity.

**Key Features:**
- Has private IP address (can be dynamic or static)
- Can have public IP attached (we're not using this)
- Associated with one subnet
- Can have NSG attached

**Commands:**
```bash
# List NICs
az network nic list \
  --resource-group $RESOURCE_GROUP \
  --output table

# Show NIC details
az network nic show \
  --resource-group $RESOURCE_GROUP \
  --name $NIC_NAME

# Get IP configuration
az network nic ip-config list \
  --resource-group $RESOURCE_GROUP \
  --nic-name $NIC_NAME \
  --output table

# Update NIC (change NSG)
az network nic update \
  --resource-group $RESOURCE_GROUP \
  --name $NIC_NAME \
  --network-security-group $NSG_NAME
```

### 5. Azure Bastion

**What it is:** Fully managed PaaS service for secure RDP/SSH connectivity to VMs without exposing them via public IPs. Similar to AWS Systems Manager Session Manager.

**Key Features:**
- Connects to VMs via private IP
- Uses HTML5 browser-based connection
- No need for NSG rules for SSH/RDP
- Requires dedicated subnet named "AzureBastionSubnet"
- Bastion itself needs a public IP (not the VMs)

**SKUs:**
- **Basic:** Standard connectivity, perfect for learning
- **Standard:** Additional features like IP-based connection, custom ports

**Commands:**
```bash
# List Bastion hosts
az network bastion list \
  --resource-group $RESOURCE_GROUP \
  --output table

# Show Bastion details
az network bastion show \
  --resource-group $RESOURCE_GROUP \
  --name $BASTION_NAME

# Delete Bastion (to save costs)
az network bastion delete \
  --resource-group $RESOURCE_GROUP \
  --name $BASTION_NAME
```

### 6. Public IP Address

**What it is:** Publicly routable IP address. In our setup, only Bastion has one.

**Key Properties:**
- **SKU:** Standard (required for Bastion)
- **Allocation:** Static (doesn't change)
- **Version:** IPv4

**Commands:**
```bash
# List public IPs
az network public-ip list \
  --resource-group $RESOURCE_GROUP \
  --output table

# Show public IP details
az network public-ip show \
  --resource-group $RESOURCE_GROUP \
  --name $BASTION_PUBLIC_IP

# Get IP address value
az network public-ip show \
  --resource-group $RESOURCE_GROUP \
  --name $BASTION_PUBLIC_IP \
  --query ipAddress \
  --output tsv
```

### 7. Virtual Machine

**What it is:** Compute resource running your workload. Similar to AWS EC2.

**Our VM Configuration:**
- **Size:** Standard_B2s (2 vCPUs, 4 GB RAM)
- **OS:** Ubuntu 24.04 LTS
- **Disk:** Managed OS disk (default 30 GB)
- **Authentication:** SSH key-based
- **Networking:** Private IP only

**Commands:**
```bash
# List VMs
az vm list \
  --resource-group $RESOURCE_GROUP \
  --output table

# Show VM details
az vm show \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME

# Get VM sizes available in region
az vm list-sizes \
  --location $LOCATION \
  --output table

# Start VM
az vm start \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME

# Stop and deallocate VM (stops billing for compute)
az vm deallocate \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME

# Restart VM
az vm restart \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME

# Delete VM
az vm delete \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --yes
```

**VM Management:**
```bash
# Get VM power state
az vm get-instance-view \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --query instanceView.statuses[1].displayStatus

# Get all VM details including IPs
az vm show \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --show-details

# Resize VM
az vm resize \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --size Standard_B4ms

# Add/update password for user
az vm user update \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --username $ADMIN_USERNAME \
  --password 'NewSecurePassword123!'
```

---

## Connecting to VMs

### Method 1: Azure Portal (Easiest for Beginners)

1. Go to **Azure Portal** (portal.azure.com)
2. Navigate to **Virtual Machines**
3. Click on your VM: `vk-demo-vm-01`
4. Click **"Connect"** → **"Connect via Bastion"**
5. Enter credentials:
   - **Username:** `azureuser`
   - **Authentication Type:** 
     - "SSH Private Key from Local File" → Browse to `~/.ssh/id_rsa`
     - OR "Password" (if you set one)
6. Click **"Connect"**

Browser opens with SSH session!

### Method 2: Azure CLI with SSH Key

```bash
# Get VM resource ID
VM_ID=$(az vm show \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --query id -o tsv)

# Connect via Bastion
az network bastion ssh \
  --name $BASTION_NAME \
  --resource-group $RESOURCE_GROUP \
  --target-resource-id $VM_ID \
  --auth-type ssh-key \
  --username $ADMIN_USERNAME \
  --ssh-key ~/.ssh/id_rsa
```

### Method 3: Azure CLI with Password

```bash
az network bastion ssh \
  --name $BASTION_NAME \
  --resource-group $RESOURCE_GROUP \
  --target-resource-id $VM_ID \
  --auth-type password \
  --username $ADMIN_USERNAME
# Will prompt for password
```

### Method 4: SSH Tunnel (Advanced)

```bash
# Terminal 1: Create tunnel
az network bastion tunnel \
  --name $BASTION_NAME \
  --resource-group $RESOURCE_GROUP \
  --target-resource-id $VM_ID \
  --resource-port 22 \
  --port 50022

# Terminal 2: SSH via tunnel
ssh $ADMIN_USERNAME@localhost -p 50022 -i ~/.ssh/id_rsa
```

### Verify Connection

Once connected, test basic commands:
```bash
# Check OS
cat /etc/os-release

# Check network
ip addr show

# Check internet connectivity
ping -c 3 8.8.8.8

# Check hostname
hostname

# Check user
whoami
```

---

## Cost Management

### Understanding Costs

**Components that incur charges:**
1. **Virtual Machines** - Charged per hour when running (~$0.02-0.05/hour for B2s)
2. **Azure Bastion** - Charged per hour (~$0.19/hour = ~$140/month)
3. **Storage** - Managed disks (~$4-5/month for 30 GB Standard SSD)
4. **Public IP** - Bastion's public IP (~$3-4/month)

**Free components:**
- VNet, subnets (no charge)
- NSGs (no charge)
- NICs (no charge)

### Stop VM When Not Using

```bash
# Stop VM (deallocate) - stops compute charges
az vm deallocate \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME

# Start VM when needed
az vm start \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME

# Check VM power state
az vm show \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --show-details \
  --query powerState
```

### Delete Resources

```bash
# Delete specific VM
az vm delete \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --yes

# Delete Bastion (saves ~$140/month)
az network bastion delete \
  --resource-group $RESOURCE_GROUP \
  --name $BASTION_NAME

# Delete entire resource group (deletes everything)
az group delete \
  --name $RESOURCE_GROUP \
  --yes \
  --no-wait
```

### Cost Optimization Tips

1. **Use B-series VMs** for development/learning (burstable, cheaper)
2. **Deallocate VMs** when not in use
3. **Delete Bastion** when done with learning session
4. **Use Standard HDD** for non-critical workloads
5. **Set up auto-shutdown** for VMs

```bash
# Enable auto-shutdown (8 PM UTC daily)
az vm auto-shutdown \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --time 2000
```

---

## AWS to Azure Comparison

| AWS Component | Azure Component | Notes |
|---------------|-----------------|-------|
| VPC | Virtual Network (VNet) | Same concept |
| Subnet | Subnet | Same concept |
| Internet Gateway | Not needed | VNet has default internet access |
| NAT Gateway | NAT Gateway | Optional for outbound internet from private subnet |
| Security Group | Network Security Group (NSG) | NSG can attach to subnet or NIC |
| Network ACL | NSG on Subnet | Azure NSGs work at both levels |
| EC2 Instance | Virtual Machine (VM) | Same concept |
| Key Pair | SSH Key | Stored locally (~/.ssh/) |
| Elastic IP | Public IP Address | Similar, but we're not using |
| Systems Manager Session Manager | Azure Bastion | Browser-based vs agent-based |
| IAM Role (for EC2) | Managed Identity | For VM to access Azure services |
| Route Table | Route Table | Same concept |
| VPC Peering | VNet Peering | Same concept |
| Transit Gateway | Virtual WAN | Hub-spoke networking |

### Common Commands Comparison

| Task | AWS CLI | Azure CLI |
|------|---------|-----------|
| List VMs | `aws ec2 describe-instances` | `az vm list` |
| VM details | `aws ec2 describe-instances --instance-ids i-xxx` | `az vm show --name vm-name` |
| Start VM | `aws ec2 start-instances --instance-ids i-xxx` | `az vm start --name vm-name` |
| Stop VM | `aws ec2 stop-instances --instance-ids i-xxx` | `az vm deallocate --name vm-name` |
| List VPCs/VNets | `aws ec2 describe-vpcs` | `az network vnet list` |
| List Security Groups/NSGs | `aws ec2 describe-security-groups` | `az network nsg list` |
| List Subnets | `aws ec2 describe-subnets` | `az network vnet subnet list --vnet-name vnet-name` |

---
