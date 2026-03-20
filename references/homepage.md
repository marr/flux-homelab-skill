# Homepage Reference

## Config organization
Keep homepage config in a ConfigMap with embedded files such as:
- `settings.yaml`
- `services.yaml`
- `widgets.yaml`
- `bookmarks.yaml`

## Important rules

### Template variables
If homepage config uses `{{HOMEPAGE_VAR_FOO}}`, the Kubernetes Secret key must also be `HOMEPAGE_VAR_FOO`.

### ConfigMap updates and subPath
If files are mounted with `subPath`, ConfigMap changes do not hot-reload into the running container. Use a restart trigger such as Stakater Reloader.

### Widget auth
Prefer internal service URLs for widgets. Store tokens/passwords in a Secret, not inline in the ConfigMap.

## Useful widget patterns

### Immich
```yaml
widget:
  type: immich
  url: http://immich-server.<namespace>.svc.cluster.local:2283
  key: "{{HOMEPAGE_VAR_IMMICH_KEY}}"
  version: 2
```
Use `version: 2` for modern Immich releases; old `/api/server-info/*` paths can 404.

### Grafana
```yaml
widget:
  type: grafana
  url: http://grafana.<namespace>.svc.cluster.local
  username: admin
  password: "{{HOMEPAGE_VAR_GRAFANA_PASS}}"
```
For some homepage Grafana stats, basic auth may work where service account tokens do not.

### Plex
```yaml
widget:
  type: plex
  url: http://plex-host:32400
  key: "{{HOMEPAGE_VAR_PLEX_TOKEN}}"
```

### Traefik
```yaml
widget:
  type: traefik
  url: https://traefik.example.com
```
If the HelmRelease sets `api.insecure: false` (recommended), the raw Service port `:8080` does **not** serve `/api/*` (you get 404). Point the widget at the same HTTPS host you use for the dashboard (an IngressRoute to `api@internal`), or set `api.insecure: true` only if you accept exposing the API on that port (often also on the LoadBalancer).

The older pattern `http://traefik.<ns>.svc.cluster.local:8080` only works when the insecure API is enabled on that listener.

### CrowdSec
```yaml
widget:
  type: crowdsec
  url: http://crowdsec-service.<namespace>.svc.cluster.local:8080
  username: <machine-name>
  password: "{{HOMEPAGE_VAR_CROWDSEC_PASS}}"
```
The widget uses **LAPI machine** login (not bouncer keys). The machine **must exist** in CrowdSec with the same password as the secret, or the homepage logs show `401` / `Failed to login to Crowdsec API`.

Register once (from the LAPI pod; use `-f` so `cscli` does not overwrite the pod’s own `local_api_credentials.yaml`):

```bash
PASS=$(kubectl get secret homepage-secrets -n homepage -o jsonpath='{.data.HOMEPAGE_VAR_CROWDSEC_PASS}' | base64 -d)
kubectl exec -n crowdsec deploy/crowdsec-lapi -- env "PASS=$PASS" sh -c \
  'cscli machines add <machine-name> -p "$PASS" -f /tmp/homepage-creds.yaml'
```

After rotating `HOMEPAGE_VAR_CROWDSEC_PASS`, either update the machine in CrowdSec (`cscli machines delete` / `add` with `--force`) or re-register with the new password.

## Theme advice
Avoid overly dark background overlays on top of a dark theme. They can look good during first paint and then collapse to nearly black after full render.
