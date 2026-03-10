---
name: azure-policy-research
description: Research and discover available Azure built-in policy definitions for a specific service category. Browse policies, view their effects, parameters, and descriptions without checking current assignments. Use this skill when asked to explore, research, browse, or list available Azure policies for a service, or when the user wants to understand what policies exist before deciding what to assign.
---

# Azure Policy Research Skill

Use this skill to help the user explore and discover available Azure built-in policy definitions for a service category. Unlike the `azure-policy-audit` skill which compares against current assignments, this skill is purely informational — it helps users understand what policies exist and what they do.

## Step 1 — Identify the target service

Ask the user which Azure service to research if not already specified. Common service keywords and their metadata categories include:

| Keyword | Metadata Category | Resource Provider |
|---|---|---|
| Storage | Storage | Microsoft.Storage |
| Compute / VM | Compute | Microsoft.Compute |
| Key Vault | Key Vault | Microsoft.KeyVault |
| SQL | SQL | Microsoft.Sql |
| Network | Network | Microsoft.Network |
| App Service | App Service | Microsoft.Web |
| Kubernetes / AKS | Kubernetes | Microsoft.ContainerService |
| Cosmos DB | Cosmos DB | Microsoft.DocumentDB |
| Container Registry | Container Registry | Microsoft.ContainerRegistry |
| Monitoring | Monitoring | Microsoft.Insights |
| Security Center | Security Center | Microsoft.Security |
| API Management | API Management | Microsoft.ApiManagement |
| Event Hub | Event Hub | Microsoft.EventHub |
| Service Bus | Service Bus | Microsoft.ServiceBus |

If the user's keyword doesn't match the table above, use the keyword as both the category filter and a DisplayName/Description search term.

## Step 2 — Retrieve built-in policy definitions

Use Az PowerShell to list built-in policy definitions for the target service. First try filtering by metadata category (most reliable):

```powershell
$policies = Get-AzPolicyDefinition | Where-Object {
    $_.PolicyType -eq 'BuiltIn' -and $_.Metadata.category -eq '<Category>'
}
```

If the category filter returns no results, fall back to matching on DisplayName and Description:

```powershell
$policies = Get-AzPolicyDefinition | Where-Object {
    $_.PolicyType -eq 'BuiltIn' -and (
        $_.DisplayName -match '<ServiceKeyword>' -or
        $_.Description -match '<ServiceKeyword>'
    )
}
```

## Step 3 — Classify and present the results

Group the policies by their effect type for easy browsing. Extract the effect from the policy rule or the default value of the effect parameter.

```powershell
$policies | ForEach-Object {
    $effectParam = $_.Parameter.effect.defaultValue
    if (-not $effectParam) {
        $effectParam = $_.PolicyRule.then.effect
    }
    [PSCustomObject]@{
        DisplayName       = $_.DisplayName
        PolicyDefinitionId = $_.Id
        Effect            = $effectParam
        Description       = $_.Description
    }
} | Sort-Object Effect, DisplayName
```

Present the results grouped by effect type:

### Deny Policies
Policies that block non-compliant resource creation or modification.

| Display Name | Policy Definition ID | Description |
|---|---|---|
| _name_ | _/providers/Microsoft.Authorization/policyDefinitions/xxx_ | _description_ |

### Audit Policies
Policies that flag non-compliant resources without blocking them.

| Display Name | Policy Definition ID | Description |
|---|---|---|
| _name_ | _/providers/Microsoft.Authorization/policyDefinitions/xxx_ | _description_ |

### DeployIfNotExists / Modify Policies
Policies that automatically remediate non-compliant resources.

| Display Name | Policy Definition ID | Description |
|---|---|---|
| _name_ | _/providers/Microsoft.Authorization/policyDefinitions/xxx_ | _description_ |

### AuditIfNotExists Policies
Policies that audit for the absence of related resources.

| Display Name | Policy Definition ID | Description |
|---|---|---|
| _name_ | _/providers/Microsoft.Authorization/policyDefinitions/xxx_ | _description_ |

### Other / Disabled Policies
Any policies with other effects or that default to Disabled.

| Display Name | Policy Definition ID | Description |
|---|---|---|
| _name_ | _/providers/Microsoft.Authorization/policyDefinitions/xxx_ | _description_ |

### Summary
- Total built-in policies found for **<Service>**: **N**
- Deny: **N**
- Audit: **N**
- DeployIfNotExists / Modify: **N**
- AuditIfNotExists: **N**
- Other: **N**

## Step 4 — Offer next steps

After presenting the results, offer the user these options:

1. **Deep dive**: If the user wants more details on a specific policy, retrieve its full definition including all parameters, policy rule conditions, and any role definitions required.

   ```powershell
   Get-AzPolicyDefinition -Id '<policyDefinitionId>' | ConvertTo-Json -Depth 20
   ```

2. **Create EPAC objects**: If the user wants to assign or group any of the discovered policies, suggest using the `epac-policy-objects` skill. Provide the selected policy definition IDs in the format that skill expects:

   ```json
   [
     {
       "displayName": "Policy display name",
       "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/xxx"
     }
   ]
   ```

3. **Audit coverage**: If the user wants to check which of these policies are already assigned in their environment, suggest using the `azure-policy-audit` skill.

## Important Notes

- Only include **BuiltIn** policy definitions unless the user explicitly asks for Custom policies.
- Some policies have deprecated status in their metadata — call these out if found.
- Policy effects may be parameterized. Show the **default** effect value, and note when the effect can be changed via parameters.
- When a policy has multiple allowed effects (e.g. Audit, Deny, Disabled), list the default value as the primary effect.
