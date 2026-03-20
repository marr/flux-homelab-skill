# Cluster hygiene (homelab)

## Legacy `Endpoints` + `EndpointSlice` for the same Service

If you ever `kubectl apply` a **v1/Endpoints** object while Git/Flux also applies an **EndpointSlice** for the same Service, kube-proxy can treat **both** as backends. If the IPs differ (e.g. after a NAS or host move), you get **~50% 502** on that route.

**Fix:** `kubectl delete endpoints <name> -n <ns>` and rely on **EndpointSlice-only** in Git. A manual Endpoints object also spawns a **mirrored** EndpointSlice (`endpointslice.kubernetes.io/managed-by: endpointslicemirroring-controller`); deleting the Endpoints removes that duplicate.

**Checked services:** `traefik/synology-dsm`, `traefik/graphstrike-backend`.

## Orphan Services (not in Git)

Review occasionally:

```bash
kubectl get svc -A -o json | jq -r '.items[] | select(.metadata.ownerReferences==null) | "\(.metadata.namespace)/\(.metadata.name)"'
```

Services with **no endpoints** and **wrong selectors** (e.g. `app: traefik` vs Helm’s `app.kubernetes.io/name: traefik`) are often leftover experiments—safe to delete if nothing references them.

**Removed 2026-03:** `traefik/dns-service` (UDP/53, selector `app: traefik` never matched Traefik pods; not in Git).

## Flux “dependency revision is not up to date”

`flux get kustomizations` may show `Ready=False` briefly while dependencies reconcile. If it **stays** stuck, reconcile parents first: `flux reconcile kustomization flux-system --with-source`.

## Hardcoded LAN IPs in Git

Plex (`homepage` config), Synology `EndpointSlice`s, Grafana dashboard PromQL, and `prometheus-stack` scrape configs use **192.168.1.x**. After infra moves, grep `192.168.1.` in `fleet-infra` and update.

## Noisy vs actionable logs (quick scan)

| Source | Pattern | Notes |
|--------|---------|--------|
| Traefik | `ForwardAuth 'maxResponseBodySize' is not configured` | Set `maxResponseBodySize` on the `voidauth` ForwardAuth middleware (bytes). |
| Traefik | `forward-auth` + `context canceled` | Usually client closed the tab; noisy, not always VoidAuth bugs. |
| Prometheus | `NFS_SUPER_MAGIC` | Prometheus on NFS PVC — [supported-with-caveats](https://prometheus.io/docs/prometheus/latest/storage/); consider local disk for prod. |
| Prometheus | `KubeQuota*` rule “many-to-many matching” | Often **two** `kube-state-metrics` replicas producing duplicate series; scale KSM to 1 or fix the rule expr. |
| Grafana | Loki `401` / `no org id` | Multi-tenant Loki expects `X-Scope-OrgID`; fix datasource or use single-tenant mode. |
| Alertmanager | `repeat_interval` \> retention | Tune `repeat_interval` or increase Alertmanager storage retention. |
| metrics-server | kubelet scrape timeout | Network/firewall/kubelet on that node; check node readiness. |
| controllers | `http2: client connection lost` (same timestamp) | Often API server restart or transient disconnect; OK if not continuous. |

```bash
# spot-check (adjust namespaces)
for ns in traefik monitoring kube-system; do
  kubectl logs -n "$ns" -l app.kubernetes.io/name=traefik --tail=200 2>/dev/null | grep -iE 'WRN|ERR' | tail -5
done
```
