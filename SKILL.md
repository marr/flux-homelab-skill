---
name: flux-homelab
description: Manage a private homelab or small-cluster Kubernetes environment that uses Flux GitOps, SOPS-encrypted secrets, and a homepage-style dashboard. Use when investigating service errors, reconciling Flux, adding apps or controllers, updating encrypted secrets, configuring homepage widgets, or debugging why config changes did not reach running pods.
---

# Flux Homelab Skill

Use this skill for repeatable GitOps homelab operations.

## Read these references as needed
- For Flux workflows and repo layout: `references/flux.md`
- For SOPS secret editing patterns: `references/secrets.md`
- For homepage widget and ConfigMap patterns: `references/homepage.md`
- For stale Endpoints, empty Services, Flux drift: `references/cluster-hygiene.md`

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
Follow `references/secrets.md`.

### Update homepage widgets
Follow `references/homepage.md`.

## Key gotchas
- Do not mix **manual v1/Endpoints** with **Git-managed EndpointSlice** for the same Service (flaky 502s if IPs drift).
- `subPath` ConfigMap mounts do not hot-reload.
- Homepage variable substitution requires matching `HOMEPAGE_VAR_*` secret keys.
- Some widgets need internal API compatibility flags or version selectors.
- Some services accept one auth mode in direct tests but require another for homepage widgets.
- SOPS has no delete-key command; use decrypt-edit-reencrypt.
- A **stuck `HelmRelease`** under a `Kustomization` with **`spec.wait: true`** blocks **`lastAppliedRevision`** from advancing and freezes **downstream `dependsOn` kustomizations**; **`helm-controller` CPU** can spike from retries. See `references/flux.md` (stuck releases, SSA + `upgrade.force`).
- **`HelmRelease.spec.install.serverSideApply`** and **`spec.upgrade.serverSideApply`** use **different types** in the Flux API (install: boolean; upgrade: `enabled` / `disabled` / `auto`). Using `disabled` on **install** fails validation. Pair **`upgrade.force: true`** with **`serverSideApply` disabled/false** or Helm may error: *force conflicts and force replace together*.
