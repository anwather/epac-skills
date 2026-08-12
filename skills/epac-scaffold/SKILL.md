---
name: epac-scaffold
description: Scaffold a complete EPAC (Enterprise Policy as Code) Definitions folder and safe test PAC environment. Use when asked to initialize, create, bootstrap, or scaffold an EPAC repository, Definitions structure, global-settings.jsonc, or starter folder layout.
---

# EPAC Scaffold Skill

Create a complete, idempotent EPAC `Definitions` structure without changing existing content.

## Inputs

Gather these values before writing:

- **Target path** (optional): the path to use as the `Definitions` root. If omitted, use `Definitions` under the current working directory.
- **deploymentRootScope**: prompt for a full management group or subscription resource ID. If omitted or blank, use `/providers/Microsoft.Management/managementGroups/REPLACE-WITH-MANAGEMENT-GROUP-ID` and clearly warn that it must be replaced.
- **managedIdentityLocation**: prompt for an Azure region. If omitted or blank, use the safe starter default `eastus2` and report that the default was used.

Resolve the target to an absolute path. If the target exists as a file rather than a directory, stop without modifying it.

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
            "managedIdentityLocation": "<provided-location-or-eastus2>"
        }
    ]
}
```

`ownedOnly` is required for this starter environment so EPAC does not assume ownership of unrelated policy resources.

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
    $TargetPath = Join-Path (Get-Location) "Definitions"
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

if ([string]::IsNullOrWhiteSpace($DeploymentRootScope)) {
    $DeploymentRootScope = "/providers/Microsoft.Management/managementGroups/REPLACE-WITH-MANAGEMENT-GROUP-ID"
    $warnings.Add("deploymentRootScope uses a placeholder and must be replaced.")
}

if ([string]::IsNullOrWhiteSpace($ManagedIdentityLocation)) {
    $ManagedIdentityLocation = "eastus2"
    $warnings.Add("managedIdentityLocation was omitted; using eastus2.")
}

$tenantId = $null
if (Get-Command -Name Get-AzContext -ErrorAction SilentlyContinue) {
    try {
        $context = Get-AzContext -ErrorAction Stop
        if ($null -ne $context -and $null -ne $context.Tenant -and $context.Tenant.Id) {
            $tenantId = [string] $context.Tenant.Id
        }
    }
    catch {
        $warnings.Add("Get-AzContext failed; tenantId will use a placeholder. $($_.Exception.Message)")
    }
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

Do not run `Connect-AzAccount`; lack of an Azure context is not an error for scaffolding.

## Report

Always summarize:

- The resolved Definitions root.
- **Created** directories and files.
- **Existing** directories left unchanged.
- **Skipped** files, especially a preserved `global-settings.jsonc`.
- Placeholder/default warnings and the exact values requiring review.

If run again against the same target, no existing content should change and the existing global settings file must be reported as skipped.
