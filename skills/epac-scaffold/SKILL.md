---
name: epac-scaffold
description: Scaffold a complete EPAC (Enterprise Policy as Code) Definitions folder and safe test PAC environment. Use when asked to initialize, create, bootstrap, or scaffold an EPAC repository, Definitions structure, global-settings.jsonc, or starter folder layout.
---

# EPAC Scaffold Skill

Create a complete, idempotent EPAC `Definitions` structure without changing existing content. Prefer safe inference for troubleshooting work and prompt only when the target or desired scope is genuinely ambiguous.

## Step 0 - Resolve the target

Resolve the `Definitions` root before gathering any other input.

1. If the user supplied a target path, use it without searching.
2. Otherwise, search the current working directory and its descendants for `Definitions/global-settings.jsonc`.
3. If exactly one is found, use its parent directory automatically.
4. If none are found, use `Definitions` under the current working directory.
5. If more than one is found, ask the user to select a target. Do not guess between existing EPAC roots.

Resolve the target to an absolute path. If the target exists as a file rather than a directory, stop without modifying it.

## Step 1 - Resolve inputs without unnecessary prompts

Gather these values before writing. Do not prompt for values that can be safely inferred.

| Input | Resolution order |
|---|---|
| **Target path** | Explicit value; one discovered `Definitions/global-settings.jsonc`; `<cwd>\Definitions`; otherwise ask because discovery was ambiguous |
| **deploymentRootScope** | Explicit value; existing `global-settings.jsonc` for the environment in use; authenticated Az subscription scope; placeholder with warning |
| **managedIdentityLocation** | Explicit value; otherwise `australiaeast` with a report |
| **tenantId** | Existing settings are preserved; for a new file, authenticated `Get-AzContext` tenant; otherwise zero-GUID placeholder with warning |
| **pacSelector** | `test` for a new scaffold; an existing selector when operating on an existing environment |
| **Troubleshooting fixtures** | Opt-in only. Never add policy fixtures unless the user explicitly requests them |

For an authenticated current Az subscription, derive the fallback scope as:

```text
/subscriptions/<current-subscription-id>
```

Do not run `Connect-AzAccount` to obtain missing values. Lack of an authenticated context is not an error.

If `global-settings.jsonc` already exists:

- Preserve it byte-for-byte.
- Do not rewrite any resolved values into it.
- Read it only to identify existing selectors and deployment scopes for reporting or a later operation.
- If it contains exactly one PAC environment, use that environment automatically.
- If it contains multiple PAC environments and a later operation needs one, ask the user to select a `pacSelector`.

For a new file, use a subscription scope when the active Az context provides one. A full management group scope remains the right explicit choice when the troubleshooting work must cover more than the active subscription.

## Required structure

Create every missing directory:

```text
Definitions/
├── policyDefinitions/
├── policySetDefinitions/
├── policyAssignments/
├── policyExemptions/
├── policyEnrollments/
├── policyDocumentations/
```

Do not delete, rename, empty, or otherwise modify existing directories or files.

## Create global-settings.jsonc

The file is `<target>\global-settings.jsonc`.

1. Check for the file before generating content.
2. If it exists, preserve it byte-for-byte. Never overwrite, merge, repair, or replace it.
3. If it does not exist:
   - Generate a new `pacOwnerId` with `[guid]::NewGuid()`.
   - If `Get-AzContext` is available, call it without attempting authentication and use its current `Tenant.Id` when present.
   - If there is no authenticated context, Az PowerShell is unavailable, or the lookup fails, use `00000000-0000-0000-0000-000000000000` and report that the tenant placeholder must be replaced.
   - Create exactly one PAC environment with `pacSelector` set to `test`, `cloud` set to `AzureCloud`, and `desiredState.strategy` set to `ownedOnly`.

Use this schema-compatible shape:

```jsonc
{
    "$schema": "https://raw.githubusercontent.com/Azure/enterprise-azure-policy-as-code/main/Schemas/global-settings-schema.json",
    "pacOwnerId": "<new-guid>",
    "pacEnvironments": [
        {
            "pacSelector": "test",
            "cloud": "AzureCloud",
            "tenantId": "<current-tenant-id-or-placeholder>",
            "deploymentRootScope": "<provided-scope-or-placeholder>",
            "desiredState": {
                "strategy": "ownedOnly",
                "keepDfcSecurityAssignments": true,
                "doNotDisableDeprecatedPolicies": false
            },
            "globalNotScopes": [],
            "managedIdentityLocation": "<provided-location-or-australiaeast>"
        }
    ]
}
```

`ownedOnly` is required for this starter environment so EPAC does not assume ownership of unrelated policy resources. The `test` selector and `australiaeast` location are stable troubleshooting defaults, not evidence that an Azure deployment has already occurred.

## Execution pattern

Use the following PowerShell pattern, adapting the parameter values from the gathered inputs:

```powershell
param(
    [string] $TargetPath,
    [string] $DeploymentRootScope,
    [string] $ManagedIdentityLocation
)

$created = [System.Collections.Generic.List[string]]::new()
$existing = [System.Collections.Generic.List[string]]::new()
$skipped = [System.Collections.Generic.List[string]]::new()
$warnings = [System.Collections.Generic.List[string]]::new()

if ([string]::IsNullOrWhiteSpace($TargetPath)) {
    $candidates = @(
        Get-ChildItem -Path (Get-Location) -Recurse -File `
            -Filter "global-settings.jsonc" -ErrorAction SilentlyContinue |
            Where-Object { $_.Directory.Name -eq "Definitions" }
    )

    if ($candidates.Count -eq 1) {
        $TargetPath = $candidates[0].Directory.FullName
    }
    elseif ($candidates.Count -gt 1) {
        throw "Multiple Definitions roots were found. Supply TargetPath to select one."
    }
    else {
        $TargetPath = Join-Path (Get-Location) "Definitions"
    }
}
$definitionsRoot = [System.IO.Path]::GetFullPath($TargetPath)

if (Test-Path -LiteralPath $definitionsRoot -PathType Leaf) {
    throw "Target path is an existing file: $definitionsRoot"
}

$directories = @(
    $definitionsRoot
    (Join-Path $definitionsRoot "policyDefinitions")
    (Join-Path $definitionsRoot "policySetDefinitions")
    (Join-Path $definitionsRoot "policyAssignments")
    (Join-Path $definitionsRoot "policyExemptions")
    (Join-Path $definitionsRoot "policyEnrollments")
    (Join-Path $definitionsRoot "policyDocumentations")
)

foreach ($directory in $directories) {
    if (Test-Path -LiteralPath $directory -PathType Container) {
        $existing.Add($directory)
    }
    else {
        $null = New-Item -ItemType Directory -Path $directory -Force
        $created.Add($directory)
    }
}

$tenantId = $null
$subscriptionId = $null
if (Get-Command -Name Get-AzContext -ErrorAction SilentlyContinue) {
    try {
        $context = Get-AzContext -ErrorAction Stop
        if ($null -ne $context -and $null -ne $context.Tenant -and $context.Tenant.Id) {
            $tenantId = [string] $context.Tenant.Id
        }
        if ($null -ne $context -and $null -ne $context.Subscription -and $context.Subscription.Id) {
            $subscriptionId = [string] $context.Subscription.Id
        }
    }
    catch {
        $warnings.Add("Get-AzContext failed; Azure-derived values will not be used. $($_.Exception.Message)")
    }
}

if ([string]::IsNullOrWhiteSpace($DeploymentRootScope)) {
    if (-not [string]::IsNullOrWhiteSpace($subscriptionId)) {
        $DeploymentRootScope = "/subscriptions/$subscriptionId"
        $warnings.Add("deploymentRootScope was inferred from the current Az subscription.")
    }
    else {
        $DeploymentRootScope = "/providers/Microsoft.Management/managementGroups/REPLACE-WITH-MANAGEMENT-GROUP-ID"
        $warnings.Add("deploymentRootScope uses a placeholder and must be replaced.")
    }
}

if ([string]::IsNullOrWhiteSpace($ManagedIdentityLocation)) {
    $ManagedIdentityLocation = "australiaeast"
    $warnings.Add("managedIdentityLocation was omitted; using australiaeast.")
}

if ([string]::IsNullOrWhiteSpace($tenantId)) {
    $tenantId = "00000000-0000-0000-0000-000000000000"
    $warnings.Add("tenantId uses a placeholder because no authenticated Az context was available.")
}

$globalSettingsPath = Join-Path $definitionsRoot "global-settings.jsonc"
if (Test-Path -LiteralPath $globalSettingsPath) {
    $skipped.Add("$globalSettingsPath (preserved existing file)")
}
else {
    $settings = [ordered]@{
        '$schema' = "https://raw.githubusercontent.com/Azure/enterprise-azure-policy-as-code/main/Schemas/global-settings-schema.json"
        pacOwnerId = ([guid]::NewGuid().ToString())
        pacEnvironments = @(
            [ordered]@{
                pacSelector = "test"
                cloud = "AzureCloud"
                tenantId = $tenantId
                deploymentRootScope = $DeploymentRootScope
                desiredState = [ordered]@{
                    strategy = "ownedOnly"
                    keepDfcSecurityAssignments = $true
                    doNotDisableDeprecatedPolicies = $false
                }
                globalNotScopes = @()
                managedIdentityLocation = $ManagedIdentityLocation
            }
        )
    }

    $content = $settings | ConvertTo-Json -Depth 10
    try {
        $stream = [System.IO.File]::Open(
            $globalSettingsPath,
            [System.IO.FileMode]::CreateNew,
            [System.IO.FileAccess]::Write,
            [System.IO.FileShare]::None
        )
        try {
            $writer = [System.IO.StreamWriter]::new(
                $stream,
                [System.Text.UTF8Encoding]::new($false)
            )
            try {
                $writer.WriteLine($content)
            }
            finally {
                $writer.Dispose()
            }
        }
        finally {
            if ($null -ne $stream) {
                $stream.Dispose()
            }
        }
        $created.Add($globalSettingsPath)
    }
    catch [System.IO.IOException] {
        $skipped.Add("$globalSettingsPath (appeared during scaffolding; preserved)")
    }
}

[pscustomobject]@{
    DefinitionsRoot = $definitionsRoot
    Created = @($created)
    Existing = @($existing)
    Skipped = @($skipped)
    Warnings = @($warnings)
}
```

Do not run `Connect-AzAccount`; lack of an Azure context is not an error for scaffolding. Adapt the discovery branch to the available tools, and report how each inferred value was resolved after execution.

## Optional troubleshooting fixtures

The empty scaffold is the default. If the user explicitly asks for runnable samples, test content, smoke-test objects, or fixtures, hand off to the `epac-test-fixtures` skill only after the scaffold and `global-settings.jsonc` exist.

The scaffold skill itself must not create policy fixtures, assignments, exemptions, or policy sets. This keeps troubleshooting starts small and avoids introducing Azure Policy objects that are unrelated to the issue being investigated.

## Report

Always summarize:

- The resolved Definitions root.
- How the root and deployment scope were resolved.
- **Created** directories and files.
- **Existing** directories left unchanged.
- **Skipped** files, especially a preserved `global-settings.jsonc`.
- Placeholder/default warnings and the exact values requiring review.

Include `managedIdentityLocation` and `deploymentRootScope` explicitly in the report. If the scope was inferred from the active Az subscription, report the subscription ID and scope. Never hide an inference in a success-only summary.

## Validation checklist

Before reporting completion, verify:

1. All six required subdirectories exist.
2. An existing `global-settings.jsonc` remains byte-for-byte unchanged.
3. A newly created file contains exactly one `test` environment with `ownedOnly`.
4. A newly created file uses `AzureCloud`.
5. `managedIdentityLocation` is the explicit value or `australiaeast`.
6. `deploymentRootScope` is explicit, derived from the active subscription, or clearly reported as a placeholder.
7. The operation is idempotent: a second run creates nothing and reports existing content and the global settings file as skipped.
8. No policy fixture is added unless it was explicitly requested.

If run again against the same target, no existing content should change and the existing global settings file must be reported as skipped.
