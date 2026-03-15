# cluster-chart: A Helm chart to generate Cluster API Cluster manifests

The chart supports the following CAPI providers, which need to be pre-installed on your CAPI management cluster:

- Proxmox CAPI Infrastructure Provider
- Talos Cluster Provider
- Talos Bootstrap Provider

## Pre-requisites

- kubectl (Kubernetes command line tool)
- clusterctl (Cluster API cluster management tool, used to install providers)
- helm (Kubernetes package manager) required unless you are using server-side apply.
- talosctl (Talos command line tool)
- flux (GitOps tool) to connect your cluster to a gitops repo
- Proxmox (Hypervisor). You will need a user token name, token value, and URL
- Talos template vm created in your proxmox cluster.
- Management Cluster: Any kubernetes cluster with network access to your hypervisor will do.

### Configuration and Installation of Cluster API

- Edit `~/.config/cluster-api/clusterctl.yaml` to match your Proxmox infrastructure
- Reference: <https://github.com/ionos-cloud/cluster-api-provider-proxmox/blob/main/docs/Usage.md>
- Install providers: `clusterctl init --ipam in-cluster --core cluster-api -c talos -b talos -i proxmox`
- Ensure the `capmox-manager-credentials` secret in the `capmox-system` namespace is present and correct in the Management Cluster.

## Values

The values used in the chart will need configuration to match your requirements. Most of them are going to need attention.

IP Address range defaults to 192.168.4.191-192.168.4.197, and Control plane VIP to 192.168.4.190.

The default machine sizing is a single 16 CPU, 16 GB RAM, and 40GB DISK control plane node, and a single 16 CPU, 16 GB RAM, and 140GB DISK worker node.

The default network uses 192.168.4.0/24 subnet, with a gateway of 192.168.4.1. The Gateway/Bastion server is available on an external network with IP address 192.168.1.80, and hostname "freyr".

The default Talos image is V1.11.1, NoCloud, amd64, with siderolabs/drbd, siderolabs/qemu-guest-agent extensions.

| Value Name                        | Description                       | Default value                                                                                                 |
|-----------------------------------|-----------------------------------|---------------------------------------------------------------------------------------------------------------|
| cluster.name                      | Cluster name                      | cluster09                                                                                                     |
| cluster.version                   | Kubernetes version                | v1.32.0                                                                                                       |
| cluster.image                     | Talos image                       | factory.talos.dev/nocloud-installer/6adc7e7fba27948460e2231e5272e88b85159da3f3db980551976bf9898ff64b:v1.11.1  |
| network.ip_ranges                 | IP range for the cluster          | [192.168.4.191-192.168.4.197]                                                                                 |
| network.ip_prefix                 | IP prefix for the cluster         | 24                                                                                                            |
| network.gateway                   | Gateway for the cluster           | 192.168.4.1                                                                                                   |
| network.bastion_host              | Bastion host                      | freyr                                                                                                         |
| network.bastion_host_endpoint_ip  | Bastion host endpoint IP          | 192.168.1.80                                                                                                  |
| network.dns_servers               | DNS servers                       | [192.168.1.201]                                                                                               |
| controlplane.endpoint_ip          | Control plane IP                  | 192.168.4.190                                                                                                 |
| controlplane.machine_count        | Control plane machines            | 1                                                                                                             |
| controlplane.boot_volume_size     | Control plane boot volume size    | 40                                                                                                            |
| controlplane.num_cores            | Control plane cores               | 4                                                                                                             |
| controlplane.num_sockets          | Control plane sockets             | 4                                                                                                             |
| controlplane.memory_mib           | Control plane memory              | 16384                                                                                                         |
| controlplane.extra_manifests      | Control plane extra manifests     | extra_manifests:                                                                                              |
|                                   |                                   |   - http://192.168.4.1/cilium.yaml                                                                            |
|                                   |                                   |   - http://192.168.4.1/kubelet-serving-cert-approver.yaml                                                     |
|                                   |                                   |   - http://192.168.4.1/metrics-server.yaml                                                                    |
|                                   |                                   |   - http://192.168.4.1/piraeus-operator.yaml                                                                  |
|                                   |                                   |   - http://192.168.4.1/gateway-api.yaml                                                                       |
|                                   |                                   |   - http://192.168.4.1/flux.yaml                                                                              |
|                                   |                                   |   - http://192.168.4.1/flux-instance-observability.yaml                                                       |
| worker.machine_count              | Worker machines                   | 1                                                                                                             |
| worker.boot_volume_size           | Worker boot volume size           | 140                                                                                                           |
| worker.num_cores                  | Worker cores                      | 4                                                                                                             |
| worker.num_sockets                | Worker sockets                    | 4                                                                                                             |
| worker.memory_mib                 | Worker memory                     | 16384                                                                                                         |
| proxmox.allowed_nodes             | Proxmox Allowed Nodes             | [venus]                                                                                                       |
| proxmox.template.sourcenode       | Proxmox Source Node               | venus                                                                                                         |
| proxmox.template.template_vmid    | Proxmox Template VM ID            | 100                                                                                                           |
| proxmox.vm.boot_volume_device     | Proxmox Boot Volume Device        | virtio0                                                                                                       |
| proxmox.vm.bridge                 | Proxmox Bridge                    | vmbr0                                                                                                         |

## Usage

### Helm CLI

- Install the chart repo: `helm repo add cluster-chart https://motttt.github.io/charts/ `
- Generate a dry-run manifest:`helm install   --dry-run   --debug   dry-run-release cluster-chart/cluster-chart  >  cluster-chart-dryrun.yaml`
- `cluster-chart-dryrun.yaml` will contain the default values and the manifests that are generated using no value overrides.
- Apply the manifest to your management cluster once you are happy with the values.
- Personalised manifests can be generated by creating your own values file, or by defining the values inline:

```bash
helm install   cluster08-release cluster-chart/cluster-chart  \ 
        --set cluster.name=cluster08 \
        --set network.ip_ranges='[192.168.4.181-192.168.4.187]' \
        --set controlplane.endpoint_ip=192.168.4.180 \
        --set controlplane.num_cores=2 \
        --set controlplane.num_sockets=2 \
        --set controlplane.memory_mib=8192 \
        --set worker.boot_volume_size=40 \
        --set worker.num_cores=2 \
        --set worker.num_sockets=2 \
        --set worker.memory_mib=8192
```

- The release notes provide sample commands for Cluster API progress monitoring, Talos and kube configuration extraction, and Flux bootstrapping:
- `helm get notes cluster08-release -n default`, will provide something to the following effect:

```text
NOTES:
Thank you for installing cluster-chart.
Your release is named cluster08-release.
Cluster API resources have been created in the cluster08 namespace.

To viw progress with the Cluster API provisioning:
  $ clusterctl describe cluster cluster08 -n cluster08 --show-conditions all --show-templates --show-resourcesets --grouping=false --echo

To extract talos configuration:
1. First ensure kubectl context is configured for (the correct) management cluster

To view the contexts already configured (in your ~/.kube/config file):
  $ kubectl config get-contexts
For example if your Management cluster is called "Management"
  $ kubectl config use-context admin@Management

2. Extract talos configuration file for the the target cluster: 
  $ kubectl get secret --namespace cluster08 cluster08-talosconfig -o jsonpath='{.data.talosconfig}' | base64 -d > cluster08-talosconfig

3. Extract kubectl config file for the the target cluster.
  $ kubectl get secret --namespace cluster08 cluster08-kubeconfig -o jsonpath='{.data.value}' | base64 -d > cluster08-kubeconfig

This step is optional, but will be required where the access configuration needs to be provided to a cluster owner, 
or where a bastion server is used.
If you are going to be accessing the kubernetes API for the target cluster via a bastion server, 
it will be necessary to adjust the generated endpoint, and port in the API Server URL in the cluster08-kubeconfig file accordingly.

4: Configure kubectl (i.e. your ~/.kube/config file) to include the context and credentials for the new cluster:
  $ talosctl kubeconfig --nodes 192.168.4.180 --endpoints 192.168.4.180 --talosconfig=./cluster08-talosconfig
This will set the kubectl context of the target cluster active.

Kubectl or k9s can be used to watch the cluster pods provisioning, and view logs. Depending on your cluster configuration this may take a while.

Once your cluster stabilises, and the pods are all running, you can bootstrap your cluster for gitops.
To bootstrap the cluster, add a secret to your cluster with the following (using the kubectl context of of the cluster08 cluster):
  $ flux create secret git flux-system --url=https://github.com/<username>/<Git repository name>.git  --username=<Git username>  --password=<Git token>
The flux operator will pick up the secret and sync your cluster to your gitops repository.
```

### Flux example

The cluster-chart helm chart was created to mechanism to add clusters to a Management cluster using gitops.

If your management cluster is using Flux, then each managed cluster can be added as a kustomization.

See an example in the cluster09 git repo <https://github.com/MoTTTT/cluster09/tree/main/gitops/cluster-charts/observability>.

## Manifest Templates

The following manifests templates are used to generate the manifests to provision a cluster using the Proxmox and Talos CAPI providers:

| Template                         | Object Kind created                        |
|----------------------------------|--------------------------------------------|
| cluster.yaml                     | CAPI Cluster                               |
| machinedeployment.yaml           | CAPI MachineDeployment                     |
| namespace.yaml                   | Namespace: Identical to the cluster name   |
| NOTES.txt                        | Helm Release Notes                         |
| proxmoxcluster.yaml              | ProxmoxCluster                             |
| proxmoxmachinetemplate-cp.yaml   | ProxmoxMachineTemplate for Control Plane   |
| proxmoxmachinetemplate-w.yaml    | ProxmoxMachineTemplate for Worker Nodes    |
| talosconfigtemplate.yaml         | Talos machine patch                        |
| taloscontrolplane.yaml           | Talos control plane patch                  |

## Networking considerations

Bastion server entries in the values file can be used to add additional entries to SAN (Subject Alternative Name) of the kube api certificate.

If the target cluster's kubernetes api server needs to be accessed indirectly, using an external hostname, IP Address or both, these can be added to the api certificates SAN, using values for `network.bastion_host` and `network.bastion_host_endpoint_ip`.

Best practice would be to avoid exposing the cluster's api endpoints externally, so the bastion server's ports should be set up for local access only, using a VPN or similar solution if remote access to the target cluster api is specifically required.

## References

- <https://github.com/kubernetes-sigs/cluster-api>
- <https://github.com/ionos-cloud/cluster-api-provider-proxmox>
- <https://github.com/siderolabs/cluster-api-bootstrap-provider-talos>
- <https://github.com/siderolabs/cluster-api-control-plane-provider-talos>
- <https://a-cup-of.coffee/blog/talos-capi-proxmox/>
