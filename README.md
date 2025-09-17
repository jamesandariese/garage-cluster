## `garage-cluster`

*Get started with Garage in Kubernetes!*

### Overview

Extremely opinionated tools to deploy Garage to Kubernetes and manage it.

Actually, these are _my_ tools.  You're welcome to borrow them.

### Creating a Garage Cluster

To create a Garage cluster, you must do the following:

* prepare your nodes:

  * Use all nodes

    For maximum flexibility, all nodes should participate even if they don't
    have storage (for load balancing, localhost endpoint for Garage, whatever).
    This repo doesn't setup anything to make use of this flexibility but it may
    or you may.  You'll want to remove capacity (`-c 10G`) from nodes which
    won't store and add `-g` for gateway instead.

  * Use LVM (even though Garage doesn't require it -- this setup does)
  
    Nodes which will have storage should have that storage added via LVM so
    that you can add storage in the future.  This DaemonSet-based Garage
    cluster does _not_ account for multiple HDDs even though they're supported
    by Garage.  To expand your storage, you'll need LVM.

  * Ensure you have Node Feature Discovery running.
  
    You might have installed this
    with the NVIDIA device plugin if you happen to use that.  If not, you can
    read more at
    [Kubernetes SIG for NFD](https://github.com/kubernetes-sigs/node-feature-discovery)

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

* copy and modify `example/kustomization.yaml`:

  * The daemonset will translate the ConfigMap entries into a `garage.toml` file for
    you.  Examples for the suggested settings are provided.

  * Settings to check:
    - `replication_factor`
    - `consistency_mode`
    - `s3_api.root_domain`
    - `s3_web.root_domain`

  * You may also include bootstrap peers here if you like.  If you do not,
    you'll need to connect them via the tools which may be more convenient
    anyway.  Don't worry about this if you're not sure what to do yet.

  * Also modify the Ingresses following the examples to change them to your own host
    instead of `example.net` and `example.com`.

* Apply the manifests:

      ./tools generate-secrets
      kubectl apply -k .
      echo "Ctrl-C to exit"
      kubectl get ds -n garage -w

* Wait for Node Feature Discovery to enumerate your storage:

      kubectl get node -o yaml | grep -q x-garage-cluster && echo ready || echo not ready

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


