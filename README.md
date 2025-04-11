## `garage-cluster`

*Get started with Garage in Kubernetes!*

### Overview

Extremely opinionated tools to deploy Garage to Kubernetes and manage it.

Actually, these are _my_ tools.  You're welcome to borrow them.

### Creating a Garage Cluster

To create a Garage cluster, you must do the following:

* prepare your nodes:

  * for maximum flexibility, all nodes should participate even if they don't
    have storage (for load balancing, localhost endpoint for Garage, whatever).
    this repo doesn't setup anything to make use of this flexibility but it may
    or you may.  you'll want to remove capacity (`-c 10G`) from nodes which
    won't store and add `-g` for gateway instead.

  * nodes which will have storage should have that storage added via LVM so
    that you can add storage in the future.  This DaemonSet-based Garage
    cluster does _not_ account for multiple HDDs even though they're supported
    by Garage.  To expand your storage, you'll need LVM.

  * partition partitions, format them, and mount them

        NVME_VG=nvme-vg
	HDD_VG=hdd-vg
        lvcreate -L 1G $NVME_VG -n garage-meta
	mkfs.btrfs /dev/$NVME_VG/garage-meta
	lvcreate -L 10G $HDD_VG -n garage-data
	mkfs.xfs /dev/$HDD_VG/garage-data
	echo "/dev/$NVME_VG/garage-meta /var/lib/garage/meta btrfs defaults 0 2" >> /etc/fstab
	echo "/dev/$HDD_VG/garage-data /var/lib/garage/data xfs defaults 0 2" >> /etc/fstab
	mount /var/lib/garage/{meta,data}

* modify the contents of `cm.yaml`:

  * The daemonset will translate these entries into a `garage.toml` file for
    you.  to place something into a section, prefix it with a period.

  * Settings to check: `replication_factor`, `s3_api.root_domain`,
    `s3_web.root_domain`

  * You may also include bootstrap peers here if you like. if you do not,
    you'll need to connect them via the tools which may be more convenient
    anyway.  don't worry about this if you're not sure what to do yet.

* Modify the contents of `ingress.yaml` to match your controller's requirements
  and/or your hostname and prefix.

* Apply the manifests:

    kubectl apply -f .
    echo "Ctrl-C to exit"
    kubectl get ds -w

* Bootstrap the nodes once they've started:

    ./tools connect

* Alias `garage` to use your kubernetes pods:

    ./tools env >> ~/.bashrc

* Generate a sample layout to start:

If you have the node feature discovery plugin, this may expose likely capacity
values.  If you do not have the plugin, it will default to 10G for every node.

You should almost certainly edit the resulting commands before applying them to
your cluster.  `$PWD/tools` is used to make the resulting script relocatable.

    $PWD/tools generate-layout -z my-zone > run-layout.sh
    $EDITOR run-layout.sh
    ./run-layout.sh
    garage layout show
    read -p 'everything look good?  press enter to continue or ctrl-c to abort'
    garage layout apply


