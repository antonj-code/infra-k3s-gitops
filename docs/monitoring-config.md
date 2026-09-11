# Scrape targets and dashboards (stage)

Prometheus scrape targets, alert rules and Grafana dashboards are managed as
code under `monitoring/configs/`, next to the kube-prometheus-stack install
in `monitoring/controllers/`:

```
monitoring/configs/
├── base/
│   └── longhorn/
│       ├── servicemonitor.yaml   # Prometheus scrapes every longhorn-manager
│       ├── longhorn.json         # the Longhorn dashboard, as Grafana exports it
│       └── kustomization.yaml    # turns the JSON into a labelled ConfigMap
└── stage/                        # pulls in base
```

It is reconciled by its own Flux Kustomization, `clusters/stage/monitoring-configs.yaml`,
with `dependsOn: monitoring`. That split is not optional: `ServiceMonitor` and
`PrometheusRule` are CRDs the kube-prometheus-stack chart installs, so on a
fresh cluster they do not exist until `monitoring` is Ready, and Flux's
dry-run of any path containing them fails until then.

## Adding a scrape target

Add a `ServiceMonitor` (or `PodMonitor`) under `base/<component>/`. It must
carry the label

```yaml
labels:
  release: kube-prometheus-stack
```

— this Prometheus only selects monitors with it, and a monitor without it is
silently ignored. The monitor can live in `monitoring` and point at another
namespace with `namespaceSelector`, as the Longhorn one does.

If a component's own chart can create its ServiceMonitor, that is usually
simpler, with one exception: anything that has to install *before*
kube-prometheus-stack. Longhorn is the case in point — monitoring's volumes
are on Longhorn, so on a rebuild Longhorn installs first, and a Longhorn
release carrying a ServiceMonitor would fail for want of the CRD. Those
monitors belong here.

Alert and recording rules work the same way, as `PrometheusRule` objects with
the same `release` label.

Check a new target on `https://prometheus.jnet.lan/targets` — it should list
one endpoint per pod behind the Service, all `UP`.

## Adding or changing a dashboard

Grafana's sidecar loads every ConfigMap labelled `grafana_dashboard: "1"`.
Keep the dashboard as the JSON file Grafana exports and let kustomize wrap it
— see `base/longhorn/kustomization.yaml`:

```yaml
configMapGenerator:
  - name: grafana-dashboard-<name>
    files:
      - <name>.json
    options:
      labels:
        grafana_dashboard: "1"
      disableNameSuffixHash: true
```

The file name becomes the file the sidecar writes, in one directory shared by
every dashboard, so it must be unique — two dashboards both called
`dashboard.json` would overwrite each other.

Dashboards loaded this way cannot be saved from the UI. To change one:

1. Edit it in Grafana, then **Save dashboard → Save JSON to file**, or
   **Export → Export as JSON**.
2. Overwrite the JSON file here with it and commit.

Keep the `uid` the same across edits; links and bookmarks use it. Dashboards
created in the UI and never exported live only in Grafana's database on its
Longhorn volume — they survive restarts, but not a lost volume or a rebuild,
and they are not in Git.

## The Longhorn dashboard

**Dashboards → Longhorn**, built from the metrics Longhorn 1.6 exposes:

- **Degraded / Faulted volumes** — volumes short of replicas, and volumes with
  none left. Degraded still serves data; it is one more failure from faulted.
- **Nodes not ready** and **Storage used** across every Longhorn disk.
- **Volume robustness** over time — when each volume lost or regained
  replicas. A worker being repaved shows as every volume going degraded at
  once, then back to healthy as Longhorn rebuilds onto the new node.
- **Volumes** — robustness, state, size and actual use per PVC.
- Per-node storage use, and per-volume throughput, IOPS and latency.

Robustness is reported as `1` healthy, `2` degraded, `3` faulted, `0`
unknown; the dashboard maps these to names.
