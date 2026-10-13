---
name: azure-sovereign-landing-zone
description: "Assess Git-Ape deployments against the Microsoft Sovereign Landing Zone (SLZ) Sovereignty Baseline control objectives SO-1 to SO-5 (data residency, operator access, customer-managed keys, confidential computing, Trusted Launch) and produce an evidence-based, scope-bound SLZ alignment report and gate result. USE FOR: sovereign, SLZ, data residency, confidential computing, CMK alignment checks after template generation. DO NOT USE FOR: legal or regulatory certification, generic security review (use azure-security-analyzer), or applying management-group policy."
license: MIT
metadata:
  author: dawright22
  source: https://github.com/dawright22/azure-sovereign-landing-zone
  maturity: experimental
  version: "1.0.0"
  argument-hint: "Deployment ID or template path, plus sovereignty profile (archetype, allowed locations, classification)"
  user-invocable: true
---

# Azure Sovereign Landing Zone (SLZ) Alignment

> Evaluate a Git-Ape ARM template and its target platform against the SLZ Sovereignty Baseline and produce a scoped, evidence-based report.

## Non-Claims

* This skill produces an **SLZ-aligned assessment**, not a sovereignty or legal-compliance certification.
* Obligations vary by jurisdiction, Azure cloud, contract, tenant configuration, operations and data flows. Use "compliant" only for a **named control with a named evidence source**, never for a deployment overall.
* Sources: the [SLZ overview](https://learn.microsoft.com/azure/azure-sovereign-clouds/public/overview-sovereign-landing-zone) and the Sovereignty Baseline policy portfolio. The `Azure/sovereign-landing-zone` repository is being archived, so do not vendor it. Re-check Learn for current control definitions on every run and record the date.

## When to Use

* "Check this deployment against the Sovereign Landing Zone"
* "Does this template meet our data residency requirements?"
* "Are we using customer-managed keys and confidential computing for the confidential landing zone?"
* A requirements file or `.azure/sovereignty.json` declares a sovereignty profile
* Never run silently: if no profile exists, ask for one (interactive) or report `Unknown` (headless)

## Procedure

### 1. Collect the sovereignty profile

| Input | Notes |
|-------|-------|
| Target cloud | Public only is supported; for anything else report unsupported, do not guess |
| Tenant, subscription, parent management group chain | Needed for platform evidence |
| Archetype | `corp`, `online`, `confidential-corp`, `confidential-online` |
| `allowedLocations` | Maps to baseline `parAllowedLocations` |
| `confidentialAllowedLocations` | Maps to `parAllowedLocationsForConfidentialComputing` (confidential archetypes only) |
| Data classification | Which data is customer-defined sensitive (drives SO-3 and SO-4 applicability) |
| Data-flow inventory | Backups, geo-replication, diagnostics, exports, global services |
| Exception register | Optional, see Constraints |

Save it to `.azure/deployments/$DEPLOYMENT_ID/sovereignty-profile.json`.

### 2. Resolve applicable controls

Use [references/slz-control-catalog.md](references/slz-control-catalog.md). Only SO-1 to SO-5 are SLZ mappings. Private endpoints, no public IP and central logging are an optional organizational overlay and must be labelled as such. SO-3 and SO-4 apply only to confidential scopes or customer-defined sensitive data.

Every check carries exactly one evidence class, and a template-only pass never implies the others:

| Class | Meaning |
|-------|---------|
| Template | Provable from an exact ARM property and value (cite the JSON path) |
| Platform | Needs a live Azure query (effective policy, tenant settings) |
| Operational | Needs human attestation (for example Customer Lockbox owner) |
| Manual | Not automatically verifiable |

Statuses are `Passed`, `Failed`, `Unknown`, `NotApplicable` and `ExceptionAccepted`. Anything unverifiable is `Unknown`, which blocks when the control is mandatory for the declared profile.

### 3. Collect template evidence (SO-1, SO-3, SO-4, SO-5)

Parse `template.json` and classify each resource's location semantics using the catalog:

* **Regional:** `location` must be in the allowlist. Also check child resources and data destinations such as diagnostic workspaces, backup vaults, replication regions and export targets.
* **Global or control plane** (Front Door, DNS zones, Traffic Manager): never auto-pass or auto-fail. Require a documented disposition or an exception.
* **Unknown type:** `Unknown`.

Resource group location is metadata, not a residency control.

For SO-3, SO-4 and SO-5, invoke `/azure-rest-api-reference` first to get the exact ARM property model per resource type. Assess separately: CMK on a supported service, key store protection, confidential capability, Trusted Launch and attested key release. Do not hard-code VM SKU families; validate SKU, image and region combinations live with `/azure-resource-availability` and `/azure-deployment-preflight`.

### 4. Collect platform evidence

Resolve the subscription's full management group ancestry. For the Sovereignty Baseline Global and Confidential initiatives report separately whether each is **assigned**, **enforced** (enforcement mode and effect), **evaluated** and **currently compliant**, including parameters, exemptions and definition versions. Missing permission or data is `Unknown`.

```bash
az account management-group entities list --query "[?type=='Microsoft.Management/managementGroups']" -o json
az policy assignment list --scope "/providers/Microsoft.Management/managementGroups/{mg}" -o json
az policy exemption list --scope "/subscriptions/{sub}" -o json
az policy state summarize --subscription {sub} -o json
```

### 5. Record operational evidence

SO-2 (Customer Lockbox) is a tenant and operational prerequisite and is never inferred from a deployment. Record the tenant, enabled state, covered services, verification time, owner and known exclusions.

### 6. Write the report and gate

Consume `/azure-security-analyzer` and `/azure-policy-advisor` output instead of re-deriving it. Write `slz-assessment.md`, `slz-assessment.json` and `sovereignty-gate.json` to `.azure/deployments/$DEPLOYMENT_ID/`. `BLOCKED` if any applicable mandatory control is `Failed` or `Unknown`.

## Outputs

| File | Content |
|------|---------|
| `sovereignty-profile.json` | Inputs from step 1 |
| `slz-assessment.md` | Per-control table (ID, evidence class, status, evidence path) with separate Template, Platform and Operational sections, then gaps and actions |
| `slz-assessment.json` | Machine-readable form of the report |
| `sovereignty-gate.json` | Gate result, shape below |

```json
{
  "profile": "confidential-online",
  "gate": "PASSED | BLOCKED | EXCEPTIONS_ACCEPTED",
  "templateSha256": "...",
  "parametersSha256": "...",
  "deploymentId": "...",
  "targetCloud": "...",
  "subscriptionId": "...",
  "ruleVersion": "...",
  "sourceCheckedAt": "...",
  "controls": [
    {"id": "SO-1", "class": "Template", "status": "Failed",
     "evidence": "resources[2].location = westus", "reason": "not in allowedLocations"}
  ],
  "exceptions": []
}
```

## Constraints

**Always:**

* Cite the exact template property for every `Passed` Template finding, and separate explicit configuration from platform defaults.
* Combine gates conjunctively: the deployment proceeds only if the Security Gate, the SLZ gate and every other mandatory gate pass. One gate's exception never clears another.
* Require structured exceptions: control ID, scope, rationale, compensating control, accountable approver, ticket reference and **expiry**. Expired exceptions fail, and `EXCEPTIONS_ACCEPTED` is never equal to `PASSED`.
* In headless CI, recompute or validate the gate against template and parameter digests rather than trusting a committed file.
* End every report with the non-claims in one line.

**Never:**

* Claim a deployment is "sovereign" or "compliant" overall.
* Emit or apply management group policy assignments from a workload deployment. Baseline policy, exemptions and Lockbox are a separate platform workflow with its own identity and approval, outside workload Deployment Stacks.
* Treat `securityType = ConfidentialVM`, secure boot or vTPM alone as proof of customer-exclusive key control (SO-4).
* Infer SO-2 or any platform control from the template.
* Accept free-text overrides.

## Attribution

* **Author:** dawright22
* **Source:** <https://github.com/dawright22/azure-sovereign-landing-zone>
* **Support:** <https://github.com/dawright22/azure-sovereign-landing-zone/issues>
