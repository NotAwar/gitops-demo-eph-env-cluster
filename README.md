# gitops-demo-eph-env-cluster

Platform (cluster) repo. Application manifests live in
[gitops-demo-eph-env-workload](https://github.com/NotAwar/gitops-demo-eph-env-workload).

## Layout

```
kind/cluster.yaml                        # local kind cluster definition
clusters/kind/flux-system/flux-instance.yaml  # FluxInstance, syncs clusters/kind from this repo
clusters/kind/tenants.yaml               # Flux Kustomization -> tenant/demo/dev
tenant/demo/dev/
  namespace.yaml                         # demo-dev control namespace
  input-provider.yaml                    # watches dev/* branches in the workload repo
  resourceset.yaml                       # per-branch ephemeral env + platform dev overlay
```

## Ephemeral dev environments

Every branch matching `dev/*` in the workload repo gets its own namespace
`demo-dev-<id>` containing:

- a scoped `flux` ServiceAccount (namespace `admin` only)
- `ResourceQuota` and `LimitRange` guardrails
- a `GitRepository` pinned to the branch HEAD commit
- a Flux `Kustomization` that builds `./deploy/base` from the workload repo and
  applies the platform dev overlay (1 replica, `environment=dev` labels,
  `${ENV_ID}`, `${ENV_NAMESPACE}`, `${GIT_BRANCH}`, `${GIT_SHA}` substitutions)

Deleting the branch removes the whole environment.

Workload repo contract: a kustomize base at `deploy/base` (namespace-agnostic).

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