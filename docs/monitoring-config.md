# Scrape targets and dashboards (stage)

Prometheus scrape targets, alert rules and Grafana dashboards are managed as
code under `monitoring/configs/`, next to the kube-prometheus-stack install
in `monitoring/controllers/`:

```
monitoring/configs/
├── base/
│   └── longhorn/
│       ├── servicemonitor.yaml   # Prometheus scrapes every longhorn-manager
│       ├── prometheusrule.yaml   # Longhorn alert rules - displayed, not sent
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

Check a new target on `https://prometheus.jnet.lan/targets` — it should list
one endpoint per pod behind the Service, all `UP`.

## Alert rules — displayed, not sent

Alert rules are `PrometheusRule` objects, and need the same
`release: kube-prometheus-stack` label as ServiceMonitors. Nothing is sent
anywhere: Alertmanager's config is the chart default, which routes every
alert to a receiver called `"null"`. Pending and firing alerts show in three
places instead:

- `https://prometheus.jnet.lan/alerts` — every rule, grouped, with its state.
- Grafana, **Alerting → Alert rules**, listed under the Prometheus data source.
  Grafana reads these from Prometheus; they are not editable there.
- The **Active alerts** panel on a dashboard, which is a table over
  Prometheus's `ALERTS` series — see the Longhorn dashboard for the pattern.

The chart ships around 150 rules of its own, covering nodes, kubelets,
workloads and Prometheus itself, and those show alongside. Two fire all the
time by design and are not a problem: `Watchdog`, a heartbeat meant to prove
the pipeline works end to end, and `InfoInhibitor`, which exists to suppress
info-level alerts.

To start sending alerts later, add a receiver and route — an
`AlertmanagerConfig` object here, or `alertmanager.config` in the HelmRelease
values — with any webhook URL or password coming from Vault through an
ExternalSecret.

### Longhorn rules

`base/longhorn/prometheusrule.yaml`:

| Alert | Fires when | For | Severity |
| --- | --- | --- | --- |
| `LonghornVolumeFaulted` | a volume has no healthy replica | 2m | critical |
| `LonghornVolumeDegraded` | a volume is short of replicas | 30m | warning |
| `LonghornNodeNotReady` | fewer ready nodes than Longhorn expects | 10m | warning |
| `LonghornDiskNotReady` | a Longhorn disk is not ready | 10m | warning |
| `LonghornStorageAlmostFull` | a node's Longhorn storage is over 80% | 15m | warning |

`LonghornVolumeDegraded` waits 30 minutes on purpose. Repaving a worker
degrades every volume for Longhorn's 10-minute replica replenishment wait
plus the rebuild, which is expected and visible on the dashboard; the alert
is for a rebuild that has stalled. 30 minutes is also how long the repave
scripts in `infra-k3s-bootstrap` wait for volumes to recover.

`LonghornNodeNotReady` counts ready nodes against `longhorn_node_count_total`
instead of looking for a not-ready status, because each `longhorn-manager`
reports only its own node — when a node goes down its status series
disappears rather than turning 0.

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
- **Nodes not ready** — nodes Longhorn expects that are not ready or not
  reporting — and **Storage used** across every Longhorn disk.
- **Active alerts** — Longhorn rules currently pending or firing.
- **Volume robustness** over time — when each volume lost or regained
  replicas. A worker being repaved shows as every volume going degraded at
  once, then back to healthy as Longhorn rebuilds onto the new node.
- **Volumes** — robustness, state, size and actual use per PVC.
- Per-node storage use, and per-volume throughput, IOPS and latency.

Robustness is reported as `1` healthy, `2` degraded, `3` faulted, `0`
unknown; the dashboard maps these to names.
