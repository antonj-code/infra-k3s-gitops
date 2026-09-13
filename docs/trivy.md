# Trivy Security Scanning (stage)

Trivy Operator monitors the cluster for vulnerabilities, exposed secrets, and bad configurations. It runs as a `Deployment` and generates custom resources (like `ConfigAuditReport`), which Prometheus immediately scrapes.

```
Trivy pod ─▶ scans workloads ─▶ creates ConfigAuditReport (CRD)
                                  │  Prometheus scrapes Trivy's /metrics port
                                  ▼
                                Prometheus (monitoring)
                                  │  Grafana queries sum(trivy_resource_configaudits)
                                  ▼
                                Grafana Compliance Dashboard
```

## Security Exclusions

Not everything should run with a locked-down security context. Infrastructure tools — like storage drivers, load balancers, and GitOps controllers — actually need `root` access and wildcard RBAC roles to function. 

Trying to force third-party Helm charts to run as unprivileged users usually just breaks them. Instead, we explicitly exclude trusted infrastructure so Trivy only alerts on workloads we write ourselves.

**To ignore a new infrastructure tool:**
Don't hack up its Helm chart. Just append its namespace to `excludeNamespaces` under the `values` block in `infrastructure/base/trivy-operator/release.yaml`.

## RBAC noise

`ClusterRoles` are global, meaning they don't belong to a namespace and cannot be filtered by `excludeNamespaces`.

The dashboard will always show dozens of Critical/High RBAC alerts. This is intentional. Core Kubernetes roles (like `system:node` or `cluster-admin`) and trusted operators (like `flux-system`) legitimately require wildcard access. Trivy flags them because they are inherently risky, but the cluster cannot function without them.

We leave them unfiltered on the dashboard as an accepted baseline. If the number jumps higher than the baseline, go check if a new custom role got too much power.

## Hardening your own apps

For the actual apps you deploy (e.g. into the `default` namespace), lock them down so they pass the CIS benchmarks. At a minimum, drop all capabilities and run as a non-root user:

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
