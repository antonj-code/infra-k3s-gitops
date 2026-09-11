# Grafana and Prometheus UIs (stage)

Both sit behind the Traefik that k3s installs, the same way the
[Longhorn UI](longhorn-ui.md) does, with one difference in what guards them:

```
grafana.jnet.lan    ─▶ 192.168.0.26 (traefik) ─▶ redirect to https
                                                  ▼
                                                Service/kube-prometheus-stack-grafana:80
                                                  (Grafana's own login)

prometheus.jnet.lan ─▶ 192.168.0.26 (traefik) ─▶ redirect to https
                                                 basicAuth ◀── Secret/basic-auth ◀── Vault
                                                  ▼
                                                Service/kube-prometheus-stack-prometheus:9090
```

- **Prometheus** has no authentication at all, so Traefik's basic auth is the
  only thing in front of it.
- **Grafana** has its own login and gets only the HTTPS redirect. Putting
  basic auth in front of it as well does not just double-prompt — Traefik
  forwards the `Authorization` header, Grafana tries it as a Grafana login and
  rejects it, and signing in breaks.

Grafana's datasource talks to Prometheus over the in-cluster Service, not
through Traefik, so basic auth does not affect dashboards.

## What Flux manages

Everything in `monitoring/controllers/stage/`, reconciled by
`clusters/stage/monitoring.yaml`:

| File | Purpose |
| --- | --- |
| `kube-prometheus-stack/ingress.yaml` | One Ingress per host, each with its own middleware chain |
| `kube-prometheus-stack/middlewares.yaml` | `redirect-https` and `basic-auth` |
| `kube-prometheus-stack/kustomization.yaml` | Patches Grafana's `root_url` and Prometheus' `externalUrl` to the hostnames above |
| `vault-secrets/externalsecret.yaml` | `grafana-admin-credentials`, and `basic-auth` for Prometheus |

Grafana used to hold its own address, `.24` out of `reserved-pool`. It is a
ClusterIP now, and `.24` is free.

## Manual steps

Do them in this order. Unlike Longhorn, the `monitoring` Kustomization has
`wait: true`, so pushing before the Vault value exists leaves it reporting
unhealthy until the ExternalSecret syncs.

### 1. Vault

Connect and log in exactly as in the Longhorn runbook
([Connecting](longhorn-ui.md#connecting), [Logging in](longhorn-ui.md#logging-in)).
The CA is the same one; either `ca-secret.yaml` works.

The credential goes into the **existing** `secret/k3s-stage/monitoring`,
beside the Grafana admin credentials. Use `patch`, not `put`:

```bash
{ printf 'admin:'; openssl passwd -apr1; } \
  | vault kv patch secret/k3s-stage/monitoring basic_auth_users=-
```

`vault kv put` replaces the whole secret and would delete
`grafana_admin_user` and `grafana_admin_password`. `patch` adds the one key and
keeps the rest. The same `-apr1` rule applies as for Longhorn — Traefik cannot
parse `openssl passwd -5`/`-6`, and the symptom is a 401 with correct
credentials.

Confirm all three keys are there:

```bash
vault kv get secret/k3s-stage/monitoring
```

No new role or policy is needed: the key lives in the path the
`k3s-stage-monitoring` role can already read. Confirm that is true rather than
assuming it:

```bash
vault policy read k3s-stage-monitoring   # expect secret/data/k3s-stage/monitoring
```

### 2. Push and check before switching DNS

Once Flux has reconciled, Traefik serves both hosts on `.26`. Test them there
before touching DNS, so `grafana.jnet.lan` is not broken in the meantime:

```bash
flux -n flux-system reconcile kustomization monitoring --with-source
kubectl -n monitoring get externalsecret,secret basic-auth
kubectl -n monitoring get svc kube-prometheus-stack-grafana   # expect ClusterIP

curl -kI --resolve grafana.jnet.lan:443:192.168.0.26 \
  https://grafana.jnet.lan                                    # expect 302 to /login
curl -kI --resolve prometheus.jnet.lan:443:192.168.0.26 \
  https://prometheus.jnet.lan                                 # expect 401
curl -kI --resolve prometheus.jnet.lan:443:192.168.0.26 \
  -u admin:'your-password' https://prometheus.jnet.lan        # expect 302 to /query
```

From the moment Grafana's Service becomes a ClusterIP, `.24` stops answering,
so there is a short outage on `grafana.jnet.lan` between this step and the
next.

### 3. DNS

- `grafana.jnet.lan` → `192.168.0.26` (was `.24`)
- `prometheus.jnet.lan` → `192.168.0.26` (new)

Both point at Traefik, as `longhorn.jnet.lan` does.

## Troubleshooting

The failure modes are the same as Longhorn's; see
[A 404 means the router was never built](longhorn-ui.md#a-404-means-the-router-was-never-built).
In short: a 404 on `prometheus.jnet.lan` means the `basic-auth` Secret does
not exist yet, so Traefik has dropped the router instead of serving it
without authentication.

If Grafana logs you in and then redirects to `.24` or to plain HTTP, the
`root_url` patch has not reached the pod. Check the rendered value:

```bash
kubectl -n monitoring get cm kube-prometheus-stack-grafana \
  -o jsonpath='{.data.grafana\.ini}' | grep root_url
```

## Rotating the Prometheus password

Same pipeline, same `patch`. Include every user you want to keep, since the
`basic_auth_users` value is replaced as a whole:

```bash
{ printf 'admin:'; openssl passwd -apr1; } \
  | vault kv patch secret/k3s-stage/monitoring basic_auth_users=-

kubectl -n monitoring annotate externalsecret basic-auth \
  force-sync="$(date +%s)" --overwrite
```

Traefik picks up the new Secret without a restart.

## Notes

- Alertmanager (`kube-prometheus-stack-alertmanager:9093`) has no login either
  and is not exposed. Exposing it is one more Ingress reusing both
  middlewares, plus `alertmanager.alertmanagerSpec.externalUrl` in the patch.
