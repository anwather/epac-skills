---
name: epac-test-fixtures
description: Scaffold a ready-to-deploy set of EPAC policy artifacts (custom initiative, built-in assignments, and a varied set of exemptions) into an existing EPAC repo so you can reproduce, exercise, or smoke-test EPAC behavior. Use this skill when asked to populate, seed, scaffold, or generate test fixtures, sample artifacts, smoke-test content, or a "demo" set of policies/assignments/exemptions in an EPAC repo — especially when validating an EPAC code change, a new switch, or a workflow end-to-end.
---

# EPAC Test Fixtures Skill

Use this skill to populate an existing (but otherwise empty) EPAC repository with a small, coherent set of policy artifacts that can be deployed against any Azure management group hierarchy without needing Log Analytics, Guest Configuration, or any other resource-side dependencies. This is the fast path for reproducing EPAC issues, validating a new EPAC feature, or smoke-testing the build/deploy/export pipeline.

The skill produces:

- 1 **custom policy set definition** (a small Storage hardening initiative) — so downstream exemptions can target `policyDefinitionReferenceIds`.
- 4 **policy assignments** (1 of the custom initiative + 3 built-ins for NSG-on-subnet, KV soft-delete, KV purge protection).
- 5 **policy exemptions** deliberately varied across every axis EPAC cares about: `Waiver` vs `Mitigated`, with/without `expiresOn`, with/without `description`, with/without `policyDefinitionReferenceIds`, with/without `resourceSelectors`, with/without `assignmentScopeValidation`, single-scope vs `scopes` array, minimal vs rich metadata.

All built-in policies used are **Audit-only by default** and have no dependency on Azure-side resources (no Log Analytics workspace, no extensions, no managed identity required).

---

## Step 0 — Detect EPAC repo context

Before creating any files, detect the EPAC repository structure automatically — identical to the pattern used by the other EPAC skills.

### Locate the Definitions folder

Search the current working directory and its children for a `Definitions/` folder containing a `global-settings.jsonc` file (glob `**/Definitions/global-settings.jsonc`). If found, use that `Definitions/` folder as the target — do NOT prompt the user.

If no `Definitions/` folder is found, ask the user where to scaffold the fixtures.

### Read global-settings.jsonc

Parse `global-settings.jsonc` and extract:

- **pacSelector**: the name of each entry in `pacEnvironments[]`.
- **deploymentRootScope**: the root scope of each entry.

### Pac selector selection logic

- If there is **exactly one** `pacEnvironment` entry, use that pac selector automatically — do NOT prompt.
- If there are **multiple**, ask the user which pac selector the fixtures should target.
- Use that environment's `deploymentRootScope` as the scope for every assignment and exemption.

---

## Step 1 — Pre-flight checks

Before writing anything, check for collisions:

1. List existing files under `Definitions/policyAssignments/`, `Definitions/policySetDefinitions/`, and `Definitions/policyExemptions/<pacSelector>/`.
2. If the repo already contains assignments or exemptions, **ask the user** whether to:
   - Add the fixtures alongside existing content (default), or
   - Stop and let them clean up first.
3. Never overwrite an existing file with the same name without confirmation.

If any of the fixture filenames already exist, append a numeric suffix (e.g. `-2`, `-3`) to the names being created so the existing files are untouched.

---

## Step 2 — Generate the fixtures

Use the exact templates below. Replace `<pacSelector>` with the resolved selector and `<deploymentRootScope>` with the full scope string (e.g. `/providers/Microsoft.Management/managementGroups/issue1278`).

Create folders as needed.

### 2a — Custom policy set definition

**Path:** `Definitions/policySetDefinitions/Storage/Enforce-Guardrails-Storage.jsonc`

```jsonc
{
    "$schema": "https://raw.githubusercontent.com/Azure/enterprise-azure-policy-as-code/main/Schemas/policy-set-definition-schema.json",
    "name": "Enforce-Guardrails-Storage",
    "properties": {
        "displayName": "Enforce recommended guardrails for Storage",
        "description": "Bundles a small set of storage account hardening policies. Used for exercising EPAC flows in tests.",
        "metadata": {
            "version": "1.0.0",
            "category": "Storage"
        },
        "parameters": {
            "storageSecureTransferEffect": {
                "type": "String",
                "metadata": {
                    "displayName": "Effect - Secure transfer",
                    "description": "Effect for the secure transfer policy"
                },
                "allowedValues": ["Audit", "Deny", "Disabled"],
                "defaultValue": "Audit"
            },
            "storagePublicAccessEffect": {
                "type": "String",
                "metadata": {
                    "displayName": "Effect - Public network access",
                    "description": "Effect for the storage public network access policy"
                },
                "allowedValues": ["Audit", "Deny", "Disabled"],
                "defaultValue": "Audit"
            },
            "storageMinTlsEffect": {
                "type": "String",
                "metadata": {
                    "displayName": "Effect - Minimum TLS version",
                    "description": "Effect for the minimum TLS version policy"
                },
                "allowedValues": ["Audit", "Deny", "Disabled"],
                "defaultValue": "Audit"
            }
        },
        "policyDefinitions": [
            {
                "policyDefinitionReferenceId": "StorageSecureTransfer",
                "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/404c3081-a854-4457-ae30-26a93ef643f9",
                "parameters": {
                    "effect": { "value": "[parameters('storageSecureTransferEffect')]" }
                }
            },
            {
                "policyDefinitionReferenceId": "StoragePublicAccessDisallowed",
                "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/4fa4b6c0-31ca-4c0d-b10d-24b96f62a751",
                "parameters": {
                    "effect": { "value": "[parameters('storagePublicAccessEffect')]" }
                }
            },
            {
                "policyDefinitionReferenceId": "StorageMinTls",
                "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/fe83a0eb-a853-422d-aac2-1bffd182c5d0",
                "parameters": {
                    "effect": { "value": "[parameters('storageMinTlsEffect')]" }
                }
            }
        ]
    }
}
```

### 2b — Policy assignments

All four assignments use `<deploymentRootScope>` as scope, `enforcementMode: Default`, and `Audit` effects. Assignment names are short (≤24 chars) and unique.

**Path:** `Definitions/policyAssignments/storage-guardrails-assignment.jsonc`

```jsonc
{
    "$schema": "https://raw.githubusercontent.com/Azure/enterprise-azure-policy-as-code/main/Schemas/policy-assignment-schema.json",
    "nodeName": "storage/guardrails",
    "assignment": {
        "name": "enf-gr-storage",
        "displayName": "Enforce Storage Guardrails",
        "description": "Assigns the custom Storage guardrails initiative."
    },
    "definitionEntry": {
        "policySetName": "Enforce-Guardrails-Storage",
        "displayName": "Enforce recommended guardrails for Storage"
    },
    "enforcementMode": "Default",
    "parameters": {
        "storageSecureTransferEffect": "Audit",
        "storagePublicAccessEffect": "Audit",
        "storageMinTlsEffect": "Audit"
    },
    "nonComplianceMessages": [
        { "message": "Storage accounts must comply with the baseline storage guardrails." }
    ],
    "scope": {
        "<pacSelector>": ["<deploymentRootScope>"]
    }
}
```

**Path:** `Definitions/policyAssignments/network-nsg-subnet-assignment.jsonc`

```jsonc
{
    "$schema": "https://raw.githubusercontent.com/Azure/enterprise-azure-policy-as-code/main/Schemas/policy-assignment-schema.json",
    "nodeName": "network/nsg-on-subnet",
    "assignment": {
        "name": "audit-nsg-subnet",
        "displayName": "Audit subnets without an NSG",
        "description": "Audits virtual network subnets that are not associated with a Network Security Group."
    },
    "definitionEntry": {
        "policyId": "/providers/Microsoft.Authorization/policyDefinitions/e71308d3-144b-4262-b144-efdc3cc90517",
        "displayName": "Subnets should be associated with a Network Security Group"
    },
    "enforcementMode": "Default",
    "parameters": { "effect": "Audit" },
    "scope": {
        "<pacSelector>": ["<deploymentRootScope>"]
    }
}
```

**Path:** `Definitions/policyAssignments/keyvault-softdelete-assignment.jsonc`

```jsonc
{
    "$schema": "https://raw.githubusercontent.com/Azure/enterprise-azure-policy-as-code/main/Schemas/policy-assignment-schema.json",
    "nodeName": "keyvault/soft-delete",
    "assignment": {
        "name": "audit-kv-softdelete",
        "displayName": "Audit Key Vaults without soft-delete",
        "description": "Audits Key Vaults that do not have soft-delete enabled."
    },
    "definitionEntry": {
        "policyId": "/providers/Microsoft.Authorization/policyDefinitions/1e66c121-a66a-4b1f-9b83-0fd99bf0fc2d",
        "displayName": "Key vaults should have soft delete enabled"
    },
    "enforcementMode": "Default",
    "parameters": { "effect": "Audit" },
    "scope": {
        "<pacSelector>": ["<deploymentRootScope>"]
    }
}
```

**Path:** `Definitions/policyAssignments/keyvault-purge-assignment.jsonc`

```jsonc
{
    "$schema": "https://raw.githubusercontent.com/Azure/enterprise-azure-policy-as-code/main/Schemas/policy-assignment-schema.json",
    "nodeName": "keyvault/purge-protection",
    "assignment": {
        "name": "audit-kv-purge",
        "displayName": "Audit Key Vaults without purge protection",
        "description": "Audits Key Vaults that do not have purge protection enabled."
    },
    "definitionEntry": {
        "policyId": "/providers/Microsoft.Authorization/policyDefinitions/0b60c0b2-2dc2-4e1c-b5c9-abbed971de53",
        "displayName": "Key vaults should have purge protection enabled"
    },
    "enforcementMode": "Default",
    "parameters": { "effect": "Audit" },
    "scope": {
        "<pacSelector>": ["<deploymentRootScope>"]
    }
}
```

### 2c — Exemptions (variety bundle)

All five exemptions go in a single file. Each exemption targets one of the assignments above using its full `policyAssignmentId` built from `<deploymentRootScope>` + `/providers/Microsoft.Authorization/policyAssignments/<assignment.name>`.

The variety is deliberate — each entry covers a different combination of optional fields so EPAC code paths that branch on them are all exercised:

| # | Name | exemptionCategory | expiresOn | description | policyDefinitionReferenceIds | resourceSelectors | assignmentScopeValidation | Scope shape | Metadata |
|---|------|-------------------|-----------|-------------|------------------------------|-------------------|---------------------------|-------------|----------|
| 1 | `exempt-storage-waiver-full` | Waiver | future | yes | — | — | — | `scope` | rich |
| 2 | `exempt-storage-mitigated-refids` | Mitigated | — | yes | yes (2 refs) | yes (resourceLocation) | — | `scope` | yes |
| 3 | `exempt-kv-softdelete-min` | Waiver | — | — | — | — | — | `scope` | — |
| 4 | `exempt-nsg-subnet-scope-validation` | Mitigated | future | yes | — | — | `DoNotValidate` | `scope` | yes |
| 5 | `exempt-kv-purge-multi-scope` | Waiver | future | `null` | — | — | — | `scopes[]` | yes |

**Path:** `Definitions/policyExemptions/<pacSelector>/exemptions.jsonc`

```jsonc
{
    "$schema": "https://raw.githubusercontent.com/Azure/enterprise-azure-policy-as-code/main/Schemas/policy-exemption-schema.json",
    "exemptions": [
        {
            "name": "exempt-storage-waiver-full",
            "displayName": "Waiver: Storage guardrails (full metadata, future expiry)",
            "description": "Approved waiver for the entire Storage guardrails initiative while the platform team migrates legacy storage accounts to compliant configuration.",
            "exemptionCategory": "Waiver",
            "expiresOn": "2099-12-31T23:59:59Z",
            "scope": "<deploymentRootScope>",
            "policyAssignmentId": "<deploymentRootScope>/providers/Microsoft.Authorization/policyAssignments/enf-gr-storage",
            "metadata": {
                "requestedBy": "platform-team",
                "ticket": "CR-12345",
                "approvedBy": "compliance-lead"
            }
        },
        {
            "name": "exempt-storage-mitigated-refids",
            "displayName": "Mitigated: Storage TLS + Public access (selected refs)",
            "description": "Compensating controls in place at the network layer; exempt only the TLS and public access member policies of the Storage guardrails initiative.",
            "exemptionCategory": "Mitigated",
            "scope": "<deploymentRootScope>",
            "policyAssignmentId": "<deploymentRootScope>/providers/Microsoft.Authorization/policyAssignments/enf-gr-storage",
            "policyDefinitionReferenceIds": [
                "StorageMinTls",
                "StoragePublicAccessDisallowed"
            ],
            "resourceSelectors": [
                {
                    "name": "RegionalRollout",
                    "selectors": [
                        {
                            "kind": "resourceLocation",
                            "in": ["australiaeast", "australiasoutheast"]
                        }
                    ]
                }
            ],
            "metadata": {
                "compensatingControl": "Private endpoints + Azure Firewall egress allowlist",
                "owner": "network-team"
            }
        },
        {
            "name": "exempt-kv-softdelete-min",
            "displayName": "Minimal waiver: Key Vault soft delete",
            "exemptionCategory": "Waiver",
            "scope": "<deploymentRootScope>",
            "policyAssignmentId": "<deploymentRootScope>/providers/Microsoft.Authorization/policyAssignments/audit-kv-softdelete"
        },
        {
            "name": "exempt-nsg-subnet-scope-validation",
            "displayName": "Mitigated: NSG audit with DoNotValidate scope",
            "description": "Subnet sits on a hub vNet where NSG is intentionally not applied; rely on Azure Firewall instead.",
            "exemptionCategory": "Mitigated",
            "expiresOn": "2099-06-30T00:00:00Z",
            "scope": "<deploymentRootScope>",
            "policyAssignmentId": "<deploymentRootScope>/providers/Microsoft.Authorization/policyAssignments/audit-nsg-subnet",
            "assignmentScopeValidation": "DoNotValidate",
            "metadata": {
                "compensatingControl": "Azure Firewall in hub vNet",
                "owner": "network-team",
                "reviewDate": "2026-12-01"
            }
        },
        {
            "name": "exempt-kv-purge-multi-scope",
            "displayName": "Waiver: KV purge protection (multi-scope)",
            "description": null,
            "exemptionCategory": "Waiver",
            "expiresOn": "2099-03-31T00:00:00Z",
            "scopes": ["<deploymentRootScope>"],
            "policyAssignmentId": "<deploymentRootScope>/providers/Microsoft.Authorization/policyAssignments/audit-kv-purge",
            "metadata": {
                "owner": "kv-platform",
                "businessJustification": "Short-lived dev key vaults are recreated on every pipeline run."
            }
        }
    ]
}
```

---

## Step 3 — Subsetting the fixtures (optional)

If the user asks for only a subset (for example: "just create some exemptions" or "I only need assignments"), generate only those parts but keep the dependency chain intact:

- Exemptions reference assignments — if you create exemption #1 or #2 you must also create the `enf-gr-storage` assignment **and** the `Enforce-Guardrails-Storage` set definition.
- Exemptions #3, #4, #5 only require the matching built-in assignments.
- Assignments have no inbound dependencies.

If the user asks for *additional* variety, prefer adjusting the existing exemptions list (e.g. add another `exempt-...-already-expired` row with an `expiresOn` in the past) over inventing more assignments.

---

## Step 4 — Build the deployment plan

After writing the files, automatically hand off to the **`epac-build-deployment-plan`** skill (or run `Build-DeploymentPlans` directly if that skill is not available) so the user sees end-to-end that the fixtures are valid. Use the same `<pacSelector>` and the repo's `Definitions/` folder. The expected summary is:

- 1 new policy set definition
- 4 new policy assignments
- 5 new policy exemptions
- 0 role assignments

> Note: `Build-DeploymentPlans -Interactive` is a `[switch]`, **not** `[bool]`. Use it as `-Interactive` (or omit it entirely for a scripted run). Passing `-Interactive $false` will fail with a positional-parameter error.

If the build fails:
- Authentication errors → suggest `Connect-AzAccount -TenantId <tenantId from global-settings>`.
- "scope not found" or exemption-validation errors on the multi-scope or `DoNotValidate` exemption → confirm `skipResourceValidationForExemptions` setting in `global-settings.jsonc`; offer to set it to `true` for fast local iteration.

---

## Step 5 — Tell the user what they can do next

After a successful build, hand back:

1. Where files were created (use repo-relative paths).
2. Where the plan file landed (`<Outputs>/plans-<pacSelector>/policy-plan.json`).
3. The deploy commands they would run if they actually want resources in Azure:

   ```powershell
   Deploy-PolicyPlan       -PacEnvironmentSelector "<pacSelector>" -DefinitionsRootFolder "./Definitions" -InputFolder "./Outputs"
   Deploy-RolesPlan        -PacEnvironmentSelector "<pacSelector>" -DefinitionsRootFolder "./Definitions" -InputFolder "./Outputs"
   ```

4. Suggested follow-up commands once deployed, depending on what they're testing — for example:

   ```powershell
   # Round-trip exemptions back into EPAC source via the new -ExportForEpac flag (issue #1278)
   .\Scripts\Operations\Get-AzExemptions.ps1 -PacEnvironmentSelector "<pacSelector>" -ExportForEpac
   ```

---

## Rules

- **Never** add policies that depend on Log Analytics workspaces, Guest Configuration / Arc, Defender plans, or any other resource-side prerequisite. Audit/Deny on resource properties only.
- **Never** include `DeployIfNotExists` or `Modify` policies in the fixtures — they require managed identities and role assignments and complicate the smoke-test.
- Assignment `name` values must be ≤24 characters.
- Always use the resolved `<pacSelector>` from `global-settings.jsonc`. Do not invent one.
- Always use the resolved `<deploymentRootScope>` for assignment scope **and** as the prefix for every exemption's `policyAssignmentId`.
- Do not modify `global-settings.jsonc` without explicit consent.
- The fixtures are **smoke-test content**, not a recommendation. Do not embed business-specific policies, custom domain names, or production scopes.

---

## Example interaction

> "Populate this empty EPAC repo so I can deploy and then re-test the new exemption export flag."

1. Detect `./Definitions/global-settings.jsonc`, single pac selector `test`, root `/providers/Microsoft.Management/managementGroups/issue1278`.
2. Confirm no existing assignments/exemptions, then write the 1 set + 4 assignments + 1 exemptions file.
3. Run `Build-DeploymentPlans -PacEnvironmentSelector test -DefinitionsRootFolder ./Definitions -OutputFolder ./Outputs`.
4. Report the new counts (1 / 4 / 5), show plan path, and remind the user of the deploy + `Get-AzExemptions -ExportForEpac` follow-up.
