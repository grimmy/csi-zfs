# csi-zfs

This is a conceptual csi driver for kubernetes using zfs pools on the nodes.
There is nothing to see here yet as this is just a dumping ground for ideas
before any code is written.

# Why?

We run local Kubernetes clusters and volumes are an issue. You can setup
something like Ceph using Rook, or Mayastor, but these all use a fair amount
of resources on your nodes and are very complicated to administer.

So the idea of ZFS pools on each node and using ZFS send/receive to move
volumes around would help a lot with that.

# Background

[This post](https://arslan.io/2018/06/21/how-to-write-a-container-storage-interface-csi-plugin/)
has a lot of good information on building a CSI driver.

The official CSI driver documentation is
[here](https://kubernetes-csi.github.io/docs/).

# Proposal

> Note: most of this proposal was written before reading the above mentioned
article and will surely need to be updated.

To bootstrap this, creation of the ZFS zpools will be left to the user. This
will allow them to configure the ZFS zpool however they want and lets us focus
on the bigger issues up front.

Right now for labels we will `csi-zfs.github.io` as our namespace.

Configuration will, right now, look at Kubernetes node labels for the ZFS
dataset to use as a parent. This would be defined in the
`csi-zfs.github.io/dataset` label. ZFS datasets will be named after the
Kubernetes PV's name under that ZFS parent dataset.

For example, if your Kubernetes PV has a name of
`pvc-f0aff9a7-d11e-474f-b902-8c1549d743a6`, and your ZFS dataset is `tank/k8s`
then the ZFS dataset for the Kubernetes PV would be named
`tank/k8s/pvc-f0aff9a7-d11e-474f-b902-8c1549d743a6`.

There will also be a headless service configured so that the CSI driver
instance can talk to each other to figure out who has the Kubernetes PV and the
ZFS dataset for it to initiate a ZFS send and recv.

When a volume mount is necessary, the daemonset running on the target node
will use the headless service to determine which node the PV currently resides
on and to get the dataset of the PV.

It will then initiate a ZFS send/recv to move the entire dataset including
snapshots and properties by using `--replicate`. Eventually we will add resume
support, but there's a lot of corner cases there that will causes issues at the
start. The transfer will be wired up through the service and ran over gRPC.
While bootstrapping this will be insecure but we will eventually add encryption
support.

For Kubernetes volume snapshots we would just wire that up to the builtin ZFS
snapshots. Resource limits would be implemented via ZFS dataset quotas, and ZFS
reservations to make sure we have the space available for the PV to make sure
Kubernetes pods only get scheduled on nodes with enough space.

Compression should probably be on by default, but perhaps we could allow
configuration via the PVC. That said, we could also create a properties
section in the PVC that would be used to enforce properties.

This will be your typical Kubernetes CSI driver written in Golang pulling in
Kubernetes client as a dependency. I'm not sure if we need/want to use etcd for
anything, but it might be nice to use to "lock" PVs as we're moving them
around.

Securing the gRPC endpoints will be difficult, but we're going to avoid this
during initial development and opt for Kubernetes network policies that will
only allow the service to talk to itself in it's own namespace.

We should also add a Prometheus listen address on a separate port that the
cluster administrator can configure for themselves. Some metrics would include
number of volumes, snapshots, IO metrics for each volume, as well as IO for
ZFS send and recv. These are just ideas and we should reference other CSI
implementations to get a better idea of what to track.

# Errata

We'd have to figure out UNIX file permissions somehow. We could use the
security context of the pod/container, but that might get weird especially if
we decide to "correct" permissions recursively. We could make these options in
the CSI specific options for the PVs.

We need to deny the sharenfs and sharesmb properties.

# Timeline

No idea on a timeline, but this is something I've been thinking a lot about
lately and wanted to get the ideas down.

# Other Ideas

Add support for Zvols. This would allow custom filesystems while still
benefiting from snapshots. This _might_ also allow
[sendfile](https://man.freebsd.org/cgi/man.cgi?sourceid=opensearch&manpath=freebsd-release-ports&query=sendfile)
to work which doesn't normally work on ZFS.

Dataset encryption via Kubernetes secrets which would also require raw sends.

Bandwidth limitations of transfers.

Allow syncing to an external pool. This will allow for backups that are not
part of the cluster. Not sure how to reseed this, but a ZFS send to the a
single node would get it back, but you would still need the Kubernetes PV,
which would create a ZFS dataset that you could overwrite with a force send.

Check for node compatibility. This could be done in the service and have it
set a node taint maybe?

# Questions

Can a CSI driver block scheduling on a node taint. If not, deployments will
just get stuck, which isn't ideal, but is fine.

Can a CSI driver add a taint toleration to a pod? If this is true, we could add
a node toleration to the pods that the CSI driver would manage on the node
itself, adding the taint when the CSI driver is up and running and removing it
on shutdown.
