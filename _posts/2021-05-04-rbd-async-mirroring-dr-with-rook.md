---
layout: post
title: DR failover and failback for RBD Async mirroring with minikube
comments: true
---

This doc assumes that you already set up the rbd mirroring between two clusters.

Please refer to [doc](../setup-rbd-async-mirroring-with-rook) on how
to setup RBD mirroring.

## Set up initial DR components

we need to Enable two new sidecar deployment in the RBD provisioner pods which
will help in achieving rbd mirroring

* OMAP Regenerator -> This is a cephcsi component that regenerates the
  required OMAP data for cephcsi to perform CSI operations after failover.

* Volume Replication operator -> Volume replication operator which exposes
  Replication CRD to perform the RBD async operations like enable and disable mirroring,
  promote, demote, force promote, and resync. Refer [operator](https://github.com/csi-addons/volume-replication-operator) for more details about it.

### Enable DR components

#### Upgrade to Rook v1.6.0+

Rook [v1.6.0](https://github.com/rook/rook/releases/tag/v1.6.0) comes with the
new volume replication support and also cephcsi v3.3.0, Install/Upgrade to Rook
v1.6.0 or higher versions.

#### Re-apply the common.yaml and crds.yaml

```bash=
kubectl apply -f crds.yaml common.yaml
```

> Once we reapply/create the crds.yaml we should see two new CRD's related to
> volume replication

```bash=
$ kubectl get crd |grep volumereplication
volumereplicationclasses.replication.storage.openshift.io   2021-04-28T05:57:57Z
volumereplications.replication.storage.openshift.io         2021-04-28T05:57:57Z
```

As we need to provide Edit `rook-ceph-operator-config` configmap and add below
two configurations to the list

```yaml
apiVersion: v1
data:
  CSI_ENABLE_OMAP_GENERATOR: "true"
  CSI_ENABLE_VOLUME_REPLICATION: "true"
```

```bash=
$kubectl edit cm rook-ceph-operator-config  -nrook-ceph
```

> Once you edit and save the configmap you will see that CSI pods are getting
> recreated, just wait until you see 8 containers in `csi-rbdplugin-provisioner-xxx`
> pod.

```bash=
kubectl get po -nrook-ceph
NAME                                            READY   STATUS      RESTARTS   AGE
csi-cephfsplugin-bkgkt                          3/3     Running     0          5d23h
csi-cephfsplugin-provisioner-6544c5b576-ndf58   6/6     Running     0          5d23h
csi-rbdplugin-provisioner-7465df5764-5g8mj      8/8     Running     0          6m7s
csi-rbdplugin-vpq4k                             3/3     Running     0          5d23h
rook-ceph-mgr-a-54bbfb54b9-pg827                1/1     Running     0          6d
rook-ceph-mon-a-79dfbdcdb6-qwssl                1/1     Running     0          6d
rook-ceph-operator-54cf7487d4-gdp5s             1/1     Running     0          6d
rook-ceph-osd-0-b99c76b7b-6cbnk                 1/1     Running     0          6d
rook-ceph-osd-prepare-minicluster2-57552        0/1     Completed   0          6d
rook-ceph-rbd-mirror-a-d449c66db-2c4hr          1/1     Running     0          5d23h
rook-ceph-tools-78cdfd976c-nm2jx                1/1     Running     0          6d
```

```bash=
kubectl logs po/csi-rbdplugin-provisioner-7465df5764-5g8mj -nrook-ceph
error: a container name must be specified for pod csi-rbdplugin-provisioner-7465df5764-5g8mj, choose one of: [csi-provisioner csi-resizer csi-attacher csi-snapshotter csi-omap-generator volume-replication csi-rbdplugin liveness-prometheus]
```

> In the above logs we can see that `csi-omap-generator` and
> `volume-replication` containers got created in
> `csi-rbdplugin-provisioner-xxx` pods.

**Note** The above steps need to be done on both clusters.

Now the initial bootstrapping is completed.

## Provision storage on cluster1

Create a `storageclass` and `PVC`

### Create storageclass on cluster1

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

### Create Volume Replication class on cluster1

```bash=
$cat <<EOF | kubectl --context=cluster1 apply -f -
apiVersion: replication.storage.openshift.io/v1alpha1
kind: VolumeReplicationClass
metadata:
  name: rbd-volumereplicationclass
spec:
  provisioner: rook-ceph.rbd.csi.ceph.com
  parameters:
    mirroringMode: snapshot
    replication.storage.openshift.io/replication-secret-name: rook-csi-rbd-provisioner
    replication.storage.openshift.io/replication-secret-namespace: rook-ceph
EOF
```

> The volume replication class holds the storage admin information required
for the volume replication operator.

### Create Volume Replication for PVC on cluster1

```bash=
$cat <<EOF | kubectl --context=cluster1 apply -f -
apiVersion: replication.storage.openshift.io/v1alpha1
kind: VolumeReplication
metadata:
  name: pvc-volumereplication2
spec:
  volumeReplicationClass: rbd-volumereplicationclass
  replicationState: primary
  dataSource:
    apiGroup: ""
    kind: PersistentVolumeClaim
    name: rbd-pvc # Name of the PVC to which mirroring to be enabled.
EOF
```

>`replicationState` is the state of the volume being referenced.
  Possible values are `primary`, `secondary`, and `resync`.
* `primary` denotes that the volume is primary
* `secondary` denotes that the volume is secondary
* `resync` denotes that the volume needs to be resynced

> Note the `VolumeReplication` is a namespace scoped object it should be in the namespace as of PVC.

Check VolumeReplication CR status

```bash=
$ kubectl get volumereplication pvc-volumereplication -oyaml
...
spec:
  dataSource:
    apiGroup: ""
    kind: PersistentVolumeClaim
    name: rbd-pvc
  replicationState: primary
  volumeReplicationClass: rbd-volumereplicationclass
status:
  conditions:
  - lastTransitionTime: "2021-05-04T07:39:00Z"
    message: ""
    observedGeneration: 1
    reason: Promoted
    status: "True"
    type: Completed
  - lastTransitionTime: "2021-05-04T07:39:00Z"
    message: ""
    observedGeneration: 1
    reason: Healthy
    status: "False"
    type: Degraded
  - lastTransitionTime: "2021-05-04T07:39:00Z"
    message: ""
    observedGeneration: 1
    reason: NotResyncing
    status: "False"
    type: Resyncing
  lastCompletionTime: "2021-05-04T07:39:00Z"
  lastStartTime: "2021-05-04T07:38:59Z"
  message: volume is marked primary
  observedGeneration: 1
  state: Primary
```

Run commands on toolbox pod on cluster1 to check to mirror is enabled on ceph block pool

```bash=
$kubectl get po --context=cluster1 -nrook-ceph -l  app=rook-ceph-tools
NAME                               READY   STATUS    RESTARTS   AGE
rook-ceph-tools-78cdfd976c-jrhjm   1/1     Running   0          3m53s
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

> You can get the RBD image name from the PV object.

#### Verify that the image is mirrored on the secondary cluster

Run commands on toolbox pod on cluster2

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

### Take a backup of PVC and PV object on cluster1

```bash=
kubectl get pvc rbd-pvc -oyaml >pvc-bkp.yaml
```

```bash=
kubectl get pv/pvc-0b93de3a-3946-4909-a189-f44dba0abc76 -oyaml >pv_bkp.yaml
```

#### Remove the claimRef on the backed up PV objects in  yaml files

sample claimRef in PV object

```yaml=
apiVersion: v1
kind: PersistentVolume
metadata:
  ...
  name: pvc-0b93de3a-3946-4909-a189-f44dba0abc76
  resourceVersion: "2516"
  selfLink: /api/v1/persistentvolumes/pvc-0b93de3a-3946-4909-a189-f44dba0abc76
  uid: 0b1a1560-2c0b-4906-8059-d6be95954bc0
spec:
  claimRef:
    apiVersion: v1
    kind: PersistentVolumeClaim
    name: rbd-pvc
    namespace: default
    resourceVersion: "2512"
    uid: 0b93de3a-3946-4909-a189-f44dba0abc76
 ...
```

> Remove the claimRef in all PV backup's

```bash=
vi pv_bkp.yaml
```

## Planned Storage Migration

### Failover

Follow the Below steps for planned migration when you want to move the workload from one cluster to another cluster

* Scale down all the application pods which are using the mirrored PVC
* Take a back up of PVC and PV object from the cluster1
  * This can be done using some backup tools like velero also
* update replicationState to secondary in VolumeReplication CR at Primary Site. When the operator sees this change, it will pass the information down to the driver via GRPC request to mark the dataSource as secondary.
* If you are manually recreating the PVC and PV on the secondary cluster. remove the claimRef section in the PV objects.
* Recreate the storageclass, PVC, and PV objects on the cluster2
  * As you are creating the static binding between PVC and PV, a new PV won't be created here, the PVC will get bind to the existing PV.
* Create the VolumeReplicationClass on the cluster2
* Create the VolumeReplications for all the PVC's for which mirroring is enabled
  * `replicationState` should be `primary` for all the PVC's on the cluster1
* Check whether the image is primary on cluster2 by looking at toolbox or VolumeReplication CR status
* Once the Image is marked as Primary create the application to use the PVC
In the case of planned migration,

#### Failback

Once the planned work is done and you want to failback, follow the above Failover steps again. the only difference, in this case, is we don't need to recreate anything we just need to change `replicationState` from `secondary` to `primary` in current cluster1.

## Storage Disaster Recovery

### Failover

In case of disaster recovery, create VolumeReplication CR at the Secondary Site. Since the connection to the Primary Site is lost, the operator automatically sends a GRPC request down to the driver to forcefully mark the dataSource as primary.

* If you are manually recreating the PVC and PV on the secondary cluster. remove the claimRef section in the PV objects.
* Recreate the storageclass, PVC, and PV objects on the cluster2
  * As you are creating the static binding between PVC and PV, a new PV won't be created here, the PVC will get bind to the existing PV.
* Create the VolumeReplicationClass and VolumeReplication CR on the cluster2
* Check whether the image is primary on cluster2 by looking at toolbox or VolumeReplication CR status
* Once the Image is marked as Primary create the application to use the PVC
In the case of planned migration,

### Failback

Once the failed cluster is recovered and you want to failback, follow the below steps

* Update the VolumeReplication CR `replicationState` state from `primary` to `secondary` in cluster1
* Scale down the applications on the cluster2
* Update the VolumeReplication CR `replicationState` state from `primary` to `secondary` in cluster2
* On the cluster1 verify that the VR status is marked as volume ready to use
* Once the volume is marked to ready to use, change the `replicationState` state from `secondary` to `primary` in cluster1
* Scale up the applications again on cluster1
