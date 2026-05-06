---
name: flux-homelab
description: Manage a private homelab or small-cluster Kubernetes environment that uses Flux GitOps, SOPS-encrypted secrets, and a homepage-style dashboard. Use when investigating service errors, reconciling Flux, adding apps or controllers, updating encrypted secrets, configuring homepage widgets, or debugging why config changes did not reach running pods.
---

# Flux Homelab Skill

Use this skill for repeatable GitOps homelab operations. This skill is designed to be lightweight and references detailed documentation in the `references/` directory to minimize context bloat.

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
See `references/secrets.md` for SOPS patterns and commands.

### Update homepage widgets
See `references/homepage.md` for widget patterns and common gotchas.

### Flux operations and troubleshooting
See `references/flux.md` for reconciliation commands, stuck kustomizations, HelmRelease patterns, and OCI chart gotchas.

### Cluster maintenance
See `references/cluster-hygiene.md` for endpoint management, orphan services, and noisy log patterns.

## Quick reference

| Task | Location |
|------|----------|
| Flux commands | `references/flux.md` |
| SOPS secrets | `references/secrets.md` |
| Homepage widgets | `references/homepage.md` |
| Cluster hygiene | `references/cluster-hygiene.md` |

---

## Key gotchas (summary)
- Do not mix **manual v1/Endpoints** with **Git-managed EndpointSlice** for the same Service (flaky 502s if IPs drift).
- `subPath` ConfigMap mounts do not hot-reload.
- Homepage variable substitution requires matching `HOMEPAGE_VAR_*` secret keys.
- Some widgets need internal API compatibility flags or version selectors.
- Some services accept one auth mode in direct tests but require another for homepage widgets.
- SOPS has no delete-key command; use decrypt-edit-reencrypt.

For detailed explanations and commands, see the `references/` directory.
