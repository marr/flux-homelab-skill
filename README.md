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
- `SKILL.md`
- `references/flux.md`
- `references/secrets.md`
- `references/homepage.md`

## Install
Use the packaged `.skill` file from the GitHub release, or clone this repo and package it with your own tooling.

## Notes
This repository includes a README for GitHub discoverability. The packaged skill itself is distributed as a `.skill` artifact in Releases.
