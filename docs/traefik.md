# Traefik (stage)

Every UI in the cluster — Longhorn, Grafana, Prometheus — is an Ingress served
by the Traefik that k3s installs. It is the one component all of them depend
on, so it runs two replicas on different nodes, behind the one MetalLB address
every hostname resolves to:

```
longhorn / grafana / prometheus.jnet.lan
  ─▶ CNAME ingress-stage.jnet.lan ─▶ 192.168.0.26
                                       │  MetalLB: one worker answers ARP for it
                                       ▼
                                     Service/traefik (kube-system)
                                       │  kube-proxy spreads connections
                               ┌───────┴───────┐
                               ▼               ▼
                          traefik pod     traefik pod      (two different workers)
                               └───────┬───────┘
                                       ▼
                                     Ingress routes ─▶ each UI's ClusterIP Service
```

Grafana and Prometheus themselves are still single pods. This makes the way in
redundant, not the UIs behind it.

## What Flux manages

| File | Purpose |
| --- | --- |
| `infrastructure/stage/traefik/helmchartconfig.yaml` | Pins `.26`, two replicas, disruption budget, one pod per node, workers only, resource requests |
| `clusters/stage/traefik.yaml` | Flux Kustomization; waits for `metallb-pool`, since `.26` must exist in a pool first |

k3s owns the Traefik install itself. Flux only applies the `HelmChartConfig`;
k3s's helm-controller then merges it into the chart's values and runs a job,
`helm-install-traefik`, to upgrade the release. Flux reports success as soon
as the `HelmChartConfig` is applied — it cannot tell whether that job worked,
so check the job, not Flux (see [Verifying](#verifying)).

## What two replicas buy

| Event | One pod | Two pods, different nodes |
| --- | --- | --- |
| Traefik pod crashes or is OOM-killed | Every UI down until it restarts | The other pod keeps serving |
| The node running Traefik dies | MetalLB moves `.26` in seconds, but no Traefik pod exists until Kubernetes evicts and reschedules it — about 5 minutes | The dead pod's endpoint is dropped within about a minute; the other serves throughout |
| Node drain, reboot, k3s upgrade | Short outage while the pod moves | The disruption budget holds one pod up throughout |
| Traefik upgrade | Rolling update, no outage | Same |

## DNS

Nothing changes. `.26` belongs to the Service, not to a pod, so the number of
pods behind it is invisible from outside. `ingress-stage.jnet.lan` stays an A
record to `192.168.0.26`, and every UI hostname a CNAME to it:

```bash
dig +short ingress-stage.jnet.lan   # 192.168.0.26
dig +short grafana.jnet.lan         # ingress-stage.jnet.lan. then 192.168.0.26
```

## Verifying

After a push, from any shell with `kubectl` pointed at stage:

```bash
flux -n flux-system reconcile kustomization traefik --with-source

# 1. k3s ran the upgrade - expect COMPLETIONS 1/1 and an AGE from just now
kubectl -n kube-system get job helm-install-traefik

# 2. Wait for the rollout to finish, then expect exactly two pods, Running,
#    on two different NODEs. Mid-rollout there are three - old and new
#    ReplicaSets side by side - and counting Ready pods then is misleading.
kubectl -n kube-system rollout status deploy/traefik
kubectl -n kube-system get pods -l app.kubernetes.io/name=traefik -o wide

# 3. Disruption budget - expect MIN AVAILABLE 1, ALLOWED DISRUPTIONS 1
kubectl -n kube-system get pdb traefik

# 4. Address unchanged, and which worker is currently answering for it
kubectl -n kube-system get svc traefik          # EXTERNAL-IP 192.168.0.26
kubectl -n metallb-system get servicel2statuses

# 5. Every UI still answers through .26
for h in longhorn grafana prometheus; do
  curl -ks -o /dev/null -w "$h %{http_code}\n" \
    --resolve "$h.jnet.lan:443:192.168.0.26" "https://$h.jnet.lan"
done
# longhorn 401, grafana 302, prometheus 401 - the auth prompts, as intended
```

`ALLOWED DISRUPTIONS 0` means one of the pods is not Ready; step 2 shows which.

### Seeing failover happen

In one terminal, poll Grafana's login page through `.26` twice a second:

```bash
while :; do
  curl -ks -o /dev/null -w '%{http_code} ' \
    --resolve grafana.jnet.lan:443:192.168.0.26 https://grafana.jnet.lan/login
  sleep 0.5
done
```

In another, delete one of the two Traefik pods:

```bash
kubectl -n kube-system delete pod \
  "$(kubectl -n kube-system get pods -l app.kubernetes.io/name=traefik -o name | head -1)"
```

The first terminal keeps printing `200`. A single `000` right at the delete is
possible — a connection kube-proxy sent to the dying pod before it stopped
routing there — but not a run of them. The Deployment replaces the pod within
seconds. Ctrl-C the loop when done.

## Troubleshooting

### The change never reached Traefik

Pods unchanged after a push, and step 1 above shows an old job or a failed
one: the helm-controller could not apply the values, most often because the
chart's schema rejected one of them. The job's log says which:

```bash
kubectl -n kube-system logs job/helm-install-traefik --tail=30
kubectl -n kube-system get helmchart traefik -o jsonpath='{.status}'; echo
```

Fix the value in `helmchartconfig.yaml` and push; the helm-controller retries on
its own. Traefik keeps running on its previous values in the meantime, so a bad
value does not take the UIs down.

### The second pod stays Pending

The spread rule is `DoNotSchedule`: the second pod may not share a node with
the first, and would rather wait than do so. `describe` names the reason:

```bash
kubectl -n kube-system describe pod -l app.kubernetes.io/name=traefik | grep -A3 Events
```

The event lists a reason per node. `didn't match Pod's node affinity/selector`
against the three control-plane nodes is expected — that is the workers-only
rule. `didn't match pod topology spread constraints` against the workers means
no other worker could take it — the rest full, cordoned or down. With three
workers that takes two of them out at once; fix the workers rather than
relaxing the rule.

## Changing the replica count

Edit `deployment.replicas` in `helmchartconfig.yaml` and push. Going back to a
single replica, also set `podDisruptionBudget.enabled: false` in the same
commit: a budget of `minAvailable: 1` over one pod never allows it to be
evicted, so every node drain — including the ones k3s upgrades do — hangs on
it forever.

## Notes

- MetalLB's L2 mode is failover, not load balancing: one worker at a time
  answers for `.26`, and another takes over within seconds if it dies. That is
  independent of how many Traefik pods there are. Spreading requests across
  pods is kube-proxy's job, and across backends Traefik's.
- The pods run on workers only, by a node affinity that mirrors the
  L2Advertisement's selector. It is needed despite the control-plane taint:
  k3s's own chart values tolerate that taint, and before the affinity both
  pods landed on the etcd nodes.
- They have resource requests (50m CPU, 128Mi) and no limits. The workers are
  what [load-test](load-test.md) saturates, and without requests Traefik
  would be BestEffort — among the first pods evicted under memory pressure. A
  memory limit would trade that for an OOM kill, which takes every UI down
  just the same.
- The Service stays `externalTrafficPolicy: Cluster`: MetalLB announces `.26`
  from any worker and kube-proxy forwards to wherever the pods are. `Local`
  would make MetalLB announce only from a node running a Traefik pod, so every
  pod reschedule would also move `.26`, and nothing here needs the client IP
  it would preserve.
- `prod` has none of this yet — nothing under `clusters/prod/` configures
  Traefik, so it runs with k3s's defaults.
