---
name: flux-homelab
description: Manage a private homelab or small-cluster Kubernetes environment that uses Flux GitOps, SOPS-encrypted secrets, and a homepage-style dashboard. Use when investigating service errors, reconciling Flux, adding apps or controllers, updating encrypted secrets, configuring homepage widgets, or debugging why config changes did not reach running pods.
---

# Flux Homelab Skill

Use this skill for repeatable GitOps homelab operations.

## Default workflow

### Investigate an issue
1. Check pod logs.
2. Check HelmRelease and kustomization health.
3. Confirm the service is reachable on its internal URL.
4. If dashboard widgets are involved, verify both the ConfigMap and the mounted file inside the pod.

### Make a change
1. Edit the GitOps repo.
2. Commit and push.
3. Reconcile the narrowest relevant Flux kustomization.
4. Confirm the change landed in the cluster.
5. Confirm the workload actually consumed the change.

### Update a secret
See the **SOPS Secrets** section below.

### Update homepage widgets
See the **Homepage** section below.

---

## Flux commands and repo structure

### Reconcile
Prefer reconciling the **GitRepository** first, then kustomizations (some Flux CLI versions omit `--with-source`):

```bash
flux reconcile source git flux-system -n flux-system
flux reconcile kustomization flux-system -n flux-system
flux reconcile kustomization infrastructure-controllers -n flux-system
flux reconcile kustomization infrastructure-configs -n flux-system
flux reconcile kustomization apps -n flux-system
```

### Stuck kustomizations and `wait: true`

If a **`Kustomization`** stays **not Ready** with **health check timeout** on a **`HelmRelease`**:

- **`kubectl describe kustomization <name> -n flux-system`** — look for `HealthCheckFailed` and which HelmRelease is blocking.
- While the blocker is unhealthy, **`status.lastAppliedRevision`** may **lag** **`lastAttemptedRevision`**: Git changes (including fixes to the same `HelmRelease`) might not apply until health recovers or you **patch the live `HelmRelease`** / **delete stuck workloads** to unblock.
- **Downstream** kustomizations with **`spec.dependsOn`** wait on upstream Ready — one bad Helm release can freeze **apps** and **configs** for the whole fleet.
- **High `helm-controller` CPU** often correlates with a **failing Helm release** in a retry loop.

### HelmRelease: `upgrade.force` and server-side apply

Helm may fail with: **`invalid operation: cannot use force conflicts and force replace together`**. On **`HelmRelease`**, set server-side apply so **`upgrade.force: true`** is usable:

```yaml
spec:
  install:
    serverSideApply: false   # boolean — not the string "disabled"
  upgrade:
    serverSideApply: disabled  # enum: enabled | disabled | auto
    force: true
```

Validate with **`kubectl apply --dry-run=server -f …`** before committing. The **install** and **upgrade** fields use **different schemas**; copying `serverSideApply: disabled` onto **both** can fail validation.

### Helm chart upgrades that change Service/Deployment shape

Charts that **remove sidecars** or **change port lists** can leave Kubernetes rejecting merged objects (**duplicate port `name`**). Mitigations:

- **`upgrade.force: true`** plus correct **`serverSideApply`** settings (above).
- If still wedged: **`kubectl delete deployment,svc`** for the release's manager workload (not CRDs), then **`flux reconcile helmrelease … --force`** so Helm recreates clean resources.

### Grafana / Loki panels on this stack

Log panels that use **LogQL** should not include a no-op **`|= \`\``** (empty line filter) — it can yield **no data** even when logs exist. Prefer `{<label selectors>} | json | …` without stray empty filters.

### Inspect status
```bash
kubectl get kustomizations -A
kubectl get helmrelease -A
kubectl describe kustomization <name> -n flux-system
kubectl describe helmrelease <name> -n <namespace>
```

### Recommended repo shape
```text
clusters/                  # Flux bootstrap
infrastructure/
  controllers/             # HelmRepository + namespaces + controller HelmReleases
  configs/                 # IngressRoutes, monitoring config, shared config
apps/
  <app>/                   # namespace.yaml, kustomization.yaml, helmrelease.yaml, secrets.sops.yaml
```

### Change workflow
1. Edit repo files.
2. Commit and push.
3. Reconcile the narrowest useful kustomization.
4. Verify HelmRelease or workload health.

### Add a controller
1. Add a HelmRepository if needed.
2. Add the namespace.
3. Add a HelmRelease file.
4. Add it to the controllers kustomization.

### Add an app
1. Create app directory.
2. Add namespace, kustomization, HelmRelease.
3. Add SOPS secret if needed.
4. Add the app directory to the apps kustomization.

### OCI chart gotchas (Flux 2.8.x)
- Use **`source.toolkit.fluxcd.io/v1` for source objects** such as `GitRepository`, `HelmRepository`, and `OCIRepository`.
- Use **`helm.toolkit.fluxcd.io/v2` for `HelmRelease`**.
- Do not read this as "use v1 everywhere" — the split is **sources = v1**, **HelmRelease = v2**.
- OCI-backed Helm charts can be wired in two different ways, and both may be valid depending on chart/repo behavior:
  1. `HelmRepository` with `spec.type: oci`, then `HelmRelease.spec.chart.spec.sourceRef.kind: HelmRepository`
  2. `OCIRepository`, then `HelmRelease.spec.chartRef.kind: OCIRepository`
- If one pattern fails unexpectedly, try the other instead of assuming the chart itself is broken.
- Keep source kind and HelmRelease reference aligned; mismatches are easy to introduce during refactors.
- When debugging OCI issues, inspect both the source object and the HelmRelease events.

---

## SOPS Secrets

### Common pattern
Use age-backed SOPS files for app and infrastructure secrets.

#### Decrypt
```bash
SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt sops -d path/to/file.sops.yaml
```

#### Add or update a value
```bash
SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt \
  sops --set '["stringData"]["KEY_NAME"] "value"' path/to/file.sops.yaml
```

#### Remove a value
SOPS has no delete command. Decrypt, edit plaintext, re-encrypt.

```bash
SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt sops -d file.sops.yaml > /tmp/plain.sops.yaml
# edit /tmp/plain.sops.yaml
cp /tmp/plain.sops.yaml /tmp/plain.sops.yaml.sops.yaml
SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt \
  sops --config .sops.yaml -e /tmp/plain.sops.yaml.sops.yaml > file.sops.yaml
rm /tmp/plain.sops.yaml /tmp/plain.sops.yaml.sops.yaml
```

### Practical advice
- Keep secret keys aligned with how the app consumes them.
- Prefer app-specific secret names over giant shared secrets.
- After changing a secret, verify the controller/app actually picked it up.
- If a workload consumes secrets via env vars, a restart may be required unless a reloader is installed.

---

## Homepage

### Config organization
Keep homepage config in a ConfigMap with embedded files such as:
- `settings.yaml`
- `services.yaml`
- `widgets.yaml`
- `bookmarks.yaml`

### Important rules

#### Template variables
If homepage config uses `{{HOMEPAGE_VAR_FOO}}`, the Kubernetes Secret key must also be `HOMEPAGE_VAR_FOO`.

#### ConfigMap updates and subPath
If files are mounted with `subPath`, ConfigMap changes do not hot-reload into the running container. Use a restart trigger such as Stakater Reloader.

#### Widget auth
Prefer internal service URLs for widgets. Store tokens/passwords in a Secret, not inline in the ConfigMap.

### Useful widget patterns

#### Immich
```yaml
widget:
  type: immich
  url: http://immich-server.<namespace>.svc.cluster.local:2283
  key: "{{HOMEPAGE_VAR_IMMICH_KEY}}"
  version: 2
```
Use `version: 2` for modern Immich releases; old `/api/server-info/*` paths can 404.

#### Grafana
```yaml
widget:
  type: grafana
  url: http://grafana.<namespace>.svc.cluster.local
  username: admin
  password: "{{HOMEPAGE_VAR_GRAFANA_PASS}}"
```
For some homepage Grafana stats, basic auth may work where service account tokens do not.

#### Plex
```yaml
widget:
  type: plex
  url: http://plex-host:32400
  key: "{{HOMEPAGE_VAR_PLEX_TOKEN}}"
```

#### Traefik
```yaml
widget:
  type: traefik
  url: https://traefik.example.com
```
If the HelmRelease sets `api.insecure: false` (recommended), the raw Service port `:8080` does **not** serve `/api/*` (you get 404). Point the widget at the same HTTPS host you use for the dashboard (an IngressRoute to `api@internal`), or set `api.insecure: true` only if you accept exposing the API on that port.

The older pattern `http://traefik.<ns>.svc.cluster.local:8080` only works when the insecure API is enabled on that listener.

#### CrowdSec
```yaml
widget:
  type: crowdsec
  url: http://crowdsec-service.<namespace>.svc.cluster.local:8080
  username: <machine-name>
  password: "{{HOMEPAGE_VAR_CROWDSEC_PASS}}"
```
The widget uses **LAPI machine** login (not bouncer keys). The machine **must exist** in CrowdSec with the same password as the secret, or the homepage logs show `401` / `Failed to login to Crowdsec API`.

Register once (from the LAPI pod; use `-f` so `cscli` does not overwrite the pod's own `local_api_credentials.yaml`):

```bash
PASS=$(kubectl get secret homepage-secrets -n homepage -o jsonpath='{.data.HOMEPAGE_VAR_CROWDSEC_PASS}' | base64 -d)
kubectl exec -n crowdsec deploy/crowdsec-lapi -- env "PASS=$PASS" sh -c \
  'cscli machines add <machine-name> -p "$PASS" -f /tmp/homepage-creds.yaml'
```

After rotating `HOMEPAGE_VAR_CROWDSEC_PASS`, either update the machine in CrowdSec (`cscli machines delete` / `add` with `--force`) or re-register with the new password.

### Theme advice
Avoid overly dark background overlays on top of a dark theme. They can look good during first paint and then collapse to nearly black after full render.

---

## Cluster hygiene

### Legacy `Endpoints` + `EndpointSlice` for the same Service

If you ever `kubectl apply` a **v1/Endpoints** object while Git/Flux also applies an **EndpointSlice** for the same Service, kube-proxy can treat **both** as backends. If the IPs differ, you get **~50% 502** on that route.

**Fix:** `kubectl delete endpoints <name> -n <ns>` and rely on **EndpointSlice-only** in Git. A manual Endpoints object also spawns a **mirrored** EndpointSlice (`endpointslice.kubernetes.io/managed-by: endpointslicemirroring-controller`); deleting the Endpoints removes that duplicate.

### Orphan Services (not in Git)

Review occasionally:

```bash
kubectl get svc -A -o json | jq -r '.items[] | select(.metadata.ownerReferences==null) | "\(.metadata.namespace)/\(.metadata.name)"'
```

Services with **no endpoints** and **wrong selectors** are often leftover experiments—safe to delete if nothing references them.

### Flux "dependency revision is not up to date"

`flux get kustomizations` may show `Ready=False` briefly while dependencies reconcile. If it **stays** stuck, reconcile parents first: `flux reconcile kustomization flux-system --with-source`.

### Hardcoded LAN IPs in Git

Plex, Synology `EndpointSlice`s, Grafana dashboard PromQL, and `prometheus-stack` scrape configs may use hardcoded LAN IPs. After infra moves, grep the IP range in the repo and update.

### Noisy vs actionable logs (quick scan)

| Source | Pattern | Notes |
|--------|---------|-------|
| Traefik | `ForwardAuth 'maxResponseBodySize' is not configured` | Set `maxResponseBodySize` on the `voidauth` ForwardAuth middleware (bytes). |
| Traefik | `forward-auth` + `context canceled` | Usually client closed the tab; noisy, not always VoidAuth bugs. |
| Prometheus | `NFS_SUPER_MAGIC` | Prometheus on NFS PVC — supported with caveats; consider local disk for prod. |
| Prometheus | `KubeQuota*` rule "many-to-many matching" | Often **two** `kube-state-metrics` replicas producing duplicate series; scale KSM to 1 or fix the rule expr. |
| Grafana | Loki `401` / `no org id` | Multi-tenant Loki expects `X-Scope-OrgID`; fix datasource or use single-tenant mode. |
| Alertmanager | `repeat_interval` > retention | Tune `repeat_interval` or increase Alertmanager storage retention. |
| metrics-server | kubelet scrape timeout | Network/firewall/kubelet on that node; check node readiness. |
| controllers | `http2: client connection lost` (same timestamp) | Often API server restart or transient disconnect; OK if not continuous. |

```bash
# spot-check (adjust namespaces)
for ns in traefik monitoring kube-system; do
  kubectl logs -n "$ns" -l app.kubernetes.io/name=traefik --tail=200 2>/dev/null | grep -iE 'WRN|ERR' | tail -5
done
```

---

## Key gotchas
- Do not mix **manual v1/Endpoints** with **Git-managed EndpointSlice** for the same Service (flaky 502s if IPs drift).
- `subPath` ConfigMap mounts do not hot-reload.
- Homepage variable substitution requires matching `HOMEPAGE_VAR_*` secret keys.
- Some widgets need internal API compatibility flags or version selectors.
- Some services accept one auth mode in direct tests but require another for homepage widgets.
- SOPS has no delete-key command; use decrypt-edit-reencrypt.
