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

Split the single `whatif` job (17 sequential what-if checks) into **3 parallel jobs**:

| Job | What-If Steps | Description |
|-----|---------------|-------------|
| `whatif-governance` | 8 | Root, parent MGs, LZ children, Platform children (connectivity, identity) |
| `whatif-governance-rbac` | 7 | Platform mgmt/security, Sandbox, Decommissioned, all RBAC |
| `whatif-infra` | 2 | Core logging, Hub networking |

All 3 jobs run **simultaneously**, reducing wall-clock time from ~60+ minutes to ~25 minutes (limited by the slowest parallel job).

Also removed `concurrency: mgmt-tfstate` from CI — what-if is read-only validation and doesn't need to block behind CD deployments.

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
| CI what-if wall time | 60-80 min | ~20-25 min |
| CD deploy wall time | 60-90 min | ~30-40 min |
| ARM API calls per run | ~17 × (enumerate + batch delete) | 0 cleanup calls |
| GitHub Actions runners | 2 concurrent | Up to 8 concurrent (CD) / 4 concurrent (CI) |

## Files Changed

- `.github/workflows/cd-template.yaml` — Parallel deploy jobs with dependency DAG
- `.github/workflows/ci-template.yaml` — 3 parallel what-if jobs
- `.github/actions/bicep-deploy/action.yaml` — Removed deployment cleanup blocks
