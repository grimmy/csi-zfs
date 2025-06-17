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

# Proposal

To bootstrap this, creation of the zpools will be left to the user. This will
allow them to configure the zpool however they want and lets us focus on the
bigger issues up front.

The csi driver would be implemented by a daemon set. Configuration will, right
now, look at node labels for the root dataset to use.

PVCs would then be created as `<root data set>/<namespace>/<pvcname>`.

There will also be a headless service configured so that the csi-driver
instance can talk to each other to figure out who has the PV and how and
dataset for it to initiate a ZFS send/recv.

When a volume mount is necessary, the daemonset running on the target node
will use the headless service to determine which node the PV currently resides
on and to get the dataset of the PV.

It will then initiate a ZFS send/recv to move the entire dataset including
snapshots and properties by using `--replicate` and `-s` to resume an
interupted transfer. This will most likely use mbuffer, but we could just run
it through the driver as well. Regardless, we should allow some methods for
encryption.

For snapshots we would just wire that up to the builtin ZFS snapshots.
Resource limits would be implement via ZFS dataset quotas, and ZFS
reservations to make sure we have the space available for the PV.

Compression should probably be on by default, but perhaps we could allow
configuration via the PVC. That said, we could also create a properties
section in the PVC that would be used to enforce properties.

This will be your typical csi-driver written in golang linking pulling in
kubernetes client as a dependency. I'm not sure if we need/want to use etcd
for anyting, but it might be nice to use to "lock" PVs as we're moving them
around.

# Errata

We'd have to figure out permissions somehow. We could use the security context
of the pod/container, but that might get weird.

# Timeline

No idea on a timeline, but this is something I've been thinking a lot about
lately and wanted to get the ideas down.

