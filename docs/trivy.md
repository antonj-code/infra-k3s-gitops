# Trivy Security Scanning

Trivy Operator watches the cluster for vulnerabilities, exposed secrets, and bad configurations. The results feed into the Grafana compliance dashboard.

## Exclusions

Not everything should run with a locked-down security context. Infrastructure tools like storage drivers and GitOps controllers actually need `root` access and wildcard RBAC roles to function. 

Trying to force third-party Helm charts to run as unprivileged users usually just breaks them. Instead, we exclude trusted infrastructure so Trivy only yells at us about our own workloads.

**To ignore a new infrastructure tool:**
Don't hack up its Helm chart. Just append its namespace to `excludeNamespaces` under `values` in `infrastructure/base/trivy-operator/release.yaml`.

## RBAC noise

`ClusterRoles` are global, so excluding namespaces doesn't hide their RBAC alerts.

You will see dozens of Critical/High RBAC alerts in the dashboard. This is intentional. Core Kubernetes roles (like `system:node` or `cluster-admin`) and operators (like `flux-system`) literally have wildcard access. Trivy flags them because they are inherently risky, but we need them.

We leave them unfiltered on the dashboard as an accepted baseline. If the number jumps higher than the baseline, go check if a new custom role got too much power.

## Hardening your own apps

For the actual apps you deploy, lock them down so they pass the CIS benchmarks. 

At a minimum, the deployment needs a `securityContext` that drops all capabilities and runs as a non-root user:

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
