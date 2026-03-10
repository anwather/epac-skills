---
name: epac-build-deployment-plan
description: Build and validate EPAC deployment plans using Build-DeploymentPlans. Runs the command, captures output, and summarizes what policies, assignments, and exemptions will be created, updated, or deleted. Use this skill when asked to test, validate, build, or preview an EPAC deployment.
---

# EPAC Build Deployment Plan Skill

Use this skill to run the EPAC `Build-DeploymentPlans` command, capture its output, and provide a clear summary of what will be deployed. This validates the EPAC definitions against the live Azure environment and produces a deployment plan without making any changes.

---

## Step 0 — Prerequisites Check

Before running the build, verify the `EnterprisePolicyAsCode` PowerShell module is available:

```powershell
if (-not (Get-Module -ListAvailable -Name EnterprisePolicyAsCode)) {
    Install-Module -Name EnterprisePolicyAsCode -Force -Scope CurrentUser
}
Import-Module EnterprisePolicyAsCode
```

If the module install fails, inform the user they need to install the `EnterprisePolicyAsCode` module manually.

---

## Step 1 — Locate the Definitions Folder

Search the current working directory and its children for a `Definitions/` folder that contains a `global-settings.jsonc` file:

```powershell
$globalSettings = Get-ChildItem -Path . -Recurse -Filter "global-settings.jsonc" | Where-Object { $_.Directory.Name -eq "Definitions" } | Select-Object -First 1
```

- If found, use the **relative path** to the parent `Definitions/` folder as the `-DefinitionsRootFolder` parameter. For example, if found at `./MyProject/Definitions/global-settings.jsonc`, use `./MyProject/Definitions`.
- If NOT found, ask the user to provide the path to the `Definitions/` folder.

Store the resolved path:

```powershell
$definitionsRoot = $globalSettings.Directory.FullName
$relativePath = Resolve-Path -Relative $definitionsRoot
```

---

## Step 2 — Parse Global Settings and Select PAC Environment

Read and parse the `global-settings.jsonc` file to extract pac environments:

```powershell
$settingsContent = Get-Content -Path (Join-Path $definitionsRoot "global-settings.jsonc") -Raw
# Remove JSONC comments for parsing
$settingsContent = $settingsContent -replace '//.*$', '' -replace '/\*[\s\S]*?\*/', ''
$settings = $settingsContent | ConvertFrom-Json
$pacEnvironments = $settings.pacEnvironments
```

### PAC Selector Selection Logic

- If there is **exactly one** `pacEnvironment` entry, use its `pacSelector` value automatically. Do NOT prompt the user.
- If there are **multiple** `pacEnvironment` entries, ask the user which pac selector to use for the build.
- If there are **zero** `pacEnvironment` entries, inform the user the global settings file is misconfigured.

Store the selected pac selector:

```powershell
if ($pacEnvironments.Count -eq 1) {
    $pacSelector = $pacEnvironments[0].pacSelector
} else {
    # Ask the user to choose
}
```

---

## Step 3 — Run Build-DeploymentPlans

Set the output folder to `Output` relative to the current working directory:

```powershell
$outputFolder = "./Output"
```

Run the `Build-DeploymentPlans` command with the resolved parameters. Capture ALL output (standard output, warnings, errors, and verbose messages):

```powershell
$result = Build-DeploymentPlans `
    -PacEnvironmentSelector $pacSelector `
    -DefinitionsRootFolder $relativePath `
    -OutputFolder $outputFolder `
    -Interactive `
    *>&1
```

**IMPORTANT:**
- Use `-Interactive` to avoid DevOps-specific variable output.
- The `*>&1` redirection captures all output streams (Success, Warning, Error, Verbose, Information) into a single variable for analysis.
- Let the command run to completion — it may take several minutes depending on the number of policy definitions and the size of the Azure environment.

If the command fails with an error, capture the error details and present them to the user with guidance on common fixes:
- Authentication errors → suggest running `Connect-AzAccount` first
- Module not found → suggest installing `EnterprisePolicyAsCode`
- Missing definitions → check the `-DefinitionsRootFolder` path

---

## Step 4 — Parse the Output and Plan Files

After the command completes, examine both the console output and the generated plan files in the output folder.

### 4a — Check for plan files

Look for the generated plan files in the output folder:

```powershell
$planFiles = Get-ChildItem -Path $outputFolder -Recurse -Filter "*.json" -ErrorAction SilentlyContinue
```

Key plan files to look for:
- `policy-plan.json` — Policy definition and policy set definition changes
- `roles-plan.json` — Role assignment changes required for managed identities
- `policy-exemptions-plan.json` — Policy exemption changes

### 4b — Parse policy-plan.json

If `policy-plan.json` exists, read and parse it to extract the deployment operations:

```powershell
$policyPlanPath = Join-Path $outputFolder "plans" "$pacSelector" "policy-plan.json"
if (Test-Path $policyPlanPath) {
    $policyPlan = Get-Content -Path $policyPlanPath -Raw | ConvertFrom-Json
}
```

The policy plan contains sections for each resource type with operation categories:
- **Policy Definitions**: `new`, `update`, `replace`, `delete`, `unchanged`
- **Policy Set Definitions**: `new`, `update`, `replace`, `delete`, `unchanged`
- **Policy Assignments**: `new`, `update`, `replace`, `delete`, `unchanged`

### 4c — Parse roles-plan.json

```powershell
$rolesPlanPath = Join-Path $outputFolder "plans" "$pacSelector" "roles-plan.json"
if (Test-Path $rolesPlanPath) {
    $rolesPlan = Get-Content -Path $rolesPlanPath -Raw | ConvertFrom-Json
}
```

### 4d — Parse exemptions plan

```powershell
$exemptionsPlanPath = Join-Path $outputFolder "plans" "$pacSelector" "policy-exemptions-plan.json"
if (Test-Path $exemptionsPlanPath) {
    $exemptionsPlan = Get-Content -Path $exemptionsPlanPath -Raw | ConvertFrom-Json
}
```

---

## Step 5 — Summarize the Deployment Plan

Present a clear, structured summary of what will be deployed. The summary MUST include:

### Deployment Plan Summary

Report the pac environment that was built:

```
PAC Environment: <pacSelector>
Definitions Root: <relativePath>
Output Folder: <outputFolder>
```

### Changes by Resource Type

For each resource type, report the count of operations:

| Resource Type | New | Update | Replace | Delete | Unchanged |
|---|---|---|---|---|---|
| Policy Definitions | N | N | N | N | N |
| Policy Set Definitions | N | N | N | N | N |
| Policy Assignments | N | N | N | N | N |
| Policy Exemptions | N | N | N | N | N |
| Role Assignments | N | N | N | N | N |

### Details of Changes

For any **new**, **update**, **replace**, or **delete** operations, list the specific resources that will be affected:

**New Resources:**
- List each new policy definition, set definition, or assignment by display name and name

**Updated Resources:**
- List each resource being updated, noting what changed if available

**Deleted Resources:**
- ⚠️ Highlight deletions prominently as they are destructive operations

**Replaced Resources:**
- Note resources being replaced (delete + recreate)

### Overall Assessment

Provide a brief assessment:
- If only new/update operations exist → "Plan looks safe to deploy"
- If deletions exist → "⚠️ Plan includes deletions — review carefully before deploying"
- If no changes → "No changes detected — Azure environment matches definitions"
- If errors occurred → "❌ Build failed — see errors above"

---

## Step 6 — Provide Next Steps

After summarizing, remind the user of next steps:

1. **Review the plan**: Check the detailed plan files in `<outputFolder>/plans/<pacSelector>/`
2. **Deploy**: If the plan looks correct, deploy with:
   ```powershell
   Deploy-PolicyPlan -PacEnvironmentSelector "<pacSelector>" -DefinitionsRootFolder "<relativePath>" -InputFolder "<outputFolder>"
   Deploy-RolesPlan -PacEnvironmentSelector "<pacSelector>" -DefinitionsRootFolder "<relativePath>" -InputFolder "<outputFolder>"
   ```
3. **Fix issues**: If errors were found, address them in the definition files and re-run the build.

---

## Important Notes

- `Build-DeploymentPlans` is a **read-only** operation — it queries Azure but does NOT make any changes.
- The command requires an authenticated Azure session (`Connect-AzAccount`).
- The output folder will be created automatically if it does not exist.
- Plan files are JSON and can be inspected manually in the output folder.
- If the user hasn't specified a pac selector and multiple exist, always ask before proceeding.
- Always show the full console output from the build command so the user can see any warnings or informational messages.
