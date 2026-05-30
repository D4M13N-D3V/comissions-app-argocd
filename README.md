# comissions.app — ArgoCD GitOps

GitOps configuration for the comissions.app platform, deployed by [Argo CD](https://argo-cd.readthedocs.io/) using the **App-of-Apps** pattern with `ApplicationSet`s and Helm charts.

## Repository layout

```
charts/
  core-api/        Helm chart for the core API (.NET) service
  ui/              Helm chart for the Next.js UI
  postgresql/      Vendored Bitnami PostgreSQL chart
dev/
  all-dev-apps.yaml    Root Application (app-of-apps) — points Argo CD at dev/apps
  apps/                One ApplicationSet per component
    core-api.yaml
    ui.yaml
    postgres.yaml
  .config/
    config.json        Per-environment cluster/project config (single source of truth)
```

## How it fits together

1. **`dev/all-dev-apps.yaml`** is the root `Application`. It watches `dev/apps/` and creates the ApplicationSets found there.
2. Each **ApplicationSet** in `dev/apps/` uses a Git *files* generator that reads **`dev/.config/config.json`**, then renders an `Application` for its component from the matching chart under `charts/`.
3. `config.json` is the single source of truth for the target cluster and Argo CD project — the ApplicationSets read `project`, `destination.server`, and `destination.namespace` from it via `goTemplate` (`missingkey=error`, so a missing key fails the render loudly).

```
all-dev-apps.yaml (root Application)
  └─ dev/apps/*.yaml (ApplicationSet, generator: config.json)
       └─ Application → charts/<component>  →  workloads in the comissions-dev namespace
```

## Bootstrap

Apply the root Application once; Argo CD reconciles everything else:

```sh
kubectl apply -f dev/all-dev-apps.yaml
```

- The root Application and the ApplicationSet objects live in the `argocd` namespace.
- Component **workloads** deploy into the **`comissions-dev`** namespace (created automatically via `CreateNamespace=true`).
- Sync is automated with `prune` and `selfHeal` enabled.

## Configuration

| Where | What |
|-------|------|
| `dev/.config/config.json` | Cluster address/name, target namespace, Argo CD project |
| `dev/apps/<component>.yaml` | Per-component Helm value overrides (image, ingress host, autoscaling, resources) |
| `charts/<component>/values.yaml` | Chart defaults |

## Secrets

> ⚠️ **Not yet implemented.** Application secrets are currently rendered as plaintext Helm values / environment variables. Externalizing them (Sealed Secrets or External Secrets Operator) and removing them from Git is tracked in the security remediation epic and **must be completed before any live deployment.**

## Conventions

- One concern per pull request; PRs targeting overlapping files are stacked so they merge cleanly in order.
- Chart changes go in `charts/`; environment-specific overrides go in `dev/apps/`.
