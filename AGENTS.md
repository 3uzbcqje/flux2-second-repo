# AGENTS.md

Guidance for AI coding agents working in this repository.

## What this repo is

A GitOps manifest repository (no application source code): Flux CD reconciles these YAML manifests onto Kubernetes clusters. It started as a fork of `fluxcd/flux2-kustomize-helm-example` — `README.md` is the upstream guide and describes the `staging`/`production` example — and has been adapted to serve as a **second** Flux `GitRepository` attached to an already-bootstrapped cluster (`clusters/test`). See `README.evans.md`: **do not run `flux bootstrap` against this repo**; the `test` cluster is wired up manually with `flux create secret git` + a GitHub deploy key + `kubectl apply -f clusters/test/flux-system/gotk-sync.yaml`.

Consequences worth knowing before editing:

- `clusters/staging/flux-system/gotk-components.yaml` and the production equivalent are one-line placeholders, not real Flux component manifests — staging and production are unbootstrapped template leftovers. `clusters/test` has no `gotk-components.yaml` at all, because Flux itself is owned by the cluster's primary repo.
- `clusters/test/` therefore diverges from the upstream pattern: its `Kustomization`s reference `sourceRef.name: flux2-second-repo` (not `flux-system`), are named with a `-flux2-second-repo` suffix to avoid colliding with the primary repo's objects in the shared `flux-system` namespace, and have no `dependsOn` (there is no infrastructure layer for `test`).

## Commands

```sh
# Validate everything (YAML syntax, cluster manifests, every kustomize overlay).
# Downloads Flux CRD schemas to /tmp/flux-crd-schemas. This is exactly what CI runs.
./scripts/validate.sh

# Validate a single overlay (the loop body of validate.sh)
kustomize build ./apps/staging --load-restrictor=LoadRestrictionsNone | \
  kubeconform -skip=Secret -strict -ignore-missing-schemas \
    -schema-location default -schema-location /tmp/flux-crd-schemas -verbose

# Render an overlay to see what Flux will apply
kustomize build ./apps/production --load-restrictor=LoadRestrictionsNone
```

`validate.sh` prerequisites: yq v4.34, kustomize v5.3, kubeconform v0.6. `--load-restrictor=LoadRestrictionsNone` mirrors kustomize-controller's build options — overlays reference files outside their own directory (`../base/podinfo`) and will not build without it.

## Architecture

Three layers, each synced by its own Flux `Kustomization` defined per cluster under `clusters/<name>/`:

| Dir | Contents | Synced by |
| --- | --- | --- |
| `infrastructure/controllers/` | namespaces + `HelmRepository`/`HelmRelease` for cert-manager, ingress-nginx | `infra-controllers` |
| `infrastructure/configs/` | CRs that depend on those controllers (e.g. `ClusterIssuer`) | `infra-configs` (`dependsOn: infra-controllers`) |
| `apps/base/` + `apps/<cluster>/` | app `HelmRelease`s and per-cluster overlays | `apps` (`dependsOn: infra-configs`) |

The `dependsOn` chain plus `wait: true` is what guarantees CRDs are registered before custom resources are applied. Preserve it when adding a layer.

**Per-cluster variation happens in two places, both patch-based — never by forking base manifests:**

1. `apps/<cluster>/kustomization.yaml` applies `podinfo-values.yaml` as a strategic-merge patch targeting `kind: HelmRelease`. This is where chart semver range and Helm values differ (`>=1.0.0-alpha` on staging to pick up pre-releases, `>=1.0.0` on production for stable only).
2. `clusters/<cluster>/infrastructure.yaml` applies JSON-patches from the Flux `Kustomization` itself — e.g. production rewrites the Let's Encrypt ACME server to the production endpoint. Base manifests always hold the *staging*-safe value.

Chart upgrades are automatic: `HelmRelease.spec.chart.spec.version` holds a semver range and `sourceRef ... interval` sets how often the chart repo index is polled. Chart versions are deliberately not pinned; don't "fix" a range to an exact version without being asked.

Namespaces carry a `toolkit.fluxcd.io/tenant` label, and the `flux-system` kustomizations stamp the same label onto everything they own.

## Repo-specific gotchas

- **`.sourceignore` excludes everything at the repo root except `/apps/`, `/clusters/`, `/infrastructure/`.** A new top-level directory of manifests is invisible to Flux until it is added there.
- `scripts/validate.sh` walks *all* `kustomization.yaml` files including `clusters/*/flux-system/`, so a broken flux-system overlay fails CI.
- The `test` overlay (`apps/test/`) is a deliberately trivial namespace + ConfigMap used to prove the second-repo wiring works; it is not a podinfo deployment.
- `README.md` is upstream text and has drifted from the manifests (it refers to `podinfo-patch.yaml`, omits the redis values in `apps/base/podinfo/release.yaml`, and predates `clusters/test`). Trust the YAML over the README.

## CI

- `.github/workflows/test.yaml` — runs `scripts/validate.sh` on PRs and pushes to any branch.
- `.github/workflows/e2e.yaml` — kind cluster + `flux install`, reconciles `./clusters/staging` with `--ignore-paths="clusters/**/flux-system/"`, waits on `infra-controllers`, `apps`, and the podinfo `HelmRelease`.
- `.github/workflows/claude.yml` and `claude-code-review.yml` — `anthropics/claude-code-action` runs. These are pinned to full commit SHAs and to `--model claude-opus-5 --effort xhigh` on purpose, and their inline comments explain the permission scoping and the `--allowed-tools` allowlist requirement. Keep the SHA pins and read those comments before changing tool/permission scope.
