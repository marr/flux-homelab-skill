# Flux Reference

## Reconcile
```bash
flux reconcile kustomization flux-system --with-source
flux reconcile kustomization infrastructure-controllers --with-source
flux reconcile kustomization infrastructure-configs --with-source
flux reconcile kustomization apps --with-source
```

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
