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
whatif (single validation job)
  └─ deploy-governance-root
       ├─ deploy-governance-parents ──┬── deploy-governance-children-lz ──────┐
       │   (LZ, Platform,            │                                        ├─ deploy-governance-rbac
       │    Sandbox, Decommissioned)  └── deploy-governance-children-platform ┘
       │
       └─ deploy-core ──── deploy-networking
```

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
- The CD pipeline's `whatif` job still runs a full ARM what-if before any real deployment
- ARM what-if slowness is specific to the `New-Az*Deployment -WhatIf` diff engine — Deployment Stacks (`New-Az*DeploymentStack`) don't have this problem because they submit templates directly to ARM without computing a full before/after diff

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
| CD deploy wall time | 60-90 min | ~30-40 min |
| ARM API calls per run | ~17 × (enumerate + batch delete) | 0 cleanup calls |
| GitHub Actions runners | 2 concurrent | Up to 8 concurrent (CD) / 1 (CI) |

## Files Changed

- `.github/workflows/cd-template.yaml` — Parallel deploy jobs with dependency DAG
- `.github/workflows/ci-template.yaml` — Removed all what-if jobs, CI is now validate-only
- `.github/actions/bicep-deploy/action.yaml` — Removed deployment cleanup blocks
