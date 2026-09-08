# Longhorn UI (stage)

Longhorn ships no authentication of its own — anyone who can reach the UI can
detach volumes — so it is not exposed directly. It sits behind the Traefik
that k3s installs, with a basic-auth middleware in front and an HTTPS redirect
in front of that, so credentials are never sent in clear text.

```
longhorn.jnet.lan ─▶ 192.168.0.26 (traefik, kube-system)
                       │  redirect to https
                       │  basicAuth ◀── Secret/longhorn-basic-auth ◀── Vault
                       ▼
                     Service/longhorn-frontend (ClusterIP, longhorn-system)
```

## What Flux manages

Everything in `infrastructure/stage/longhorn/`, reconciled by
`clusters/stage/longhorn.yaml`:

| File | Purpose |
| --- | --- |
| `ingress.yaml` | `longhorn.jnet.lan` → `longhorn-frontend:80`, chaining both middlewares |
| `middlewares.yaml` | `longhorn-redirect-https` and `longhorn-basic-auth` |
| `vault-secrets/` | SecretStore, ServiceAccount, CA and the ExternalSecret that materialises `longhorn-basic-auth` |

The Traefik address itself is pinned by `infrastructure/stage/traefik/`, which
claims `.26` out of `reserved-pool`.

## Manual steps

Three things live outside the repo and must exist before any of the above
works. Do them in this order.

### 1. Vault

#### Connecting

```bash
export VAULT_ADDR='https://vault.jnet.lan:8200'
```

Vault is fronted by the internal JNET root CA, which your shell will not trust
by default. The CA is already committed here (external-secrets needs it too),
so point Vault at it rather than disabling verification:

```bash
sed -n '/BEGIN CERTIFICATE/,/END CERTIFICATE/p' \
  infrastructure/stage/longhorn/vault-secrets/ca-secret.yaml \
  | sed 's/^ *//' > ~/.vault-jnet-ca.crt
export VAULT_CACERT=~/.vault-jnet-ca.crt
```

`export VAULT_SKIP_VERIFY=true` also works and is quicker, but it disables
certificate checking on a session where you are about to type a root token —
prefer the CA. Confirm the endpoint is up and unsealed before logging in:

```bash
vault status
```

#### Logging in

```bash
vault login
Token (will be hidden):
```

The steps below write a policy and an auth role, which needs a token with
`sudo` on `sys/policies/acl/*` and `auth/kubernetes/role/*` — in practice the
root token or an admin token. A workload token will not do.

Verify what you got, and note the TTL — a short-lived token that expires
partway through leaves the policy created and the role missing:

```bash
vault token lookup
```

#### Creating the credential

Traefik's basic-auth middleware reads an htpasswd file out of the `users` key.
external-secrets copies values through verbatim and cannot hash anything, so
what Vault holds is the finished `user:hash` line, not a plaintext password.

```bash
htpasswd -nbB admin 'your-password'      # → admin:$2y$05$...
```

`htpasswd` comes from `httpd-tools` (`dnf install httpd-tools`). `-B` selects
bcrypt. Unlike the nginx-ingress equivalent, the `$` characters need no
escaping anywhere in this path.

```bash
vault kv put secret/k3s-stage/longhorn basic_auth_users='admin:$2y$05$...'
```

The KV mount is `secret`, version 2 — which is why the policy below grants
`secret/data/...` and not `secret/...`. Check it landed:

```bash
vault kv get secret/k3s-stage/longhorn
```

For more than one login, the value is newline-separated — pass it from a file
rather than trying to quote a literal newline:

```bash
htpasswd -nbB admin 'pw1'  > /tmp/users
htpasswd -nbB viewer 'pw2' >> /tmp/users
vault kv put secret/k3s-stage/longhorn basic_auth_users=@/tmp/users
shred -u /tmp/users
```

#### Granting the cluster access

```bash
vault policy write k3s-stage-longhorn - <<'POLICY'
path "secret/data/k3s-stage/longhorn" { capabilities = ["read"] }
POLICY

vault write auth/kubernetes/role/k3s-stage-longhorn \
  bound_service_account_names=longhorn-vault-auth \
  bound_service_account_namespaces=longhorn-system \
  policies=k3s-stage-longhorn ttl=1h
```

The role name, service account and namespace must match
`vault-secrets/secretstore.yaml` and `vault-secrets/serviceaccount.yaml`
exactly; a mismatch surfaces as a permission denied on the ExternalSecret and
nowhere else. Read them back to confirm:

```bash
vault read auth/kubernetes/role/k3s-stage-longhorn
vault policy read k3s-stage-longhorn
```

This assumes the Kubernetes auth method is already enabled at `kubernetes/`
and pointed at the stage cluster — it is, via `vault-auth-delegator` in
`infrastructure/base/`, which gives Vault the TokenReview permission it needs
to validate the service account. `vault auth list` should show it.

### 2. DNS

Point `longhorn.jnet.lan` at `192.168.0.26` — Traefik's address, not
Longhorn's. Nothing in `longhorn-system` holds an external address any more.

### 3. Push, then unblock MetalLB

This last step is a one-off correction, not part of normal operation. The
`metallb-pool` Kustomization has been failing since the stage/prod pool split:
MetalLB's webhook rejects `reserved-pool` for overlapping the live
`primary-pool`, and Flux dry-runs every object against live state before
applying any of them, so the commit that would shrink `primary-pool` never
gets far enough to run. Deleting the stale pool breaks the cycle.

```bash
# Protect Traefik's address across the gap, before reserved-pool exists.
kubectl -n kube-system annotate svc traefik \
  metallb.universe.tf/loadBalancerIPs=192.168.0.26 --overwrite

kubectl -n metallb-system delete ipaddresspool primary-pool
flux -n flux-system reconcile kustomization metallb-pool
```

Between those two commands, Grafana and Traefik hold addresses belonging to no
pool and MetalLB may reassign them, so keep them close together. Once
`metallb-pool` reports Ready the `traefik` Kustomization comes off its
`dependsOn` and applies the pin.

## Verifying

`flux get kustomization longhorn` is not a useful health check here — the
Kustomization has no `wait: true`, and a failing ExternalSecret still applies
cleanly, so Flux reports success while the UI returns 500. Check the secret
itself:

```bash
kubectl -n longhorn-system get externalsecret longhorn-basic-auth
kubectl -n longhorn-system get secret longhorn-basic-auth
kubectl -n metallb-system get ipaddresspools \
  -o custom-columns='NAME:.metadata.name,ADDR:.spec.addresses'
curl -kI https://longhorn.jnet.lan          # expect 401
curl -kI -u admin:'your-password' https://longhorn.jnet.lan   # expect 200
```

## Rotating the password

Write the new hash to the same Vault key. external-secrets picks it up within
`refreshInterval` (1h), or immediately with:

```bash
kubectl -n longhorn-system annotate externalsecret longhorn-basic-auth \
  force-sync="$(date +%s)" --overwrite
```

Traefik watches the Secret and reloads the middleware on its own; no restart.

## Notes

- Traefik serves its own self-signed certificate, so browsers will warn. The
  `tls:` block in `ingress.yaml` marks where a cert-manager-issued Secret goes
  once there is an issuer for `jnet.lan`.
- `middlewares.yaml` uses `traefik.io/v1alpha1`, correct for Traefik v2.10 and
  later. On an older bundled Traefik the group is `traefik.containo.us/v1alpha1`,
  and a wrong group fails silently — the middleware is simply never applied.
  Confirm with `kubectl get crd middlewares.traefik.io`.
- Longhorn's data path does not touch MetalLB or Traefik. Engine and replica
  traffic runs over the pod network and the `10.20.20.x` node addresses, so
  none of the above affects volumes or attachments.
