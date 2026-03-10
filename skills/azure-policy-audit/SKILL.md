---
name: azure-policy-audit
description: Audit Azure Policy definitions for a specific service (e.g. Storage, Compute, KeyVault). Finds built-in policy definitions related to the service, checks which are currently assigned in the subscription, and returns the unassigned policies with their names and IDs. Use this skill when asked about Azure policy coverage, missing policies, unassigned policies, or policy gaps for an Azure service.
---

# Azure Policy Audit Skill

When the user asks you to audit Azure policies for a specific service, follow these steps exactly.

## Step 1 — Identify the target service

Ask the user which Azure service to audit if not already specified. Common service keywords include:
- **Storage** → `Microsoft.Storage`
- **Compute** → `Microsoft.Compute`
- **KeyVault** → `Microsoft.KeyVault`
- **SQL** → `Microsoft.Sql`
- **Network** → `Microsoft.Network`
- **App Service** → `Microsoft.Web`
- **Kubernetes** → `Microsoft.ContainerService`
- **Cosmos DB** → `Microsoft.DocumentDB`

Use the resource provider namespace to match policy definitions accurately.

## Step 2 — Retrieve built-in policy definitions for the service

Use Az PowerShell to list built-in policy definitions related to the target service. Replace `<ServiceKeyword>` with the service name the user specified (e.g. "Storage", "Key Vault").

First, try filtering by the policy metadata category which is the most reliable method:

```powershell
Get-AzPolicyDefinition | Where-Object { $_.PolicyType -eq 'BuiltIn' -and $_.Metadata.category -eq '<ServiceKeyword>' } | Select-Object -Property DisplayName, Name, @{N='PolicyDefinitionId';E={$_.Id}}
```

If the category filter returns no results, fall back to matching on DisplayName and Description:

```powershell
Get-AzPolicyDefinition | Where-Object { $_.PolicyType -eq 'BuiltIn' -and ($_.DisplayName -match '<ServiceKeyword>' -or $_.Description -match '<ServiceKeyword>') } | Select-Object -Property DisplayName, Name, @{N='PolicyDefinitionId';E={$_.Id}}
```

Store the resulting list of policy definition IDs and display names.

## Step 3 — Retrieve current policy assignments

Use Az PowerShell to retrieve all policy assignments scoped at the **tenant root management group** and below. This captures every assignment inherited across the entire tenant.

First, get the tenant root management group ID and the current subscription ID:

```powershell
$tenantId = (Get-AzContext).Tenant.Id
$subId = (Get-AzContext).Subscription.Id
$rootScope = "/providers/Microsoft.Management/managementGroups/$tenantId"
```

Then retrieve policy assignments from both the root management group and the subscription (with descendants). The `-IncludeDescendent` switch is **not supported** at management group scope, so we query each scope separately and deduplicate:

```powershell
$mgAssignments = Get-AzPolicyAssignment -Scope $rootScope
$subAssignments = Get-AzPolicyAssignment -Scope "/subscriptions/$subId" -IncludeDescendent
$assignments = @($mgAssignments) + @($subAssignments) | Sort-Object -Property Id -Unique
```

From the assignment results, extract the `PolicyDefinitionId` from each assignment. These are the policies that are currently assigned.

```powershell
$assignedPolicyDefIds = @()

foreach ($assignment in $assignments) {
    $defId = $assignment.PolicyDefinitionId

    if ($defId -match 'policySetDefinitions') {
        # This is an initiative — resolve the individual policy definitions inside it
        try {
            $setDef = Get-AzPolicySetDefinition -Id $defId -ErrorAction Stop
            foreach ($p in $setDef.PolicyDefinition) {
                $assignedPolicyDefIds += $p.policyDefinitionId
            }
        } catch {
            Write-Warning "Could not resolve initiative: $defId"
        }
    } else {
        $assignedPolicyDefIds += $defId
    }
}

$assignedPolicyDefIds = $assignedPolicyDefIds | Sort-Object -Unique
```

**IMPORTANT:** Policy assignments can reference either individual policy definitions or policy set definitions (initiatives). When checking initiatives, you need to look inside each initiative's `policyDefinitions` array to find the individual policy definition references contained within. A policy definition is considered "assigned" if:
- It is directly assigned as a standalone policy assignment, OR
- It is included within an assigned initiative (policy set definition)

## Step 4 — Compare and identify unassigned policies

Compare the list of built-in policy definitions from Step 2 against the assigned policy definition IDs from Step 3.

A policy is **unassigned** if its definition ID does not appear in any current assignment, either directly or within an initiative.

## Step 5 — Return the results

Present the results in a clear, structured format. Always include two sections:

### Assigned Policies
List policies that ARE currently assigned:

| Display Name | Policy Definition ID | Assignment Type |
|---|---|---|
| _name_ | _/providers/Microsoft.Authorization/policyDefinitions/xxx_ | Direct / Via Initiative |

### Unassigned Policies
List policies that are NOT currently assigned:

| Display Name | Policy Definition ID |
|---|---|
| _name_ | _/providers/Microsoft.Authorization/policyDefinitions/xxx_ |

### Summary
- Total built-in policies found for the service: **N**
- Currently assigned: **N** (direct: N, via initiative: N)
- Not assigned: **N**

## Important Notes

- Only include **BuiltIn** policy definitions. Do not include Custom policies unless the user explicitly asks.
- The policy definition ID is the full resource ID (e.g. `/providers/Microsoft.Authorization/policyDefinitions/<guid>`).
- If the user asks to assign the unassigned policies, provide the policy definition IDs in a format that can be consumed by other skills or scripts.
- When returning results for consumption by other skills, output a JSON array:

```json
[
  {
    "displayName": "Storage accounts should restrict network access",
    "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/xxx"
  }
]
```
