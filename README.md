# ALZ-Bicep-Notebook (archived)

> **This repository is archived and superseded.** It is kept for reference only. The
> approach it demonstrates – deploying an Azure Landing Zone by hand from a notebook,
> using the classic `Azure/ALZ-Bicep` modules – is no longer how Microsoft recommends
> building landing zones. Use the **Azure Landing Zones IaC Accelerator** instead:
> <https://azure.github.io/Azure-Landing-Zones/accelerator/>.

## What this was

A [Polyglot Notebook](alz-bicep-notebook.ipynb) that stepped through the
[Azure Landing Zone conceptual architecture](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/landing-zone/#azure-landing-zone-conceptual-architecture),
one platform layer at a time – management groups, custom policy and role definitions,
logging and Sentinel, management-group diagnostic settings, hub networking, RBAC
assignments, subscription placement, policy assignments and spoke networking. Each step
built a PowerShell splat hashtable and called `New-AzTenantDeployment`,
`New-AzManagementGroupDeployment` or `New-AzResourceGroupDeployment` against a module from
a local clone of [Azure/ALZ-Bicep](https://github.com/Azure/ALZ-Bicep), with a `-WhatIf`
preview cell before each real deployment.

In its day it was a reasonable way to learn how the pieces fit together and to run
controlled proof-of-concept deployments. It has since fallen behind on every front it
depended on.

## Why it is archived

- **The recommended approach has changed.** Microsoft now points people first at the
  **ALZ IaC Accelerator**, which bootstraps a Git repository, a CI/CD pipeline, OIDC
  managed identities and drift management, then deploys the platform from that pipeline.
  Driving deployments by hand, cell by cell, is precisely what the Accelerator replaces –
  it gives you a review gate, an audit trail and repeatability that a notebook cannot.
- **The classic modules are retiring.** `Azure/ALZ-Bicep` is now branded "Classic". It was
  removed from the Accelerator on 16 February 2026 and the repository is scheduled to be
  archived on 16 February 2027. Its successor is
  [Bicep Azure Verified Modules for Platform Landing Zone](https://azure.github.io/Azure-Landing-Zones/bicep/)
  (`avm/ptn/alz/*`), which reached general availability in early 2026. Governance content –
  policy definitions, policy assignments, role definitions and archetypes – has moved into
  the shared [Azure Landing Zones Library](https://github.com/Azure/Azure-Landing-Zones-Library),
  so the hand-maintained policy JSON this notebook relied on is obsolete.
- **The deployment mechanics have moved on.** Typed
  [`.bicepparam` parameter files](https://learn.microsoft.com/azure/azure-resource-manager/bicep/parameter-files)
  have replaced JSON parameter files.
  [Deployment Stacks](https://learn.microsoft.com/azure/azure-resource-manager/bicep/deployment-stacks)
  (generally available since May 2024) have replaced plain deployments wherever lifecycle
  management and resource locking matter – both AVM and the Accelerator use them. The
  `AzureAD` PowerShell module used in the RBAC cells was retired in mid-2025; the
  replacement is the Microsoft Graph PowerShell SDK (`Connect-MgGraph`, `Get-MgGroup`,
  `Get-MgServicePrincipal`).
- **The notebook runtime itself is deprecated.** The Polyglot Notebooks VS Code extension
  was deprecated on 27 March 2026 and the `dotnet/interactive` project was archived on
  27 April 2026. The extension still runs with a current .NET SDK, but it receives no
  further fixes.

## What to use instead – the ALZ IaC Accelerator

The Accelerator is a guided bootstrap for a landing-zone deployment that runs from source
control rather than from your workstation. You install the `ALZ` PowerShell module
(`Install-Module -Name ALZ`) and run `Deploy-Accelerator`. With no parameters it runs an
interactive wizard; for a repeatable run you pass `Deploy-Accelerator -inputs inputs.yaml`.
It supports GitHub (Actions) and Azure DevOps (Pipelines), and has a "Local" output mode
that simply writes the generated Bicep to disk if you want to deploy it by hand – the
closest equivalent to what this notebook did.

### What it sets up

Running the Accelerator provisions the scaffolding around the deployment, not just the
landing zone:

- User-assigned managed identities with federated credentials, so the pipeline
  authenticates to Azure over OIDC with no stored secrets.
- Optional Terraform state storage (resource group, storage account and container),
  optionally behind private networking.
- Optional self-hosted runners or agents, hosted on Azure Container Instances.
- Two Git repositories – one for your infrastructure module, one holding governed pipeline
  templates – with CI/CD pipelines, environments, branch policies and OIDC service
  connections wired up.
- Role assignments for the managed identity on the target management groups.

### How it works

The process runs in stages. **Planning** – choose the IaC language (Bicep or Terraform)
and the version-control system, and gather credentials and the platform subscriptions.
**Bootstrap** – the `ALZ` PowerShell module applies a Terraform bootstrap that provisions
everything listed above; note that the bootstrap runs on Terraform even when you have
chosen Bicep, because Bicep is used only for the platform landing zone itself, not for the
scaffolding. **Run** – you customise the starter module, the CI pipeline runs a `what-if`,
and the CD pipeline applies the change as a deployment stack.

### The Bicep starter

The current Bicep starter is "Bicep – Complete", composed of roughly nineteen Azure
Verified Modules (sixteen resource modules and three pattern modules), with parameters
supplied as `.bicepparam` and lifecycle managed through
`New-AzManagementGroupDeploymentStack`. The starter templates live in
[Azure/alz-bicep-accelerator](https://github.com/Azure/alz-bicep-accelerator), organised
into four areas: ALZ core (management groups and policies), ALZ management (Log Analytics
and monitoring), hub networking, and Virtual WAN.

### If you want to stay hands-on

You do not have to adopt the full pipeline. The
[`avm/ptn/alz/*`](https://github.com/Azure/bicep-registry-modules/tree/main/avm/ptn/alz)
pattern modules can be consumed directly from the public Bicep registry with your own
`.bicepparam` files and deployment stacks – this is the supported "custom build" path, and
it is the modern version of what this notebook attempted. For teams without IaC skills
there is also a [portal-based accelerator](https://azure.github.io/Azure-Landing-Zones/accelerator/).
Either way, take your policy, archetype and role content from the
[Azure Landing Zones Library](https://github.com/Azure/Azure-Landing-Zones-Library) at a
pinned release rather than hand-authoring it.

### One architecture change worth knowing

The baseline ALZ management-group hierarchy has grown since this notebook was written. There
is now a **Security** management group and subscription under Platform, alongside Identity,
Management and Connectivity, intended for Microsoft Sentinel and security tooling separate
from operational logging. There is also a **Local** management group under the application
landing zones, alongside Corp and Online, for Azure Local clusters and their workloads.

## Reference

- [CAF – platform landing zone implementation options](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/landing-zone/implementation-options)
- [CAF – what is an Azure landing zone](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/landing-zone/)
- [ALZ documentation – Bicep](https://azure.github.io/Azure-Landing-Zones/bicep/)
- [ALZ documentation – IaC Accelerator](https://azure.github.io/Azure-Landing-Zones/accelerator/)
- [Azure/ALZ-PowerShell-Module](https://github.com/Azure/ALZ-PowerShell-Module) – the `ALZ` module and `Deploy-Accelerator`
- [Azure/alz-bicep-accelerator](https://github.com/Azure/alz-bicep-accelerator) – AVM Bicep starter templates
- [Azure/bicep-registry-modules – `avm/ptn/alz`](https://github.com/Azure/bicep-registry-modules/tree/main/avm/ptn/alz)
- [Azure/Azure-Landing-Zones-Library](https://github.com/Azure/Azure-Landing-Zones-Library) – shared policy, archetype and role content
- [Azure/bicep-lz-vending](https://github.com/Azure/bicep-lz-vending) – subscription vending (`avm/ptn/lz/sub-vending`)
- [Deployment Stacks](https://learn.microsoft.com/azure/azure-resource-manager/bicep/deployment-stacks)
- [Bicep parameter files (`.bicepparam`)](https://learn.microsoft.com/azure/azure-resource-manager/bicep/parameter-files)
- [Azure/ALZ-Bicep](https://github.com/Azure/ALZ-Bicep) – the classic modules this notebook used, for historical context
