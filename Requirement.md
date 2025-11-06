# Requirement

In order to leverage GitOps in the provisioning of clusters, a working Cluster API cluster template has been refactored into a Helm chart.

An original Cluster API template could be used with `clusterctl` commands to generate the cluster manifests, requiring:

- The editing of target cluster configuration information in ~/.config/cluster-api/clusterctl.yaml
- On an operator console with clusterctl installed, using the management cluster kubectl context.
- Manually adding the generated cluster manifest into a GitOps repo for a Cluster API management cluster, in order to provision a cluster.
- Manual changes to generated manifests would then be used to scale clusters or modify configuration.

This helm chart, used with gitops (tested with flux kustomizations), removes the necessity for management cluster access in order to provision new clusters, and adjust (some 
aspects of) existing ones.
