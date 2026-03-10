# EPAC Skills — Copilot CLI Plugin

A [GitHub Copilot CLI plugin](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/plugins-creating) that helps manage Azure Policy using the [Enterprise Policy as Code (EPAC)](https://aka.ms/epac) framework.

## EPAC Repo Awareness

When launched inside an EPAC repository, all skills automatically detect the `Definitions/` folder and read `global-settings.jsonc` to extract pac selectors and deployment root scopes. If only one pac selector is configured, it is used automatically without prompting.

## Available Skills

### azure-policy-audit

Audits Azure Policy definitions for a specific service (e.g. Storage, Compute, KeyVault). It finds built-in policy definitions related to the service, checks which are currently assigned across your tenant, and returns the unassigned policies with their names and IDs.

**Use when:** You want to identify policy coverage gaps for an Azure service.

**Example prompts:**

- `Find unassigned Storage policies in my environment`
- `Which Compute policies am I missing?`
- `Audit Key Vault policy coverage`

**Requirements:** Az PowerShell module (`Az.Resources`) authenticated to your Azure tenant.

### azure-policy-research

Discovers and browses available Azure built-in policy definitions for a service category. Shows policy effects, parameters, and descriptions grouped by effect type — without checking current assignments.

**Use when:** You want to explore what policies exist for a service before deciding what to assign.

**Example prompts:**

- `What Storage policies are available?`
- `Show me all built-in Kubernetes policies`
- `Research available Network security policies`
- `What Deny policies exist for Key Vault?`

**Requirements:** Az PowerShell module (`Az.Resources`) authenticated to your Azure tenant.

### epac-policy-objects

Creates Azure Policy objects in EPAC format — policy definitions, policy set definitions (initiatives), and policy assignments as JSON/JSONC files following EPAC conventions and schemas. Automatically detects your EPAC repo structure and pac selectors.

**Use when:** You want to generate EPAC-compatible policy files for your `Definitions/` folder.

**Example prompts:**

- `Create a policy set for Storage security guardrails`
- `Generate an EPAC assignment for the Allowed Locations policy`
- `Build a custom policy definition that denies storage accounts without TLS 1.2`

**Works with:** The `azure-policy-audit` and `azure-policy-research` skills — research or audit first to find policies, then use this skill to generate the EPAC files to deploy them.

## Installation

Install as a Copilot CLI plugin:

### From a local clone

```shell
git clone https://github.com/<owner>/epac-skills.git
copilot plugin install ./epac-skills
```

### From GitHub (once published)

```shell
copilot plugin install <owner>/epac-skills
```

### Verifying installation

After installing, launch or restart GitHub Copilot CLI and run:

```
/plugin list
```

You should see `epac-skills` in the list. You can also verify the skills loaded:

```
/skills list
```

You should see `azure-policy-audit` and `epac-policy-objects` in the list.

### Updating

```shell
copilot plugin update epac-skills
```

### Uninstalling

```shell
copilot plugin uninstall epac-skills
```

## Prerequisites

- [GitHub Copilot CLI](https://docs.github.com/en/copilot/how-tos/set-up/install-copilot-cli)
- [Az PowerShell module](https://learn.microsoft.com/en-us/powershell/azure/install-azure-powershell) — `Install-Module Az -Scope CurrentUser`
- An Azure tenant with permissions to read policy definitions and assignments
- An active Azure session — authenticate with `Connect-AzAccount` before using the skills
