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

### Name

The BareMetalHost on the managed cluster will get the same name as its
corresponding BareMetalAsset on the hub. In cases where the BareMetalHost is
being created on the managed cluster during installation, or any other case
where it is not created by the inventory controller, it is important that the
name be set to match the BareMetalAsset.

### Role

The Role can be either "worker", "master", or "". When a management application
is looking for available BareMetalAssets to add to a cluster, it will know
which role it is trying to fill. It should prefer to use BareMetalAssets whose
role equals the desired role, and if none are found it can use BareMetalAssets
with no role.

A BareMetalAsset with a specified role should not be used to fill a different
role.

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

## Alternative APIs

Other options were considered for the API.

### BareMetalHost

We considered using a sparse BareMetalHost on the hub as the inventory data
structure. These are the main reasons not to do that:

* The BMH API includes many features that wouldn't be supported on the hub,
  such as provisioning an image, turning power on and off, etc. It's awkward to
  expose those APIs without implementing the behavior.
* The BMH status doesn't make sense for inventory.
* One hub cluster may need to eventually be able to populate multiple versions
  of the BMH API onto clusters that are at different versions. Tracking the BMH
  CRD on the hub vs. what versions are on each cluster could get complex.
* Then consider that the hub itself may have real BMHs in use as its own Nodes
  with its own baremetal-operator. Not only would that make the versioning
  matrix more complex; it means the same API would have vastly different
  behaviors on the same cluster depending on what namespace a resource was in.
  It also means the inventory controller would have to just work with whatever
  BMH CRD version was in use by its local baremetal-operator.

### Secret

A BareMetalAsset does not contain a large number of fields in its `Spec`.
Rather than have both a BareMetalAsset and a Secret to hold BMC credentials, we
could put all of the inventory information into the Secret. This would halve
the number of keys that need to be stored in etcd.

Reasons not to do this:
* There would be no clear opportunity to represent status.
* The data model would not be concretetely structured, making validation
  more difficult.

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
