# Pipeline Optimisation Changes

## Problem

CI/CD pipelines were consistently taking over an hour to complete, with the CI what-if job frequently timing out. Root causes identified:

1. **Sequential execution** — All 17 deployment steps ran one after another in a single job
2. **Pre-deployment cleanup** — Every deployment step enumerated and deleted all existing ARM deployments (`Get-Az*Deployment` + `Remove-Az*Deployment`) before creating the deployment stack
3. **No parallelism** — Independent deployments (e.g. Landing Zones children vs Platform children) waited for each other unnecessarily

## Changes

### CD Template (`cd-template.yaml`)

Replaced the monolithic 2-job workflow (whatif + deploy) with **8 parallel jobs** following a dependency DAG:

```
deploy-governance-root
     ├─ deploy-governance-parents ──┬── deploy-governance-children-lz ──────┐
     │   (LZ, Platform,            │                                        ├─ deploy-governance-rbac
     │    Sandbox, Decommissioned)  └── deploy-governance-children-platform ┘
     │
     └─ deploy-core ──── deploy-networking
```

The `whatif` job is retained but **skipped by default** (`skip_what_if: true`). It can be re-enabled for a specific run by passing `skip_what_if: false` via `workflow_dispatch`. Deploy jobs no longer depend on it.

| Wave | Jobs | Runs in parallel with |
|------|------|-----------------------|
| 1 | `deploy-governance-root` | — |
| 2 | `deploy-governance-parents`, `deploy-core` | Each other |
| 3 | `deploy-governance-children-lz`, `deploy-governance-children-platform` | Each other + `deploy-networking` |
| 4 | `deploy-governance-rbac` | — |

### CI Template (`ci-template.yaml`)

Removed all ARM what-if jobs from CI entirely. The what-if checks were consistently timing out (60+ minutes) due to Azure ARM API limitations with management group scope deployments — particularly the `int-root` template which evaluates the entire management group hierarchy.

**Why this is safe:**
- The `validate` job (`bicep build` + lint) catches syntax errors, type mismatches, and missing parameters in seconds
- ARM what-if slowness is specific to the `New-Az*Deployment -WhatIf` diff engine — Deployment Stacks (`New-Az*DeploymentStack`) don't have this problem because they submit templates directly to ARM without computing a full before/after diff
- The CD `whatif` job is skipped by default for the same reason — by the time CD runs you've committed to deployment, and Deployment Stacks go straight to ARM

**CI now runs a single job:**
| Job | Description | Duration |
|-----|-------------|----------|
| `validate` | `bicep build` + lint all `.bicep` files | ~1-2 min |

### Bicep Deploy Action (`bicep-deploy/action.yaml`)

Removed deployment cleanup blocks from all 3 deployment types (management group, subscription, resource group). These blocks were:

```powershell
# REMOVED — was running before every deployment stack operation
$allDeployments = Get-AzManagementGroupDeployment -ManagementGroupId $id
$allDeployments | ForEach-Object -Parallel {
    Remove-AzManagementGroupDeployment -ManagementGroupId $id -Name $_.DeploymentName
} -ThrottleLimit 100
```

This enumeration was hitting ARM API rate limits and adding 2-5 minutes per step. Deployment Stacks manage their own lifecycle and don't need manual cleanup of ARM deployments.

## Expected Impact

| Metric | Before | After (estimated) |
|--------|--------|--------------------|
| CI wall time | 60-80 min (what-if timeouts) | ~1-2 min (validate only) |
| CD deploy wall time (full run) | 60-90 min | ~30-40 min |
| CD deploy wall time (targeted) | 60-90 min (always full) | ~5-15 min (path-based change detection) |
| ARM API calls per run | ~17 × (enumerate + batch delete) | 0 cleanup calls |
| GitHub Actions runners | 2 concurrent | Up to 8 concurrent (CD) / 1 (CI) |

## Files Changed

- `.github/workflows/cd-template.yaml` — Parallel deploy jobs with dependency DAG; `skip_what_if` defaults to `true`
- `.github/workflows/ci-template.yaml` — Removed all what-if jobs, CI is now validate-only
- `.github/actions/bicep-deploy/action.yaml` — Removed deployment cleanup blocks

---

### Caller Workflow: Path-Based Change Detection (`cmalz-mgmt` repo)

The caller `cd.yaml` in the `cmalz-mgmt` repo now includes a **`detect-changes`** job that uses [`dorny/paths-filter@v3`](https://github.com/dorny/paths-filter) to compare the push commit against the previous commit. Only deployment groups whose templates actually changed are sent to the reusable workflow as `true`.

**On push to main** — change detection runs automatically and skips unchanged deployment groups.

**On workflow_dispatch** — change detection is skipped entirely and the manual input toggles control which jobs run (all default to `true`).

#### Path Filter Mapping

| Filter group | Triggers when these paths change |
|---|---|
| `governance-int-root` | `templates/core/governance/mgmt-groups/int-root/**` |
| `governance-landingzones` | `templates/core/governance/mgmt-groups/landingzones/main.bicep`, `main.bicepparam` |
| `governance-landingzones-children` | `landingzones-corp/**`, `landingzones-online/**` |
| `governance-platform` | `templates/core/governance/mgmt-groups/platform/main.bicep`, `main.bicepparam` |
| `governance-platform-children` | `platform-connectivity/**`, `platform-identity/**`, `platform-management/**`, `platform-security/**` |
| `governance-sandbox` | `templates/core/governance/mgmt-groups/sandbox/**` |
| `governance-decommissioned` | `templates/core/governance/mgmt-groups/decommissioned/**` |
| `governance-rbac` | `**/main-rbac.bicep`, `**/main-rbac.bicepparam` |
| `core` | `templates/core/logging/**` |
| `networking` | `templates/networking/**` |

> **Shared files** — Every filter group also triggers on changes to `parameters.json`, `bicepconfig.json`, `templates/core/alzCoreType.bicep`, and `templates/core/governance/lib/**` (the ALZ policy library), since these are consumed by all or most templates.

#### Example

A PR that only changes `templates/networking/hubnetworking/main.bicep` will result in:

- `networking` → `true` (only this group deploys)
- All governance and core groups → `false` (skipped)

This avoids unnecessary ARM round-trips for unchanged stacks, saving several minutes of runner time per deployment.

#### Files Changed (caller repo `cmalz-mgmt`)

- `.github/workflows/cd.yaml` — Added `detect-changes` job; `plan_and_apply` wired to use path filter outputs on push, manual inputs on dispatch; `skip_what_if` defaults to `true`
