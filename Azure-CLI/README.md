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

