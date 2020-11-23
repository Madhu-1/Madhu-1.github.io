---
layout: post
title: Play with RBD Async mirroring with minikube
comments: true
---

This doc assumes that you already have a minikube [installed](https://minikube.sigs.k8s.io/docs/start/), minikube is configured to use `kvm2` to create a virtual machine. You can refer [kvm2](https://minikube.sigs.k8s.io/docs/drivers/kvm2/) to install the required prerequisites.

> Install [`kubectl`](https://kubernetes.io/docs/tasks/tools/install-kubectl/) on the machine where you want to create kubernetes clusters and from where you want to access kubernetes
To Play with RBD async mirroring we need to have two kubernetes/OCP clusters as it's for just let's use minikube to create 2 kubernetes clusters.

## Create Kubernetes cluster

### Create kubernetes cluster1

```bash=
$minikube start --force --memory="4096" --cpus="2" -b kubeadm --kubernetes-version="v1.19.2" --driver="kvm2" --feature-gates="BlockVolume=true,CSIBlockVolume=true,VolumeSnapshotDataSource=true,ExpandCSIVolumes=true" --profile="cluster1"
😄  [cluster1] minikube v1.14.0 on Fedora 32
❗  minikube skips various validations when --force is supplied; this may lead to unexpected behavior
✨  Using the kvm2 driver based on user configuration
🛑  The "kvm2" driver should not be used with root privileges.
💡  If you are running minikube within a VM, consider using --driver=none:
📘    https://minikube.sigs.k8s.io/docs/reference/drivers/none/
👍  Starting control plane node cluster1 in cluster cluster1
🔥  Creating kvm2 VM (CPUs=2, Memory=4096MB, Disk=20000MB) ...
🐳  Preparing Kubernetes v1.19.2 on Docker 19.03.12 ...
🔎  Verifying Kubernetes components...
🌟  Enabled addons: storage-provisioner, default-storageclass
🏄  Done! kubectl is now configured to use "cluster1" by default
```

>Note:- `--profile` is the name of the minikube VM being used. This can be set to allow having multiple instances of minikube independently. (default "minikube, so we are going to set it to `cluster1` as we are planning to create 2 minikube clusters

### Create kubernetes cluster2

We are going to use the same `minikube start` command to create kubernetes cluster but we are going to change the `--profile` name.

```bash=
$minikube start --force --memory="4096" --cpus="2" -b kubeadm --kubernetes-version="v1.19.2" --driver="kvm2" --feature-gates="BlockVolume=true,CSIBlockVolume=true,VolumeSnapshotDataSource=true,ExpandCSIVolumes=true" --profile="cluster2"
😄  [cluster2] minikube v1.14.0 on Fedora 32
❗  minikube skips various validations when --force is supplied; this may lead to unexpected behavior
✨  Using the kvm2 driver based on user configuration
🛑  The "kvm2" driver should not be used with root privileges.
💡  If you are running minikube within a VM, consider using --driver=none:
📘    https://minikube.sigs.k8s.io/docs/reference/drivers/none/
👍  Starting control plane node cluster2 in cluster cluster2
🔥  Creating kvm2 VM (CPUs=2, Memory=4096MB, Disk=20000MB) ...
🐳  Preparing Kubernetes v1.19.2 on Docker 19.03.12 ...
🔎  Verifying Kubernetes components...
🌟  Enabled addons: storage-provisioner, default-storageclass
🏄  Done! kubectl is now configured to use "cluster2" by default
```

### Verify kubernetes clusters

As we have created the 2 kubernetes clusters, Let's verify it. Am going to use `--context` with kubectl to talk to different clusters

```bash=
[root@dhcp53-215 ~]$ kubectl get nodes --context=cluster1
NAME       STATUS   ROLES    AGE     VERSION
cluster1   Ready    master   8m14s   v1.19.2
[root@dhcp53-215 $ kubectl get nodes --context=cluster2
NAME       STATUS   ROLES    AGE     VERSION
cluster2   Ready    master   3m19s   v1.19.2
```

> The value  for the `--context` will be the name we passed in `--profile` when creating kubernetes clusters

As we got the kubernetes cluster, we need to add a disk to minikube VM as Rook needs a raw device to create ceph OSD and also we need to create a directory that is required to store monitor details.

### Add Device to minikube VM's

```bash=
minikube ssh "sudo mkdir -p /mnt/vda1/var/lib/rook;sudo ln -s /mnt/vda1/var/lib/rook /var/lib/rook" --profile="cluster1"
```

> Repeat the same step for cluster2 minikube VM also just change the `--profile` value to `cluster2`

Now let's add a device to minikube vm's

```bash=
$sudo -S qemu-img create -f raw /var/lib/libvirt/images/minikube-box2-vm-disk-"cluster1"-50G 50G
$virsh -c qemu:///system attach-disk "cluster1" --source /var/lib/libvirt/images/minikube-box2-vm-disk-"cluster1"-50G --target vdb --cache none
$virsh -c qemu:///system reboot --domain "cluster1"
```

### Verify that thedevice is present in the minikube VM

```bash=
$minikube ssh --profile=cluster1
                         _             _
            _         _ ( )           ( )
  ___ ___  (_)  ___  (_)| |/')  _   _ | |_      __
/' _ ` _ `\| |/' _ `\| || , <  ( ) ( )| '_`\  /'__`\
| ( ) ( ) || || ( ) || || |\`\ | (_) || |_) )(  ___/
(_) (_) (_)(_)(_) (_)(_)(_) (_)`\___/'(_,__/'`\____)

$ ls /dev/vd
vda   vda1  vdb
$ ls /dev/vd
vda   vda1  vdb
$ ls /dev/vdb
/dev/vdb
$ exit
```

> If the device is not visible inside the minikube VM, you need to start the minikube with the same command `minikube start` we used above to create the kubernetes cluster.

> Repeat the same steps to add a  device to the other kubernetes cluster  `cluster2`, don't forget to change `--profile` value to `cluster2`.


## Install Rook

Now we have the basic infrastructure to create a Ceph cluster. let's install Rook on both kubernetes clusters

```bash=
$git clone git@github.com:rook/rook.git
$ cd rook
```

### Create Required RBAC and CRD
```bash=
$kubectl create -f cluster/examples/kubernetes/ceph/common.yaml --context=cluster1
$kubectl create -f cluster/examples/kubernetes/ceph/crds.yaml --context=cluster1
```

### Install Rook operator

```bash=
$kubectl create -f cluster/examples/kubernetes/ceph/operator.yaml --context=cluster1
configmap/rook-ceph-operator-config created
deployment.apps/rook-ceph-operator created
```

As you already know that for RBD async mirroring RBD daemon need to talk to each other so am going to use hostNetworking for now to create a Ceph cluster

### Create ceph cluster

```bash=
$cat <<EOF | kubectl --context=cluster1 apply -f -
kind: ConfigMap
apiVersion: v1
metadata:
  name: rook-config-override
  namespace: rook-ceph
data:
  config: |
    [global]
    osd_pool_default_size = 1
    mon_warn_on_pool_no_redundancy = false
---
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: my-cluster
  namespace: rook-ceph
spec:
  dataDirHostPath: /var/lib/rook
  cephVersion:
    image: ceph/ceph:v15
    allowUnsupported: true
  mon:
    count: 1
    allowMultiplePerNode: true
  dashboard:
    enabled: true
  crashCollector:
    disable: true
  storage:
    useAllNodes: true
    useAllDevices: true
  network:
    provider: host
  healthCheck:
    daemonHealth:
      mon:
        interval: 45s
        timeout: 600s
EOF
```
### Create toolbox pod

```bash=
$kubectl create -f cluster/examples/kubernetes/ceph/toolbox.yaml --context=cluster1
deployment.apps/rook-ceph-tools created
```

### Verify all pods are in running state

```bash=
$kubectl get po --context=cluster1 -nrook-ceph
NAME                                            READY   STATUS      RESTARTS   AGE
csi-cephfsplugin-gzgxm                          3/3     Running     0          21m
csi-cephfsplugin-provisioner-6cfdb89f9b-5bqbf   6/6     Running     0          21m
csi-cephfsplugin-provisioner-6cfdb89f9b-d2bp6   0/6     Pending     0          21m
csi-rbdplugin-ffpsh                             3/3     Running     0          21m
csi-rbdplugin-provisioner-785d8b7967-wxfp7      0/7     Pending     0          21m
csi-rbdplugin-provisioner-785d8b7967-zrv5x      7/7     Running     0          21m
rook-ceph-mgr-a-7c74849566-ghmqp                1/1     Running     0          24m
rook-ceph-mon-a-5bb9f8787d-v5n4m                1/1     Running     0          24m
rook-ceph-operator-d758658ff-4ldqs              1/1     Running     0          33m
rook-ceph-osd-0-56d989d9cc-6rsxl                1/1     Running     0          24m
rook-ceph-osd-prepare-cluster1-hpxfp            0/1     Completed   0          24m
rook-ceph-tools-78cdfd976c-jj98r                1/1     Running     0          3m30s
rook-discover-p4dfs                             1/1     Running     0          32m
```

> Repeat the same steps we did to install rook. Now install Rook on cluster2. Don't forget to change the `--context` value


Once we have Rook installed on both the clusters, we can create pools, configure mirroring and try out rbd async replication

### Create RBD Block Pool

Rook allows the creation and customization of storage pools through the custom resource definitions (CRDs), Let's create a Pool with Name `replicapool` with mirroring Enabled

```bash=
$cat <<EOF | kubectl --context=cluster1 apply -f -
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  replicated:
    size: 1
  mirroring:
    enabled: true
    mode: image
    # schedule(s) of snapshot
    snapshotSchedules:
      - interval: 24h # daily snapshots
        startTime: 14:00:00-05:00
EOF
```

Once mirroring is enabled, Rook will by default create its own bootstrap peer token so that it can be used by another cluster. The bootstrap peer token can be found in a Kubernetes Secret. The name of the Secret is present in the Status field of the CephBlockPool CR:

```bash=
$kubectl get cephblockpools.ceph.rook.io replicapool --context=cluster1 -nrook-ceph -o jsonpath='{.status.info}'
{"rbdMirrorBootstrapPeerSecretName":"pool-peer-token-replicapool"}
```

This secret can then be fetched like so:

```bash=
$kubectl get secret -n rook-ceph pool-peer-token-replicapool --context=cluster2 -o jsonpath='{.data.token}'|base64 -d
eyJmc2lkIjoiZjA1YzEwMGYtNjJkYS00YzU4LWI4OTktMzQyZTM0ZDg4MDNkIiwiY2xpZW50X2lkIjoicmJkLW1pcnJvci1wZWVyIiwia2V5IjoiQVFBNFU3dGZndDl6T3hBQUZXbEorYUFNUFo5MXpqNlRwUE84V3c9PSIsIm1vbl9ob3N0IjoiW3YyOjE5Mi4xNjguMzkuODQ6MzMwMCx2MToxOTIuMTY4LjM5Ljg0OjY3ODldIn0=
```

### Create RBD mirror daemon CRD

Rook allows the creation and updating of the rbd-mirror daemon(s) through the custom resource definitions (CRDs). RBD images can be asynchronously mirrored between two Ceph clusters.


#### Configure RBD mirroring on cluster1

Let's configure the RBD mirror daemon on the cluster1, To do that we need to get the pool secret and site_name from the cluster2

##### Get site_name from the cluster2

```bash=
$kubectl get cephblockpools.ceph.rook.io replicapool --context=cluster2 -nrook-ceph -o jsonpath='{.status.mirroringInfo.summary.summary.site_name}'
e1877a97-6607-4baa-b477-64b0c61f268f-rook-ceph
```

##### Get token from the cluster2

```bash=
$kubectl get secret -n rook-ceph pool-peer-token-replicapool --context=cluster2 -o jsonpath='{.data.token}'|base64 -d
eyJmc2lkIjoiZTE4NzdhOTctNjYwNy00YmFhLWI0NzctNjRiMGM2MWYyNjhmIiwiY2xpZW50X2lkIjoicmJkLW1pcnJvci1wZWVyIiwia2V5IjoiQVFCZ1U3dGZydUh5THhBQVBMRE9EajJRMEcybjRCd2tDbmxLQUE9PSIsIm1vbl9ob3N0IjoiW3YyOjE5Mi4xNjguMzkuMTYwOjMzMDAsdjE6MTkyLjE2OC4zOS4xNjA6Njc4OV0ifQ==
```

When the peer token is available, you need to create a Kubernetes Secret. Our `e1877a97-6607-4baa-b477-64b0c61f268f-rook-ceph` will have to be created manually, like so:

##### Create secret for RBD mirroring on cluster1

```bash=
$kubectl -n rook-ceph create secret generic --context=cluster1 "e1877a97-6607-4baa-b477-64b0c61f268f-rook-ceph" \
--from-literal=token=eyJmc2lkIjoiZTE4NzdhOTctNjYwNy00YmFhLWI0NzctNjRiMGM2MWYyNjhmIiwiY2xpZW50X2lkIjoicmJkLW1pcnJvci1wZWVyIiwia2V5IjoiQVFCZ1U3dGZydUh5THhBQVBMRE9EajJRMEcybjRCd2tDbmxLQUE9PSIsIm1vbl9ob3N0IjoiW3YyOjE5Mi4xNjguMzkuMTYwOjMzMDAsdjE6MTkyLjE2OC4zOS4xNjA6Njc4OV0ifQ== \
--from-literal=pool=replicapool
secret/e1877a97-6607-4baa-b477-64b0c61f268f-rook-ceph created
```

> Pass the correct pool name and token retrieved from cluster2

Rook will read both token and pool keys of the Data content of the Secret. Rook also accepts the destination key, which specifies the mirroring direction. It defaults to rx-tx for bidirectional mirroring, but can also be set to rx-only for unidirectional mirroring.

##### Create mirroring CRD on cluster1

You can now inject the rbdmirror CR:

```bash=
$cat <<EOF | kubectl --context=cluster1 apply -f -
apiVersion: ceph.rook.io/v1
kind: CephRBDMirror
metadata:
  name: my-rbd-mirror
  namespace: rook-ceph
spec:
  count: 1
  peers:
    secretNames:
      - "e1877a97-6607-4baa-b477-64b0c61f268f-rook-ceph"
EOF
cephrbdmirror.ceph.rook.io/my-rbd-mirror created
```
> Update the secretName in Mirror CRD with the one you created above.

##### Validate mirroring health on cluster1

Once we create mirror CRD we need to validate two things.
1. Mirroring Pod in  rook-ceph namespace
2. Blockpool mirroring info

###### Check mirroring pod status

```bash=
$kubectl get po -nrook-ceph --context=cluster1
NAME                                            READY   STATUS      RESTARTS   AGE
csi-cephfsplugin-8jj5b                          3/3     Running     0          28m
csi-cephfsplugin-provisioner-7dc78747bf-426cp   6/6     Running     0          28m
csi-rbdplugin-64w97                             3/3     Running     0          28m
csi-rbdplugin-provisioner-54d48757b4-4hzml      6/6     Running     0          28m
rook-ceph-mgr-a-f59fd65f4-bwnlb                 1/1     Running     0          27m
rook-ceph-mon-a-7f754c4768-rwz29                1/1     Running     0          28m
rook-ceph-operator-559b6fcf59-77tct             1/1     Running     0          29m
rook-ceph-osd-0-6774f57f64-qmjqx                1/1     Running     0          27m
rook-ceph-osd-prepare-cluster1-n8vqb            0/1     Completed   0          27m
rook-ceph-rbd-mirror-a-6cbdcd6cc9-cfsj2         1/1     Running     0          6m33s
```

###### Check blockpool mirroring status

```bash=
$kubectl get cephblockpools.ceph.rook.io replicapool --context=cluster1 -nrook-ceph -o jsonpath='{.status.mirroringStatus.summary.summary}'
{"daemon_health":"OK","health":"OK","image_health":"OK","states":{}}
```

> You need to wait till all fields in `summary` become `OK`

Wow!! Everything seems fine let's configure the mirroring daemon on cluster2

#### Configure RBD mirroring on cluster2

Configure the RBD mirror daemon on the cluster2, To do that we need to get the pool secret and site_name from the cluster1

##### Get site_name from the cluster1

```bash=
$kubectl get cephblockpools.ceph.rook.io replicapool --context=cluster1 -nrook-ceph -o jsonpath='{.status.mirroringInfo.summary.summary.site_name}'
f05c100f-62da-4c58-b899-342e34d8803d-rook-ceph
```

##### Get token from the cluster1

```bash=
$kubectl get secret -n rook-ceph pool-peer-token-replicapool --context=cluster1 -o jsonpath='{.data.token}'|base64 -d
eyJmc2lkIjoiZjA1YzEwMGYtNjJkYS00YzU4LWI4OTktMzQyZTM0ZDg4MDNkIiwiY2xpZW50X2lkIjoicmJkLW1pcnJvci1wZWVyIiwia2V5IjoiQVFBNFU3dGZndDl6T3hBQUZXbEorYUFNUFo5MXpqNlRwUE84V3c9PSIsIm1vbl9ob3N0IjoiW3YyOjE5Mi4xNjguMzkuODQ6MzMwMCx2MToxOTIuMTY4LjM5Ljg0OjY3ODldIn0=
```

When the peer token is available, you need to create a Kubernetes Secret. Our `f05c100f-62da-4c58-b899-342e34d8803d-rook-ceph` will have to be created manually, like so:

##### Create secret for RBD mirroring on cluster2

```bash=
$kubectl -n rook-ceph create secret generic --context=cluster2 "f05c100f-62da-4c58-b899-342e34d8803d-rook-ceph" \
--from-literal=token=eyJmc2lkIjoiZjA1YzEwMGYtNjJkYS00YzU4LWI4OTktMzQyZTM0ZDg4MDNkIiwiY2xpZW50X2lkIjoicmJkLW1pcnJvci1wZWVyIiwia2V5IjoiQVFBNFU3dGZndDl6T3hBQUZXbEorYUFNUFo5MXpqNlRwUE84V3c9PSIsIm1vbl9ob3N0IjoiW3YyOjE5Mi4xNjguMzkuODQ6MzMwMCx2MToxOTIuMTY4LjM5Ljg0OjY3ODldIn0= \
--from-literal=pool=replicapool
secret/f05c100f-62da-4c58-b899-342e34d8803d-rook-ceph created
```

> Pass the correct pool name and token retrieved from cluster1

##### Create mirroring CRD on cluster2

You can now inject the rbdmirror CR:

```bash=
$cat <<EOF | kubectl --context=cluster2 apply -f -
apiVersion: ceph.rook.io/v1
kind: CephRBDMirror
metadata:
  name: my-rbd-mirror
  namespace: rook-ceph
spec:
  count: 1
  peers:
    secretNames:
      - "f05c100f-62da-4c58-b899-342e34d8803d-rook-ceph"
EOF
cephrbdmirror.ceph.rook.io/my-rbd-mirror created
```
> Update the secretName in CRD with the one you created.

##### Validate mirroring health on cluster1

Once we create mirror CRD we need to validate two things.
1. Mirroring Pod in  rook-ceph namespace
2. Blockpool mirroring info

###### Check mirroring pod status

```bash=
$kubectl get po -nrook-ceph --context=cluster2
NAME                                            READY   STATUS      RESTARTS   AGE
csi-cephfsplugin-pkrhb                          3/3     Running     0          38m
csi-cephfsplugin-provisioner-7dc78747bf-gr99q   6/6     Running     0          38m
csi-rbdplugin-provisioner-54d48757b4-4kpw5      6/6     Running     0          38m
csi-rbdplugin-xwz6j                             3/3     Running     0          38m
rook-ceph-mgr-a-565774f8fb-b66xs                1/1     Running     0          38m
rook-ceph-mon-a-88b5d6d95-vkgsb                 1/1     Running     0          38m
rook-ceph-operator-559b6fcf59-mzbpb             1/1     Running     0          39m
rook-ceph-osd-0-557d5f5948-9qrvq                1/1     Running     0          37m
rook-ceph-osd-prepare-cluster2-j9ccs            0/1     Completed   0          38m
rook-ceph-rbd-mirror-a-7c6d49f67c-b2vd2         1/1     Running     0          83s
```

###### Check blockpool mirroring status

```bash=
$kubectl get cephblockpools.ceph.rook.io replicapool --context=cluster2 -nrook-ceph -o jsonpath='{.status.mirroringStatus.summary.summary}'
{"daemon_health":"OK","health":"OK","image_health":"OK","states":{}}
```

> You need to wait till all fields in `summary` become `OK`

Now the installation is complete, Let's see we can create rbd images and able to mirror it to the `secondary` cluster.

### Provision storage

Create a `storageclass` and `PVC`

#### Create storageclass

```bash=
$cat <<EOF | kubectl --context=cluster1 apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: rook-ceph-block
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
    clusterID: rook-ceph
    pool: replicapool
    imageFormat: "2"
    imageFeatures: layering
    csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
    csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
    csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
    csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
    csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
    csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
    csi.storage.k8s.io/fstype: ext4
reclaimPolicy: Retain
EOF
storageclass.storage.k8s.io/rook-ceph-block created
```

> Create a storageclass with `Retain` `reclaimPolicy` so that we can use static binding for
DR.

### Create PersistentVolumeClaim on cluster1

```bash=
$cat <<EOF | kubectl --context=cluster1 apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rbd-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: rook-ceph-block
EOF
```

Check PVC is in bound state

```bash=
$kubectl get pvc --context=cluster1
NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
rbd-pvc   Bound    pvc-57cfca36-cb76-4d79-b258-a107bc5c4d15   1Gi        RWO            rook-ceph-block   44s
```

Get the RBD image name created for this PVC

```bash=
$kubectl get pv --context=cluster1 $(kubectl get pvc rbd-pvc  --context=cluster1 -o jsonpath='{.spec.volumeName}') -o jsonpath='{.spec.csi.volumeAttributes.imageName}'
csi-vol-1b82d349-2d5a-11eb-b8d5-0242ac110004
```

Run commands on toolbox pod on cluster1 to enable mirroring

```bash=
$kubectl get po --context=cluster1 -nrook-ceph -l  app=rook-ceph-tools
NAME                               READY   STATUS    RESTARTS   AGE
rook-ceph-tools-78cdfd976c-jrhjm   1/1     Running   0          3m53s
```

```bash=
$kubectl exec --context=cluster1 -nrook-ceph rook-ceph-tools-78cdfd976c-jrhjm -- rbd info csi-vol-1b82d349-2d5a-11eb-b8d5-0242ac110004 --pool=replicapool
rbd image 'csi-vol-1b82d349-2d5a-11eb-b8d5-0242ac110004':
	size 1 GiB in 256 objects
	order 22 (4 MiB objects)
	snapshot_count: 0
	id: 13a672b41f5f
	block_name_prefix: rbd_data.13a672b41f5f
	format: 2
	features: layering
	op_features:
	flags:
	create_timestamp: Mon Nov 23 07:04:21 2020
	access_timestamp: Mon Nov 23 07:04:21 2020
	modify_timestamp: Mon Nov 23 07:04:21 2020
```

```bash=
kubectl exec --context=cluster1 -nrook-ceph rook-ceph-tools-78cdfd976c-jrhjm -- rbd mirror image enable csi-vol-1b82d349-2d5a-11eb-b8d5-0242ac110004 snapshot --pool=replicapool
Mirroring enabled
```

Verify that mirroring enabled on the image

```bash=
$kubectl exec --context=cluster1 -nrook-ceph rook-ceph-tools-78cdfd976c-jrhjm -- rbd info csi-vol-1b82d349-2d5a-11eb-b8d5-0242ac110004 --pool=replicapool
rbd image 'csi-vol-1b82d349-2d5a-11eb-b8d5-0242ac110004':
	size 1 GiB in 256 objects
	order 22 (4 MiB objects)
	snapshot_count: 1
	id: 13a672b41f5f
	block_name_prefix: rbd_data.13a672b41f5f
	format: 2
	features: layering
	op_features:
	flags:
	create_timestamp: Mon Nov 23 07:04:21 2020
	access_timestamp: Mon Nov 23 07:04:21 2020
	modify_timestamp: Mon Nov 23 07:04:21 2020
	mirroring state: enabled
	mirroring mode: snapshot
	mirroring global id: 7da10ec2-aa67-4b05-86f0-487b4fa9fdbb
	mirroring primary: true
```

#### Verify that image is mirrored on the secondary cluster

Run commands on toolbox pod on cluster1 to enable mirroring

```bash=
$kubectl get po --context=cluster2 -nrook-ceph -l  app=rook-ceph-tools
NAME                               READY   STATUS    RESTARTS   AGE
rook-ceph-tools-78cdfd976c-2566m   1/1     Running   0          3m53s
```

```bash=
$kubectl exec --context=cluster2 -nrook-ceph rook-ceph-tools-78cdfd976c-2566m -- rbd info csi-vol-1b82d349-2d5a-11eb-b8d5-0242ac110004 --pool=replicapool
rbd image 'csi-vol-1b82d349-2d5a-11eb-b8d5-0242ac110004':
	size 1 GiB in 256 objects
	order 22 (4 MiB objects)
	snapshot_count: 1
	id: 12a55edc8361
	block_name_prefix: rbd_data.12a55edc8361
	format: 2
	features: layering, non-primary
	op_features:
	flags:
	create_timestamp: Mon Nov 23 07:12:33 2020
	access_timestamp: Mon Nov 23 07:12:33 2020
	modify_timestamp: Mon Nov 23 07:12:33 2020
	mirroring state: enabled
	mirroring mode: snapshot
	mirroring global id: 7da10ec2-aa67-4b05-86f0-487b4fa9fdbb
	mirroring primary: false
```
