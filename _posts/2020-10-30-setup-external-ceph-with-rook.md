---
layout: post
title: Set up external ceph cluster with Rook
comments: true
---

This blog will help you out understand the various configuration we need to do
to manage the external ceph cluster with Rook.

## Checkout released Rook branch

```bash=
[🎩︎]mrajanna@localhost $]git clone https://github.com/rook/rook
[🎩︎]mrajanna@localhost $]git checkout v1.4.6
[🎩︎]mrajanna@localhost $]cd rook/cluster/examples/kubernetes/ceph
```

Let's step into the ceph directory as we are more intrested in configuring the
Rook for ceph

## Check network connectivity on your kubernetes nodes

```bash=
[🎩︎]mrajanna@localhost $]cat < /dev/tcp/192.168.39.101/6789
ceph v027���'e����'^C
[🎩︎]mrajanna@localhost $] cat < /dev/tcp/192.168.39.101/3300
ceph v2
^C
```

> Press ctrl+c as the command didn't return.

## Create Required RBAC for Rook

```bash=
[🎩︎]mrajanna@localhost $]kubectl create -f common.yaml
```

## Create operator deployment

```bash=
[🎩︎]mrajanna@localhost $]kubectl create -f operator.yaml
```

> Verify Rook operator is running

 ```bash=
[🎩︎]mrajanna@localhost $]kuberc get po
NAME                                READY   STATUS    RESTARTS   AGE
rook-ceph-operator-86756d44-vdr8b   1/1     Running   0          3m20s
rook-discover-sfjrf                 1/1     Running   0          2m47s
```

## Importing external ceph cluster

### Create RBAC for external ceph cluster

> If Rook is not managing any existing cluster in the `rook-ceph` namespace do:
> kubectl create -f common.yaml
> kubectl create -f operator.yaml
> kubectl create -f cluster-external.yaml (you need to change the namespace to `rook-ceph`)

> If there is already a cluster managed by Rook in `rook-ceph` then do:
> kubectl create -f common-external.yaml
> kubectl create -f cluster-external-management.yaml

In my case Rook is not managing any ceph cluster in `rook-ceph` namespace i
will create both `common-external.yaml` and `cluster-external-management.yaml`

```bash=
[🎩︎]mrajanna@localhost $]kubectl create -f common-external.yaml
```

### Import external ceph cluster

Export few pieces of information required for importing external ceph cluster

```bash=
export NAMESPACE=rook-ceph-external ---> Namespace where we are planning use of external ceph cluster
export ROOK_EXTERNAL_FSID=db747e90-2ede-4867-9aee-ba233aa1db55 ---> Run `ceph fsid` on your ceph cluster to get this
export ROOK_EXTERNAL_ADMIN_SECRET=AQA2cSJfOMblMRAAeHroW3THSZukFGtpkIhZ1w== ---> Run `ceph auth get-key client.admin`
on external ceph cluster
export ROOK_EXTERNAL_CEPH_MON_DATA=mon.ceph-node1=192.168.39.101:6789 ---> Run `ceph mon dump` to get the list of monitors(passing one Monitor IP should be enough)
```

>Make sure you pass correct monitor name, Provided an example below
> `mon.ceph-node1` is the monitor name for `192.168.39.101:6789`

```bash=
[🎩︎]mrajanna@localhost $]ceph mon dump
dumped monmap epoch 1
epoch 1
fsid fbafbb58-51d3-4f44-bedd-b3b728bc5766
last_changed 2020-10-27 09:48:52.247843
created 2020-10-27 09:48:52.247843
min_mon_release 14 (nautilus)
0: [v2:192.168.39.101:3300/0,v1:192.168.39.101:6789/0] mon.ceph-node1
1: [v2:192.168.39.102:3300/0,v1:192.168.39.102:6789/0] mon.ceph-node2
2: [v2:192.168.39.103:3300/0,v1:192.168.39.103:6789/0] mon.ceph-node3

```

```bash=
[🎩︎]mrajanna@localhost $] bash import-external-cluster.sh
```

### Create ceph cluster CRD

As we have created  secrets required for the external ceph cluster, Let's create
ceph cluster CRD.

```yaml
[🎩︎]mrajanna@localhost $] cat cluster-external-management.yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph-external
  namespace: rook-ceph-external
spec:
  external:
    enable: true -> set this to false if you want Rook to manage your external ceph cluster
  dataDirHostPath: /var/lib/rook
  # providing an image is required, if you want to create other CRs (rgw, mds, nfs)
  # In latest Rook release we dont need to provide cephVersion we can skip this one.
  cephVersion:
    image: ceph/ceph:v14.2.12 # Should match external cluster version
```

```bash=
[🎩︎]mrajanna@localhost $]kubectl create -f cluster-external-management.yaml
```

Let us check the status of the externa ceph cluster

```bash=
[🎩︎]mrajanna@localhost $]kubectl get cephcluster -nrook-ceph-external
NAME                 DATADIRHOSTPATH   MONCOUNT   AGE   PHASE        MESSAGE                 HEALTH
rook-ceph-external   /var/lib/rook                16s   Connecting   Cluster is connecting
```

> If the status is in the `Connecting` Phase for a longtime please check Rook
> operator pod which will be running in `rook-ceph` namespace.

If you don't see any useful logs in the operator pod, increase the log level of
the Rook operator deployment.

```bash=
[🎩︎]mrajanna@localhost $]kubectl edit deployment rook-ceph-operator -nrook-ceph
```

> Change `ROOK_LOG_LEVEL` from `INFO` to `DEBUG`

```yaml
- name: ROOK_LOG_LEVEL
  value: DEBUG
```

And watch for the operator logs again of any issue

> 2020-10-30 12:25:40.271439 E | cephclient: ceph username is empty
2020-10-30 12:25:40.275572 D | op-config: CephCluster "rook-ceph-external" status: "Failure". "Failed to configure external ceph cluster"

If you see an error like `ceph username` we need to remove the unwanted entry in
the secret created from the `import-external-cluster.sh`

```bash=
[🎩︎]mrajanna@localhost $]kubectl edit secret rook-ceph-mon -nrook-ceph-external
```

```yaml
apiVersion: v1
data:
  admin-secret: QVFBMmNTSmZPTWJsTVJBQWVIcm9XM1RIU1p1a0ZHdHBrSWhaMXc9PQ==
  ceph-secret: ""
  ceph-username: ""
  cluster-name: cm9vay1jZXBoLWV4dGVybmFs
  fsid: ZGI3NDdlOTAtMmVkZS00ODY3LTlhZWUtYmEyMzNhYTFkYjU1
  mon-secret: bW9uLXNlY3JldA==
kind: Secret
```

As the secret contains empty `ceph-username` and `ceph-secret` the ceph cluster
is not getting connected just remove those 2 entries from the secret and save it.

Start Watching for the operator pod log again, to see if there are any other
issues

We have got one more issue in Rook operator related to updating secret failure

> 2020-10-30 12:32:55.493617 E | ceph-cluster-controller: failed to reconcile.
> failed to reconcile cluster "rook-ceph-external": failed to configure
> external ceph cluster: failed to create csi kubernetes secrets: failed to
> create kubernetes csi secret: failed to create kubernetes secret
> map["userID":"csi-rbd-provisioner"
> "userKey":"AQAMBphfoeniNhAAtE7ZO00ZPBiJLwHU1hZnAw=="] for cluster
> "rook-ceph-external": failed to update secret for rook-csi-rbd-provisioner:
> Secret "rook-csi-rbd-provisioner" is invalid: type: Invalid value:
> "kubernetes.io/rook": field is immutable

To fix this issue lets delete all the secrets created by
`import-external-cluster.sh` script and let `Rook` create required secrets

```bash=
[🎩︎]mrajanna@localhost $]kubectl delete secret rook-csi-cephfs-node rook-csi-cephfs-provisioner rook-csi-rbd-node rook-csi-rbd-provisioner -nrook-ceph-external
secret "rook-csi-cephfs-node" deleted
secret "rook-csi-cephfs-provisioner" deleted
secret "rook-csi-rbd-node" deleted
secret "rook-csi-rbd-provisioner" deleted
```

Once we delete the secrets, let's restart the `Rook` operator pod

```bash=
[🎩︎]mrajanna@localhost $]kubectl delete po/rook-ceph-operator-675fdbc9d9-g6mjm -nrook-ceph
```

Start Watching for the operator pod log again, to see if there are any other
issues

Meanwhile, start checking the `cephclusters` status

```bash=
[🎩︎]mrajanna@localhost $]kubectl get cephclusters.ceph.rook.io -nrook-ceph-external
NAME                 DATADIRHOSTPATH   MONCOUNT   AGE   PHASE       MESSAGE                          HEALTH
rook-ceph-external   /var/lib/rook                32m   Connected   Cluster connected successfully   HEALTH_OK
```

Wow!!! Now Rook is connected to the external ceph cluster.
