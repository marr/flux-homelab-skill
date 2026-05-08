# Grafana Dashboards Reference

## Custom Dashboards Deployed

Four custom Grafana dashboards are deployed as ConfigMaps in the `monitoring` namespace:

| Dashboard | Purpose | Location |
|-----------|---------|----------|
| `homelab-overview-dashboard` | Cluster health gauges, CPU/memory, disk usage | `infrastructure/configs/grafana-dashboards.yaml` |
| `k8s-cluster-health-dashboard` | Kubernetes resource health, pod counts, namespace metrics | `infrastructure/configs/grafana-dashboards.yaml` |
| `immich-deep-dive-dashboard` | Photo library metrics: uploads, storage, transcoding | `infrastructure/configs/grafana-dashboards.yaml` |
| `traefik-site-health-dashboard` | Request rates, response times, error rates | `infrastructure/configs/traefik-dashboard.yaml` |

## Deployment Pattern

Dashboards are deployed as ConfigMaps with the `grafana_dashboard: "1"` label:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: <dashboard-name>
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  <dashboard-name>.json: |
    { ... dashboard JSON ... }
```

Grafana's dashboard provisioning automatically discovers and loads these ConfigMaps.

## Why Custom Dashboards?

Standard Kubernetes dashboards show raw metrics. Custom dashboards provide:

- **Service context** — See your services, not just cluster internals
- **Actionable metrics** — Focus on what matters for your workloads
- **Quick triage** — When Immich is slow, immediately see if it's storage, transcoding, or database queries

## Maintenance

- **Add a dashboard** — Add a new ConfigMap to `grafana-dashboards.yaml` or `traefik-dashboard.yaml`
- **Update a dashboard** — Edit the JSON in the ConfigMap, Flux reconciles automatically
- **Remove a dashboard** — Delete the ConfigMap section, Flux removes it from the cluster

## Related

- [Flux Web UI](flux.md#flux-web-ui) — Built-in Flux operator dashboard
- [Traefik Routes](traefik-routes.md) — How services are exposed
