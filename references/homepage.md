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
  url: http://traefik.<namespace>.svc.cluster.local:8080
```
This usually requires enabling Traefik's internal API/dashboard for in-cluster access.

### CrowdSec
```yaml
widget:
  type: crowdsec
  url: http://crowdsec-service.<namespace>.svc.cluster.local:8080
  username: <machine-name>
  password: "{{HOMEPAGE_VAR_CROWDSEC_PASS}}"
```
The widget expects watcher/machine auth. Bouncer keys may not work for all endpoints.

## Theme advice
Avoid overly dark background overlays on top of a dark theme. They can look good during first paint and then collapse to nearly black after full render.
