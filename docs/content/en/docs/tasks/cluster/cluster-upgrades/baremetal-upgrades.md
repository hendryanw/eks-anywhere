---
title: "Upgrade Bare Metal cluster"
linkTitle: "Upgrade Bare Metal cluster"
weight: 20
date: 2017-01-05
description: >
  How to perform a cluster upgrade for Bare Metal cluster
---
EKS Anywhere provides the command `upgrade`, which allows you to `upgrade` various aspects of your EKS Anywhere cluster.
When you run `eksctl anywhere upgrade cluster -f ./cluster.yaml`, EKS Anywhere runs a set of preflight checks to ensure your cluster is ready to be upgraded.
EKS Anywhere then performs the upgrade, modifying your cluster to match the updated specification. 
The upgrade command also upgrades core components of EKS Anywhere and lets the user enjoy the latest features, bug fixes and security patches.

>**_NOTE:_** Currently only Minor Version Upgrades are support for Bare Metal clusters. No other aspects of the cluster upgrades are currently supported.
>


### Minor Version Upgrades

Kubernetes has minor releases [three times per year](https://kubernetes.io/releases/release/) and EKS Distro follows a similar cadence.
EKS Anywhere will add support for new EKS Distro releases as they are released, and you are advised to upgrade your cluster when possible.

Cluster upgrades are not handled automatically and require administrator action to modify the cluster specification and perform an upgrade.
You are advised to upgrade your clusters in development environments first and verify your workloads and controllers are compatible with the new version.

Cluster upgrades are performed using a rolling upgrade process (similar to Kubernetes Deployments).
Upgrades can only happen one minor version at a time (e.g. `1.24` -> `1.25`).
Control plane components will be upgraded before worker nodes.

### Prerequisites

This type of upgrade requires you to have one spare hardware server for control plane upgrade and one for each worker node group upgrade.
The spare hardware server is provisioned with the new version and then an old server is deprovisioned. The deprovisioned server is then reprovisioned with
the new version while another old server is deprovisioned. This happens one at a time until all the control plane components have been upgraded, followed by
worker node upgrades.

### Core component upgrades

EKS Anywhere `upgrade` also supports upgrading the following core components:

* Core CAPI
* CAPI providers
* Cilium CNI plugin
* Cert-manager
* Etcdadm CAPI provider
* EKS Anywhere controllers and CRDs
* GitOps controllers (Flux) - this is an optional component, will be upgraded only if specified

The latest versions of these core EKS Anywhere components are embedded into a bundles manifest that the CLI uses to fetch the latest versions 
and image builds needed for each component upgrade. 
The command detects both component version changes and new builds of the same versioned component.
If there is a new Kubernetes version that is going to get rolled out, the core components get upgraded before the Kubernetes
version. 
Irrespective of a Kubernetes version change, the upgrade command will always upgrade the internal EKS
Anywhere components mentioned above to their latest available versions. All upgrade changes are backwards compatible.

### Check upgrade components
Before you perform an upgrade, check the current and new versions of components that are ready to upgrade by typing:

```bash
eksctl anywhere upgrade plan cluster -f cluster.yaml
```

The output should appear similar to the following:

```
Worker node group name not specified. Defaulting name to md-0.
Warning: The recommended number of control plane nodes is 3 or 5
Worker node group name not specified. Defaulting name to md-0.
Checking new release availability...
NAME                     CURRENT VERSION                 NEXT VERSION
EKS-A                    v0.0.0-dev+build.1000+9886ba8   v0.0.0-dev+build.1105+46598cb
cluster-api              v1.0.2+e8c48f5                  v1.0.2+1274316
kubeadm                  v1.0.2+92c6d7e                  v1.0.2+aa1a03a
vsphere                  v1.0.1+efb002c                  v1.0.1+ef26ac1
kubadm                   v1.0.2+f002eae                  v1.0.2+f443dcf
etcdadm-bootstrap        v1.0.2-rc3+54dcc82              v1.0.0-rc3+df07114
etcdadm-controller       v1.0.2-rc3+a817792              v1.0.0-rc3+a310516
```
To format the output in json, add `-o json` to the end of the command line.

### Check hardware availability

Next, you must ensure you have enough available hardware for the rolling upgrade operation to function. This type of upgrade requires you to have one spare hardware server for control plane upgrade and one for each worker node group upgrade. Check [prerequisites]({{< relref "baremetal-upgrades/#prerequisites" >}}) for more information.
Available hardware could have been fed to the cluster as extra hardware during a prior create command, or could be fed to the cluster during the upgrade process by providing the hardware CSV file to the [upgrade cluster command]({{< relref "baremetal-upgrades/#upgrade-cluster-command" >}}).

To check if you have enough available hardware for rolling upgrade, you can use the `kubectl` command below to check if there are hardware objects with the selector labels corresponding to the controlplane/worker node group and without the `ownerName` label. 

```bash
kubectl get hardware -n eksa-system --show-labels
```

For example, if you want to perform upgrade on a cluster with one worker node group with selector label `type=worker-group-1`, then you must have an additional hardware object in your cluster with the label `type=controlplane` (for control plane upgrade) and one with `type=worker-group-1` (for worker node group upgrade) that doesn't have the `ownerName` label. 

In the command shown below, `eksa-worker2` matches the selector label and it doesn't have the `ownerName` label. Thus, it can be used to perform rolling upgrade of `worker-group-1`. Similarly, `eksa-controlplane-spare` will be used for rolling upgrade of control plane.

```bash
kubectl get hardware -n eksa-system --show-labels 
NAME                STATE       LABELS
eksa-controlplane               type=controlplane,v1alpha1.tinkerbell.org/ownerName=abhnvp-control-plane-template-1656427179688-9rm5f,v1alpha1.tinkerbell.org/ownerNamespace=eksa-system
eksa-controlplane-spare         type=controlplane
eksa-worker1                    type=worker-group-1,v1alpha1.tinkerbell.org/ownerName=abhnvp-md-0-1656427179689-9fqnx,v1alpha1.tinkerbell.org/ownerNamespace=eksa-system
eksa-worker2                    type=worker-group-1
```

If you don't have any available hardware that match this requirement in the cluster, you can [setup a new hardware CSV]({{< relref "../../../reference/baremetal/bare-preparation/#prepare-hardware-inventory" >}}). You can feed this hardware inventory file during the [upgrade cluster command]({{< relref "baremetal-upgrades/#upgrade-cluster-command" >}}).

### Performing a cluster upgrade

To perform a cluster upgrade you can modify your cluster specification `kubernetesVersion` field to the desired version.

As an example, to upgrade a cluster with version 1.24 to 1.25 you would change your spec as follows:

```
apiVersion: anywhere.eks.amazonaws.com/v1alpha1
kind: Cluster
metadata:
  name: dev
spec:
  controlPlaneConfiguration:
    count: 1
    endpoint:
      host: "198.18.99.49"
    machineGroupRef:
      kind: TinkerbellMachineConfig
      name: dev
      ...
  kubernetesVersion: "1.25"
      ...
```

>**_NOTE:_** If you have a custom machine image for your nodes in your cluster config yaml you may also need to update 
your [`TinkerbellDatacenterConfig`]({{< relref "../../../reference/clusterspec/baremetal/#tinkerbelldatacenterconfig-fields" >}}) with a new [`osImageURL`]({{< relref "../../../reference/clusterspec/baremetal/#osimageurl" >}}).

and then you will run the [upgrade cluster command]({{< relref "baremetal-upgrades/#upgrade-cluster-command" >}}).


#### Upgrade cluster command

##### With hardware CSV

```
eksctl anywhere upgrade cluster -f cluster.yaml --hardware-csv <hardware.csv>
```

##### Without hardware CSV

```
eksctl anywhere upgrade cluster -f cluster.yaml
```

This will upgrade the cluster specification (if specified), upgrade the core components to the latest available versions and apply the changes using the provisioner controllers.


#### Output

Example output:

```
✅ control plane ready
✅ worker nodes ready
✅ nodes ready
✅ cluster CRDs ready
✅ cluster object present on workload cluster
✅ upgrade cluster kubernetes version increment
✅ validate immutable fields
🎉 all cluster upgrade preflight validations passed
Performing provider setup and validations
Pausing EKS-A cluster controller reconcile
Pausing Flux kustomization
GitOps field not specified, pause flux kustomization skipped
Creating bootstrap cluster
Installing cluster-api providers on bootstrap cluster
Moving cluster management from workload to bootstrap cluster
Upgrading workload cluster
Moving cluster management from bootstrap to workload cluster
Applying new EKS-A cluster resource; resuming reconcile
Resuming EKS-A controller reconciliation
Updating Git Repo with new EKS-A cluster spec
GitOps field not specified, update git repo skipped
Forcing reconcile Git repo with latest commit
GitOps not configured, force reconcile flux git repo skipped
Resuming Flux kustomization
GitOps field not specified, resume flux kustomization skipped
```

During the upgrade process, EKS Anywhere pauses the cluster controller reconciliation by adding the paused annotation `anywhere.eks.amazonaws.com/paused: true` to the EKS Anywhere cluster, provider datacenterconfig and machineconfig resources, before the components upgrade. After upgrade completes, the annotations are removed so that the cluster controller resumes reconciling the cluster.

Though not recommended, you can manually pause the EKS Anywhere cluster controller reconciliation to perform extended maintenance work or interact with Cluster API objects directly. To do it, you can add the paused annotation to the cluster resource:

```bash
kubectl annotate clusters.anywhere.eks.amazonaws.com ${CLUSTER_NAME} -n ${CLUSTER_NAMESPACE} anywhere.eks.amazonaws.com/paused=true
```

After finishing the task, make sure you resume the cluster reconciliation by removing the paused annotation, so that EKS Anywhere cluster controller can continue working as expected.

```bash
kubectl annotate clusters.anywhere.eks.amazonaws.com ${CLUSTER_NAME} -n ${CLUSTER_NAMESPACE} anywhere.eks.amazonaws.com/paused-
```

### Upgradeable cluster attributes

`Cluster`:
- `kubernetesVersion`

### Advanced configuration for rolling upgrade

EKS Anywhere allows an optional configuration to customize the behavior of upgrades. 

It allows the specification of 
Two parameters that control the desired behavior of rolling upgrades: 
* maxSurge - The maximum number of machines that can be scheduled above the desired number of machines. When not specified, the current CAPI default of 1 is used.
* maxUnavailable - The maximum number of machines that can be unavailable during the upgrade. When not specified, the current CAPI default of 0 is used.

Example configuration:

```bash
upgradeRolloutStrategy:
  rollingUpgradeParams:
    maxSurge: 1
    maxUnavailable: 0    # only configurable for worker nodes
```

'upgradeRolloutStrategy' configuration can be specified separately for control plane and for each worker node group. This template contains an example for control plane under the 'controlPlaneConfiguration' section and for worker node group under 'workerNodeGroupConfigurations':

```bash
apiVersion: anywhere.eks.amazonaws.com/v1alpha1
kind: Cluster
metadata:
  name: my-cluster-name
spec:
  clusterNetwork:
    cniConfig:
      cilium: {}
    pods:
      cidrBlocks:
      - 192.168.0.0/16
    services:
      cidrBlocks:
      - 10.96.0.0/12
  controlPlaneConfiguration:
    count: 1
    endpoint:
      host: "10.61.248.209"
    machineGroupRef:
      kind: TinkerbellMachineConfig
      name: my-cluster-name-cp
  upgradeRolloutStrategy:
      rollingUpgradeParams:
        maxSurge: 1
  datacenterRef:
    kind: TinkerbellDatacenterConfig
    name: my-cluster-name
  kubernetesVersion: "1.25"
  managementCluster:
    name: my-cluster-name 
  workerNodeGroupConfigurations:
  - count: 2
    machineGroupRef:
      kind: TinkerbellMachineConfig
      name: my-cluster-name 
    name: md-0
  upgradeRolloutStrategy:
      rollingUpgradeParams:
        maxSurge: 1
        maxUnavailable: 0

---
...
```

#### upgradeRolloutStrategy
Configuration parameters for upgrade strategy.

#### rollingUpgradeParams
This field accepts configuration parameters for customizing rolling upgrade behavior.

#### maxSurge
Default: 1

This can not be 0 if maxUnavailable is 0.

The maximum number of machines that can be scheduled above the desired number of machines. 

Example: When this is set to n, the new worker node group can be scaled up immediately by n when the rolling upgrade starts. Total number of machines in the cluster (old + new) never exceeds (desired number of machines + n). Once scale down happens and old machines are brought down, the new worker node group can be scaled up further ensuring that the total number of machines running at any time does not exceed the desired number of machines + n.

#### maxUnavailable
Default: 0

This can not be 0 if MaxSurge is 0.

The maximum number of machines that can be unavailable during the upgrade.

Example: When this is set to n, the old worker node group can be scaled down by n machines immediately when the rolling upgrade starts. Once new machines are ready, old worker node group can be scaled down further, followed by scaling up the new worker node group, ensuring that the total number of machines unavailable at all times during the upgrade never falls below n.

### Rolling upgrades with no additional hardware

When maxSurge is set to 0 and maxUnavailable is set to 1, it allows for a rolling upgrade without need for additional hardware. Use this configuration if your workloads can tolerate node unavailability.

>**_NOTE:_** This could ONLY be used if unavailability of a maximum of 1 node is acceptable. For single node clusters, an additional temporary machine is a must. Alternatively, you may recreate the single node cluster for upgrading and handle data recovery manually.

With this kind of configuration, the rolling upgrade will proceed node by node, deprovision and delete a node fully before re-provisioning it with upgraded version, and re-join it to the cluster. This means that any point during the course of the rolling upgrade, there could be one unavailable node.


### Troubleshooting

Attempting to upgrade a cluster with more than 1 minor release will result in receiving the following error.

```
✅ validate immutable fields
❌ validation failed    {"validation": "Upgrade preflight validations", "error": "validation failed with 1 errors: WARNING: version difference between upgrade version (1.21) and server version (1.19) do not meet the supported version increment of +1", "remediation": ""}
Error: failed to upgrade cluster: validations failed
```

For more errors you can see the [troubleshooting section]({{< relref "../../troubleshoot" >}}).
