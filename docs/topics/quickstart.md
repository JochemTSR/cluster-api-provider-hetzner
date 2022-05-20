# Quickstart Guide

This guide goes through all necessary steps to create a cluster on Hetzner infrastructure (on HCloud & on Hetzner Dedicated).

## Preparing Hetzner

You have the two options of either creating a pure HCloud cluster or a hybrid cluster with Hetzner dedicated (bare metal) servers. For a full list of flavors please check out the [release page](https://github.com/syself/cluster-api-provider-hetzner/releases).

To create a workload cluster we need to do some preparation:

- Set up the projects and credentials in HCloud.
- Create the management/bootstrap cluster.
- Export variables needed for cluster-template.
- Create a secret with the credentials.

For more information about this step, please see [here](./preparation.md)

## Generate your cluster.yaml

The clusterctl generate cluster command returns a YAML template for creating a workload cluster.
It generates a YAML file named `my-cluster.yaml` with a predefined list of Cluster API objects (`Cluster`, `Machines`, `MachineDeployments`, etc.) to be deployed in the current namespace. With the `--target-namespace` flag, you can specify a different target namespace.
See also `clusterctl generate cluster --help`.

```shell
clusterctl generate cluster my-cluster --kubernetes-version v1.23.4 --control-plane-machine-count=3 --worker-machine-count=3  > my-cluster.yaml
```

You can also use different flavors, e.g. to create a cluster with private network:

```shell
clusterctl generate cluster my-cluster --kubernetes-version v1.23.4 --control-plane-machine-count=3 --worker-machine-count=3  --flavor hcloud-network > my-cluster.yaml
```

All pre-configured flavors can be found on the [release page](https://github.com/syself/cluster-api-provider-hetzner/releases). The cluster-templates start with `cluster-template-`. The flavor name is the suffix.

## Hetzner Dedicated / Bare Metal Server

If you want to create a cluster with bare metal servers, you need to additionally set up the robot credentials in the preparation step. As described in the [reference](/docs/reference/hetzner-bare-metal-machine-template.md), you need to manually buy bare metal servers before-hand.
Then you need to create a `HetznerBareMetalHost` object for each bare metal server that you bought and specify its server ID in the specs. If you already know the WWN of the storage device you want to choose for booting, then specify it in `rootDeviceHints`. If not, then you can wait for the host to start provisioning. It will show you an error that you did not specify the WWN. If you reach this point, have a look at the status of `HetznerBareMetalHost`. There you find `hardwareDetails`, in which you can see a list of all relevant storage devices as well as their properties. You can just copy+paste the WWN of your favorite storage device into `rootDeviceHints`.  ....

## Apply the workload cluster

```shell
kubectl apply -f my-cluster.yaml
```

### Accessing the workload cluster

The cluster will now start provisioning. You can check status with:

```shell
kubectl get cluster
```

You can also view cluster and its resources at a glance by running:

```shell
clusterctl describe cluster my-cluster
```

To verify the first control plane is up, use this command:

```shell
kubectl get kubeadmcontrolplane
```

> The control plane won’t be `ready` until we install a CNI in the next step.

After the first control plane node is up and running, we can retrieve the kubeconfig of the workload cluster:

```shell
export CAPH_WORKER_CLUSTER_KUBECONFIG=/tmp/workload-kubeconfig
clusterctl get kubeconfig my-cluster > $CAPH_WORKER_CLUSTER_KUBECONFIG
```

## Deploy a CNI solution

```shell
helm repo add cilium https://helm.cilium.io/

KUBECONFIG=$CAPH_WORKER_CLUSTER_KUBECONFIG helm upgrade --install cilium cilium/cilium --version 1.10.5 \
--namespace kube-system \
-f templates/cilium/cilium.yaml
```

You can, of course, also install an alternative CNI, e.g. calico.

> There is a bug in ubuntu that requires the older version of cilium for this quickstart guide.

## Deploy the CCM

### Deploy HCloud Cloud Controller Manager - *hcloud only*

This make command will install the CCM in your workload cluster.

`make install-manifests-ccm-hcloud PRIVATE_NETWORK=false`

```shell
# For a cluster without private network:
helm repo add syself https://charts.syself.com
helm repo update syself

KUBECONFIG=$CAPH_WORKER_CLUSTER_KUBECONFIG helm upgrade --install ccm syself/ccm-hcloud --version 1.0.10 \
	--namespace kube-system \
	--set secret.name=hetzner \
	--set secret.tokenKeyName=hcloud \
	--set privateNetwork.enabled=false
```

### Deploy Hetzner Cloud Controller Manager

> This requires a secret containing access credentials to both Hetzner Robot and HCloud

`make install-manifests-ccm-hetzner PRIVATE_NETWORK=false`

```shell
helm repo add syself https://charts.syself.com
helm repo update syself

KUBECONFIG=$CAPH_WORKER_CLUSTER_KUBECONFIG)helm upgrade --install ccm syself/ccm-hetzner --version 1.1.2 \
--namespace kube-system \
--set image.tag=latest \
--set privateNetwork.enabled=false
```

## Deploy the CSI (optional)

```shell
cat << EOF > csi-values.yaml
storageClasses:
- name: hcloud-volumes
  defaultStorageClass: true
  reclaimPolicy: Retain
EOF

KUBECONFIG=$CAPH_WORKER_CLUSTER_KUBECONFIG helm upgrade --install csi syself/csi-hcloud --version 0.2.0 \
--namespace kube-system -f csi-values.yaml
```

## Clean Up

Delete workload cluster.

```shell
kubectl delete cluster my-cluster
```

> **IMPORTANT**: In order to ensure a proper clean-up of your infrastructure, you must always delete the cluster object. Deleting the entire cluster template with kubectl delete -f capi-quickstart.yaml might lead to pending resources that have to be cleaned up manually.

Delete management cluster with

```shell
kind delete cluster
```

## Next Steps

### Switch to the workload cluster

```shell
export KUBECONFIG=/tmp/workload-kubeconfig
```

### Moving components

To move the cluster-api objects from your bootstrap cluster to the new management cluster, you need first to install the Cluster API controllers. To install the components with the latest version, please run:

```shell
clusterctl init --core cluster-api --bootstrap kubeadm --control-plane kubeadm --infrastructure hetzner

```

If you want a specific version, then use the flag `--infrastructure hetzner:vX.X.X`.

Now you can switch back to the management cluster, for example with

```shell
export KUBECONFIG=~/.kube/config
```

You can now move the objects into the new cluster by using:

```shell
clusterctl move --to-kubeconfig $CAPH_WORKER_CLUSTER_KUBECONFIG
```

Clusterctl Flags:

| Flag                      | Description                                                                                                                   |
| ------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| _--namespace_             | The namespace where the workload cluster is hosted. If unspecified, the current context's namespace is used. |
| _--kubeconfig_            | Path to the kubeconfig file for the source management cluster. If unspecified, default discovery rules apply. |
| _--kubeconfig-context_    | Context to be used within the kubeconfig file for the source management cluster. If empty, current context will be used. |
| _--to-kubeconfig_         | Path to the kubeconfig file to use for the destination management cluster. |
| _--to-kubeconfig-context_ | Context to be used within the kubeconfig file for the destination management cluster. If empty, current context will be used. |
