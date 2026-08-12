---
name: epac-documentation
description: Create and optionally run EPAC policy documentation configurations for assignment reports and policy set comparisons. Use when asked to generate, configure, scaffold, build, or update policyDocumentations JSON/JSONC files, document all or representative Azure Policy assignments, compare initiatives, or run Build-PolicyDocumentation.
---

# EPAC Policy Documentation Skill

Create schema-valid EPAC documentation configuration files under `Definitions/policyDocumentations/`. Generate configuration only by default; run `Build-PolicyDocumentation` only when the user explicitly asks to build, execute, preview, or validate the documentation.

## Step 0 — Detect EPAC context

1. Search the current directory and its children for `Definitions/global-settings.jsonc`.
2. If exactly one is found, use its parent as the Definitions root.
3. If multiple are found, use the one in the user's stated project; otherwise ask which repository to use.
4. If none is found, ask for the Definitions root.
5. Parse `global-settings.jsonc` and collect every `pacEnvironments[].pacSelector`.

### PAC selector logic

- One PAC selector: select it automatically.
- Multiple selectors:
  - Use the selector explicitly named by the user.
  - Otherwise infer it only when the requested documentation folder, existing configuration, or referenced objects unambiguously identify one selector.
  - If still ambiguous, ask the user to choose.
- Zero selectors: stop and report that `global-settings.jsonc` is invalid for this workflow.
- Every `pacEnvironment` value in the generated configuration must exactly match a configured `pacSelector`.

### Documentation folder auto-detection

Use `Definitions/policyDocumentations/` by default.

- If it contains no subfolders, place configuration files directly in it.
- If it contains one subfolder and its name matches the selected PAC selector, use it and pass `-PacSelector`.
- If the sole subfolder does not match the selector, use it only when its configuration is unambiguous and omit `-PacSelector` so EPAC can auto-select the sole folder; otherwise create or select a matching selector folder.
- If it contains multiple subfolders, use `<policyDocumentations>/<pacSelector>/`.
- With multiple subfolders, `-PacSelector` selects the subfolder whose name matches the selector and filters `documentAllAssignments` entries to that PAC environment. Do not place mixed-environment configuration in a selector-specific subfolder.

`Build-PolicyDocumentation` recursively reads every `.json` and `.jsonc` file in the selected folder; filenames do not control processing.

## Step 1 — Select the documentation mode

Use one configuration file for one or both top-level topics:

| Request | Configuration |
|---|---|
| Every deployed assignment | `documentAssignments.documentAllAssignments` |
| Representative assignments grouped as prod/test/dev/etc. | `documentAssignments.environmentCategories` |
| Compare policy set definitions | `documentPolicySets` |
| Assignment reports plus policy set comparison | Include both `documentAssignments` and `documentPolicySets` |

Always include:

```jsonc
"$schema": "https://raw.githubusercontent.com/Azure/enterprise-azure-policy-as-code/main/Schemas/policy-documentation-schema.json"
```

## Step 2 — Gather mode-specific inputs

Ask only for information that cannot be derived safely.

### All assignments

Required:

- `pacEnvironment`
- Output `fileNameStem` and Markdown `title`, either in `documentationSpecifications` or as global defaults

Optional:

- `enabled`
- `fileNameStemPrefix`
- `skipPolicyAssignments`: assignment resource IDs to exclude
- `skipPolicyDefinitions`: policy or policy set definition resource IDs whose assignments should be excluded
- `overrideEnvironmentCategory`: object mapping category names to arrays of assignment scopes

Without `overrideEnvironmentCategory`, each directly assigned scope becomes an environment category. Use `fileNameStemPrefix` when multiple environments could otherwise produce the same output names.

### Representative assignments / environment categories

For each category, gather:

- `pacEnvironment`
- `environmentCategory`: output column/category name
- `representativeAssignments`: one or more `{ "shortName", "id" }` assignment references
- Optional `scopes`: display-only text for the Markdown Scopes section
- Optional category-level `documentationSpecifications`

At `documentAssignments.documentationSpecifications`, gather:

- `fileNameStem`
- `title`
- The `environmentCategories` to show when supported by the installed EPAC version; use names declared above

Representative assignment `id` values must be full assignment resource IDs. Keep `shortName` concise and unique because it appears in CSV and Markdown output.

### Policy sets

For each `documentPolicySets` item, gather:

- `pacEnvironment`
- `fileNameStem`
- `title`
- `policySets`: one or more entries containing `shortName` and exactly one of:
  - `id` for a built-in or fully qualified policy set resource ID
  - `name` for a custom policy set known to the selected EPAC environment
- `environmentColumnsInCsv`: one or more starter parameter column names

Optional:

- `includePolicySetList`: omit the policy set list when `false`
- `policySetEffectsOneColumn`: consolidate effects into one column when `true`

Never use the legacy `initiatives` property.

### Combined configurations

A single file may contain `globalDocumentationSpecifications`, `documentAssignments`, and `documentPolicySets`. Ensure every entry is valid independently and that selector-subfolder filtering will not remove intended entries.

## Step 3 — Apply global and Markdown options

Use top-level `globalDocumentationSpecifications` for shared defaults. A property on a specific assignment documentation specification or policy set item overrides the global value.

Supported global/shared properties:

| Property | Purpose |
|---|---|
| `fileNameStem` | Default assignment output stem when assignment specifications are omitted |
| `title` | Default assignment Markdown title when assignment specifications are omitted |
| `markdownAddToc` | Add a Markdown table of contents |
| `markdownAdoWiki` | Format for Azure DevOps Wiki and implicitly enable the TOC |
| `markdownAdoWikiConfig` | One object or an array of `{ adoOrganization, adoProject, adoWiki }` |
| `markdownNoEmbeddedHtml` | Use commas instead of `<br/>` in Markdown tables |
| `markdownIncludeComplianceGroupNames` | Include policy definition group names |
| `markdownSuppressParameterSection` | Omit the Markdown parameter section |
| `markdownMaxParameterLength` | Truncate long parameter names; default `42`, minimum `16` |

If assignment `documentationSpecifications` is omitted, `globalDocumentationSpecifications.fileNameStem` and `.title` are required. EPAC then synthesizes one assignment specification using discovered environment categories.

Set `markdownAdoWikiConfig` only when Azure DevOps Wiki output is intended. Do not place PATs, tokens, or other credentials in configuration files.

## Step 4 — Generate safely

1. Create `Definitions/policyDocumentations/` or the selected selector subfolder if needed.
2. Use a descriptive kebab-case filename ending in `.jsonc`, such as:
   - `document-all-assignments.jsonc`
   - `document-representative-assignments.jsonc`
   - `document-policy-sets.jsonc`
   - `document-policy-landscape.jsonc`
3. Reject path traversal, rooted paths, and invalid filename characters.
4. Inspect all existing documentation configurations and output stems before writing.
5. Never overwrite an existing file. If the desired path exists, preserve it and use a meaningful variant or numeric suffix.
6. Keep every effective `fileNameStem` unique. Consider global defaults and `fileNameStemPrefix` when calculating effective names.
7. Do not use path separators or extensions in `fileNameStem`.
8. Validate the generated structure against the local `Schemas/policy-documentation-schema.json` when available. Also enforce semantic rules described here that the schema may not fully express.

Minimal all-assignments shape:

```jsonc
{
    "$schema": "https://raw.githubusercontent.com/Azure/enterprise-azure-policy-as-code/main/Schemas/policy-documentation-schema.json",
    "globalDocumentationSpecifications": {
        "fileNameStem": "azure-policy-assignments",
        "title": "Current Azure Policy assignments"
    },
    "documentAssignments": {
        "documentAllAssignments": [
            {
                "pacEnvironment": "<pacSelector>",
                "skipPolicyAssignments": [],
                "skipPolicyDefinitions": []
            }
        ]
    }
}
```

Minimal representative-assignment shape:

```jsonc
{
    "$schema": "https://raw.githubusercontent.com/Azure/enterprise-azure-policy-as-code/main/Schemas/policy-documentation-schema.json",
    "documentAssignments": {
        "environmentCategories": [
            {
                "pacEnvironment": "<pacSelector>",
                "environmentCategory": "prod",
                "scopes": ["Production"],
                "representativeAssignments": [
                    {
                        "shortName": "Baseline",
                        "id": "<full-policy-assignment-resource-id>"
                    }
                ]
            }
        ],
        "documentationSpecifications": [
            {
                "fileNameStem": "azure-policy-assignments",
                "title": "Azure Policy assignments"
            }
        ]
    }
}
```

Minimal policy-set shape:

```jsonc
{
    "$schema": "https://raw.githubusercontent.com/Azure/enterprise-azure-policy-as-code/main/Schemas/policy-documentation-schema.json",
    "documentPolicySets": [
        {
            "pacEnvironment": "<pacSelector>",
            "fileNameStem": "policy-set-comparison",
            "title": "Policy set comparison",
            "includePolicySetList": true,
            "policySetEffectsOneColumn": false,
            "policySets": [
                {
                    "shortName": "ASB",
                    "id": "/providers/Microsoft.Authorization/policySetDefinitions/<guid>"
                }
            ],
            "environmentColumnsInCsv": ["prod"]
        }
    ]
}
```

## Step 5 — Optionally run Build-PolicyDocumentation

Do not execute merely because a configuration was created. Execute only when explicitly requested.

### Prerequisites

Check and import the EPAC module without installing it automatically:

```powershell
if (-not (Get-Module -ListAvailable -Name EnterprisePolicyAsCode)) {
    throw "EnterprisePolicyAsCode PowerShell module is not installed."
}
Import-Module EnterprisePolicyAsCode -ErrorAction Stop
```

If unavailable, surface the error and tell the user to install the module. Do not modify their PowerShell environment without a separate request.

Use an explicitly requested output folder; otherwise use `<repository-root>\Output`. EPAC writes generated files beneath `<OutputFolder>\policy-documentation\`, including service-category Markdown under `services\`.

Run non-interactively and pass resolved paths. Pass `-PacSelector` when selecting a matching selector subfolder, filtering multi-environment `documentAllAssignments`, or otherwise disambiguating the PAC environment:

```powershell
Build-PolicyDocumentation `
    -DefinitionsRootFolder "<DefinitionsRoot>" `
    -OutputFolder "<OutputFolder>" `
    -PacSelector "<pacSelector>" `
    -Interactive:$false `
    -ErrorAction Stop
```

When intentionally using a sole subfolder whose name does not match the PAC selector, omit `-PacSelector`; EPAC auto-selects that sole folder. Do this only when the files inside identify one unambiguous PAC environment.

Add switches only when requested:

- `-WindowsNewLineCells`: Excel-friendly multiline CSV cells on Windows
- `-SuppressConfirmation`: suppress per-definition processing prompts in interactive use
- `-IncludeManualPolicies`: include policies with the `Manual` effect
- `-OnlyCheckManagedAssignments`: include only assignments managed by the configured PAC owner
- `-StrictMode`: fail when referenced policy definitions are missing
- `-StrictMode:$false`: warn and continue for missing referenced definitions
- `-WikiClonePat <value>` or `-WikiSPN`: publish configured Markdown to Azure DevOps Wiki; never expose the PAT in output

Do not combine `-WikiClonePat` and `-WikiSPN`. Wiki execution also requires `markdownAdoWiki` and `markdownAdoWikiConfig`.

### Execution safety and reporting

- Before execution, check for files under `<OutputFolder>\policy-documentation\` with stems this run will generate. If collisions exist, do not run until the user explicitly approves replacement or selects a different output folder or stem.
- Do not suppress, reinterpret, or hide command failures. Return the direct PowerShell error, including the failing specification filename when available.
- Authentication failures should be surfaced directly with a concise suggestion to run `Connect-AzAccount`.
- After success, summarize:
  - PAC selector
  - Definitions root and selected documentation folder
  - Output folder
  - Configuration files processed
  - Generated `.md`, `.csv`, and `.jsonc` files grouped by stem
  - Whether service pages or Azure DevOps Wiki publishing were used

Expected assignment outputs include `<stem>-summary.md`, `<stem>-full.md`, `<stem>-full.csv`, and `<stem>-parameters.csv`. Policy set outputs include `<stem>-full.md`, `<stem>-full.csv`, `<stem>-parameters.csv`, and `<stem>.jsonc`.
