# Get started with Azure CLI

## Installation

https://learn.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest

### mac

```sh
brew update && brew install azure-cli
```

### windows

```sh
winget install -e --id Microsoft.AzureCLI
```

https://learn.microsoft.com/en-us/cli/azure/?view=azure-cli-latest

## Install or run in Azure Cloud Shell

```sh
az version
```

### az account set
Set a subscription to be the current active subscription

```sh
az account set --name --subscription
```

Required Parameters

```--name --subscription -n -s```

Name or ID of subscription.

### az account show
Get the details of a subscription.

If the subscription isn't specified, shows the details of the default subscription.

```sh
az account show [--name --subscription]
```


### 🔐 Authentication & Subscription

```bash
az login
```

**Description:**
Logs in to Azure using a web browser and authenticates the Azure CLI session.


```bash
az account set --subscription ad8d2cb5-236c-46fb-bcf2-fc088a93d462
```

**Description:**
Sets the active Azure subscription context for the current CLI session.
Used when you have access to multiple subscriptions (e.g., sandbox, production).


```bash
az account show
```

**Description:**
Displays details of the currently active Azure account, including:

* Subscription ID
* Subscription name
* Tenant (Directory) ID


#### ✅ Best Practice Notes (Add This Section)

* Always run `az account show` after login to confirm:

  * Correct subscription
  * Correct tenant
* Use `az account set` **before creating resources** to avoid deploying to the wrong subscription.
* Never hard-code subscription IDs in scripts without documentation.


#### 📌 Optional Enhancement (Recommended)

Add a **Quick Check Commands** section:

```bash
az account list
```

**Description:**
Lists all Azure subscriptions you have access to.


```bash
az logout
```

**Description:**
Logs out of the current Azure CLI session.

---

## kubelogin – Azure Kubernetes Service Authentication


**Reference:**
Microsoft Learn – *Use kubelogin to authenticate in Azure Kubernetes Service (AKS)*

https://learn.microsoft.com/en-us/azure/aks/kubelogin-authentication?tabs=environment-variables&pivots=device-code


### What is kubelogin?

`kubelogin` is a **client-go credential plugin** used by `kubectl` to authenticate to **Azure Kubernetes Service (AKS)** using **Microsoft Entra ID (Azure AD)**.

It enables:

* Secure, identity-based access to AKS clusters
* Integration with **Azure RBAC**
* Authentication features not available directly in `kubectl`


### Why kubelogin is Needed in AKS

* AKS clusters are often configured with **Microsoft Entra ID authentication**
* Access to the cluster is controlled using **Azure RBAC**
* `kubectl` alone cannot handle Entra ID authentication
* `kubelogin` bridges this gap by handling token acquisition and renewal

---

### Installation Guide

#### Windows (Using winget)

#### Required Software

| Software  | Command                                            | Purpose                        |
| --------- | -------------------------------------------------- | ------------------------------ |
| kubectl   | `winget install --id=Kubernetes.kubectl -e`        | Kubernetes CLI                 |
| kubelogin | `winget install --id=Microsoft.Azure.Kubelogin -e` | Entra ID authentication plugin |

**OS:** Windows
**Install Method:** winget

##### macOS (Using Homebrew)

```bash
brew install Azure/kubelogin/kubelogin
```

**OS:** macOS
**Install Method:** Homebrew
**Purpose:** Installs kubelogin for AKS authentication using Microsoft Entra ID


#### How Authentication Works (High-Level Flow)

1. User runs `kubectl` command
2. `kubectl` invokes `kubelogin`
3. `kubelogin` authenticates the user via Microsoft Entra ID
4. Access token is generated
5. Token is validated against Azure RBAC permissions
6. Authorized access to the AKS cluster is granted


#### Key Notes / Best Practices

* Ensure **Azure CLI (`az login`)** is completed before using `kubectl`
* User must have appropriate **Azure RBAC roles** (e.g., AKS Cluster User Role)
* `kubelogin` must match the authentication mode used by the AKS cluster
* Recommended for **enterprise and production environments**


#### Common Use Case

> AKS instances are configured with Microsoft Entra ID authentication with Azure RBAC.
> When daily access is granted, appropriate RBAC roles and permissions allow authenticated users to interact with the cluster using `kubectl`.


---

## kubelogin Cheat Sheet (AKS – Microsoft Entra ID)

#### Prerequisites

* Azure CLI installed and logged in (`az login`)
* `kubelogin` installed
* `kubectl` installed
* Network access enabled (e.g., Zscaler VPN if required)


#### Core Commands

```bash
az aks get-credentials \
  --resource-group ${RESOURCE_GROUP_NAME} \
  --name ${AKS_INSTANCE_NAME} \
  --overwrite-existing
```

**Description:**
Downloads the AKS cluster credentials and merges them into the local `kubeconfig` file.
The `--overwrite-existing` flag ensures existing credentials for the cluster are updated.


```bash
kubelogin convert-kubeconfig -l azurecli
```

**Description:**
Converts the kubeconfig to use **Microsoft Entra ID authentication** via the Azure CLI login.
This enables `kubectl` to authenticate using the identity from `az login`.


#### Typical Connection Flow (Recommended)

1. Authenticate to Azure:

   ```bash
   az login
   ```
2. Set correct subscription:

   ```bash
   az account set --subscription <SUBSCRIPTION_ID>
   ```
3. Download AKS credentials:

   ```bash
   az aks get-credentials --resource-group <RG> --name <AKS_NAME>
   ```
4. Enable Entra ID authentication:

   ```bash
   kubelogin convert-kubeconfig -l azurecli
   ```
5. Verify access:

   ```bash
   kubectl get nodes
   ```


#### Authentication Notes

* AKS is configured with **Microsoft Entra ID authentication** and **Azure RBAC**
* User access is controlled via Azure role assignments (e.g., *AKS Cluster User Role*)
* `kubelogin` handles token retrieval and renewal
* `kubectl` alone cannot authenticate to Entra ID–enabled clusters


#### Azure Portal Tip

The **Connect** option in the Azure Portal (AKS → Connect) provides:

* Cluster-specific connection commands
* Authentication method details
* A reliable reference for first-time setup


#### Best Practices

* Always confirm the active subscription before connecting
* Use `--overwrite-existing` when switching between clusters
* Keep Azure CLI and kubelogin versions up to date
* Prefer Entra ID + Azure RBAC for production clusters


#### Optional Add-On (Good to Document Later)

```bash
kubelogin --help
```

**Purpose:**
Lists supported login modes (`azurecli`, `devicecode`, `spn`, etc.)

<img width="1598" height="924" alt="image" src="https://github.com/user-attachments/assets/4d9bc3ad-53f5-44b9-a98d-6ca4b4f8a0b4" />

Once authenticated you can proceed to run ```kubectl``` commands:


```sh
PS C:\Users\csiler> az aks get-credentials --resource-group rg-fsc-dev-app-eaus-001 --name aks-fsc-dev-app-eaus-001 --overwrite-existing
Merged "aks-fsc-dev-app-eaus-001" as current context in C:\Users\csiler\.kube\config
PS C:\Users\csiler> kubelogin convert-kubeconfig -l azurecli
PS C:\Users\csiler> kubectl get deployments --all-namespaces=true
NAMESPACE                      NAME                                          READY   UP-TO-DATE   AVAILABLE   AGE
account-manager                account-manager                               1/1     1            1           42d
accounts-management-server     accounts-management-server                    1/1     1            1           46d
admin-api-service              admin-api-service                             2/2     2            2           40d
api-service                    api-service                                   2/2     2            2           40d
asset-identity                 asset-identity                                1/1     1            1           31d
asset-orchestrator             asset-orchestrator                            1/1     1            1           13d
asset-query                    asset-query                                   2/2     2            2           26d
```

Example: kubectl output

<img width="1229" height="215" alt="image" src="https://github.com/user-attachments/assets/c9e30eac-74a7-4a00-bff2-b070e534c003" />


---

