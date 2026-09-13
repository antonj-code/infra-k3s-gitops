# Trivy Security Scanning

Trivy Operator continuously monitors the cluster for vulnerabilities, exposed secrets, RBAC misconfigurations, and workload compliance (CIS Benchmarks).

## Security Exclusions

Not all workloads can (or should) adhere to strict Kubernetes security contexts. Low-level infrastructure operators (like CNI plugins, GitOps controllers, and storage drivers) explicitly require elevated host access, `root` privileges, and wildcard RBAC roles to do their jobs.

To prevent alert fatigue and keep the compliance dashboard actionable, trusted third-party infrastructure is globally excluded from Trivy's config audits. 

### Adding a new trusted namespace

If you install a new infrastructure component that throws expected security alerts (e.g., a new CSI driver), you should exclude its namespace rather than attempting to rewrite its Helm chart `securityContext`.

1. Open `infrastructure/base/trivy-operator/release.yaml`.
2. Append the new namespace to the comma-separated `excludeNamespaces` list under the `values` block.
3. Commit and push.

### RBAC Assessments (ClusterRoles)

Because `ClusterRoles` are non-namespaced, they cannot be filtered out via `excludeNamespaces`. 

By design, this repository does **not** filter out core Kubernetes roles (like `system:node`) or trusted operator roles (like `cluster-admin` or `flux-system`) from the Grafana dashboard or Trivy reports. These roles are mathematically true risks (e.g., they literally have `*` wildcard access), and are left visible as a documented, accepted baseline risk. 

If the baseline number of RBAC alerts climbs unexpectedly, it means a new role has been provisioned with overly broad permissions and should be audited.

## Hardening Workloads

For standard web services and applications you deploy yourself, they should be strictly hardened to pass Trivy's CIS benchmarks.

At a minimum, ensure the deployment's Pod template includes a strict `securityContext`:

```yaml
    spec:
      securityContext:
        runAsUser: 10000
        runAsGroup: 10000
        fsGroup: 10000
        runAsNonRoot: true
      containers:
      - name: my-app
        securityContext:
          runAsNonRoot: true
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          privileged: false
          capabilities:
            drop:
              - ALL
```
