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

### V0.1.35

- `vm.max_map_count`: hard-coded opinionated value `1048576` on all nodes unconditionally (was `262144`, was conditional on `storage: in-cluster`). Value suits all known use-cases including OpenSearch.
- `network.certSANs`: new array value replaces `network.bastion_host` + `network.bastion_host_endpoint_ip`. Add any external access endpoint IPs/hostnames per cluster. Bastion terminology removed from chart.
- `controlplane.cloudProviderManifests`: new array value surfaces the hard-coded external cloud provider manifest URL. Configurable per cluster.
- `storage` flag comment updated: controls drbd kernel modules only (sysctl is now unconditional).
- `controlplane.extra_manifests` removed from values.yaml (dead config since v0.1.33 removed ExtraManifests rendering).
- NOTES.txt: removed bastion terminology; updated GitOps bootstrap section (gitopsapi manages bootstrap — no manual ExtraManifests step); added Cluster Configuration section.

### V0.1.34

- Surface `machine.installDisk` to values.yaml (default `/dev/vda`); replaces hard-coded disk path in TalosControlPlane and TalosConfigTemplate.
- Add `storage` capability flag: `""` (default, vanilla install) or `"in-cluster"` — conditionally renders drbd kernel modules (`drbd`, `drbd_transport_tcp`) in both CP and worker patches.
- Add `cni` capability flag: `""` (default kube-proxy) or `"cilium"` — conditionally renders `cluster.network.cni.name: none` + `cluster.proxy.disabled: true` in both CP and worker patches.
- Both flags are Tier 3 (CNI) / Tier 2 (storage) immutable per the cluster mutability model — set at provision time via gitopsapi-generated values.yaml.
- PROJ-025/T-002 + T-004 complete.

### V0.1.33

- Remove cluster.network.cni, proxy.disabled, and extraManifests from TalosControlPlane entirely (not commented — Helm directives in block scalar comments cause invalid YAML output).

### V0.1.32

- Revert NTP server override — remove network.ntp_servers from values and templates.
- Comment out cluster.network.cni, cluster.proxy.disabled, and cluster.extraManifests in TalosControlPlane — removes Cilium from the equation for network diagnostics.

### V0.1.31

- Add `network.ntp_servers` value — rendered into `machine.time.servers` in TalosControlPlane and TalosConfigTemplate.
- Default NTP servers: `216.239.35.0`, `216.239.35.4` (Google NTP — routable via internet masquerade, no DNS required).

### V0.1.30

- `cluster.image` carries the full image URL including version tag — independent of `cluster.talos_version`.
- `cluster.talos_version` is used only for the `talosVersion` field (major.minor, e.g. `v1.10`).
- Default `talos_version` reverted to `v1.11`; default `image` restored to include `:v1.11.5` tag.

### V0.1.29

- Wire `cluster.talos_version` into `talosVersion` field in TalosControlPlane and TalosConfigTemplate (was hardcoded `v1.10`).
- Image tag derived from `cluster.talos_version` — strip version from `cluster.image`; templates render `image:talos_version`.
- Default `talos_version` updated to `v1.11.5`.

### V0.1.20

- Parameterise ProxmoxMachineTemplate names: add `controlplane.machine_template_suffix` and `worker.machine_template_suffix` values (defaults: `controlplane`, `worker`). Enables hash-based template name generation in T-020 (ClusterSpec rolling update semantics).

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
