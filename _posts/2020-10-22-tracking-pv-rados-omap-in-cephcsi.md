---
layout: post
title: Track PV to RADOS omap data mapping stored by cephcsi
comments: true
---

This blog will help you to understand how to track the internal rados omap data
stored by cephcsi for cephfs and rbd pvc.

Note: This blog assumes that you have rook cluster up and running and few
CephFS and RBD PVC's are created.

```bash
[🎩︎]mrajanna@ceph $]kubectl get pvc
NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
cephfs-pvc   Bound    pvc-88919d42-ecdf-4737-a805-065eacdfd34f   1Gi        RWO            rook-cephfs       68s
rbd-pvc      Bound    pvc-4a4c2fa3-086b-49fc-a1ef-fd8b0768f4b1   1Gi        RWO            rook-ceph-block   79s
```

In the above PVC list we have one RBD and one CephFS PVC, let us first track the
rados omap for RBD

## Track RBD omap data

To get the omap details we need to know the rbd pool in which cephcsi
stores the omap data. let's first get the pool name which is stored in PV CSI
spec

```bash
[🎩︎]mrajanna@ceph $]kubectl get pv pvc-4a4c2fa3-086b-49fc-a1ef-fd8b0768f4b1 -o jsonpath='{.spec.csi}'
{
	"controllerExpandSecretRef": {
		"name": "rook-csi-rbd-provisioner",
		"namespace": "rook-ceph"
	},
	"driver": "rook-ceph.rbd.csi.ceph.com",
	"fsType": "ext4",
	"nodeStageSecretRef": {
		"name": "rook-csi-rbd-node",
		"namespace": "rook-ceph"
	},
	"volumeAttributes": {
		"clusterID": "rook-ceph",
		"imageFeatures": "layering",
		"imageFormat": "2",
		"imageName": "csi-vol-92837648-1431-11eb-8990-0242ac110005",
		"journalPool": "replicapool",
		"pool": "replicapool",
		"radosNamespace": "",
		"storage.kubernetes.io/csiProvisionerIdentity": "1603348800044-8081-rook-ceph.rbd.csi.ceph.com"
	},
	"volumeHandle": "0001-0009-rook-ceph-0000000000000002-92837648-1431-11eb-8990-0242ac110005"
}
```

In the above PV `replicapool` is the journal pool name so let's step into the toolbox
pod which helps us to connect to the ceph cluster

**_NOTE:_** If you are using a standalone ceph cluster you can execute these commands
from your ceph cluster

> Cephcsi internal design created 2 omap mapping
> * one is request ID(PV name) to unique ID mapping
>   * This helps to make cephcsi idempotent even if we get the same request
      we return the existing data.
> * One is uniqueID and image details mapping
>   * This helps cephcsi to extract the image details   when volumeID is passed in
      the request, cephcsi will decode the volumeID and get the omap
      details to extract image/volume name etc.

### RBD PV name and unique ID mapping

PV name is the request name, let's get the unique ID mapped to the request name

```bash
[🎩︎]mrajanna@ceph $]kubectl exec -it rook-ceph-tools-6c984f579-qqh7n sh -nrook-ceph
sh-4.4# rados getomapval csi.volumes.default csi.volume.pvc-4a4c2fa3-086b-49fc-a1ef-fd8b0768f4b1 --pool=replicapool
value (36 bytes) :
00000000  39 32 38 33 37 36 34 38  2d 31 34 33 31 2d 31 31  |92837648-1431-11|
00000010  65 62 2d 38 39 39 30 2d  30 32 34 32 61 63 31 31  |eb-8990-0242ac11|
00000020  30 30 30 35                                       |0005|
00000024
```

`csi.volumes.default` is the object created by cephcsi to store the request
name to unique ID mapping

`92837648-1431-11eb-8990-0242ac110005` is the unique ID mapped to request name
`pvc-4a4c2fa3-086b-49fc-a1ef-fd8b0768f4b1`

### RBD unique ID and omap details mapping

Let's get the list of keys cephcsi stores with this unique ID

```bash
sh-4.4# rados listomapkeys csi.volume.92837648-1431-11eb-8990-0242ac110005 --pool=replicapool
csi.imageid
csi.imagename
csi.volname
```

`csi.volume.92837648-1431-11eb-8990-0242ac110005` is the object cephcsi creates
to store individual image details and its always unique and last part of the
object is the unique ID that we got from Request Name mapping.

For RBD, cephcsi stores 3 keys in the unique object

1. csi.imageid is the key which holds the imageid which is required by cephcsi
   to deletion operation.
2. csi.imagename which holds the RBD image name.
3. csi.volname holds the request name.

Let's get all the values of these keys

**Note:** Am listing all the values stored in the
`csi.volume.92837648-1431-11eb-8990-0242ac110005` object

```bash
sh-4.4# rados listomapvals csi.volume.92837648-1431-11eb-8990-0242ac110005 --pool=replicapool
csi.imageid
value (11 bytes) :
00000000  31 30 39 36 39 35 32 32  61 35 63                 |10969522a5c|
0000000b

csi.imagename
value (44 bytes) :
00000000  63 73 69 2d 76 6f 6c 2d  39 32 38 33 37 36 34 38  |csi-vol-92837648|
00000010  2d 31 34 33 31 2d 31 31  65 62 2d 38 39 39 30 2d  |-1431-11eb-8990-|
00000020  30 32 34 32 61 63 31 31  30 30 30 35              |0242ac110005|
0000002c

csi.volname
value (40 bytes) :
00000000  70 76 63 2d 34 61 34 63  32 66 61 33 2d 30 38 36  |pvc-4a4c2fa3-086|
00000010  62 2d 34 39 66 63 2d 61  31 65 66 2d 66 64 38 62  |b-49fc-a1ef-fd8b|
00000020  30 37 36 38 66 34 62 31                           |0768f4b1|
00000028
```

We have seen how to track the PV -> rados omap mapping for the RBD,let's see how
to track the CephFS rados omap data

## Track CephFS PVC omap data

To get the omap details we need to know the cephfs metadata pool in which
cephcsi stores the omap data. Let's get the filesystem name which is
stored in PV CSI spec

```bash
[🎩︎]mrajanna@ceph $]kubectl get pv pvc-88919d42-ecdf-4737-a805-065eacdfd34f -o jsonpath='{.spec.csi}'
{
	"controllerExpandSecretRef": {
		"name": "rook-csi-cephfs-provisioner",
		"namespace": "rook-ceph"
	},
	"driver": "rook-ceph.cephfs.csi.ceph.com",
	"fsType": "ext4",
	"nodeStageSecretRef": {
		"name": "rook-csi-cephfs-node",
		"namespace": "rook-ceph"
	},
	"volumeAttributes": {
		"clusterID": "rook-ceph",
		"fsName": "myfs",
		"pool": "myfs-data0",
		"storage.kubernetes.io/csiProvisionerIdentity": "1603348799452-8081-rook-ceph.cephfs.csi.ceph.com",
		"subvolumeName": "csi-vol-bff8a308-1431-11eb-b0fd-0242ac110006"
	},
	"volumeHandle": "0001-0009-rook-ceph-0000000000000001-bff8a308-1431-11eb-b0fd-0242ac110006"
}
```

In the above PV `myfs` is the CephFS filesystem name so let's step into the
toolbox pod which helps us to connect to ceph cluster

**Note:** If you are using standalone ceph cluster you can execute these commands
from your ceph cluster

```text
To know the metadata pool for the filesystem, run
sh-4.4# ceph fs ls
name: myfs, metadata pool: myfs-metadata, data pools: [myfs-data0 ]
```

### CephFS PV name and unique ID mapping

PV name is the request name, let's get the unique ID mapped to the request name

```bash
[🎩︎]mrajanna@ceph $]kubectl exec -it rook-ceph-tools-6c984f579-qqh7n sh -nrook-ceph
sh-4.4# rados getomapval csi.volumes.default csi.volume.pvc-88919d42-ecdf-4737-a805-065eacdfd34f --pool=myfs-metadata --namespace=csi
value (36 bytes) :
00000000  62 66 66 38 61 33 30 38  2d 31 34 33 31 2d 31 31  |bff8a308-1431-11|
00000010  65 62 2d 62 30 66 64 2d  30 32 34 32 61 63 31 31  |eb-b0fd-0242ac11|
00000020  30 30 30 36                                       |0006|
```

`csi.volumes.default` is the object created by cephcsi to store the request
name to unique ID mapping, For CephFS cephcsi uses `csi` namespace by
default,for RBD its `default` rados namespace.

`bff8a308-1431-11eb-b0fd-0242ac110006` is the unique ID mapped to request name
`pvc-88919d42-ecdf-4737-a805-065eacdfd34f`

### CephFS unique ID and omap details mapping

Let's get the list of keys cephcsi stores with this unique ID

```bash
sh-4.4# rados listomapkeys csi.volume.bff8a308-1431-11eb-b0fd-0242ac110006 --pool=myfs-metadata --namespace=csi
csi.imagename
csi.volname
```

`csi.volume.bff8a308-1431-11eb-b0fd-0242ac110006` is the object cephcsi creates
to store indivisual image details and its always unique and last part of the
object is the unique ID that we got from Request Name mapping.

For CephFS, cephcsi stores 2 keys in the unique object

1. csi.imagename which holds the CephFS subvolume name.
2. csi.volname holds the request name.

Let's get all the values of these keys

Note:- Am listing all the values stored in the
`csi.volume.bff8a308-1431-11eb-b0fd-0242ac110006` object

```bash
sh-4.4# rados listomapvals csi.volume.bff8a308-1431-11eb-b0fd-0242ac110006 --pool=myfs-metadata --namespace=csi
csi.imagename
value (44 bytes) :
00000000  63 73 69 2d 76 6f 6c 2d  62 66 66 38 61 33 30 38  |csi-vol-bff8a308|
00000010  2d 31 34 33 31 2d 31 31  65 62 2d 62 30 66 64 2d  |-1431-11eb-b0fd-|
00000020  30 32 34 32 61 63 31 31  30 30 30 36              |0242ac110006|
0000002c

csi.volname
value (40 bytes) :
00000000  70 76 63 2d 38 38 39 31  39 64 34 32 2d 65 63 64  |pvc-88919d42-ecd|
00000010  66 2d 34 37 33 37 2d 61  38 30 35 2d 30 36 35 65  |f-4737-a805-065e|
00000020  61 63 64 66 64 33 34 66                           |acdfd34f|
00000028
```
