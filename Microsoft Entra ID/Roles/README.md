Azure built-in roles

https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles


## RBAC


• It’s always better to assign roles to groups and not individual users

• Easier maintenance


## Understanding the Two RBAC Systems

Azure has **two separate but complementary** RBAC systems - this is crucial to understand:

### 1. Azure RBAC (for Azure Resources)
Controls access to Azure resources like VMs, Storage Accounts, Databases, etc.

### 2. Entra ID Roles (for Identity Management)
Controls who can manage the identity system itself - users, groups, app registrations, etc.

**Think of it this way:**
- **Azure RBAC** = "Can you deploy a VM or read from a storage account?"
- **Entra ID Roles** = "Can you create users or reset passwords?"

---

## Groups in Azure

Groups are your primary tool for organizing identities and assigning permissions at scale.

### Types of Groups

**Security Groups:**
- Used for access management
- Can contain users, devices, other groups, and service principals
- Used in both Azure RBAC and Entra ID role assignments

**Microsoft 365 Groups:**
- Collaboration-focused (Teams, SharePoint, Outlook)
- Include email, shared calendar, files
- Can also be used for Azure RBAC

### Membership Types

**Assigned (Static):**

**Dynamic (Rule-based):**
Automatically manages membership based on user/device attributes.

```
Real example - Dynamic group rule:
(user.department -eq "Engineering") -and (user.jobTitle -contains "DevOps")

This auto-adds anyone with department=Engineering AND job title containing "DevOps"
```

**Why dynamic groups matter for DevOps:**
- New engineer joins → automatically gets access to dev resources
- Engineer changes teams → access automatically removed
- No manual group management needed

---

## Azure RBAC Deep Dive

### Role Assignment Components

✅ Every Azure RBAC assignment has three parts:

1. **Security Principal** (WHO) - User, Group, Service Principal, Managed Identity
2. **Role Definition** (WHAT) - Set of permissions
3. **Scope** (WHERE) - Level at which access applies

### Scope Hierarchy

```
Management Group (highest)
    │
    ├─ Subscription
    │     │
    │     ├─ Resource Group
    │     │     │
    │     │     └─ Resource (lowest)
    │
    └─ Subscription
          └─ Resource Group
                └─ Resource
```

**Inheritance flows downward:** If you grant "Reader" at subscription level, the principal can read all resource groups and resources within.

### Built-in Azure RBAC Roles

**Common roles you'll use:**

**Owner:**
- Full access to resources
- Can manage access (assign roles to others)
- Use case: Team leads, admins

**Contributor:**
- Create/manage resources
- Cannot manage access
- Use case: Developers, DevOps engineers who deploy infrastructure

**Reader:**
- View resources only
- Cannot make changes
- Use case: Auditors, support staff

**Specialized roles examples:**
- `Virtual Machine Contributor` - Manage VMs but not networking
- `Key Vault Secrets User` - Read secrets from Key Vault
- `Storage Blob Data Contributor` - Read/write/delete blobs
- `AcrPull` - Pull container images from ACR
- `Kubernetes Cluster Admin` - Full access to AKS cluster


<img width="1974" height="1168" alt="image" src="https://github.com/user-attachments/assets/0198ca88-b64c-4b7d-a31a-21a63c1b6ef2" />

e.g.

<img width="1820" height="1156" alt="image" src="https://github.com/user-attachments/assets/1cba14ad-8e91-4543-a31b-7d83b06b9dab" />

<img width="2088" height="1086" alt="image" src="https://github.com/user-attachments/assets/57a692d0-8a42-47f9-868f-dcd5be04cb84" />


**Result:** Everyone in DevOps-Team can:
- Deploy/manage resources in production-rg
- Read secrets from prod-kv
- But cannot delete the Key Vault itself or manage access to it

### Custom Roles

When built-in roles don't fit, create custom ones:

```json
{
  "Name": "Virtual Machine Operator",
  "Description": "Can start/stop VMs but not delete or modify them",
  "Actions": [
    "Microsoft.Compute/virtualMachines/start/action",
    "Microsoft.Compute/virtualMachines/restart/action",
    "Microsoft.Compute/virtualMachines/powerOff/action",
    "Microsoft.Compute/virtualMachines/read"
  ],
  "NotActions": [],
  "AssignableScopes": [
    "/subscriptions/{subscription-id}"
  ]
}
```

---

## Entra ID Roles Deep Dive

These roles control **Entra ID management operations**, not Azure resources.

### Common Entra ID Roles

**Global Administrator:**
- Highest privilege
- Full access to all Entra ID features
- Can manage all Azure subscriptions
- ⚠️ Use sparingly - this is like AWS root account

**User Administrator:**
- Create/manage users and groups
- Reset passwords for non-admin users
- Cannot manage other administrators

**Application Administrator:**
- Register/manage applications
- Grant app permissions
- Manage enterprise applications

**Security Administrator:**
- Manage security features
- Read security reports
- Manage conditional access policies

**Cloud Application Administrator:**
- Similar to Application Administrator
- Cannot manage App Proxy or federated apps

---

## Service Principals & Managed Identities

https://learn.microsoft.com/en-us/azure/devops/integrate/get-started/authentication/service-principal-managed-identity?view=azure-devops

### What Are They?

**Both are "identities for applications"** - like IAM roles/users for apps instead of humans.

### Service Principal

**Manual identity** you create with credentials (secret or certificate).

**When to use:**
- CI/CD pipelines (GitHub Actions, Jenkins, Terraform)
- Apps running outside Azure
- Need explicit credential control

**Downside:** You manage credentials, rotation, storage


### Managed Identity

**Automatic identity** managed by Azure - **no credentials to handle**.

#### System-Assigned
- Tied to one resource (VM, App Service, Container)
- Created/deleted with the resource
- Use when: Single resource needs access


#### Quick Comparison

| Feature | Service Principal | System-Assigned MI | User-Assigned MI |
|---------|------------------|-------------------|------------------|
| **Credentials** | Manual (secret/cert) | None (automatic) | None (automatic) |
| **Lifecycle** | Independent | Tied to resource | Independent |
| **Reusability** | Yes | No | Yes |
| **Use case** | Outside Azure | Single resource | Multiple resources |
| **Management** | You handle | Azure handles | Azure handles |

---

#### Real Example: Web App Accessing Database

**With Service Principal (old way):**
```
1. Create SP, get secret
2. Store secret in app config
3. Rotate secret every 90 days
4. Update app config
5. App uses secret to authenticate
```

**With Managed Identity (modern way):**
```
1. Enable managed identity on App Service
2. Grant it "SQL DB Contributor" role
3. App authenticates automatically
4. No secrets, no rotation
```

#### Key Takeaway

**Always prefer Managed Identity when running in Azure**. Only use Service Principals when you must (external systems, CI/CD outside Azure).


**When to use which:**

- **System-assigned:** Single VM/App Service needs access to resources
- **User-assigned:** Multiple resources need same access, or you need identity to persist even if resource is recreated
- **Service Principal:** CI/CD pipelines, on-premises apps, third-party tools

---
