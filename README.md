# EPAC Skills — Copilot CLI Plugin

A [GitHub Copilot CLI plugin](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/plugins-creating) that helps manage Azure Policy using the [Enterprise Policy as Code (EPAC)](https://aka.ms/epac) framework.

## EPAC Repo Awareness

When launched inside an EPAC repository, all skills automatically detect the `Definitions/` folder and read `global-settings.jsonc` to extract pac selectors and deployment root scopes. If only one pac selector is configured, it is used automatically without prompting.

## Available Skills

### epac-scaffold

Creates a complete EPAC `Definitions/` structure and a safe starter `global-settings.jsonc`. It uses the active Az PowerShell context for the tenant ID when available, prompts for values that cannot be inferred, and preserves any existing files.

**Use when:** You want to initialize a new EPAC project before adding policy resources.

**Example prompts:**

- `Scaffold an EPAC project in this directory`
- `Create a new EPAC Definitions structure under ./policy`
- `Initialize an EPAC test environment`

**Requirements:** Az PowerShell is optional. If no authenticated context is available, the skill writes a clearly marked tenant ID placeholder.

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

Creates Azure Policy objects in EPAC format — policy definitions, policy set definitions (initiatives), policy assignments, and preview Policy Enrollments as JSON/JSONC files following EPAC conventions and schemas. Automatically detects your EPAC repo structure and pac selectors.

**Use when:** You want to generate EPAC-compatible policy files for your `Definitions/` folder.

**Example prompts:**

- `Create a policy set for Storage security guardrails`
- `Generate an EPAC assignment for the Allowed Locations policy`
- `Build a custom policy definition that denies storage accounts without TLS 1.2`
- `Create an enroll-mode assignment and policy enrollment for these subscriptions`

**Works with:** The `azure-policy-audit` and `azure-policy-research` skills — research or audit first to find policies, then use this skill to generate the EPAC files to deploy them.

### epac-documentation

Creates EPAC `policyDocumentations/*.jsonc` instruction files for assignment inventories, environment comparisons, and policy set comparisons. It can also run `Build-PolicyDocumentation` on request and summarize the generated Markdown, CSV, and JSONC outputs.

**Use when:** You want to configure or build documentation for deployed EPAC policy assignments and policy sets.

**Example prompts:**

- `Create documentation config for all assignments in my test PAC environment`
- `Compare these policy sets in EPAC documentation`
- `Build the policy documentation for this repository`

**Requirements:** Creating configuration files only requires an EPAC repository. Running the build requires the `EnterprisePolicyAsCode` PowerShell module and an authenticated Azure session.

### epac-build-deployment-plan

Builds and validates EPAC deployment plans using `Build-DeploymentPlans`. It captures command output and summarizes what policy definitions, initiatives, assignments, exemptions, and related role changes will be created, updated, replaced, deleted, or unchanged.

**Use when:** You want to preview and validate EPAC deployment impact before running deploy commands.

**Example prompts:**

- `Build an EPAC deployment plan for my repo`
- `Preview what policy changes will be deployed for prod`
- `Validate my EPAC definitions and summarize changes`

**Requirements:** `EnterprisePolicyAsCode` PowerShell module installed and imported, authenticated Azure session, and an EPAC repo with a valid `Definitions/global-settings.jsonc`.

### epac-test-fixtures

Scaffolds a ready-to-deploy set of EPAC artifacts — one custom Storage initiative, four built-in policy assignments (NSG-on-subnet, Key Vault soft-delete, Key Vault purge protection, and the custom initiative), and five exemptions deliberately varied across every optional field (`expiresOn`, `description`, `policyDefinitionReferenceIds`, `resourceSelectors`, `assignmentScopeValidation`, `scope` vs `scopes`, minimal vs rich metadata). All policies are Audit-only and have no Log Analytics / Guest Configuration / extension dependencies, so they deploy cleanly against any management group. Then runs `Build-DeploymentPlans` to confirm the fixtures are valid.

**Use when:** You want to populate an otherwise-empty EPAC repo so you can deploy real artifacts to reproduce, exercise, or smoke-test EPAC behavior (e.g. validating a new EPAC flag, walking through an end-to-end deploy/export round-trip, or reproducing an issue).

**Example prompts:**

- `Populate this empty EPAC repo with some test fixtures`
- `Create a few assignments and varied exemptions I can deploy`
- `Scaffold smoke-test content for the EPAC change I'm working on`

**Works with:** The `epac-build-deployment-plan` skill — runs it automatically once the fixtures are written so you immediately see the build summary.

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

You should see:

- `azure-policy-audit`
- `azure-policy-research`
- `epac-scaffold`
- `epac-policy-objects`
- `epac-documentation`
- `epac-build-deployment-plan`
- `epac-test-fixtures`

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
- [EnterprisePolicyAsCode PowerShell module](https://www.powershellgallery.com/packages/EnterprisePolicyAsCode) for deployment-plan and documentation builds
- An Azure tenant with permissions to read policy definitions and assignments
- An active Azure session for skills that query Azure or run EPAC builds — authenticate with `Connect-AzAccount`
