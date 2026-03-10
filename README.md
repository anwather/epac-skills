# EPAC Skills for GitHub Copilot CLI

This folder contains custom [agent skills](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/create-skills) for GitHub Copilot CLI that help manage Azure Policy using the [Enterprise Policy as Code (EPAC)](https://aka.ms/epac) framework.

## Available Skills

### azure-policy-audit

Audits Azure Policy definitions for a specific service (e.g. Storage, Compute, KeyVault). It finds built-in policy definitions related to the service, checks which are currently assigned across your tenant, and returns the unassigned policies with their names and IDs.

**Use when:** You want to identify policy coverage gaps for an Azure service.

**Example prompts:**

- `Find unassigned Storage policies in my environment`
- `Which Compute policies am I missing?`
- `Audit Key Vault policy coverage`

**Requirements:** Az PowerShell module (`Az.Resources`) authenticated to your Azure tenant.

### epac-policy-objects

Creates Azure Policy objects in EPAC format — policy definitions, policy set definitions (initiatives), and policy assignments as JSON/JSONC files following EPAC conventions and schemas.

**Use when:** You want to generate EPAC-compatible policy files for your `Definitions/` folder.

**Example prompts:**

- `Create a policy set for Storage security guardrails`
- `Generate an EPAC assignment for the Allowed Locations policy`
- `Build a custom policy definition that denies storage accounts without TLS 1.2`

**Works with:** The `azure-policy-audit` skill — audit first to find unassigned policies, then use this skill to generate the EPAC files to deploy them.

## Installation

Copy the skill folders to one of the supported skill locations:

### Personal skills (all projects)

```powershell
Copy-Item -Recurse .\azure-policy-audit "$HOME\.copilot\skills\azure-policy-audit"
Copy-Item -Recurse .\epac-policy-objects "$HOME\.copilot\skills\epac-policy-objects"
```

### Repository skills (single project)

```powershell
Copy-Item -Recurse .\azure-policy-audit ".\.github\skills\azure-policy-audit"
Copy-Item -Recurse .\epac-policy-objects ".\.github\skills\epac-policy-objects"
```

### Organisation skills (all repos in org)

Copy the skill folders to the `/skills` directory in your organisation's `.github-private` repository.

## Verifying Installation

After copying, launch or restart GitHub Copilot CLI and run:

```
/skills list
```

If you've added skills during an active session, reload without restarting:

```
/skills reload
```

You should see `azure-policy-audit` and `epac-policy-objects` in the list.

## Prerequisites

- [GitHub Copilot CLI](https://docs.github.com/en/copilot/how-tos/set-up/install-copilot-cli)
- [Az PowerShell module](https://learn.microsoft.com/en-us/powershell/azure/install-azure-powershell) — `Install-Module Az -Scope CurrentUser`
- An Azure tenant with permissions to read policy definitions and assignments
- An active Azure session — authenticate with `Connect-AzAccount` before using the skills
