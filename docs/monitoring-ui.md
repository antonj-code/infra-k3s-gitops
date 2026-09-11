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
assuming it. The policy is not named after the role, as it is for Longhorn, so
go through the role to find it:

```bash
vault read -field=token_policies auth/kubernetes/role/k3s-stage-monitoring
# [k3s-stage-monitoring-read]
vault policy read k3s-stage-monitoring-read   # expect secret/data/k3s-stage/monitoring
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

Both become CNAMEs to `ingress-stage.jnet.lan`, Traefik's A record, as
`longhorn.jnet.lan` is:

- `grafana.jnet.lan` → CNAME `ingress-stage.jnet.lan` (was an A record to
  `.24` — delete it first, a CNAME cannot share a name with another record)
- `prometheus.jnet.lan` → CNAME `ingress-stage.jnet.lan` (new)

```bash
dig +short grafana.jnet.lan      # ingress-stage.jnet.lan. then 192.168.0.26
dig +short prometheus.jnet.lan
```

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

## Grafana admin credentials

The Grafana admin login lives beside the Prometheus password:
`grafana_admin_user` and `grafana_admin_password` in
`secret/k3s-stage/monitoring`, synced by the `grafana-admin-credentials`
ExternalSecret and handed to Grafana as `GF_SECURITY_ADMIN_USER` and
`GF_SECURITY_ADMIN_PASSWORD`.

### Retrieving the admin login

From the repo root, since the CA path is relative:

```bash
export VAULT_ADDR='https://vault.jnet.lan:8200'
sed -n '/BEGIN CERTIFICATE/,/END CERTIFICATE/p' \
  infrastructure/stage/longhorn/vault-secrets/ca-secret.yaml \
  | sed 's/^ *//' > ~/.vault-jnet-ca.crt
export VAULT_CACERT=~/.vault-jnet-ca.crt
vault status                      # expect Sealed: false

vault login                       # the token prompt is hidden

vault kv get -field=grafana_admin_user     secret/k3s-stage/monitoring
vault kv get -field=grafana_admin_password secret/k3s-stage/monitoring
```

The token needs `read` on `secret/data/k3s-stage/monitoring` — in practice the
root token or an admin token. The `k3s-stage-monitoring` role's tokens can read
it too, but those are issued to Grafana's ServiceAccount, not to people.

Without Vault — it is down, or no token is to hand — the same values are in
the cluster, in the Secret the ExternalSecret syncs them into:

```bash
kubectl -n monitoring get secret grafana-admin-credentials \
  -o jsonpath='{.data.admin-user}' | base64 -d; echo
kubectl -n monitoring get secret grafana-admin-credentials \
  -o jsonpath='{.data.admin-password}' | base64 -d; echo
```

If that password does not log you in, it was changed inside Grafana and not in
Vault; Grafana reads Vault's value only on first start. Step 3 of
[Changing the password](#changing-the-password--vault-alone-is-not-enough)
resets it without needing the old one, and steps 1 and 2 put Vault back in
step.

The Prometheus login cannot be retrieved at all. Vault holds only its `apr1`
hash, in `basic_auth_users`; the password itself was never stored. If it is
lost, set a new one with
[Rotating the Prometheus password](#rotating-the-prometheus-password).

### First-time setup

They were seeded — together with the `k3s-stage-monitoring` role and policy —
by `infra-k3s-bootstrap`'s `vault_seed_apps.sh` (see
[Vault integration](vault.md#layer-2-one-role-and-policy-per-app)):

```bash
bash scripts/vault_seed_apps.sh stage monitoring monitoring grafana-vault-auth \
  --secret grafana_admin_user=admin \
  --secret grafana_admin_password="$(openssl rand -base64 24)"
```

The password is random and never shown;
[Retrieving the admin login](#retrieving-the-admin-login) reads it back.

### Changing the password — Vault alone is not enough

Grafana applies `GF_SECURITY_ADMIN_PASSWORD` only when it creates the admin
user, on first start with an empty database. Its database lives on a
persistent Longhorn volume, so after that a new value in Vault reaches the
Secret and the pod's environment but never the login itself. Change it in
Vault, then set the same password inside Grafana:

```bash
# 1. A new random password into Vault, never on a command line
openssl rand -base64 24 | tr -d '\n' \
  | vault kv patch secret/k3s-stage/monitoring grafana_admin_password=-

# 2. Sync it into the cluster now rather than within the hour
kubectl -n monitoring annotate externalsecret grafana-admin-credentials \
  force-sync="$(date +%s)" --overwrite
kubectl -n monitoring get externalsecret grafana-admin-credentials   # wait for a fresh LAST SYNC

# 3. Set it in Grafana's database, piped straight from the Secret
kubectl -n monitoring get secret grafana-admin-credentials \
    -o jsonpath='{.data.admin-password}' | base64 -d \
  | kubectl -n monitoring exec -i deploy/kube-prometheus-stack-grafana -c grafana -- \
      grafana cli admin reset-admin-password --password-from-stdin
```

`tr -d '\n'` keeps `openssl rand`'s trailing newline out of the password. To
choose one yourself instead, swap the first pipeline for a silent prompt —
`read -rs p; printf '%s' "$p" | vault kv patch … grafana_admin_password=-;
unset p` — `printf` is a shell builtin, so the value still never reaches `ps`.

Step 3 is what actually changes the login; steps 1 and 2 keep Vault in step
with it, so that if the volume is ever lost, Grafana starts over with the
password Vault holds. `grafana_admin_user` has the same first-start-only
behaviour — to rename the admin after the fact, change it in Vault and rename
the user in Grafana's own user administration to match.

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
