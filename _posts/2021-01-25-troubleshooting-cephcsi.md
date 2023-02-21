The issue in Provisioning the CephFS/RBD PVC or Mounting the CephFS/RBD PVC to
application pods can happen for Many reasons like **`Network connectivity
between csi pods and ceph`**, **`cluster health issue`**,**`Slow operations`**,
**`kubernetes issue`**, etc. Even the issue can be at the Ceph-CSI layer
itself. Hope the below-documented steps might help you to understand and Debug
the cephcsi issues.

## How to identify network issues

Get the monitor IP from the configmap you have created when starting the
cephcsi pods or from the ceph cluster.

```bash=
sh-4.4# ceph mon dump
dumped monmap epoch 1
epoch 1
fsid ba41ac93-3b55-4f32-9e06-d3d8c6ff7334
last_changed 2021-01-20T12:28:37.972517+0000
created 2021-01-20T12:28:37.972517+0000
min_mon_release 15 (octopus)
0: [v2:10.111.136.166:3300/0,v1:10.111.136.166:6789/0] mon.a
```

**Note:-** `10.111.136.166`is the Monitor IP and `6789` is the monitor port
that will be used by CephCSI to connect to the ceph cluster.

> Ceph Monitors normally listen on port 3300 for the new v2 protocol, and 6789
> for the old v1 protocol. Check if your cluster is listening on v2 or v1
> protocol.

### Check network connectivity on your kubernetes nodes

If you are seeing the issue in Provisioning the PVC then you need to check the
network connectivity from the Provisioner pods.

* CephFS

If it's **CephFS PVC** then you need to check network connectivity from the
**csi-cephfsplugin** container of the **csi-cephfsplugin-provisioner** Pods.

* RBD

If it's **RBD PVC** then you need to check network connectivity from the
**csi-rbdplugin** container of the **csi-rbdplugin-provisioner** Pods.

> Based on the kubernetes nodes count or the configuration there might be 1 or
> more provisioner pods will be running. It's good to validate you can check
> the ceph cluster from all provisioner pods.

```bash=
[🎩︎]mrajanna@localhost $]cat < /dev/tcp/10.111.136.166/6789
ceph v027���'e����'^C
[🎩︎]mrajanna@localhost $] cat < /dev/tcp/10.111.136.166/3300
ceph v2
^C
```

> If you have multiple monitors make sure you check network connectivity for
> all monitor IP's and ports which are passed to cephcsi.

> If the above commands do not return any response then it will be a network
> issue to connect to the ceph cluster.

## check ceph cluster is healhty

Sometimes unhealthy ceph cluster can contribute to the issues in Creating or
mounting the pvc. Please make sure your ceph cluster is healthy

```bash=
sh-4.4# ceph health detail
HEALTH_OK
```

## check for slow ops

Even slow ops in the ceph cluster can contribute to the issues. Please make
sure that No slow Ops are present and the ceph cluster is healthy

```bash=
sh-4.4# ceph -s
  cluster:
    id:     ba41ac93-3b55-4f32-9e06-d3d8c6ff7334
    health: HEALTH_WARN
            30 slow ops, oldest one blocked for 10624 sec, mon.a has slow ops
```

Some tips on what to check.

* Look in the monitor logs
* Look in the OSD logs
  * Check Disk Health
* Check Network Health

> How to debug different issues in the ceph cluster is documented at [**ceph
> troubleshooting**](https://access.redhat.com/documentation/en-us/red_hat_ceph_storage/4/html/troubleshooting_guide/index)

## Pool or subvolumegroup exists in the ceph cluster

* RBD

Make sure the pool you have specified in the
[storageclass.yaml](https://github.com/rook/rook/blob/master/cluster/examples/kubernetes/ceph/csi/rbd/storageclass.yaml#L34)
Exists in the ceph cluster.

Suppose the pool name is mentioned in the storageclass.yaml is replicapool. It
can be verified as below.

```bash=
sh-4.4# ceph osd lspools
1 device_health_metrics
2 replicapool
```

>**If the pool does not exists make sure you create the pool before proceeding.**

* CephFS

Make sure the filesystem and pool you have specified in the
[storageclass.yaml](https://github.com/rook/rook/blob/master/cluster/examples/kubernetes/ceph/csi/cephfs/storageclass.yaml)
Exist in the ceph cluster.

Suppose the filesystem name mentioned in the storageclass.yaml is `myfs`. It
can be verified as below.

```bash=
sh-4.4# ceph fs ls
name: myfs, metadata pool: myfs-metadata, data pools: [myfs-data0 ]
```

> In case, if you have a specified pool name make sure it is available in the
> above data pools list.

## Check cephcsi can create subvolumegroup

If the subvolumegroup is not specified in the cephcsi configmap (where you have
passed the ceph monitor information). cephcsi creates the default
subvolumegroup when name `csi`.

```bash=
sh-4.4# ceph fs subvolumegroup ls myfs
[
    {
        "name": "csi"
    }
]
```

> **This is not applicable for Rook Users.**

Couple for cephfs issue which might help  to debug cephfs network issue
[4251](https://github.com/rook/rook/issues/4251),[6183](https://github.com/rook/rook/issues/6183)
If you don't have any issues with your ceph cluster, we can start debugging the
issue from the CSI side.

## Issue in Provisioning Volume

Most of the time the issue can also exist in the cephcsi or the sidecar
containers we are using in cephcsi.

For CephFS or RBD cephcsi has included number of sidecar containers in the
provisioner pods example **`csi-attacher` `csi-resizer` `csi-provisioner`
`csi-cephfsplugin` `csi-snapshotter` `liveness-prometheus`**

> If it's a CephFS provisioner pod you will see `csi-cephfsplugin` as one of
> the container names. If its an RBD provisioner you will see `csi-rbdplugin`
> as the container name

Let us see the functionality of the sidecar containers we use in cephcsi
provisioner pods.

* **csi-provisioner**

  The external-provisioner is a sidecar container that dynamically provisions
  volumes by calling ControllerCreateVolume and ControllerDeleteVolume
  functions of CSI drivers. More details about external-provisioner can be
  found [here](https://github.com/kubernetes-csi/external-provisioner).

  If any issue exists in PVC create or Delete operation you can check the logs
  of the csi-provisioner sidecar container.

```bash=
kubectl logs po/csi-rbdplugin-provisioner-d857bfb5f-ddctl -c csi-provisioner
```

* **csi-resizer**

  The CSI external-resizer is a sidecar container that watches the Kubernetes
  API server for PersistentVolumeClaim updates and triggers
  **ControllerExpandVolume** operations against a CSI endpoint if the user
  requested more storage on the PersistentVolumeClaim object. More details
  about external-provisioner can be found
  [here](https://github.com/kubernetes-csi/external-resizer).

  If any issue exists in PVC expansion you can check the logs of the
  csi-resizer sidecar container.

```bash=
kubectl logs po/csi-rbdplugin-provisioner-d857bfb5f-ddctl -c csi-resizer
```

* **csi-snapshotter**

  The CSI external-snapshotter sidecar only watches for VolumeSnapshotContent
  create/update/delete events. It will talk to cephcsi containers to create or
  delete snapshots. More details about external-snapshotter can be found
  [here](https://github.com/kubernetes-csi/external-snapshotter).

> In Kubernetes 1.17 when the volume snapshot feature is promoted to beta. In
> Kubernetes 1.20, the feature gate is enabled by default on standard
> Kubernetes deployments and cannot be turned off.

Make sure you have installed the right snapshotter CRD version.

```bash=
kubectl get crd |grep snapshot
volumesnapshotclasses.snapshot.storage.k8s.io    2021-01-25T11:19:38Z
volumesnapshotcontents.snapshot.storage.k8s.io   2021-01-25T11:19:39Z
volumesnapshots.snapshot.storage.k8s.io          2021-01-25T11:19:40Z
```

Check above CRD's is having Right version and you are using the Correct Version
in your **snapshotclass.yaml** or **snapshot.yaml** as well. Or else
**VolumeSnapshot** and **VolumesnapshotContent** might not get created.

The snapshot controller is responsible for creating both VolumeSnapshot and
VolumesnapshotContent object. If the objects are not getting created, You may
need to check the logs of the snapshot-controller pod.

> **Rook** only installs the snapshotter sidecar container, not the
> controller. It is strongly recommended that Kubernetes distributors bundle
> and deploy the controller and CRDs as part of their Kubernetes cluster
> management process (independent of any CSI Driver).

> If your Kubernetes distribution does not bundle the snapshot controller,
> you may manually install these
> [components](https://github.com/kubernetes-csi/external-snapshotter#usage)

If any issue exists in the snapshot Create/Delete operation you can check the
logs of the csi-snapshotter sidecar container.

```bash=
kubectl logs po/csi-rbdplugin-provisioner-d857bfb5f-ddctl -c csi-snapshotter
```

> If you see error like `GRPC error: rpc error: code = Aborted desc = an operation with the given Volume ID 0001-0009-rook-ceph-0000000000000001-8d0ba728-0e17-11eb-a680-ce6eecc894de already exists`.

```go
The issue mostly exists in ceph cluster or network connectivity. If the issue is
in Provisioning the PVC Restarting the Provisioner pods help(for CephFS issue
restart `csi-cephfsplugin-provisioner-xxxxxx` CephFS Provisioner. For RBD, restart
`csi-rbdplugin-provisioner-xxxxxx` pod. If the issue is in mounting the PVC,
restarting the csi-rbdplugin-xxxxx for RBD issue and csi-cephfsplugin-xxxxx pod
for CephFS issue helps sometimes not always.
```

IF the restarting didnt help you can execute below commands from the cephfs/rbd
provisioner pods

* For RBD provisioner pod

```bash
$kubectl exec -t csi-rbdplugin-provisioner -c csi-rbdplugin -- rbd create --size 1024 pool-name/testimage101 -m=10.11.111.11:6789 --user=csi-rbd-provisioner --key="AQC+t/5fqCS2DhAAqheqlRIIMz6mjWQ4g8mUnw==" --debug_ms=20 --debug_rbd=20

$kubectl exec -t csi-rbdplugin-provisioner -c csi-rbdplugin -- rados setomapval test testkey testval -p poolname -m=10.11.111.11:6789 --user=csi-rbd-provisioner --key="xx/xxx+MSr6O5ZMarRHw=="
```

Note:- update monitor IP, csi-rbdplugin-provisioner pod name, pool-name and
key in above command

* For CephFS provisioner pod

```bash
$kubectl exec -t csi-cephfsplugin-provisioner -c csi-cephfsplugin -- ceph fs subvolume ls <fs-name> csi -m=10.11.111.11:6789 --user=csi-cephfs-provisioner --key="AQC+t/5fqCS2DhAAqheqlRIIMz6mjWQ4g8mUnw==" --debug_ms=20

$kubectl exec -t csi-cephfsplugin-provisioner -c csi-cephfsplugin -- rados setomapval test testkey testval -p myfs-metadata --namespace=csi -m=10.11.111.11:6789 --user=csi-cephfs-provisioner --key="xx/xxx+MSr6O5ZMarRHw=="
```

Note:- update monitor IP, csi-cephfsplugin-provisioner pod name,filesystem name, pool-name and
key in above command

## Issue in Mounting the pvc to application pods

When a user requests to create the application pod with PVC. It's a three-step
process

* csi-driver Registration
* Create volume attachment object
* Stage and Publish Volume.

### csi-driver registration

`csi-cephfsplugin-xxxx` or `csi-rbdplugin-xxxx` is a daemonset pod running on
all the nodes where your application gets scheduled. If the plugin pods are not
running on the node where your application is scheduled might cause the issue,
Make sure plugin pods are always running.

Each plugin pod has 2 important container one is `driver-registrar` and
`csi-rbdplugin` or `csi-cephfsplugin`. sometimes we also deploy a
`liveness-prometheus` container.

* **driver-registrar**

  The node-driver-registrar is a sidecar container that registers the CSI
  driver with Kubelet. More datils can be found
  [here](https://github.com/kubernetes-csi/node-driver-registrar)

If any issue exists in attaching the PVC to the application pod check logs from
*driver-registrar* sidecar container in plugin pod where your application pod
is scheduled.

```bash=
$ kubectl logs po/csi-rbdplugin-vh8d5 -c driver-registrar
I0120 12:28:34.231761  124018 main.go:112] Version: v2.0.1
I0120 12:28:34.233910  124018 connection.go:151] Connecting to unix:///csi/csi.sock
I0120 12:28:35.242469  124018 node_register.go:55] Starting Registration Server at: /registration/rook-ceph.rbd.csi.ceph.com-reg.sock
I0120 12:28:35.243364  124018 node_register.go:64] Registration Server started at: /registration/rook-ceph.rbd.csi.ceph.com-reg.sock
I0120 12:28:35.243673  124018 node_register.go:86] Skipping healthz server because port set to: 0
I0120 12:28:36.318482  124018 main.go:79] Received GetInfo call: &InfoRequest{}
I0120 12:28:37.455211  124018 main.go:89] Received NotifyRegistrationStatus call: &RegistrationStatus{PluginRegistered:true,Error:,}
E0121 05:19:28.658390  124018 connection.go:129] Lost connection to unix:///csi/csi.sock.
E0125 07:11:42.926133  124018 connection.go:129] Lost connection to unix:///csi/csi.sock.
```

> You should see `RegistrationStatus{PluginRegistered:true,Error:,}` response
> in the logs to confirm that plugin is registered with kubelet.

> If you see a driver not found an error in the application pod describe
> output. restarting the csi-xxxxplugin-xxx pod on the node helps sometimes.

### Check Issue in Volume attachment

Each provisioner pod also consists of a sidecar container called csi-attacher.

* **csi-attacher**

  The external-attacher is a sidecar container that attaches volumes to nodes
  by calling ControllerPublish and ControllerUnpublish functions of CSI
  drivers. It is necessary because the internal Attach/Detach controller
  running in Kubernetes controller-manager does not have any direct interfaces
  to CSI drivers. More datils can be found
  [here](https://github.com/kubernetes-csi/external-attacher)

If any issue exists in attaching the PVC to the application pod first check the
`volumettachment` object created and also log from *csi-attacher* sidecar
container in provisioner pod.

```bash=
$ kubectl get volumeattachment
NAME                                                                   ATTACHER                        PV                                         NODE       ATTACHED   AGE
csi-75903d8a902744853900d188f12137ea1cafb6c6f922ebc1c116fd58e950fc92   rook-ceph.cephfs.csi.ceph.com   pvc-5c547d2a-fdb8-4cb2-b7fe-e0f30b88d454   minikube   true       4m26s
```

```bash=
kubectl logs po/csi-rbdplugin-provisioner-d857bfb5f-ddctl -c csi-attacher
```

### How to identify stale operations

* CephFS Check for any stale mount commands on the csi-cephfsplugin-xxxx pod on
  the node where your application pod is scheduled.

You need to exec in the `csi-cephfsplugin-xxxx` pod and grep for stale `mount`
operators.

> Identify the csi-cephfsplugin-xxxx pod running on the node where your
> application is scheduled with `kubectl get po -owide` and match the node
> names.

```bash=
$ kubectl exec -it csi-cephfsplugin-tfk2g -c csi-cephfsplugin -- sh
sh-4.4# ps -ef |grep mount
root          67      60  0 11:55 pts/0    00:00:00 grep mount
sh-4.4# ps -ef |grep ceph
root           1       0  0 Jan20 ?        00:00:26 /usr/local/bin/cephcsi --nodeid=minikube --type=cephfs --endpoint=unix:///csi/csi.sock --v=0 --nodeserver=true --drivername=rook-ceph.cephfs.csi.ceph.com --pidlimit=-1 --metricsport=9091 --forcecephkernelclient=true --metricspath=/metrics --enablegrpcmetrics=true
root          69      60  0 11:55 pts/0    00:00:00 grep ceph
```

If any commands are stuck check the **`dmesg`** logs from the node, Restarting
the csi-cephfsplugin pod also might work sometime.

If you don't see any stuck command, Make sure about network connectivity and
ceph health, and slow ops.

* RBD

Check for any stale map/mkfs/mount commands on the csi-rbdplugin-xxxx pod on
the node where your application pod is scheduled.

You need to exec in the `csi-rbdplugin-xxxx` pod and grep for stale operators
like (**`rbd map`**, **`rbd unmap`**, **`mkfs`**, **`mount`** and
**`umount`**).

> Identify the csi-rbdplugin-xxxx pod running on the node where your
> application is scheduled with `kubectl get po -owide` and match the node
> names.

```bash=
$ kubectl exec -it csi-rbdplugin-vh8d5 -c csi-rbdplugin -- sh
sh-4.4# ps -ef |grep map
root     1297024 1296907  0 12:00 pts/0    00:00:00 grep map
sh-4.4# ps -ef |grep mount
root        1824       1  0 Jan19 ?        00:00:00 /usr/sbin/rpc.mountd
ceph     1041020 1040955  1 07:11 ?        00:03:43 ceph-mgr --fsid=ba41ac93-3b55-4f32-9e06-d3d8c6ff7334 --keyring=/etc/ceph/keyring-store/keyring --log-to-stderr=true --err-to-stderr=true --mon-cluster-log-to-stderr=true --log-stderr-prefix=debug  --default-log-to-file=false --default-mon-cluster-log-to-file=false --mon-host=[v2:10.111.136.166:3300,v1:10.111.136.166:6789] --mon-initial-members=a --id=a --setuser=ceph --setgroup=ceph --client-mount-uid=0 --client-mount-gid=0 --foreground --public-addr=172.17.0.6
root     1297115 1296907  0 12:00 pts/0    00:00:00 grep mount
sh-4.4# ps -ef |grep mkfs
root     1297291 1296907  0 12:00 pts/0    00:00:00 grep mkfs
sh-4.4# ps -ef |grep umount
root     1298500 1296907  0 12:01 pts/0    00:00:00 grep umount
sh-4.4# ps -ef |grep unmap
root     1298578 1296907  0 12:01 pts/0    00:00:00 grep unmap
```

If any commands are stuck check the **`dmesg`** logs from the node, Restarting
the csi-rbdplugin pod also might work sometime.

If you don't see any stuck command, Make sure about network connectivity and
ceph health, and slow ops.

### How to check dmesg logs

Checking the *dmesg* logs on the node where pvc mounting is failing or the
csi-rbdplugin container of the csi-rbdplugin-xxxx pod on that node always
helps.

```bash=
dmesg
```

* If nothing helps get the last executed command from the cephcsi pod logs and
  run it manually inside provisioner or plugin pod

```bash=
rbd ls --id=csi-rbd-node -m=10.111.136.166:6789 --key=AQDpIQhg+v83EhAAgLboWIbl+FL/nThJzoI3Fg==
```

> Need to pass the exact user ID, key and monitor IP's and port when executing
> the command.

> If you see error like `GRPC error: rpc error: code = Aborted desc = an operation with the given Volume ID 0001-0009-rook-ceph-0000000000000001-8d0ba728-0e17-11eb-a680-ce6eecc894de already exists`.

```go
The issue mostly exists in ceph cluster or network connectivity.
If the issue is in mounting the PVC, restarting the csi-rbdplugin-xxxxx
for RBD issue and csi-cephfsplugin-xxxxx pod for CephFS issue helps.
```

IF the restarting didnt help you can execute below commands from the cephfs/rbd
plugin pod where you are seeing above error message

* For RBD plugin pod

```bash
$kubectl exec -t csi-rbdplugin-xxx -c csi-rbdplugin -- rbd ls pool-name -m=10.11.111.11:6789 --user=csi-rbd-node --key="AQC+t/5fqCS2DhAAqheqlRIIMz6mjWQ4g8mUnw==" --debug_ms=20 --debug_rbd=20

$kubectl exec -t csi-rbdplugin-xxxx -c csi-rbdplugin -- rados listomapvals test -p replicapool -m=10.11.111.11:6789 --user=csi-rbd-node --key="xx/xxx+MSr6O5ZMarRHw=="
```

Note:- update monitor IP, csi-rbdplugin pod name, pool-name and
key in above command

* For CephFS plugin pod

```bash
$kubectl exec -t csi-cephfsplugin-xxx -c csi-cephfsplugin -- ceph fs subvolume ls <fs-name> csi -m=10.11.111.11:6789 --user=csi-cephfs-node --key="AQC+t/5fqCS2DhAAqheqlRIIMz6mjWQ4g8mUnw==" --debug_ms=20

$kubectl exec -t csi-cephfsplugin-xxx -c csi-cephfsplugin -- rados getomapvals test -p poolname --namespace=csi -m=10.11.111.11:6789 --user=csi-cephfs-node --key="xx/xxx+MSr6O5ZMarRHw=="
```

Note:- update monitor IP, csi-cephfsplugin pod name,filesystem name, pool-name and
key in above command
