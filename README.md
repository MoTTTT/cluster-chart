# cluster-chart: A Helm chart to generate Cluster API managed Cluster manifests

## Requirement

In order to leverage GitOps in the provisioning of clusters, a working Cluster API cluster template has been refactored into a Helm chart.

An original Cluster API template could be used with `clusterctl` commands to generate the cluster manifests, requiring:

- The editing of target cluster configuration information in ~/.config/cluster-api/clusterctl.yaml
- On an operator console with clusterctl installed, using the management cluster kubectl context.
- Manually adding the generated cluster manifest into a GitOps repo for a Cluster API management cluster, in order to provision a cluster.
- Manual changes to generated manifests would then be used to scale clusters or modify configuration.

This helm chart removes the necessity for management cluster access in order to provision new clusters, and adjust existing ones.

The chart supports the following CAPI providers, which need to be pre-installed on your CAPI management cluster:

- Proxmox CAPI Infrastructure Provider
- Talos Cluster Provider
- Talos Bootstrap Provider

### Installation of Cluster API onto an existing cluster, creating a "Management Cluster"

- kubectl installed, using management cluster context
- clusterctl installation
- Edit `~/.config/cluster-api/clusterctl.yaml` to match your Proxmox infrastructure, as described here: <https://github.com/ionos-cloud/cluster-api-provider-proxmox/blob/main/docs/Usage.md>
- NOTE: Configuring Proxmox access credentials via the Helm chart is being replaced by use of the `capmox-manager-credentials` secret created in the `capmox-system` namespace at provider installation time.
- Install providers: `clusterctl init --ipam in-cluster --core cluster-api -c talos -b talos -i proxmox`

### Usage

The following manifests templates are used to generate the manifests to provision a cluster using these CAPI providers:

| Template                         | Object Kind created                        |
|----------------------------------|--------------------------------------------|
| cluster.yaml                     | CAPI Cluster                               |
| machinedeployment.yaml           | CAPI MachineDeployment                     |
| namespace.yaml                   | Namespace: Identical to the cluster name   |
| NOTES.txt                        | Helm Release Notes                         |
| proxmoxcluster.yaml              | ProxmoxCluster                             |
| proxmoxmachinetemplate-cp.yaml   | ProxmoxMachineTemplate for Control Plane   |
| proxmoxmachinetemplate-w.yaml    | ProxmoxMachineTemplate for Worker Nodes    |
| secret.yaml (deprecated)         | capmox-manager-credentials Secret          |
| talosconfigtemplate.yaml         | Talos machine patch                        |
| taloscontrolplane.yaml           | Talos control plane patch                  |

### Values

The values used in the chart will need configuration to match your infrastructure.

| Value Name                        | Description                       | Default value                                                                                                  |
|-----------------------------------|-----------------------------------|----------------------------------------------------------------------------------------------------------------|
| cluster.name                      | Cluster name                      | cluster09                                                                                                      |
| cluster.version                   | Kubernetes version                | v1.32.0                                                                                                        |
| cluster.image                     | Talos image                       | factory.talos.dev/nocloud-installer/6adc7e7fba27948460e2231e5272e88b85159da3f3db980551976bf9898ff64b:v1.11.1   |
| network.ip_ranges                 | IP range for the cluster          | [192.168.4.101-192.168.4.107]                                                                                  |
| network.ip_prefix                 | IP prefix for the cluster         | 24                                                                                                             |
| network.gateway                   | Gateway for the cluster           | 192.168.4.1                                                                                                    |
| network.bastion_host              | Bastion host                      | freyr                                                                                                          |
| network.bastion_host_endpoint_ip  | Bastion host endpoint IP          | 192.168.1.80                                                                                                   |
| network.dns_servers               | DNS servers                       | [192.168.1.201]                                                                                                |
| controlplane.endpoint_ip          | Control plane IP                  | 192.168.4.100                                                                                                  |
| controlplane.machine_count        | Control plane machines            | 1                                                                                                              |
| controlplane.boot_volume_size     | Control plane boot volume size    | 40                                                                                                             |
| controlplane.num_cores            | Control plane cores               | 4                                                                                                              |
| controlplane.num_sockets          | Control plane sockets             | 4                                                                                                              |
| controlplane.memory_mib           | Control plane memory              | 16384                                                                                                          |
| worker.machine_count              | Worker machines                   | 1                                                                                                              |
| worker.boot_volume_size           | Worker boot volume size           | 140                                                                                                            |
| worker.num_cores                  | Worker cores                      | 4                                                                                                              |
| worker.num_sockets                | Worker sockets                    | 4                                                                                                              |
| worker.memory_mib                 | Worker memory                     | 16384                                                                                                          |
| proxmox.url                       | Proxmox URL (deprecated)          | https://proxmox:8006                                                                                           |
| proxmox.token                     | Proxmox Token (deprecated)        | Token name                                                                                                     |
| proxmox.allowed_nodes             | Proxmox Allowed Nodes             | [venus]                                                                                                        |
| proxmox.secret                    | Proxmox Secret  (deprecated)      | Token value                                                                                                    |
| proxmox.template.sourcenode       | Proxmox Source Node               | venus                                                                                                          |
| proxmox.template.template_vmid    | Proxmox Template VM ID            | 100                                                                                                            |
| proxmox.vm.boot_volume_device     | Proxmox Boot Volume Device        | virtio0                                                                                                        |
| proxmox.vm.bridge                 | Proxmox Bridge                    | vmbr0                                                                                                          |

### Bastion server

If the target cluster's kubernetes api server needs to be accessed indirectly, using an external hostname, IP Address or both, these can be added to the api certificates SAN.

Best practice would be to avoid exposing the cluster's api endpoints externally, so the bastion server's ports should be set up for local access only.

### References

- <https://github.com/kubernetes-sigs/cluster-api>
- <https://github.com/ionos-cloud/cluster-api-provider-proxmox>
- <https://github.com/siderolabs/cluster-api-bootstrap-provider-talos>
- <https://github.com/siderolabs/cluster-api-control-plane-provider-talos>
- <https://a-cup-of.coffee/blog/talos-capi-proxmox/>
