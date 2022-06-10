---
layout: post
title: Consuming Storage From external ceph cluster in Rook
comments: true
---

This doc assumes that you already set up the Rook with an external ceph cluster.

Please refer to [doc](../setup-external-ceph-cluster-with-rook) on how to setup
external ceph cluster with Rook.

This blog will help you to Understand how to consume storage from imported
external ceph cluster in Rook and how to create and consume Below listed
resources in ceph cluster.

* CephBlockPool
* CephFileSystem
* SubvolumeGroup
* RadosNamespace

## Create resources specified above in the ceph cluster

**Note** You can skip this step and create the ceph resources with Rook CRs if
you are using the ceph cluster deployed with Rook.

### Create RBD Pool

```bash=
ceph osd pool create replicapool 128
rbd pool init
ceph osd pool set replicapool size 1 --yes-i-really-mean-it
```

### Create CephFS Filesystem

```console
ceph osd pool create myfs-metadata 128
ceph osd pool create myfs-data 124
ceph fs new myfs myfs-metadata myfs-data
```

### Create SubVolumeGroup

Let's create a SubVolumeGroup in the Filesystem `myfs` created above

```bash=
ceph fs subvolumegroup create myfs volgroup
```

### Create RadosNamespace

Let's create a RadosNamespace in the BlockPool `replicapool` created above

```bash=
rbd namespace create replicapool/testnamespace
```

checkout Rook 1.9.5 Repository

```bash=
cd rook/deploy/examples
```

## Consume above created ceph resources in kubernetes cluster

### CephBlockPool

To create RBD PVC, we need to first have the RBD pool created in the ceph
cluster and ceph users to access the pool and a StorageClass to point to the
Pool where to create the PVC.

*Note* Above create a pool with the name `replicapool` with  PG Num `128` and
setting replica size to `1` As I have only one OSD. If you have a minimum `3`
OSD please set it to `3`.

As this ceph cluster is deployed in the `rook-ceph-external` namespace, we need
to update a few details in RBD storageclass

```bash=
$ cd csi/rbd
$ cat storageclass-test.yaml
allowVolumeExpansion: true
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block
parameters:
  clusterID: rook-ceph-external  # Namespace where ceph cluster is deployed
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph-external  # Namespace where ceph cluster is deployed
  csi.storage.k8s.io/fstype: ext4
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph-external  # Namespace where ceph cluster is deployed
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph-external  # Namespace where ceph cluster is deployed
  imageFeatures: layering
  imageFormat: "2"
  pool: replicapool
provisioner: rook-ceph.rbd.csi.ceph.com
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

change the above parameters in the rbd storageclass-test.yaml and keep all
other configurations as it is (Dont forget to remove pool.yaml from
storageclass-test.yaml).

Let's create RBD storageclass, PVC, Pod and verify it on the ceph cluster.

```bash=
[🎩︎]mrajanna@fedora rbd $]kubectl create -f storageclass-test.yaml
storageclass.storage.k8s.io/rook-ceph-block created
[🎩︎]mrajanna@fedora rbd $]kubectl create -f pvc.yaml
persistentvolumeclaim/rbd-pvc created
[🎩︎]mrajanna@fedora rbd $]kubectl get pvc
NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
rbd-pvc   Bound    pvc-a1a802fd-a577-45e4-b52d-687b8333cbef   1Gi        RWO            rook-ceph-block   4s
[🎩︎]mrajanna@fedora rbd $]kubectl create -f pod.yaml
pod/csirbd-demo-pod created
[🎩︎]mrajanna@fedora rbd $]kubectl get po
NAME              READY   STATUS    RESTARTS   AGE
csirbd-demo-pod   1/1     Running   0          43s
```

Let's verify we have the rbd image in the ceph cluster.

```bash=
sh-4.4$ rbd ls replicapool
csi-vol-9475df3e-e895-11ec-bbf0-9a1ab8207c3a
```

### CephFilesystem

To create CephFS PVC, we need to first have the CephFS Filesystem created in
the ceph cluster and ceph users to access the Filesystem and a StorageClass to
point to the Filesystem where to create the PVC.

*Note* Above a Filesystem with the name `myfs`, let's use the same for testing

As this ceph cluster is deployed in the `rook-ceph-external` namespace, we need
to update a few details in CephFS storageclass

```bash=
$ cd csi/cephfs
$ cat storageclass.yaml
allowVolumeExpansion: true
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-cephfs
parameters:
  clusterID: rook-ceph-external # Namespace where ceph cluster is deployed
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph-external # Namespace where ceph cluster is deployed
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-cephfs-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph-external # Namespace where ceph cluster is deployed
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph-external # Namespace where ceph cluster is deployed
  fsName: myfs
  pool: myfs-data
provisioner: rook-ceph.cephfs.csi.ceph.com
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

change the above parameters in the cephfs storageclass.yaml and keep all other
configurations as it is.

Let's create CephFS storageclass, PVC, Pod and verify it on the ceph cluster.

```bash=
[🎩︎]mrajanna@fedora cephfs $]kubectl create  -f cephfs/storageclass.yaml
storageclass.storage.k8s.io/rook-cephfs created
[🎩︎]mrajanna@fedora cephfs $]kubectl create -f pvc.yaml
persistentvolumeclaim/cephfs-pvc created
[🎩︎]mrajanna@fedora cephfs $]kubectl get pvc
NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
cephfs-pvc   Bound    pvc-52186bd0-92dd-433c-af00-e0a60baf71a1   1Gi        RWO            rook-cephfs       2s
[🎩︎]mrajanna@fedora cephfs $]kubectl create -f pod.yaml
pod/csicephfs-demo-pod created
[🎩︎]mrajanna@fedora cephfs $]kubectl get po
NAME                 READY   STATUS    RESTARTS   AGE
csicephfs-demo-pod   1/1     Running   0          24s
```

Let's verify we have cephfs Subvolume created in the ceph cluster.

```bash=
sh-4.4$ ceph fs subvolume ls myfs --group_name=csi
[
    {
        "name": "csi-vol-9bcadef0-e896-11ec-93d2-8a68b48559cc"
    }
]
```

**Note** `csi` is the default subvolumegroup cephcsi creates.

### SubvolumeGroup

To create CephFS PVC in a subvolumeGroup, we need to first have subvolumegroup
created in the CephFS Filesystem created in the ceph cluster and ceph users to
access the subvolumegroup and a StorageClass to point to the clusterID of
subvolumegroup where to create the PVC.

*Note* Above we have subvolumegroup with name `volgroup` in `myfs` Filesystem,
Let's use the same for testing

#### Create SubVolumeGroup CR

```bash=
[🎩︎]mrajanna@fedora examples $]cat <<EOFcat <<EOF | kubectl apply -f -
---
apiVersion: ceph.rook.io/v1
kind: CephFilesystemSubVolumeGroup
metadata:
  name: volgroup
  namespace: rook-ceph-external
spec:
  filesystemName: myfs
EOF
cephfilesystemsubvolumegroup.ceph.rook.io/volgroup created
```

**Note** We are creating SubVolumeGroup CR to generate the SubVolumeGroup Name
and ClusterID mapping, it won't create SubVolumeGroup in the Ceph cluster.

#### Check SubVolumeGroup details

```bash=
[🎩︎]mrajanna@fedora examples $]kubectl get cephfilesystemsubvolumegroup.ceph.rook.io/volgroup -nrook-ceph-external
NAME       PHASE
volgroup   Ready
[🎩︎]mrajanna@fedora examples $]kubectl get cephfilesystemsubvolumegroup.ceph.rook.io/volgroup -nrook-ceph-external -o jsonpath='{.status.info}'
{"clusterID":"728ed87ac1a8809431b4549f4a4897aa"}
```

The above clusterID `728ed87ac1a8809431b4549f4a4897aa` is the one we need to
use it in subvolumegroup storageclass.

The reason we have a new clusterID for subvolumegroup is we cannot specify
subvolumegroup name in the storageclass, This clusterID has the mapping to the
subvolumegroup

#### Check mapping between clusterID and SubVolumeGroup

```bash=
[🎩︎]mrajanna@fedora examples $]kubectl get cm rook-ceph-csi-config -nrook-ceph -oyaml
apiVersion: v1
data:
  csi-cluster-config-json: '[{"clusterID":"rook-ceph-external","monitors":["192.168.122.243:6789"],"namespace":"rook-ceph-external"},{"clusterID":"728ed87ac1a8809431b4549f4a4897aa","monitors":["192.168.122.243:6789"],"namespace":"rook-ceph-external","cephFS":{"subvolumeGroup":"volgroup"}}]'
kind: ConfigMap
metadata:
  creationTimestamp: "2022-06-10T07:57:13Z"
  name: rook-ceph-csi-config
  namespace: rook-ceph
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: false
    controller: true
    kind: Deployment
    name: rook-ceph-operator
    uid: 5c8f3d9a-d973-4b65-b3f3-edbee10cd6b3
  resourceVersion: "5973"
  uid: 7ac9d0d9-a9ef-4e04-b322-63d01513c072
```

#### Create StorageClass, PVC and Pod

```bash=
$cat <<EOF | kubectl apply -f -
allowVolumeExpansion: true
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-cephfs-subvol-group
parameters:
  clusterID: 728ed87ac1a8809431b4549f4a4897aa # Subvolumegroup clusterID
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph-external # Namespace where ceph cluster is deployed
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-cephfs-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph-external # Namespace where ceph cluster is deployed
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph-external # Namespace where ceph cluster is deployed
  fsName: myfs
  pool: myfs-data
provisioner: rook-ceph.cephfs.csi.ceph.com
reclaimPolicy: Delete
volumeBindingMode: Immediate
EOF
storageclass.storage.k8s.io/rook-cephfs-subvol-group created
```

```bash=
[🎩︎]mrajanna@fedora cephfs $]kubectl create -f pvc.yaml
persistentvolumeclaim/cephfs-subvol-pvc created
[🎩︎]mrajanna@fedora cephfs $]kubectl get pvc
NAME                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS               AGE
cephfs-subvol-pvc   Bound    pvc-556acc86-ba24-43d6-ae19-85fa21a046a4   1Gi        RWO            rook-cephfs-subvol-group   2s
[🎩︎]mrajanna@fedora cephfs $]kubectl create -f pod.yaml
pod/csicephfssubvol-demo-pod created
[🎩︎]mrajanna@fedora cephfs $]kubectl get po
NAME                       READY   STATUS    RESTARTS   AGE
csicephfssubvol-demo-pod   1/1     Running   0          70s
```

**Note**. Dont forget to change storageclass in the PVC yaml to point to
subvolumegroup storageClass

Let's verify we have cephfs Subvolume in the specified group created in the
ceph cluster.

```bash=
sh-4.4$ ceph fs subvolume ls myfs --group_name=volgroup
[
    {
        "name": "csi-vol-e0905660-e898-11ec-93d2-8a68b48559cc"
    }
]
```

**Note** `volgroup` is the SubVolumeGroup we created.

## RadosNamespaces

To create RBD PVC in a RadosNamespaces, we need to first have RadosNamespaces
created in the RBD Pool created in the ceph cluster and ceph users to access
the RadosNamespaces and a StorageClass to point to the clusterID of
RadosNamespaces where to create the PVC.

*Note* Above we have RadosNamespaces with the name `testnamespace` in the
`replicapool` Pool, Let's use the same for testing

### Create RadosNamespaces CR

```bash=
[🎩︎]mrajanna@fedora examples $]cat <<EOF | kubectl apply -f -
---
apiVersion: ceph.rook.io/v1
kind: CephBlockPoolRadosNamespace
metadata:
  name: testnamespace
  namespace: rook-ceph-external
spec:
  blockPoolName: replicapool
EOF
cephblockpoolradosnamespace.ceph.rook.io/testnamespace created
```

**Note** We are creating RadosNamespace CR to generate the RadosNamespace Name
and ClusterID mapping, it won't create RadosNamespace in the Ceph cluster.

#### Get RadosNamespaces details

```bash=
kubectl get cephblockcephblockpoolradosnamespace.ceph.rook.io/testnamespace -nrook-ceph-external
NAME            AGE
testnamespace   28s
[🎩︎]mrajanna@fedora examples $]kubectl get cephblockpoolradosnamespace.ceph.rook.io/testnamespace -nrook-ceph-external -o jsonpath='{.status.info}'
{"clusterID":"6165a9e9a61346a8b8e629bdfb583414"}
```

The above clusterID `6165a9e9a61346a8b8e629bdfb583414` is the one we need to
use it in RadosNamespace storageclass.

The reason we have a new clusterID for RadosNamespace is we cannot specify
RadosNamespace name in the storageclass, This clusterID has the mapping to the
RadosNamespace

#### Check mapping between clusterID and RadosNamespace

```bash=
[🎩︎]mrajanna@fedora examples $]kubectl get cm rook-ceph-csi-config -nrook-ceph -oyaml
apiVersion: v1
data:
  csi-cluster-config-json: '[{"clusterID":"rook-ceph-external","monitors":["192.168.122.243:6789"],"namespace":"rook-ceph-external"},{"clusterID":"728ed87ac1a8809431b4549f4a4897aa","monitors":["192.168.122.243:6789"],"namespace":"rook-ceph-external","cephFS":{"subvolumeGroup":"volgroup"}},{"clusterID":"6165a9e9a61346a8b8e629bdfb583414","monitors":["192.168.122.243:6789"],"namespace":"rook-ceph-external","rbd":{"radosNamespace":"testnamespace"}}]'
kind: ConfigMap
metadata:
  creationTimestamp: "2022-06-10T07:57:13Z"
  name: rook-ceph-csi-config
  namespace: rook-ceph
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: false
    controller: true
    kind: Deployment
    name: rook-ceph-operator
    uid: 5c8f3d9a-d973-4b65-b3f3-edbee10cd6b3
  resourceVersion: "7913"
  uid: 7ac9d0d9-a9ef-4e04-b322-63d01513c072

```

#### Create StorageClass, PVC, and Pod

```bash=
$cat <<EOF | kubectl apply -f -
allowVolumeExpansion: true
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-blockpool-radosnamespace
parameters:
  clusterID: 6165a9e9a61346a8b8e629bdfb583414 # RadosNamespace clusterID
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph-external
  csi.storage.k8s.io/fstype: ext4
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph-external
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph-external
  imageFeatures: layering
  imageFormat: "2"
  pool: replicapool
provisioner: rook-ceph.rbd.csi.ceph.com
reclaimPolicy: Delete
volumeBindingMode: Immediate
EOF
storageclass.storage.k8s.io/rook-blockpool-radosnamespace created
```

```bash=
[🎩︎]mrajanna@fedora rbd $]kubectl create -f pvc.yaml
persistentvolumeclaim/rbd-radosnamespace-pvc created
[🎩︎]mrajanna@fedora rbd $]kubectl get pvc
NAME                     STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                    AGE
rbd-radosnamespace-pvc   Bound    pvc-e664ea28-c625-41d3-aa81-aead7449e989   1Gi        RWO            rook-blockpool-radosnamespace   3s
[🎩︎]mrajanna@fedora rbd $]kubectl create -f pod.yaml
pod/csirbd-namespace-demo-pod created
[🎩︎]mrajanna@fedora rbd $]kubectl get po
NAME                        READY   STATUS    RESTARTS   AGE
csirbd-namespace-demo-pod   1/1     Running   0          26s

```

**Note**. Dont forget to change storageclass in the PVC yaml to point to
RadosNamespace storageClass

Let's verify we have the rbd image created in the specified radosnamespace in
the ceph cluster.

```bash=
sh-4.4$ rbd ls replicapool/testnamespace
csi-vol-3ca55f72-e89a-11ec-bbf0-9a1ab8207c3a
```

Yay!, We have created StorageClass's, PVC's, Pod's and verfied for all the ceph
resources listed above.
