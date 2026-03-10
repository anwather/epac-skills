---
name: epac-policy-objects
description: Create Azure Policy objects in EPAC (Enterprise Policy as Code) format. Generates policy definitions, policy set definitions (initiatives), and policy assignments as JSON/JSONC files following EPAC conventions. Use this skill when asked to create, generate, or scaffold EPAC policy files, policy sets, assignments, or any Azure Policy as Code object.
---

# EPAC Policy Object Creation Skill

Use this skill to generate Azure Policy objects in EPAC format. EPAC organizes policy resources under a `Definitions/` folder structure:

```
Definitions/
├── global-settings.jsonc
├── policyDefinitions/{Category}/
├── policySetDefinitions/{Category}/
├── policyAssignments/
├── policyExemptions/
└── policyDocumentations/
```

Before creating any files, ask the user which `Definitions/` folder to write to if not already specified.

Before creating any files, ask the user which `Definitions/` folder to write to if not already specified.

---

## Object Type 1 — Custom Policy Definition

Create files under `Definitions/policyDefinitions/{Category}/` where `{Category}` matches the policy's metadata category (e.g. `Storage`, `Network`, `General`).

### Template

```json
{
    "$schema": "https://raw.githubusercontent.com/Azure/enterprise-azure-policy-as-code/main/Schemas/policy-definition-schema.json",
    "name": "<kebab-case-name>",
    "properties": {
        "displayName": "<Human readable name>",
        "description": "<What this policy does>",
        "mode": "All",
        "metadata": {
            "version": "1.0.0",
            "category": "<Category>"
        },
        "parameters": {
            "effect": {
                "type": "String",
                "metadata": {
                    "displayName": "Effect",
                    "description": "Enable or disable the execution of the policy"
                },
                "allowedValues": [
                    "<PrimaryEffect>",
                    "Disabled"
                ],
                "defaultValue": "<PrimaryEffect>"
            }
        },
        "policyRule": {
            "if": {
                "<condition>"
            },
            "then": {
                "effect": "[parameters('effect')]"
            }
        }
    }
}
```

### Rules

- **name**: Use kebab-case. Convention is `{Effect}-{Service}-{Feature}` (e.g. `Deny-Storage-minTLS`).
- **mode**: Use `All` for resource properties, `Indexed` for tags/location only.
- **effect parameter**: Always parameterize the effect. The `then.effect` MUST be `"[parameters('effect')]"`.
- **allowedValues** for effect must use valid values: `Disabled`, `Audit`, `Deny`, `Modify`, `Append`, `AuditIfNotExists`, `DeployIfNotExists`, `DenyAction`, `Manual`.
- **metadata.version**: Start at `1.0.0`.
- **metadata.category**: Must match the subfolder name.
- Additional parameters beyond `effect` are allowed — use camelCase names.
- For `DeployIfNotExists` or `Modify` effects, include the `details` block inside `then` with the required `roleDefinitionIds`, `deployment`, or `operations`.

### Example — Deny Storage accounts without minimum TLS

```json
{
    "$schema": "https://raw.githubusercontent.com/Azure/enterprise-azure-policy-as-code/main/Schemas/policy-definition-schema.json",
    "name": "Deny-Storage-minTLS",
    "properties": {
        "displayName": "Storage accounts should have the specified minimum TLS version",
        "description": "Deny storage accounts that do not meet the minimum TLS version requirement.",
        "mode": "All",
        "metadata": {
            "version": "1.0.0",
            "category": "Storage"
        },
        "parameters": {
            "effect": {
                "type": "String",
                "metadata": {
                    "displayName": "Effect",
                    "description": "Enable or disable the execution of the policy"
                },
                "allowedValues": ["Deny", "Audit", "Disabled"],
                "defaultValue": "Deny"
            },
            "minimumTlsVersion": {
                "type": "String",
                "metadata": {
                    "displayName": "Minimum TLS Version",
                    "description": "The minimum TLS version required"
                },
                "allowedValues": ["TLS1_0", "TLS1_1", "TLS1_2"],
                "defaultValue": "TLS1_2"
            }
        },
        "policyRule": {
            "if": {
                "allOf": [
                    {
                        "field": "type",
                        "equals": "Microsoft.Storage/storageAccounts"
                    },
                    {
                        "field": "Microsoft.Storage/storageAccounts/minimumTlsVersion",
                        "notEquals": "[parameters('minimumTlsVersion')]"
                    }
                ]
            },
            "then": {
                "effect": "[parameters('effect')]"
            }
        }
    }
}
```

---

## Object Type 2 — Policy Set Definition (Initiative)

Create files under `Definitions/policySetDefinitions/{Category}/`.

### Template

```json
{
    "$schema": "https://raw.githubusercontent.com/Azure/enterprise-azure-policy-as-code/main/Schemas/policy-set-definition-schema.json",
    "name": "<kebab-case-name>",
    "properties": {
        "displayName": "<Human readable name>",
        "description": "<What this initiative does>",
        "metadata": {
            "version": "1.0.0",
            "category": "<Category>"
        },
        "parameters": {
            "<paramName>": {
                "type": "String",
                "metadata": {
                    "displayName": "<Display Name>",
                    "description": "<Description>"
                },
                "defaultValue": "<default>"
            }
        },
        "policyDefinitions": [
            {
                "policyDefinitionReferenceId": "<unique-reference-id>",
                "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/<guid>",
                "parameters": {
                    "effect": {
                        "value": "[parameters('<paramName>')]"
                    }
                }
            }
        ]
    }
}
```

### Rules

- **name**: Use kebab-case. Convention is `{Effect}-Guardrails-{Category}` for guardrail sets (e.g. `Enforce-Guardrails-Storage`).
- **policyDefinitions[]**: Each member MUST have a unique `policyDefinitionReferenceId`.
- To reference a **built-in** policy, use `policyDefinitionId` with the full resource ID (e.g. `/providers/Microsoft.Authorization/policyDefinitions/<guid>`).
- To reference a **custom** EPAC policy, use `policyDefinitionName` with the `name` from the policy definition file (e.g. `Deny-Storage-minTLS`).
- **Parameters**: Define parameters at the set level and pass them through to member policies using `"value": "[parameters('<name>')]"`.
- A common pattern is to have one effect parameter per member policy (e.g. `storageMinTlsEffect`, `storagePublicAccessEffect`) so each can be controlled independently.
- `definitionVersion` can optionally pin a built-in policy version (e.g. `"1.*.*"`).

### Example — Storage guardrails initiative

```json
{
    "$schema": "https://raw.githubusercontent.com/Azure/enterprise-azure-policy-as-code/main/Schemas/policy-set-definition-schema.json",
    "name": "Enforce-Guardrails-Storage",
    "properties": {
        "displayName": "Enforce recommended guardrails for Storage",
        "description": "This policy initiative enforces recommended guardrails for Azure Storage accounts.",
        "metadata": {
            "version": "1.0.0",
            "category": "Storage"
        },
        "parameters": {
            "storageMinTlsEffect": {
                "type": "String",
                "metadata": {
                    "displayName": "Effect - Minimum TLS version",
                    "description": "Effect for minimum TLS version policy"
                },
                "allowedValues": ["Deny", "Audit", "Disabled"],
                "defaultValue": "Deny"
            },
            "storageSecureTransferEffect": {
                "type": "String",
                "metadata": {
                    "displayName": "Effect - Secure transfer",
                    "description": "Effect for secure transfer policy"
                },
                "allowedValues": ["Deny", "Audit", "Disabled"],
                "defaultValue": "Deny"
            }
        },
        "policyDefinitions": [
            {
                "policyDefinitionReferenceId": "StorageMinTls",
                "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/fe83a0eb-a853-422d-aac2-1bffd182c5d0",
                "parameters": {
                    "effect": {
                        "value": "[parameters('storageMinTlsEffect')]"
                    }
                }
            },
            {
                "policyDefinitionReferenceId": "StorageSecureTransfer",
                "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/404c3081-a854-4457-ae30-26a93ef643f9",
                "parameters": {
                    "effect": {
                        "value": "[parameters('storageSecureTransferEffect')]"
                    }
                }
            }
        ]
    }
}
```

---

## Object Type 3 — Policy Assignment

Create files under `Definitions/policyAssignments/`. Assignments use a recursive tree structure with `children[]` for scope inheritance.

### Template — Simple (single scope)

```jsonc
{
    "$schema": "https://raw.githubusercontent.com/Azure/enterprise-azure-policy-as-code/main/Schemas/policy-assignment-schema.json",
    "nodeName": "<path/assignment-name>",
    "assignment": {
        "name": "<max-24-char-name>",
        "displayName": "<Human readable name>",
        "description": "<What this assignment does>"
    },
    "definitionEntry": {
        "policySetName": "<custom-set-name>",
        "displayName": "<Display name for reference>"
    },
    "enforcementMode": "Default",
    "parameters": {
        "<paramName>": "<value>"
    },
    "nonComplianceMessages": [
        {
            "message": "<Message shown when non-compliant>"
        }
    ],
    "scope": {
        "<pacSelector>": [
            "/providers/Microsoft.Management/managementGroups/<mg-name>"
        ]
    }
}
```

### Template — Tree structure (multiple scopes/policies)

```jsonc
{
    "$schema": "https://raw.githubusercontent.com/Azure/enterprise-azure-policy-as-code/main/Schemas/policy-assignment-schema.json",
    "nodeName": "root/",
    "scope": {
        "<pacSelector>": [
            "/providers/Microsoft.Management/managementGroups/<mg-name>"
        ]
    },
    "children": [
        {
            "nodeName": "child-assignment-1",
            "assignment": {
                "name": "<max-24-char>",
                "displayName": "<Display Name>",
                "description": "<Description>"
            },
            "definitionEntry": {
                "policySetId": "/providers/Microsoft.Authorization/policySetDefinitions/<guid>",
                "displayName": "<Display name>"
            },
            "parameters": {},
            "nonComplianceMessages": [
                {
                    "message": "<Non-compliance message>"
                }
            ]
        },
        {
            "nodeName": "child-assignment-2",
            "assignment": {
                "name": "<max-24-char>",
                "displayName": "<Display Name>",
                "description": "<Description>"
            },
            "definitionEntry": {
                "policyName": "<custom-policy-name>"
            },
            "parameters": {}
        }
    ]
}
```

### Rules

- **assignment.name**: Maximum 24 characters. Must be unique within the scope.
- **nodeName**: Hierarchical path describing the assignment tree node (e.g. `platform/security`).
- **definitionEntry**: Use exactly ONE of these reference types:
  - `policyId` — full resource ID of a built-in policy definition
  - `policySetId` — full resource ID of a built-in policy set definition
  - `policySetName` — name of a custom EPAC policy set definition
  - `policyName` — name of a custom EPAC policy definition
- **scope**: Object keyed by `pacSelector` values from `global-settings.jsonc`. Each value is an array of management group, subscription, or resource group scopes.
- **enforcementMode**: `"Default"` (enforce) or `"DoNotEnforce"` (audit only).
- **parameters**: Flat object of parameter name/value pairs. Parameters set at parent nodes are inherited by all `children`.
- **notScopes**: Object keyed by `pacSelector`. Each value is an array of scopes to exclude.
- **children[]**: Array of child nodes. Each child inherits `scope`, `notScopes`, and `parameters` from the parent and can override or extend them.
- **nonComplianceMessages**: Array of `{ "message": "..." }` objects. For policy sets, add `"policyDefinitionReferenceId"` to target a specific member policy.
- **managedIdentityLocations**: Required when assigning `DeployIfNotExists` or `Modify` policies. Object keyed by `pacSelector` with Azure region as the value.
- **additionalRoleAssignments**: Required when the managed identity needs roles beyond what the policy defines.

### Example — Assign a custom initiative to a management group

```jsonc
{
    "$schema": "https://raw.githubusercontent.com/Azure/enterprise-azure-policy-as-code/main/Schemas/policy-assignment-schema.json",
    "nodeName": "storage/guardrails",
    "assignment": {
        "name": "Enf-GR-Storage",
        "displayName": "Enforce Storage Guardrails",
        "description": "Assigns the Storage guardrails initiative to enforce security best practices."
    },
    "definitionEntry": {
        "policySetName": "Enforce-Guardrails-Storage",
        "displayName": "Enforce recommended guardrails for Storage"
    },
    "enforcementMode": "Default",
    "parameters": {
        "storageMinTlsEffect": "Deny",
        "storageSecureTransferEffect": "Deny"
    },
    "nonComplianceMessages": [
        {
            "message": "Storage guardrails must be enforced. See https://aka.ms/yourwiki for details."
        }
    ],
    "scope": {
        "test": [
            "/providers/Microsoft.Management/managementGroups/alz-landingzones"
        ]
    }
}
```

---

## Workflow Guidance

When the user asks to create EPAC policy objects, follow this process:

1. **Clarify the request**: Determine which object type(s) are needed (definition, set, assignment).
2. **Identify the target Definitions folder**: Ask the user which `Definitions/` directory to write files to.
3. **Gather inputs**:
   - For **definitions**: What should the policy enforce? Which resource type and properties? What effect?
   - For **sets**: Which policies to include? Built-in (provide IDs from the azure-policy-audit skill) or custom?
   - For **assignments**: Which policy/set to assign? What scope (management group, subscription)? What `pacSelector` from their global settings?
4. **Generate the JSON file(s)**: Use the templates and rules above. Validate naming conventions and required fields.
5. **Write to disk**: Save files to the correct subfolder under the user's `Definitions/` directory.
6. **Remind the user**: After creating files, remind them to run `Build-DeploymentPlans` to validate before deploying.

### Integration with azure-policy-audit Skill

When building a policy set from unassigned built-in policies discovered by the `azure-policy-audit` skill:
1. Take the list of `policyDefinitionId` values from the audit results.
2. Generate a policy set definition with each policy as a member.
3. Create a corresponding assignment file.
4. Use individual effect parameters (one per member policy) for granular control.
