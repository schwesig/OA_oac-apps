# Workload Cluster Deployment

## Infrastructure Configuration

A workload cluster only needs nodes for additional worker nodes (since its control plane runs on its associated infra cluster). The number of needed worker nodes is not fixed, and is dependent on the desired capacity of the workload cluster.

* **Networking**
   * A workload cluster requires a cluster network and a storage network
      * [Add cluster and storage networks to the MOC inventory](network-inventory-and-configuration.md#network-inventory)
      * [Attach these networks to each node](hardware-inventory-and-configuration.md#network-configuration)
   * Perform the [appropriate firewall configurations](network-inventory-and-configuration.md#firewall) for the cluster and storage networks
* **`open-accelerator-infra` inventory**
   * Modify [`inventory/00hosts.yaml`](https://github.com/CCI-MOC/open-accelerator-infra/blob/main/infra/inventory/00hosts.yaml) to add the new workload cluster and its associated nodes
* **Add nodes as agents to the infra cluster**
   * Agent discovery image
      * Obtain the agent discovery image from the infra cluster: `oc get infraenv -n hardware-inventory hardware-inventory -o jsonpath='{.status.isoDownloadURL}'`
      * Rename the agent discovery image to `<cluster>-agent-discovery.iso` and copy it to the `/srv/boot/` directory of the bastion host
   * [Boot the nodes](hardware-inventory-and-configuration.md#boot-configuration) with the discovery image `http://10.2.0.82/boot/<cluster>-agent-discovery.iso`
   * These nodes should eventually appear as agents in the infra cluster: `oc get agents -n hardware-inventory`
   * Label these agents with additional metadata: `ansible-playbook update-agents.yaml -l <cluster>`

## Cluster Deployment and Configuration

Workload cluster configuration and deployment is managed through an ArgoCD instance running on the infra cluster, and the [`oac-apps` repository](https://github.com/CCI-MOC/oac-apps).

* `hosted-clusters/<infra-cluster>/<cluster>/values.yaml`: workload cluster configuration
* `values/<infra-cluster>/`: workload cluster component configuration
   * `<component>.yaml`: component configurations applicable hub-wide
   * `<cluster>/<component>.yaml`: component configurations applicable to workload cluster
* `apps/<infra-cluster>/`:  additional changes to be applied to the ArgoCD instance running on the infra cluster (which also manages workload clusters)

Make the desired customizations, and then submit them as a PR. Once the PR is merged, the ArgoCD instance running on the infra cluster will pick up the changes and deploy/configure the workload cluster.

Some shared services require additional work prior to configuration. These are detailed here:

* [Pure storage configuration](pure-storage-configuration.md)
* [IDP configuration](idp-configuration.md)
* [Cluster certificates](network-inventory-and-configuration.md#cluster-certificates)
* [Public IPs](network-inventory-and-configuration.md#public-ips)
* [External DNS](network-inventory-and-configuration.md#external-dns)
* [Logging](#logging)
* ??

After the workload cluster is deployed, you may optionally want to [acquire a public IP](network-inventory-and-configuration.md#public-ips) and [expose the cluster's ingress and API IPs to the public internet](network-inventory-and-configuration.md#firewall).

## Logging

Each workload cluster with logging keeps its own logs. The `cluster-logging` chart installs a LokiStack (`openshift-logging/logging-loki`) and a ClusterLogForwarder (`collector`) that collects application, infrastructure, and node audit logs (auditd and OVN). It also adds Observe > Logs to the console. Project members with `view` or `edit` in a namespace see that namespace's application logs, through a ClusterRole the chart aggregates into those roles; the infrastructure and audit logs are readable only by the groups in `viewers.groups` (by default `open-accelerator-admins`).

Storage is Pure: the log chunks go to a FlashBlade bucket claimed through COSI (`fb-bucket`), and the LokiStack volumes use `pure-fb-nfsv4`. Retention is 15 days for application and infrastructure logs and 90 days for audit logs.

A workload cluster's API server audit does not exist on the workload cluster: its API servers run in `clusters-<cluster>` on the infra cluster. The infra cluster collects it into its own LokiStack (see `apps/<infra-cluster>/hub-cluster-logging.yaml` and `values/<infra-cluster>/local-cluster/cluster-logging.yaml`), where it is kept for the audit retention. Query it there with `{kubernetes_namespace_name="clusters-<cluster>", kubernetes_container_name="audit-logs"}`.

To enable logging on a workload cluster:

1. Make sure the cluster's Pure S3 path works: it needs a `kustomize/pure-cosi/overlays/<cluster>` overlay and working `object-storage-proxy` values, so that a `BucketClaim` becomes ready.
2. Create `values/<infra-cluster>/<cluster>/cluster-logging.yaml` with at least the LokiStack size, e.g. `lokiStack.size: 1x.demo` on dev. The chart has no default size.
3. Add `massopen.cloud/feature-logging: "true"` to `clusterLabels` in `hosted-clusters/<infra-cluster>/<cluster>/values.yaml`. The `logging-clusters` placement then selects the cluster, and the `logging-clusters` ApplicationSet deploys the `loki-operator` and `cluster-logging` charts to it.
4. To collect the cluster's API audit, add `clusters-<cluster>` to `collector.inputs.hostedControlPlaneAudit.namespaces` in `values/<infra-cluster>/local-cluster/cluster-logging.yaml`. Only `oac-dev-infra` has the hub-side logging apps so far; another infra cluster needs its own `apps/<infra-cluster>/hub-loki-operator.yaml` and `hub-cluster-logging.yaml` first.

Removing the label or the chart keeps the bucket and its logs: the `BucketClaim` is never pruned or deleted by ArgoCD, so deleting it is a deliberate manual step.

## Example: OAC Prod Workload0 Cluster

* **Infrastructure Configuration**
   * VLANs
      * [Cluster network](https://github.com/CCI-MOC/ansible-switches/blob/main/group_vars/all/vlans.yaml#L99-L104)
      * [Storage network](https://github.com/CCI-MOC/ansible-switches/blob/main/group_vars/all/vlans.yaml#L2052-L2058)
   * Network configuration
      * [`MOC-R4PAC10-SW-TORS-A`](https://github.com/CCI-MOC/ansible-switches/blob/main/host_vars/MOC-R4PAC10-SW-TORS-A/interfaces.yaml) (search for `OAC Prod`)
      * [`MOC-R4PAC10-SW-TORS-B`](https://github.com/CCI-MOC/ansible-switches/blob/main/host_vars/MOC-R4PAC10-SW-TORS-B/interfaces.yaml) (search for `OAC Prod`)
      * Additional nodes are either manually configured, or managed through ESI
   * `open-accelerator-infra` hosts inventory
      * [`00hosts.yaml`](https://github.com/CCI-MOC/open-accelerator-infra/blob/main/infra/inventory/00hosts.yaml#L107-L146)
* **Cluster Deployment and Configuration**
   * `moc-keycloak` IDP configuration
      * [`main.tf`](https://github.com/CCI-MOC/moc-keycloak/blob/main/main.tf) (search for `oac_prod_workload0`)
   * `everpure-moc` Pure storage configuration
      * [`ansible/group_vars/all/realms.yaml`](https://github.com/CCI-MOC/everpure-moc/blob/main/ansible/group_vars/all/realms.yaml) (search for `oac-prod-workload0`)
   * `documentation` Public IP reservation
      * [`kaizen-networks.md`](https://github.com/CCI-MOC/documentation/blob/master/docs/source/networking/kaizen-networks.md#public-neu-vlan-127) (search for `oac_prod_workload0`)
   * `moc-aws` cluster certificates and external DNS configuration
      * [`main.tf`](https://github.com/CCI-MOC/moc-aws/blob/main/main.tf) (search for `oac_prod_workload0`)
      * [`cert-manager-policies.tf`](https://github.com/CCI-MOC/moc-aws/blob/main/cert-manager-policies.tf) (search for `oac_prod_workload0`)
   * `oac-apps` cluster configuration
      * `hosted-clusters/oac-prod-infra/`
         * [`values.yaml`](https://github.com/CCI-MOC/oac-apps/blob/main/hosted-clusters/oac-prod-workload0/values.yaml)
         * [`oac-prod-workload0/values.yaml`](https://github.com/CCI-MOC/oac-apps/blob/main/hosted-clusters/oac-prod-infra/oac-prod-workload0/values.yaml)
      * `values/oac-prod-infra/`
         * [`hcp-config.yaml`](https://github.com/CCI-MOC/oac-apps/blob/main/values/oac-prod-infra/hcp-config.yaml)
         * [`oac-prod-workload0/`](https://github.com/CCI-MOC/oac-apps/blob/main/values/oac-prod-infra/oac-prod-workload0)
            * [`portworx.yaml`](https://github.com/CCI-MOC/oac-apps/blob/main/values/oac-prod-infra/oac-prod-workload0/portworx.yaml)
            * [`cluster-certificates.yaml`](https://github.com/CCI-MOC/oac-apps/blob/main/values/oac-prod-infra/oac-prod-workload0/cluster-certificates.yaml)
            * additional component configuration
