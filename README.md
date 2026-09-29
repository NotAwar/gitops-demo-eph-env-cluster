# gitops-demo-eph-env-cluster

Platform (cluster) repo. Application manifests live in
[gitops-demo-eph-env-workload](https://github.com/NotAwar/gitops-demo-eph-env-workload).

## Environments

| Env     | Workload branch          | Namespace              | Source path     | Replicas |
|---------|--------------------------|------------------------|-----------------|----------|
| dev     | `dev/*` (one env each)   | `demo-dev-<id>`        | `deploy/base`   | 1        |
| staging | `staging`, `staging/*`   | `demo-staging-<id>`    | `deploy/base`   | 2        |
| test    | `test`                   | `demo-test`            | `deploy/base`   | 1        |
| prod    | `main`                   | `star-wars`            | `deploy/prod`   | 2        |

## Layout

```
kind/cluster.yaml                             # local kind cluster definition
clusters/kind/flux-system/flux-instance.yaml  # FluxInstance, syncs clusters/kind from this repo
clusters/kind/tenants.yaml                    # one Flux Kustomization per tenant env
tenant/demo/base/ephemeral/                   # shared branch-per-env machinery (input provider + ResourceSet)
tenant/demo/dev/                              # overlay: dev/* branches
tenant/demo/staging/                          # overlay: staging branches
tenant/demo/test/                             # static env tracking the test branch
tenant/demo/prod/                             # static env tracking main with the workload prod overlay
```

## Ephemeral environments (dev, staging)

Every matching branch in the workload repo gets its own namespace
`demo-<env>-<id>` containing:

- a scoped `flux` ServiceAccount (namespace `admin` only)
- `ResourceQuota` and `LimitRange` guardrails
- a `GitRepository` pinned to the branch HEAD commit
- a Flux `Kustomization` that builds `./deploy/base` from the workload repo and
  applies the platform overlay (replicas, `environment` label,
  `${ENV_ID}`, `${ENV_NAMESPACE}`, `${GIT_BRANCH}`, `${GIT_SHA}` substitutions)

Deleting the branch removes the whole environment.

Workload repo contract: a namespace-agnostic kustomize base at `deploy/base`
and a prod overlay at `deploy/prod`.

## Bootstrap

```sh
kind create cluster --config kind/cluster.yaml
helm install flux-operator oci://ghcr.io/controlplaneio-fluxcd/charts/flux-operator \
  --namespace flux-system --create-namespace --wait
kubectl apply -f clusters/kind/flux-system/flux-instance.yaml
```

Inspect:

```sh
flux-operator get instance flux -n flux-system
kubectl -n demo-dev get resourcesetinputprovider,resourceset
kubectl get ns -l ephemeral=true
```