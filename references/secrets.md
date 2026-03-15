# SOPS Secrets Reference

## Common pattern
Use age-backed SOPS files for app and infrastructure secrets.

### Decrypt
```bash
SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt sops -d path/to/file.sops.yaml
```

### Add or update a value
```bash
SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt \
  sops --set '["stringData"]["KEY_NAME"] "value"' path/to/file.sops.yaml
```

### Remove a value
SOPS has no delete command. Decrypt, edit plaintext, re-encrypt.

```bash
SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt sops -d file.sops.yaml > /tmp/plain.sops.yaml
# edit /tmp/plain.sops.yaml
cp /tmp/plain.sops.yaml /tmp/plain.sops.yaml.sops.yaml
SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt \
  sops --config .sops.yaml -e /tmp/plain.sops.yaml.sops.yaml > file.sops.yaml
rm /tmp/plain.sops.yaml /tmp/plain.sops.yaml.sops.yaml
```

## Practical advice
- Keep secret keys aligned with how the app consumes them.
- Prefer app-specific secret names over giant shared secrets.
- After changing a secret, verify the controller/app actually picked it up.
- If a workload consumes secrets via env vars, a restart may be required unless a reloader is installed.
