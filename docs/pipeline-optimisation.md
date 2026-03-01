# Pipeline Optimisation Changes

## Problem

CI/CD pipelines were taking over an hour to complete, with CI what-if jobs frequently timing out.

1. **Sequential execution** — All 17 deployment steps ran in a single job
2. **Pre-deployment cleanup** — Every step enumerated and deleted ARM deployments before creating the deployment stack
3. **No parallelism** — Independent deployments waited for each other unnecessarily

## Changes

### CD Template (`cd-template.yaml`)

Replaced the monolithic 2-job workflow with **8 parallel jobs** following a dependency DAG:

```
deploy-governance-root
     ├─ deploy-governance-parents ──┬── deploy-governance-children-lz ──────┐
     │   (LZ, Platform,            │                                        ├─ deploy-governance-rbac
     │    Sandbox, Decommissioned)  └── deploy-governance-children-platform ┘
     │
     └─ deploy-core ──── deploy-networking
```

The governance and networking chains run fully in parallel. The `whatif` job is retained but **skipped by default** (`skip_what_if: true`), re-enabled via `workflow_dispatch`.

| Wave | Jobs | Runs in parallel with |
|------|------|-----------------------|
| 1 | `deploy-governance-root` | — |
| 2 | `deploy-governance-parents`, `deploy-core`, `deploy-networking` | Each other |
| 3 | `deploy-governance-children-lz`, `deploy-governance-children-platform` | Each other |
| 4 | `deploy-governance-rbac` | — |

### CI Template (`ci-template.yaml`)

Removed all ARM what-if jobs. CI runs a single `validate` job (`bicep build` + lint), ~1-2 minutes.

### Bicep Deploy Action (`bicep-deploy/action.yaml`)

Removed deployment cleanup blocks (`Get-Az*Deployment` + `Remove-Az*Deployment`). Deployment Stacks manage their own lifecycle.

## Impact

| Metric | Before | After |
|--------|--------|-------|
| CI wall time | 60-80 min | ~1-2 min |
| CD wall time (full run) | 60-90 min | ~30-40 min |
| CD wall time (targeted) | 60-90 min | ~5-15 min |

## Files Changed (templates repo)

- `.github/workflows/cd-template.yaml` — Parallel deploy jobs with dependency DAG; `skip_what_if` defaults to `true`
- `.github/workflows/ci-template.yaml` — Validate-only
- `.github/actions/bicep-deploy/action.yaml` — Removed deployment cleanup blocks

---

## Path-Based Change Detection (caller `cd.yaml` in `cmalz-mgmt`)

The caller `cd.yaml` includes a **`detect-changes`** job using [`dorny/paths-filter@v3`](https://github.com/dorny/paths-filter). Only deployment groups whose templates changed are sent to the reusable workflow.

- **On push to main** — change detection runs automatically; unchanged groups are skipped
- **On workflow_dispatch** — change detection is skipped; manual input toggles control which jobs run

### Path Filter Mapping

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

> **Shared files** — Every filter group also triggers on changes to `parameters.json`, `bicepconfig.json`, `templates/core/alzCoreType.bicep`, and `templates/core/governance/lib/**`.

### Files Changed (caller repo `cmalz-mgmt`)

- `.github/workflows/cd.yaml` — Added `detect-changes` job; `plan_and_apply` wired to path filter outputs on push, manual inputs on dispatch
