# Load test (stage)

A stress-ng StatefulSet for putting CPU, memory and Longhorn I/O load on the
worker nodes — added to kick the tires on a freshly built cluster, and parked
at zero replicas the rest of the time. It is committed at `replicas: 0`, so
nothing runs unless you ask for it.

Manifests are in `infrastructure/base/load-test/`, reconciled by
`clusters/stage/load-test.yaml`. Only stage has that Kustomization, so despite
living under `base/` it does not run on prod.

## What each pod does

`StatefulSet/worker-stress` in the `load-test` namespace runs
`ghcr.io/alexei-led/stress-ng:0.20.01` with:

| Stressor | Setting | Loads |
| --- | --- | --- |
| `--cpu 2` | two CPU workers | CPU |
| `--vm 1 --vm-bytes 512M` | one worker cycling 512M | memory |
| `--iomix 1 --iomix-bytes 256M` | mixed reads/writes on `/data` | Longhorn |

`/data` is a dedicated 10Gi Longhorn volume per pod, from the StatefulSet's
`volumeClaimTemplates`, so the I/O goes through Longhorn's replication to the
other workers rather than to local disk. Each pod requests 250m CPU / 512Mi
and is capped at 2 CPU / 2Gi, so it cannot starve a node outright.

Placement is strict: pods only schedule on nodes without the control-plane
role, and required anti-affinity keeps it to one pod per node. Stage has three
workers, so **3 is the maximum** — a fourth pod stays `Pending` rather than
doubling up.

## Running it

The normal way is a commit, like any other change. Set `replicas` in
`infrastructure/base/load-test/stress-statefulset.yaml` — 1 to 3 — push, and
set it back to 0 when done. The pods run until then; the StatefulSet restarts
stress-ng whenever it exits.

For a quick ad-hoc run without two commits, suspend the Kustomization first.
Without that, `kubectl scale` works only until Flux next re-applies the
committed `replicas: 0`, within ten minutes:

```bash
flux -n flux-system suspend kustomization load-test
kubectl -n load-test scale statefulset worker-stress --replicas=3

# ...when done - resuming puts replicas back to the committed 0 on its own
flux -n flux-system resume kustomization load-test
```

## Watching it

```bash
kubectl -n load-test get pods -o wide   # one per worker
kubectl top nodes
kubectl -n load-test top pods
```

In Grafana, *Node Exporter / Nodes* shows the load per worker, and
*Kubernetes / Compute Resources / Namespace (Pods)* with namespace `load-test`
shows it per pod. The Longhorn UI shows the `load-test` volumes and their
replicas.

## Cleaning up

Scaling to 0 stops the pods but **does not delete their volumes** —
StatefulSets keep their PVCs on scale-down by design. Every pod that has ever
run leaves a 10Gi Longhorn volume, replicated three times, counting against
Longhorn's schedulable space on the workers. Nothing else lives in the
namespace, so once it is scaled to 0:

```bash
kubectl -n load-test get pvc
kubectl -n load-test delete pvc --all
```

The Longhorn StorageClass reclaims on delete, so this removes the volumes and
their replicas too. The next run provisions fresh ones.
