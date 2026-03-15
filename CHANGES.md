# Changes

## TODO

- [X] Extract talos patches into values file: extraManifests, and Registries
- [X] SOPS for secrets. Potential candidates: Proxmox credentials; Flux GIT token.
- [X] Roll cluster-chart into cluster09
- [X] Known issue: Deleting a cluster deletes the CAPMOX cluster credentials: Mitigation: add a credential per cluster, and to management cluster flux, encrypted with sops.
- [X] Known issue: `error creating rbac.authorization.k8s.io/v1/Role/cilium-ingress-secrets: namespaces \"cilium-secrets\" not found`: Only experienced once, may have been wan related.
- [X] Issue: Deleting a cluster removes `capmox-manager-credentials`: Workaround is to create and re-apply a manifest for the secret. Sorted

## Issues

- [X] Name and Namespace not rendering correctly: values.yaml file variable formats fixed

## Versions

### V0.1.19

- Default Talos: v1.11.5; Kubernetes: v1.34.2

### V0.1.18

- Default flux repo secret pull from local server.

### V0.1.17

- Proxmox secret (encrypted) in cluster namespace.

### V0.1.16

- Fix local manifest URL

### V0.1.15

- Regress to local manifest load.

### V0.1.14

- Manifest load order

### V0.1.13

- Use github pages for machine extraManifests.

### V0.1.12 (not released)

- Eternalise registries configuration, fix harbor address (V2...)
- Add talos command cheat sheet to NOTES.md, refactored flux notes.

### V0.1.11

- Fix harbour address

### V0.1.10

- Debug name and namespace

### V0.1.9

- Retest harbor mirror config, added entries for `machine.registries.mirrors` to `talosconfigtemplate.yaml`, and `taloscontrolplane.yaml` template files.

## Releasing charts to the repo

- Bump the version in the Chart.yaml file.
- In project root directory, run: `helm package cluster-chart`
- A new tgz file is created.
- Run `helm repo index .` to update the `index.yaml` file.
- Check in, and merge into gh-pages.
