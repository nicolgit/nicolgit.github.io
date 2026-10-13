---
title: "Assigning Contributor to a Customer Managed Identity through Azure Lighthouse"
date: 2026-10-13 10:00
tags: [Azure, Lighthouse, Managed Identity, CLI]
excerpt: "Proximity project - a sandbox for practicing real cloud architecture, experimenting with new frameworks, and stress-testing patterns like serverless, IaC, and AI-assisted development in a space where I can make deliberate, opinionated decisions end to end."


---
Azure Lighthouse lets a service provider manage delegated Azure subscriptions or resource groups in a customer's tenant while remaining authenticated in the provider's own tenant. The customer retains ownership of the resources and controls which provider principals receive access and which roles they can use.

In this post, we will use **Azure Cloud Shell Bash on the provider side** to:

1. Create a user-assigned managed identity in a customer's resource group.
2. Create a storage account in the same resource group.
3. Assign **Contributor** to that managed identity, scoped only to the storage account.

The interesting part is the last step: managing resources through Lighthouse and assigning Azure RBAC roles through Lighthouse are different capabilities.

## The special case for managed identities

A Lighthouse operator cannot use the delegation as a general-purpose mechanism for assigning roles to customer users, groups, or arbitrary service principals.

However, Lighthouse supports a specific exception: an authorized provider principal can assign an explicitly permitted set of built-in roles to **managed identities in the customer tenant**.

For this scenario, the provider principal needs a Lighthouse authorization with:

- **User Access Administrator**: `18d7d88d-d35e-4fb5-a5c3-7773c20a72d9`.
- A `delegatedRoleDefinitionIds` array containing the roles it may assign. Here, that is **Contributor**: `b24988ac-6180-42a0-ab88-20f7382dd24c`.

The following entry belongs in the registration definition's `authorizations` array:

```json
{
  "principalId": "<PROVIDER-PRINCIPAL-OBJECT-ID>",
  "principalIdDisplayName": "Managed identity role operator",
  "roleDefinitionId": "18d7d88d-d35e-4fb5-a5c3-7773c20a72d9",
  "delegatedRoleDefinitionIds": [
    "b24988ac-6180-42a0-ab88-20f7382dd24c"
  ]
}
```

Replace `<PROVIDER-PRINCIPAL-OBJECT-ID>` with the object ID of the provider user, group, or service principal. For a group authorization, the operator must be a member of that group.

> Lighthouse limits this authorization to the supported managed-identity role-assignment scenario. Other permissions normally associated with the role do not apply.

### Does the operator also need Contributor?

**NO**, The permission to assign a role is separate from the permission to manage resources.

For the complete walkthrough, however, the operator also needs permissions to create the identity and storage account. A separate **Contributor** authorization on the delegated resource group provides those permissions:

```json
{
  "principalId": "<PROVIDER-PRINCIPAL-OBJECT-ID>",
  "principalIdDisplayName": "Resource operator",
  "roleDefinitionId": "b24988ac-6180-42a0-ab88-20f7382dd24c"
}
```

### Allowing the provider to remove its delegation

Microsoft also recommends including an authorization with **Managed Services Registration Assignment Delete Role**:

```json
{
  "principalId": "<PROVIDER-PRINCIPAL-OBJECT-ID>",
  "principalIdDisplayName": "Delegation removal operator",
  "roleDefinitionId": "91c1777a-f3dc-4fae-b103-61d183457e46"
}
```

This allows the authorized provider principal to remove its access to the delegation when needed. It is not required for assigning Contributor to the managed identity.

Because removing a delegation revokes provider access, grant this role deliberately to the principal or group responsible for offboarding.

## Prepare and approve the Lighthouse delegation

In the **service provider tenant**, open the Azure portal and navigate to:

**Azure Lighthouse > Create ARM template**

Prepare the offer with the provider tenant ID, the principals to authorize, and the required resource-management roles.

> **Important:** the portal template-generation UI does not expose `delegatedRoleDefinitionIds`. Edit the generated ARM template and explicitly add the User Access Administrator authorization shown above before sending it to the customer.

In the **customer tenant**, an authorized customer administrator then opens:

**Service providers > Service provider offers > Add offer via template**

Upload or paste the prepared template, review the requested permissions, and deploy it. The customer deployment requires the permissions described in the [Lighthouse onboarding documentation](https://learn.microsoft.com/azure/lighthouse/how-to/onboard-customer#deploy-the-azure-resource-manager-template), including the ability to assign roles at the relevant scope.

Once onboarding is complete, the authorized provider operator can work with the delegated customer resource group.

## Why not use the portal to assign the role?

Although the provider can manage many customer resources through the portal, Microsoft documents that these **cross-tenant role assignments must be created through APIs, not the Azure portal**.

The Lighthouse managed-identity exception requires the role-assignment request to identify the customer resource that owns the managed identity through `delegatedManagedIdentityResourceId`. The normal portal IAM workflow is not the supported path for this operation.

We will use `az rest` in Cloud Shell to send the request directly to Azure Resource Manager.

## Create the identity and storage account from Cloud Shell

Open **Cloud Shell in Bash mode** while signed in to the provider tenant as an authorized operator.

If you are using Azure CLI outside the portal's authenticated Cloud Shell session, explicitly sign in to the provider tenant:

```bash
az login --tenant "<PROVIDER-TENANT-ID>"
```

Do not sign in to the customer tenant for the provider-side workflow. Instead, select the delegated customer subscription:

```bash
SUBSCRIPTION_ID="<CUSTOMER-SUBSCRIPTION-ID>"
RESOURCE_GROUP="customer-managed-rg"
LOCATION="westeurope"
IDENTITY_NAME="workload-identity"
STORAGE_NAME="<GLOBALLY-UNIQUE-STORAGE-NAME>"

az account set --subscription "$SUBSCRIPTION_ID"

az identity create \
    --subscription "$SUBSCRIPTION_ID" \
    --resource-group "$RESOURCE_GROUP" \
    --name "$IDENTITY_NAME" \
    --location "$LOCATION"

az storage account create \
    --subscription "$SUBSCRIPTION_ID" \
    --resource-group "$RESOURCE_GROUP" \
    --name "$STORAGE_NAME" \
    --location "$LOCATION" \
    --sku Standard_LRS \
    --kind StorageV2
```

> Replace all placeholders before running the commands. Storage account names must be globally unique and contain 3-24 lowercase letters and numbers.

## Assign Contributor to the managed identity

Save the following script as `assign-role-contributor.sh`, update the variables, and run it from provider-side Cloud Shell Bash:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Provider-side Cloud Shell Bash: fill in the variables and run the script.
# Assigns Contributor to the identity on the storage account.
# The delegation must include User Access Administrator with Contributor allowed.
SUBSCRIPTION_ID="<CUSTOMER-SUBSCRIPTION-ID>"
RESOURCE_GROUP="customer-managed-rg"
STORAGE_NAME="<GLOBALLY-UNIQUE-STORAGE-NAME>"
IDENTITY_NAME="workload-identity"
CONTRIBUTOR_ROLE_ID="b24988ac-6180-42a0-ab88-20f7382dd24c"
API_VERSION="2022-04-01"

az account set --subscription "$SUBSCRIPTION_ID"

BASE_SCOPE="/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${RESOURCE_GROUP}/providers"
IDENTITY_RESOURCE_ID="${BASE_SCOPE}/Microsoft.ManagedIdentity/userAssignedIdentities/${IDENTITY_NAME}"
STORAGE_SCOPE="${BASE_SCOPE}/Microsoft.Storage/storageAccounts/${STORAGE_NAME}"
PRINCIPAL_ID=$(az identity show --ids "$IDENTITY_RESOURCE_ID" --query principalId --output tsv)
PRINCIPAL_ID="${PRINCIPAL_ID//$'\r'/}"
ASSIGNMENT_ID=$(cat /proc/sys/kernel/random/uuid)

az rest --method put \
    --url "https://management.azure.com${STORAGE_SCOPE}/providers/Microsoft.Authorization/roleAssignments/${ASSIGNMENT_ID}?api-version=${API_VERSION}" \
    --body "{
      \"properties\": {
        \"roleDefinitionId\": \"/subscriptions/${SUBSCRIPTION_ID}/providers/Microsoft.Authorization/roleDefinitions/${CONTRIBUTOR_ROLE_ID}\",
        \"principalId\": \"${PRINCIPAL_ID}\",
        \"principalType\": \"ServicePrincipal\",
        \"delegatedManagedIdentityResourceId\": \"${IDENTITY_RESOURCE_ID}\"
      }
    }"
```

Run it without command-line arguments:

```bash
bash assign-role-contributor.sh
```

The script intentionally performs no existence checks or follow-up verification. Azure CLI errors remain visible, and `set -euo pipefail` stops the script when a command fails.

Three details matter:

- **`principalId` is the managed identity's service principal object ID**, not its `clientId`.
- **`delegatedManagedIdentityResourceId` identifies the user-assigned managed identity resource in the customer tenant.** This is the key Lighthouse-specific property.
- **The role-assignment scope is the storage account**, not the entire resource group or subscription.

The UUID is generated using a Linux facility available in Cloud Shell Bash. This script is intended for that environment, not Windows PowerShell.

> **Terminology:** this assigns a role *to* the managed identity *on* the storage account. It does not attach the managed identity to the storage account as the storage account's own identity.

## What a successful response looks like

A successful request returns a role-assignment resource similar to the following. This is an abbreviated, anonymized example; angle-bracket values represent identifiers from your environment.

```json
{
  "id": "/subscriptions/<CUSTOMER-SUBSCRIPTION-ID>/resourceGroups/<RESOURCE-GROUP>/providers/Microsoft.Storage/storageAccounts/<STORAGE-ACCOUNT>/providers/Microsoft.Authorization/roleAssignments/<ROLE-ASSIGNMENT-ID>",
  "name": "<ROLE-ASSIGNMENT-ID>",
  "properties": {
    "delegatedManagedIdentityResourceId": "/subscriptions/<CUSTOMER-SUBSCRIPTION-ID>/resourceGroups/<RESOURCE-GROUP>/providers/Microsoft.ManagedIdentity/userAssignedIdentities/<MANAGED-IDENTITY>",
    "principalId": "<MANAGED-IDENTITY-PRINCIPAL-ID>",
    "principalType": "ServicePrincipal",
    "roleDefinitionId": "/subscriptions/<CUSTOMER-SUBSCRIPTION-ID>/providers/Microsoft.Authorization/roleDefinitions/b24988ac-6180-42a0-ab88-20f7382dd24c",
    "scope": "/subscriptions/<CUSTOMER-SUBSCRIPTION-ID>/resourceGroups/<RESOURCE-GROUP>/providers/Microsoft.Storage/storageAccounts/<STORAGE-ACCOUNT>"
  },
  "type": "Microsoft.Authorization/roleAssignments"
}
```

The Contributor role GUID remains in the example because it is a public built-in role identifier, not an environment-specific value.

The response confirms the assignment returned by the creation request. Effective permissions may take time to propagate.

## Verification: creation and visibility are different

One important Lighthouse limitation is that **successfully creating a customer role assignment does not guarantee that the provider can subsequently see it in the portal or retrieve it through CLI/API reads**.

Do not interpret a provider-side empty list or `RoleAssignmentNotFound` as proof that the assignment does not exist.

> **For an independent check, use an account authenticated in the customer tenant with the required RBAC read permissions.** Inspect the storage account's **Access control (IAM) > Role assignments**, or use Azure CLI from that customer context.

## Takeaways

Azure Lighthouse can support an end-to-end provider workflow that creates customer resources and grants a managed identity a permitted management-plane role.

The essential combination is:

- Resource-management permissions for creating the identity and storage account.
- A **User Access Administrator** Lighthouse authorization with **Contributor** in `delegatedRoleDefinitionIds`.
- An API role-assignment request containing `delegatedManagedIdentityResourceId`.
- A resource-level scope to limit the identity's permissions to the intended storage account.

Finally, keep creation and visibility separate: a provider can create the assignment without being able to reliably inspect it afterward through Lighthouse.

## References

- [Azure Lighthouse overview](https://learn.microsoft.com/azure/lighthouse/overview)
- [Onboard a customer to Azure Lighthouse](https://learn.microsoft.com/azure/lighthouse/how-to/onboard-customer)
- [Tenants, users, and roles in Azure Lighthouse scenarios](https://learn.microsoft.com/azure/lighthouse/concepts/tenants-users-roles)
- [Authorize role assignments to customer managed identities](https://learn.microsoft.com/azure/lighthouse/how-to/deploy-policy-remediation#create-a-user-who-can-assign-roles-to-a-managed-identity-in-the-customer-tenant)
- [Role Assignments REST API: Create](https://learn.microsoft.com/rest/api/authorization/role-assignments/create?view=rest-authorization-2022-04-01)
- [List Azure role assignments using Azure CLI](https://learn.microsoft.com/azure/role-based-access-control/role-assignments-list-cli)
