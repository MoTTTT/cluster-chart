# cluster-chart

A Helm chart that generates [Cluster API](https://cluster-api.sigs.k8s.io/) manifests for provisioning Talos Linux clusters on Proxmox hypervisors.

## Features

- **Single values file** — all cluster configuration expressed in one `values.yaml`; the chart renders the complete set of CAPI, Talos, and Proxmox manifests ready to apply to a management cluster.
- **Vanilla-first defaults** — the default install is a minimal single control-plane node cluster with no external dependencies: no CNI override, no registry mirrors, no extra manifests.
- **Capability flags** — `cni: cilium` and `storage: in-cluster` flags wire the correct Talos MachineConfig settings atomically. Each flag is documented with its mutability tier (Tier 2 rolling update / Tier 3 reprovision).
- **Configurable certSANs** — `network.certSANs` accepts an array of IPs and hostnames added to the API server TLS certificate. Empty by default; add entries for VPN gateways or jump hosts as needed.
- **Registry mirrors** — `registries.enabled` gates Talos pull-through proxy configuration. Off by default; a commented example is provided in `values.yaml`.
- **Control plane scheduling** — `cluster.controlplaneScheduling: true` (default) allows workload pods on control plane nodes, enabling single-node dev/test clusters without worker nodes.
- **Schema validated** — `values.schema.json` enforces types, required fields, and valid enum values at `helm install` / `helm upgrade` time.
- **gitopsapi-ready** — `values.yaml` is the stable API contract consumed by [gitopsapi](https://github.com/MoTTTT/gitopsapi) to provision clusters programmatically. The chart can also be used standalone via Helm CLI.

---

## Quick Start

This guide walks through provisioning a cluster directly with Helm, without gitopsapi, registry mirrors, a networking service provider, or a VPN gateway.

### Prerequisites

| Dependency | Purpose |
| --- | --- |
| [kubectl](https://kubernetes.io/docs/tasks/tools/) | Apply manifests and extract secrets from the management cluster |
| [helm](https://helm.sh/docs/intro/install/) v3 | Install and manage the chart |
| [talosctl](https://www.talos.dev/latest/introduction/getting-started/) | Interact with Talos nodes; activate kubeconfig |
| [clusterctl](https://cluster-api.sigs.k8s.io/user/quick-start) | Monitor CAPI provisioning progress |
| **CAPI management cluster** | Any Kubernetes cluster reachable from your Proxmox hosts. Must have the Proxmox, Talos, and in-cluster IPAM providers installed (see below). |
| **Proxmox hypervisor** | One or more Proxmox nodes. Requires a Talos Linux VM template and a CAPMOX API token. |

#### Install CAPI providers on the management cluster

```bash
# Point kubectl at your management cluster, then:
clusterctl init \
  --ipam in-cluster \
  --core cluster-api \
  --control-plane talos \
  --bootstrap talos \
  --infrastructure proxmox
```

Ensure the `capmox-manager-credentials` secret is present in the `capmox-system` namespace with your Proxmox API token credentials. See the [CAPMOX usage guide](https://github.com/ionos-cloud/cluster-api-provider-proxmox/blob/main/docs/Usage.md) for the full secret format.

#### Create a Talos VM template in Proxmox

Download a Talos factory image and register it as a Proxmox VM template. The factory image URL encodes the Talos system extension set for your hardware. Generate one at [factory.talos.dev](https://factory.talos.dev) — select the `nocloud` platform and any required extensions (e.g. `qemu-guest-agent`).

Note the template VMID and the Proxmox node name — you will need both in your values file.

---

### 1. Add the Helm repository

```bash
helm repo add cluster-chart https://motttt.github.io/charts/
helm repo update
```

---

### 2. Create a cluster values file

Create a `my-cluster.yaml` with the values specific to your environment. Only site-specific overrides are needed — everything else inherits safe defaults.

```yaml
cluster:
  name: my-cluster
  # Talos factory image URL for your hardware — generate at factory.talos.dev
  image: factory.talos.dev/nocloud-installer/<your-schematic-id>:v1.11.5

network:
  # IP pool for cluster VMs — must be within your LAN subnet and not in use
  ip_ranges: [192.168.x.x-192.168.x.x]
  ip_prefix: 24
  gateway: 192.168.x.1

controlplane:
  # VIP for the control plane — must be in the subnet but outside ip_ranges
  endpoint_ip: "192.168.x.x"

proxmox:
  allowed_nodes: [your-proxmox-node]
  template:
    sourcenode: your-proxmox-node
    template_vmid: 100   # VMID of your Talos template
```

> **Tip:** The cluster09 IP block (`192.168.4.190–197`) is the chart default and can be used for a first test install if that range is available on your network.

---

### 3. Install the chart

```bash
helm install my-cluster cluster-chart/cluster-chart -f my-cluster.yaml
```

Helm validates your values against `values.schema.json` before rendering. Fix any reported errors before proceeding.

To preview the generated manifests without applying:

```bash
helm template my-cluster cluster-chart/cluster-chart -f my-cluster.yaml > my-cluster-manifests.yaml
```

---

### 4. Monitor provisioning

Run against the management cluster:

```bash
clusterctl describe cluster my-cluster -n my-cluster \
  --show-conditions all --show-templates --show-resourcesets --grouping=false --echo
```

Provisioning is complete when all control plane nodes reach `Ready`.

---

### 5. Extract credentials

```bash
# Talos configuration
kubectl get secret -n my-cluster my-cluster-talosconfig \
  -o jsonpath='{.data.talosconfig}' | base64 -d > my-cluster-talosconfig

# Kubeconfig
kubectl get secret -n my-cluster my-cluster-kubeconfig \
  -o jsonpath='{.data.value}' | base64 -d > my-cluster-kubeconfig

# Activate kubeconfig via talosctl
talosctl kubeconfig \
  --nodes <controlplane-endpoint-ip> \
  --endpoints <controlplane-endpoint-ip> \
  --talosconfig=./my-cluster-talosconfig
```

You can now use `kubectl` or `k9s` against the new cluster.

---

## Values Reference

### Defaults summary

The default install provisions a **single control plane node with no workers**. Control plane scheduling is enabled, so workloads can run on the control plane node — suitable for dev/test. For production, set `worker.machine_count: 3` (or more) and `cluster.controlplaneScheduling: false`.

Default VM sizing: 4 sockets × 4 cores, 16 GB RAM, 40 GB boot volume.

### Values table

| Value | Description | Default |
| --- | --- | --- |
| `cluster.name` | Unique cluster name. Used as the CAPI namespace and resource name prefix. | `YourNewCluster` |
| `cluster.kubernetes_version` | Kubernetes version to deploy. | `v1.34.2` |
| `cluster.talos_version` | Talos major.minor version (e.g. `v1.11`). Used in the `talosVersion` field. | `v1.11` |
| `cluster.image` | Full Talos factory installer image URL including tag. | factory.talos.dev image `v1.11.5` |
| `cluster.hostname` | Public-facing ingress hostnames. See [Networking](#networking). | `[]` |
| `cluster.internalhost` | Internal-only ingress hostnames. See [Networking](#networking). | `[]` |
| `cluster.controlplaneScheduling` | Allow workloads to schedule on control plane nodes. | `true` |
| `cni` | CNI selection: `""` (default kube-proxy) or `"cilium"`. Tier 3 immutable. | `""` |
| `storage` | Storage flag: `""` (none) or `"in-cluster"` (adds drbd kernel modules). Tier 2. | `""` |
| `machine.installDisk` | Install disk device path. `/dev/vda` for Proxmox virtio; `/dev/sda` for physical. | `/dev/vda` |
| `network.ip_ranges` | IP pool for cluster VMs, allocated by the in-cluster IPAM provider. | `[192.168.4.191-192.168.4.197]` |
| `network.ip_prefix` | Subnet prefix length (e.g. `24` = /24). | `24` |
| `network.gateway` | Default gateway for cluster VMs. | `192.168.4.1` |
| `network.dns_servers` | DNS resolvers for cluster VMs. | `[8.8.8.8, 8.8.4.4]` |
| `network.certSANs` | Additional SANs for the API server TLS certificate. Empty = no external access endpoint. | `[]` |
| `controlplane.endpoint_ip` | Control plane VIP — floats across CP nodes; must be in the subnet but outside `ip_ranges`. | `192.168.4.190` |
| `controlplane.machine_count` | Number of control plane nodes. Use `1` for dev/test, `3` for HA. | `1` |
| `controlplane.machine_template_suffix` | Suffix for the ProxmoxMachineTemplate name. | `controlplane` |
| `controlplane.boot_volume_size` | Boot volume size in GB. | `40` |
| `controlplane.num_cores` | vCPU cores per socket. | `4` |
| `controlplane.num_sockets` | vCPU sockets. | `4` |
| `controlplane.memory_mib` | RAM in MiB. | `16384` |
| `controlplane.cloudProviderManifests` | Manifests applied by the Talos external cloud provider at boot. Includes the CCM. | CCM URL |
| `worker.machine_count` | Number of worker nodes. `0` = single-node cluster (uses CP scheduling). | `0` |
| `worker.machine_template_suffix` | Suffix for the worker ProxmoxMachineTemplate name. | `worker` |
| `worker.boot_volume_size` | Worker boot volume size in GB. | `40` |
| `worker.num_cores` | Worker vCPU cores per socket. | `4` |
| `worker.num_sockets` | Worker vCPU sockets. | `4` |
| `worker.memory_mib` | Worker RAM in MiB. | `16384` |
| `proxmox.allowed_nodes` | Proxmox nodes eligible to host cluster VMs. | `[venus]` |
| `proxmox.template.sourcenode` | Proxmox node holding the Talos VM template. | `venus` |
| `proxmox.template.template_vmid` | VMID of the Talos VM template. | `100` |
| `proxmox.vm.boot_volume_device` | Proxmox disk device for the boot volume. | `virtio0` |
| `proxmox.vm.bridge` | Proxmox bridge interface for VM networking. | `vmbr0` |
| `registries.enabled` | Enable Talos registry mirrors. Requires `registries.mirrors` to be configured. | `false` |
| `registries.mirrors` | Mirror configuration map. See commented example in `values.yaml`. | `{}` |

### Capability flags

Two capability flags control Talos MachineConfig features that are set at provision time and have different mutability characteristics.

| Flag | Values | Effect | Mutability |
| --- | --- | --- | --- |
| `cni` | `""` \| `"cilium"` | `cilium` sets `cluster.network.cni.name: none` + `cluster.proxy.disabled: true` in the Talos patch for all nodes. | **Tier 3** — cannot be changed on a live cluster without reprovisioning. |
| `storage` | `""` \| `"in-cluster"` | `in-cluster` loads the `drbd` and `drbd_transport_tcp` kernel modules on all nodes. Required for Linstor/Piraeus in-cluster storage operators. | **Tier 2** — requires rolling node replacement. |

---

## Manifest Templates

| Template | Object kind |
| --- | --- |
| `cluster.yaml` | `Cluster` (CAPI) |
| `machinedeployment.yaml` | `MachineDeployment` (CAPI) |
| `namespace.yaml` | `Namespace` — identical to `cluster.name` |
| `proxmoxcluster.yaml` | `ProxmoxCluster` |
| `proxmoxmachinetemplate-cp.yaml` | `ProxmoxMachineTemplate` — control plane |
| `proxmoxmachinetemplate-w.yaml` | `ProxmoxMachineTemplate` — workers |
| `talosconfigtemplate.yaml` | `TalosConfigTemplate` — worker Talos MachineConfig patch |
| `taloscontrolplane.yaml` | `TalosControlPlane` — control plane Talos MachineConfig patch |
| `NOTES.txt` | Helm release notes — provisioning commands and cluster configuration summary |

---

## Networking

### Public ingress (`cluster.hostname`)

Hostnames listed under `cluster.hostname` are intended for public-facing ingress. Two routing approaches are supported:

- **On-premise routing** — configure your internet-facing load balancer or reverse proxy to forward traffic to the cluster Gateway IP. Hostnames must be resolvable from the internet.
- **Network Service Provider (e.g. Cloudflare)** — proxied DNS records provide WAF, DDoS protection, and TLS termination without exposing cluster IPs. Recommended where no on-premise Web Application Firewall is available.

### Internal ingress (`cluster.internalhost`)

Hostnames listed under `cluster.internalhost` are served on port 443 with TLS issued by a DNS-01 ACME ClusterIssuer. DNS-01 supports wildcard certificates without enumerating individual hostnames. Clients resolve these names via internal DNS (a local resolver or split-horizon DNS zone).

### API server access (`network.certSANs`)

The `network.certSANs` array adds IPs and hostnames to the API server's TLS certificate SAN list. Add entries for any endpoint through which the cluster API will be accessed externally — VPN gateway, jump host, or load balancer address.

Best practice is to restrict API server access to the cluster network and use a VPN for remote access, eliminating the need for external SANs. The `certSANs` default is empty (`[]`).

---

## Using with gitopsapi or Flux

In production, `cluster-chart` is typically consumed by [gitopsapi](https://github.com/MoTTTT/gitopsapi), which generates the `values.yaml` from a `ClusterSpec` and applies the Helm release to the management cluster automatically.

For GitOps-managed management clusters using Flux, each cluster can also be added as a `HelmRelease` kustomization pointing at the `cluster-chart` chart with a cluster-specific values file.

---

## Schema Validation

The chart includes `values.schema.json` which Helm uses to validate your values at `install` / `upgrade` / `template` time. Validation catches type errors, missing required fields, and invalid capability flag values before any manifests are rendered.

To test your values file against the schema without installing:

```bash
helm template my-cluster cluster-chart/cluster-chart -f my-cluster.yaml
```

Helm will report schema violations and exit non-zero if validation fails.

---

## References

- [Cluster API](https://github.com/kubernetes-sigs/cluster-api)
- [Cluster API Provider Proxmox (CAPMOX)](https://github.com/ionos-cloud/cluster-api-provider-proxmox)
- [Cluster API Bootstrap Provider Talos](https://github.com/siderolabs/cluster-api-bootstrap-provider-talos)
- [Cluster API Control Plane Provider Talos](https://github.com/siderolabs/cluster-api-control-plane-provider-talos)
- [Talos Linux](https://www.talos.dev)
- [Talos Factory](https://factory.talos.dev)
