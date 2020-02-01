## Assumptions

* A management cluster serves as a "hub" from which bare metal clusters are
  created and managed.
* A management application on the hub can provision and lifecycle bare metal
  clusters.
* The hub cluster needs to maintain an inventory of physical hosts that can be
  used by the bare metal clusters it manages.

## API

The BareMetalAsset CR (example shown below) contains the basic information
needed to create a usable BareMetalHost. The BareMetalAsset will be used on a
hub cluster to track an inventory of physical hosts that are available to be
assigned to a managed cluster and provisioned.

A temporary API namespace, “midas.io”, is in use until we determine what API
namespace will be used by multicluster-related projects.

### Example

```
---
apiVersion: midas.io/v1alpha1
kind: BareMetalAsset
metadata:
  name: baremetalasset-worker-0
spec:
  bmc:
    address: ipmi://192.168.122.1:6233
    credentialsName: worker-0-bmc-secret
  bootMACAddress: "00:1B:44:11:3A:B7"
  hardwareProfile: "hardwareProfile"
  role: "worker"
  clusterName: "cluster0"
---
apiVersion: v1
kind: Secret
metadata:
  name: worker-0-bmc-secret
type: Opaque
data:
  username: YWRtaW4=
  password: cGFzc3dvcmQ=
```

### Usage

An inventory system will create BareMetalAsset resources in a hub cluster.
Each BareMetalAsset will have a corresponding Secret that contains BMC
credentials.

To assign a BareMetalAsset to a cluster, populate the `ClusterName` field of the
Spec.

To remove a BareMetalAsset from a cluster, set its `ClusterName` to `""`.

### Labels

When a BareMetalAsset gets created, the `Spec.Role` is used as the value for a
label with key `midas.io/role`. This enables a cluster management application
or user to query for inventory that is intended for use as a particular role.

When a BareMetalAsset gets assigned to a cluster, the `Spec.ClusterName` is
used as the value for a label with key `midas.io/cluster`. This enables a
cluster management application or user to query for inventory that is
associated with a particular cluster.

### Status

The status will expose these conditions:
* Has the required credentials secret been found?
* Is creation of the BareMetalHost on the managed cluster in progress?
* Has the BareMetalHost been created on the managed cluster?

## Controller

The inventory controller that watches BareMetalAssets will likely become a part
of the management application that coordinates the lifecycle of clusters.

## User Stories

### Creation

A real piece of hardware is tracked in a user configuration management database (CMDB) or similar system. It is
physically installed in a data center. Some tool or automation uses a record in
the CMDB to create a BareMetalAsset and corresponding Secret in a hub cluster.

### Addition to Existing Managed Cluster

A controller on the hub cluster watches BareMetalAsset records. When one
appears, it uses `Spec.ClusterName` to determine if it has been assigned to a
Cluster. It then uses either a hive SyncSet or other mechanism (decision on
that is yet to be made pending specific questions) to ensure that a
corresponding BareMetalHost and Secret exist on the managed cluster.

The MachineSet on the managed cluster will be automatically scaled up by the
cluster-api-provider-baremetal running on that cluster.

### Provisioning a New Cluster

The user creates a Cluster record (likely through automation) that includes a
label selector. When provisioning of that cluster is initiated, the cluster
management application looks for matching (based on the label selector)
BareMetalAsset records whose role is “master” and uses them to provision a new
cluster. This includes creating the install-config.yaml file that gets passed
to the openshift installer. It also sets the ClusterName in the spec of each
inventory record.

Once the cluster is running, an inventory controller on the hub will begin
reconciling the state of the masters’ BareMetalHost resources on the managed
cluster using the same mechanism as described above in “Addition to Existing
Managed Cluster”.

Once the cluster has been provisioned, additional inventory designated as
workers will be added in the manner described in the “Addition to Existing
Managed Cluster” section.

### Updating Host Information

At times some information about a host may need to be updated. For example,
if the BMC credentials were incorrect, they would need to be modified.

The update in the user’s inventory gets processed by the inventory-add tool.
That tool identifies the existing resource and updates it accordingly.

The controller that delivers inventory to managed clusters will then use either
a Subscription or SyncSet to apply changes to the BareMetalHost and its Secret
on the managed cluster.

### Removal from Cluster

When a BareMetalAsset resource that is assigned to a cluster has its
`ClusterName` field set to `""`, a controller will ensure that the
BareMetalHost on the managed cluster will be deprovisioned and deleted
according to the documented steps.

The MachineSet on the managed cluster will be automatically scaled down by the
cluster-api-provider.
