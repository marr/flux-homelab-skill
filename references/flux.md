# Flux Reference

## Reconcile
Prefer reconciling the **GitRepository** first, then kustomizations (some Flux CLI versions omit `--with-source`):

```bash
flux reconcile source git flux-system -n flux-system
flux reconcile kustomization flux-system -n flux-system
flux reconcile kustomization infrastructure-controllers -n flux-system
flux reconcile kustomization infrastructure-configs -n flux-system
flux reconcile kustomization apps -n flux-system
```

## Stuck kustomizations and `wait: true`

If a **`Kustomization`** stays **not Ready** with **health check timeout** on a **`HelmRelease`**:

- **`kubectl describe kustomization <name> -n flux-system`** — look for `HealthCheckFailed` and which HelmRelease is blocking.
- While the blocker is unhealthy, **`status.lastAppliedRevision`** may **lag** **`lastAttemptedRevision`**: Git changes (including fixes to the same `HelmRelease`) might not apply until health recovers or you **patch the live `HelmRelease`** / **delete stuck workloads** to unblock.
- **Downstream** kustomizations with **`spec.dependsOn`** wait on upstream Ready — one bad Helm release can freeze **apps** and **configs** for the whole fleet.
- **High `helm-controller` CPU** often correlates with a **failing Helm release** in a retry loop.

## HelmRelease: `upgrade.force` and server-side apply

Helm may fail with: **`invalid operation: cannot use force conflicts and force replace together`**. On **`HelmRelease`**, set server-side apply so **`upgrade.force: true`** is usable (same idea as kube-prometheus-stack in many fleets):

```yaml
spec:
  install:
    serverSideApply: false   # boolean — not the string "disabled"
  upgrade:
    serverSideApply: disabled  # enum: enabled | disabled | auto
    force: true
```

Validate with **`kubectl apply --dry-run=server -f …`** before committing. The **install** and **upgrade** fields use **different schemas**; copying `serverSideApply: disabled` onto **both** can fail validation.

## Helm chart upgrades that change Service/Deployment shape

Charts that **remove sidecars** or **change port lists** (e.g. OpenTelemetry operator **0.109 → 0.110**, kube-rbac-proxy removal) can leave Kubernetes rejecting merged objects (**duplicate port `name`**). Mitigations:

- **`upgrade.force: true`** plus correct **`serverSideApply`** settings (above).
- If still wedged: **`kubectl delete deployment,svc`** for the release’s manager workload (not CRDs), then **`flux reconcile helmrelease … --force`** so Helm recreates clean resources.

## Grafana / Loki panels on this stack

Log panels that use **LogQL** should not include a no-op **`|= \`\``** (empty line filter) — it can yield **no data** even when logs exist. Prefer `{<label selectors>} | json | …` without stray empty filters.

## Inspect status
```bash
kubectl get kustomizations -A
kubectl get helmrelease -A
kubectl describe kustomization <name> -n flux-system
kubectl describe helmrelease <name> -n <namespace>
```

## Recommended repo shape
```text
clusters/                  # Flux bootstrap
infrastructure/
  controllers/             # HelmRepository + namespaces + controller HelmReleases
  configs/                 # IngressRoutes, monitoring config, shared config
apps/
  <app>/                   # namespace.yaml, kustomization.yaml, helmrelease.yaml, secrets.sops.yaml
```

## Change workflow
1. Edit repo files.
2. Commit and push.
3. Reconcile the narrowest useful kustomization.
4. Verify HelmRelease or workload health.

## Add a controller
1. Add a HelmRepository if needed.
2. Add the namespace.
3. Add a HelmRelease file.
4. Add it to the controllers kustomization.

## Add an app
1. Create app directory.
2. Add namespace, kustomization, HelmRelease.
3. Add SOPS secret if needed.
4. Add the app directory to the apps kustomization.

## OCI chart gotchas (Flux 2.8.x)
- Use **`source.toolkit.fluxcd.io/v1` for source objects** such as `GitRepository`, `HelmRepository`, and `OCIRepository`.
- Use **`helm.toolkit.fluxcd.io/v2` for `HelmRelease`**.
- Do not read this as “use v1 everywhere” — the split is **sources = v1**, **HelmRelease = v2**.
- OCI-backed Helm charts can be wired in two different ways, and both may be valid depending on chart/repo behavior:
  1. `HelmRepository` with `spec.type: oci`, then `HelmRelease.spec.chart.spec.sourceRef.kind: HelmRepository`
  2. `OCIRepository`, then `HelmRelease.spec.chartRef.kind: OCIRepository`
- If one pattern fails unexpectedly, try the other instead of assuming the chart itself is broken.
- Keep source kind and HelmRelease reference aligned; mismatches are easy to introduce during refactors.
- When debugging OCI issues, inspect both the source object and the HelmRelease events.
