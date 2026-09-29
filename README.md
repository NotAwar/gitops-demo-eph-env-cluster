# gitops-demo-eph-env-cluster

Platform (cluster) repo. Application manifests live in
[gitops-demo-eph-env-workload](https://github.com/NotAwar/gitops-demo-eph-env-workload).

## Environments

| Env     | Example branch   | Namespace                  | 
|---------|------------------|----------------------------|
| dev     | `dev/login`      | `star-wars-dev-login`      | 
| staging | `staging`        | `star-wars-staging`        |
| staging | `staging/rc1`    | `star-wars-staging-rc1`    |
| test    | `test`           | `star-wars-test`           |
| prod    | `main`           | `star-wars`                |

## Layout

```
kind/cluster.yaml                             # local kind cluster definition
clusters/kind/flux-system/flux-instance.yaml  # FluxInstance, syncs clusters/kind from this repo
clusters/kind/tenants.yaml                    # one Flux Kustomization per tenant env
tenant/demo/base/ephemeral/                   # shared branch-per-env machinery (input provider + ResourceSet)
tenant/demo/dev/                              # overlay: dev/* branches
tenant/demo/staging/                          # overlay: staging branches
tenant/demo/test/                             # overlay: test branch
tenant/demo/prod/                             # static env tracking main with the workload prod overlay
```

## Branch environments (dev, staging, test)

Every matching branch in the workload repo gets its own namespace
`<app>-<env>[-<branch suffix>]` (`app` = the workload's prod namespace) containing:

- a scoped `flux` ServiceAccount (namespace `admin` only)
- `ResourceQuota` and `LimitRange` guardrails
- a `GitRepository` pinned to the branch HEAD commit
- a Flux `Kustomization` that builds `./deploy/base` from the workload repo and
  applies the platform overlay (replicas, `environment` label,
  `${ENV_ID}`, `${ENV_NAMESPACE}`, `${GIT_BRANCH}`, `${GIT_SHA}` substitutions)

Deleting the branch deletes the namespace and everything in it.

## Bootstrap

```sh
kind create cluster --config kind/cluster.yaml
helm install flux-operator oci://ghcr.io/controlplaneio-fluxcd/charts/flux-operator \
  --namespace flux-system --create-namespace --wait
kubectl apply -f clusters/kind/flux-system/flux-instance.yaml
```