# Vault integration (stage)

Every Kubernetes Secret this repo pulls from Vault — the Longhorn and
Prometheus basic-auth users, the Grafana admin login — goes through the same
chain:

```
ExternalSecret ─▶ SecretStore/vault-backend (one per namespace)
                    │  logs in as that namespace's ServiceAccount
                    ▼
                  Vault auth/kubernetes/role/k3s-stage-<app>
                    │  Vault checks the ServiceAccount token with a TokenReview
                    │  against https://192.168.0.43:6443, using the
                    │  vault-auth-delegator token
                    ▼
                  policy ─▶ read secret/data/k3s-stage/<app>
```

It is set up in two layers, and the Vault side of both is owned by the sibling
repo `infra-k3s-bootstrap`, not this one. This repo holds the cluster side:
the delegator RBAC, and a `vault-secrets/` directory per component.

## Layer 1: registering the cluster with Vault

`infra-k3s-bootstrap`'s `ansible/roles/k3s_vault_kubernetes_auth` does this.
It is the last play in `ansible/playbooks/site.yaml`, which CI runs in
`stage:ansible:configure` — before `stage:flux:bootstrap`, so the auth method
already works when Flux starts reconciling anything that needs it. The role:

1. applies a `ServiceAccount`, a token `Secret` and a `ClusterRoleBinding` to
   `system:auth-delegator`, all named `vault-auth-delegator` in `kube-system`;
2. enables Vault's `kubernetes` auth method, if it is not already;
3. writes `auth/kubernetes/config`: `kubernetes_host` (the kube-vip address,
   `https://192.168.0.43:6443`), the cluster CA, and the delegator's token as
   `token_reviewer_jwt`.

The same three objects are declared here in
`infrastructure/base/vault-auth-delegator/`. Flux adopts the ones Ansible
created and keeps them in place from then on.

Vault runs outside the cluster, so it has no service account of its own to call
the TokenReview API with. It borrows the delegator's, and `system:auth-delegator`
is the permission to do exactly that and nothing more.

### Rebuilds re-register on their own

A rebuilt cluster has a new CA and a new delegator token, so Vault's config
goes stale with every rebuild. The role rewrites it on every `site.yaml` run
for that reason, and nothing needs doing by hand — **as long as `VAULT_TOKEN`
is set**. Without it the role skips with a warning instead of failing, the
pipeline still goes green, and every SecretStore in the new cluster is left
unable to log in.

That token has to be admin-level: enabling an auth method and writing its
config are well outside what the `k3s-bootstrap` policy in
`infra-k3s-bootstrap`'s `docs/vault-integration.md` grants.

### Do not recreate the delegator token

`token_reviewer_jwt` is a copy of the token in
`kube-system/vault-auth-delegator`. If that Secret is ever deleted and
recreated — deleted by hand, or removed from this repo and pruned by Flux — the
token changes, Vault's copy stops working, and **every** Vault login in the
cluster fails at once.

Re-running the whole playbook (`make configure ENV=stage` in
`infra-k3s-bootstrap`) fixes it. The quicker fix is to rewrite just the config,
from the same sources the role uses, with kubectl pointed at stage:

```bash
vault write auth/kubernetes/config \
  kubernetes_host=https://192.168.0.43:6443 \
  kubernetes_ca_cert=@<(kubectl config view --raw --minify --flatten \
    -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' | base64 -d) \
  token_reviewer_jwt=@<(kubectl -n kube-system get secret vault-auth-delegator \
    -o jsonpath='{.data.token}' | base64 -d)
```

`@<(...)` hands Vault a file descriptor to read rather than the value itself,
so the token never appears in shell history or `ps`.

### Checking it

```bash
vault read auth/kubernetes/config   # kubernetes_host, token_reviewer_jwt_set: true
kubectl get secretstores -A         # every vault-backend Valid and Ready
```

## Layer 2: one role and policy per app

`infra-k3s-bootstrap`'s `scripts/vault_seed_apps.sh` does this:

```bash
bash scripts/vault_seed_apps.sh <env> <app> <namespace> <service_account> \
  [--secret KEY=VALUE ...] [--ttl 1h] [--force]
```

For `<app>` it writes:

- policy `k3s-<env>-<app>-read` — `read` on `secret/data/k3s-<env>/<app>` and
  its metadata, nothing else;
- role `k3s-<env>-<app>` — binds that policy to exactly one ServiceAccount in
  one namespace, with a 1h token TTL;
- any `--secret` fields, merged into what is already at that path. `--force`
  replaces the whole secret instead.

The other half lives here, in the component's `vault-secrets/` directory: a
ServiceAccount with that name, a `vault-backend` SecretStore naming the role,
the Vault CA, and the ExternalSecrets. The names on both sides must match
exactly.

### What exists on stage

| App | Vault path | Role | Policy | Bound ServiceAccount | Keys |
| --- | --- | --- | --- | --- | --- |
| monitoring | `secret/k3s-stage/monitoring` | `k3s-stage-monitoring` | `k3s-stage-monitoring-read` | `monitoring/grafana-vault-auth` | `grafana_admin_user`, `grafana_admin_password`, `basic_auth_users` |
| longhorn | `secret/k3s-stage/longhorn` | `k3s-stage-longhorn` | `k3s-stage-longhorn` | `longhorn-system/longhorn-vault-auth` | `basic_auth_users` |

Monitoring was set up with the script. Longhorn was set up by hand from
[its runbook](longhorn-ui.md#granting-the-cluster-access), which is why its
policy has no `-read` suffix; both work the same way. To find a role's policy
without guessing at its name:

```bash
vault read -field=token_policies auth/kubernetes/role/k3s-stage-<app>
```

### Adding a new app

1. In `infra-k3s-bootstrap`, create the role and policy:

   ```bash
   bash scripts/vault_seed_apps.sh stage <app> <namespace> <service_account>
   ```

   Leave off `--secret` for passwords and hashes. It puts the value on the
   script's command line, and the script hands it on to `curl` the same way,
   so it is visible in `ps` while it runs. Write those from stdin instead —
   `vault kv put` for the first key at a new path, `vault kv patch` after
   that, since `put` replaces every key already there:

   ```bash
   <command that prints the value> | vault kv patch secret/k3s-stage/<app> <key>=-
   ```

2. In this repo, copy an existing `vault-secrets/` directory — monitoring's or
   Longhorn's — and change the ServiceAccount name, the `role` in
   `secretstore.yaml`, and the ExternalSecrets. `ca-secret.yaml` is the same in
   every namespace.
3. Give the component's Flux Kustomization `dependsOn` both `external-secrets`
   and `vault-auth-delegator`, as `longhorn` and `monitoring` have.

## The Vault CA

Each `vault-secrets/ca-secret.yaml` carries the JNET root CA certificate, so
the SecretStores verify `https://vault.jnet.lan:8200` rather than skipping TLS
checks the way the bootstrap tooling does. It is a public certificate, not a
secret, which is why it is committed.

## Troubleshooting

`kubectl -n <ns> describe secretstore vault-backend` and
`kubectl -n <ns> describe externalsecret <name>` carry Vault's error. Where it
fails narrows down why:

- **SecretStore not Valid, in every namespace at once** — layer 1 is stale: a
  rebuild ran without `VAULT_TOKEN`, or the delegator token changed. See
  above.
- **SecretStore not Valid, in one namespace** — the role's bound
  ServiceAccount or namespace does not match the SecretStore. Compare
  `vault read auth/kubernetes/role/k3s-stage-<app>` with `vault-secrets/`.
- **SecretStore Valid, ExternalSecret not synced** — login works, the read
  does not. Either the policy does not cover the path (read it with the lookup
  above) or the key does not exist (`vault kv get secret/k3s-stage/<app>`).

The `longhorn` Kustomization has no `wait: true`, so a failure there does not
show in `flux get kustomizations`; `monitoring` does wait, and will.
