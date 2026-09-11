# k3s add-ons (stage)

k3s installs a few components itself — CoreDNS, metrics-server and the
local-path provisioner — from manifests under
`/var/lib/rancher/k3s/server/manifests` on each control-plane node. It
re-applies them every time k3s starts. They are not Flux's, and must not
become Flux's:

- Deleting one from here would be undone by k3s at its next restart.
- CoreDNS especially must never be taken over or pruned: Flux resolves
  `gitbox.jnet.lan` through it, so a Flux mistake there would cut Flux off from
  the very repo that could fix it.

So instead of owning them, Flux **patches individual fields** on them, the
same way the `HelmChartConfig` adjusts k3s's Traefik without owning it.

## What Flux manages

Everything in `infrastructure/stage/k3s-addons/`, reconciled by
`clusters/stage/k3s-addons.yaml`:

| File | Patches | Change |
| --- | --- | --- |
| `metrics-server.yaml` | `Deployment/metrics-server` | Workers only (node affinity) |

Not patched, and left as k3s deploys them:

- **CoreDNS** — a single replica, on the first control-plane node. Losing
  that node stops DNS inside the cluster for about 5 minutes, until Kubernetes
  reschedules it. Two replicas and a disruption budget would fix it, via a
  patch here.
- **local-path-provisioner** — unused (every volume is on Longhorn), and its
  `local-path` StorageClass is marked default alongside `longhorn`. Not
  confirmed yet whether k3s's own manifest sets an affinity, which would undo
  a move on every restart.

## How a patch works

Each file is a partial object: `apiVersion`, `kind`, name and namespace, plus
only the fields to change. Flux applies it with server-side apply, which
merges it into the live object and leaves every other field — k3s's — alone.
Each file also carries

```yaml
metadata:
  annotations:
    kustomize.toolkit.fluxcd.io/prune: disabled
```

so that removing the file, or the whole Kustomization, never deletes the
k3s object.

Only patch fields k3s's manifest does **not** set. k3s re-applies its
manifest on every start and resets any field it sets itself; fields it leaves
out survive. To see what k3s sets — it keeps its last-applied manifest in an
annotation:

```bash
kubectl -n kube-system get deploy metrics-server \
  -o jsonpath='{.metadata.annotations.objectset\.rio\.cattle\.io/applied}' \
  | base64 -d | gunzip \
  | jq '{replicas: .spec.replicas, affinity: .spec.template.spec.affinity}'
# {"replicas": null, "affinity": null} - neither is set, both safe to patch
```

Swap `metrics-server` for `coredns` or `local-path-provisioner` to check those.

## Verifying

From any shell with `kubectl` pointed at stage:

```bash
flux -n flux-system reconcile kustomization k3s-addons --with-source

# 1. The patch reached the Deployment - expect DoesNotExist
kubectl -n kube-system get deploy metrics-server \
  -o jsonpath='{.spec.template.spec.affinity.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution.nodeSelectorTerms[0].matchExpressions[0].operator}'; echo

# 2. Rollout finished, and the pod is on a worker (k3s-wk-*)
kubectl -n kube-system rollout status deploy/metrics-server
kubectl -n kube-system get pods -l k8s-app=metrics-server -o wide

# 3. The API server can reach it there - expect AVAILABLE True
kubectl get apiservice v1beta1.metrics.k8s.io

# 4. And it can reach every kubelet - expect all six nodes listed
kubectl top nodes
```

`kubectl top` goes blank for a few seconds while the pod moves; nothing else
uses metrics-server (Prometheus and Grafana collect their own metrics).

After a k3s restart or upgrade on the control-plane nodes, re-run step 2. The
pod should still be on a worker; if it has moved back, k3s's manifest has
started setting an affinity of its own — check with the command in
[How a patch works](#how-a-patch-works).

## Troubleshooting

### `kubectl top` says the Metrics API is not available

The pod is running, but the API server cannot reach it on the worker:

```bash
kubectl get apiservice v1beta1.metrics.k8s.io
kubectl describe apiservice v1beta1.metrics.k8s.io | grep -A3 Conditions
kubectl -n kube-system logs deploy/metrics-server --tail=30
```

While that APIService is unavailable, deleting a namespace can hang: the
namespace controller cannot list metrics resources in it. Not expected here —
the API server already reaches webhooks on the workers, and pods on the workers
already reach every kubelet — but if it happens, undo the move (below) and it
recovers.

## Undoing a patch

Removing the file and pushing is not enough on its own. Because of
`prune: disabled`, Flux leaves the object as it is, affinity included. Remove
the file from `kustomization.yaml`, push, then take the field off by hand:

```bash
kubectl -n kube-system patch deploy metrics-server --type=json \
  -p='[{"op":"remove","path":"/spec/template/spec/affinity"}]'
```

The pod is then free to schedule anywhere again, as k3s deployed it.
