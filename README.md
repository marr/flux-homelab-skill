# flux-homelab

A sanitized skill for managing a small Kubernetes homelab using:
- Flux GitOps
- SOPS-encrypted secrets
- a homepage-style dashboard

This version is safe to share publicly. It focuses on reusable workflows and gotchas without including private infrastructure details.

## What it helps with
- investigating service errors
- reconciling Flux changes
- updating encrypted secrets with SOPS
- configuring homepage widgets
- debugging ConfigMap/Secret changes that did not reach running pods

## Contents
- `SKILL.md` — lightweight skill definition with reference links
- `references/` — detailed documentation for Flux, SOPS, Homepage, and cluster hygiene

## Install

Install via npx skills:

```bash
npx skills add marr/flux-homelab-skill
```

## Notes
This repository includes a README for GitHub discoverability. The packaged skill itself is distributed as a `.skill` artifact in Releases.
