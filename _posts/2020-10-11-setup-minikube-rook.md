---
layout: post
title: Install Rook in minikube
comments: true
---

This blog will help you out to install and setup rook in a minikube vm, before
you continue make sure you have installed minikube on your local system

## Install minikube

```bash=
[🎩︎]mrajanna@localhost $]curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
[🎩︎]mrajanna@localhost $]sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

## create kubernetes cluster using minikube

```bash=
[🎩︎]mrajanna@localhost $]minikube start --force --memory="4096" --cpus="2" -b kubeadm --kubernetes-version="v1.19.2" --driver="kvm2" --feature-gates="BlockVolume=true,CSIBlockVolume=true,VolumeSnapshotDataSource=true,ExpandCSIVolumes=true"
```

I will be using `kvm2` in this blog as a vmdriver to  create minikube vm, rook
expects us to have a raw device on the nodes where we are creating a ceph
cluster. using kvm2 we can attach devices to the minikube vm.You can also
select different vm drivers when installing the minikube.

### Create a folder for rook to store the ceph information

```bash=
[🎩︎]mrajanna@localhost $]minikube ssh "sudo mkdir -p /mnt/vda1/var/lib/rook;sudo ln -s /mnt/vda1/var/lib/rook /var/lib/rook"
```

### Add a disk to minikube vm

```bash=
[🎩︎]mrajanna@localhost $]sudo -S qemu-img create -f raw /var/lib/libvirt/images/minikube-box-vm-disk-50G 50G
[🎩︎]mrajanna@localhost $]virsh -c qemu:///system attach-disk minikube --source /var/lib/libvirt/images/minikube-box-vm-disk-50G --target vdb --cache none
[🎩︎]mrajanna@localhost $]virsh -c qemu:///system reboot --domain minikube
```

Assuming you have already installed libvirt virsh etc. Am attaching a device
called `vdb` to the minikube vm to create ceph cluster.

Note:

```note
You can do `minikube ssh` and step inside the minikube vm and check `/dev/vdb` is created.
sometimes the disk wont show up immidiately for that you can need to  start
the minikube again.
```

```bash=
[🎩︎]mrajanna@localhost $]minikube ssh
$ ls /dev/vdb
$ exit
[🎩︎]mrajanna@localhost $]minikube start --force --memory="4096" --cpus="2" -b kubeadm --kubernetes-version="v1.19.2" --driver="kvm2" --feature-gates="BlockVolume=true,CSIBlockVolume=true,VolumeSnapshotDataSource=true,ExpandCSIVolumes=true"
```

to verify the kubernetes cluster  is created you can run below command

```console
[🎩︎]mrajanna@localhost $]minikube kubectl -- cluster-info
```

### Install Rook

As the kubernetes cluster is installed we can start installing the rook now,
for that we need to first download the  rook github project and check out the
release branch.

```bash=
[🎩︎]mrajanna@localhost $]git clone git@github.com:rook/rook.git
[🎩︎]mrajanna@localhost $]git checkout v1.4.5
```

All the kubernetes templates which are required for the rook installation are
localted at `cluster/examples/kubernetes/ceph`

```bash=
[🎩︎]mrajanna@localhost $]cd rook/cluster/examples/kubernetes/ceph
```

To have a complete ceph cluster, we need to install below yaml files

```text
common.yaml         ----> CRD's and RBAC's required for operator
operator.yaml       ----> Operator deployment
cluster-test.yaml   ----> The ceph cluster CRD
pool-test.yaml      ----> block pool CRD
filesystem.yaml     ----> ceph filesystem CRD
toolbox.yaml        ----> toolbox deployment to execute ceph commands
```

Files ending with *_test.yaml* should be used only for testing not for
production.

Lets create all the kubernetes templates to create ceph cluster

```bash=
kubectl create -f common.yaml
kubectl create -f operator.yaml
kubectl create -f cluster-test.yaml
kubectl create -f pool-test.yaml
kubectl create -f filesystem.yaml
kubectl create -f toolbox.yaml
```

Lets wait for few minutes and Verify all the pods are running

```bash=
[🎩︎]mrajanna@localhost $]kubectl get po -nrook-ceph
NAME                                            READY   STATUS      RESTARTS   AGE
csi-cephfsplugin-7llbt                          3/3     Running     0          22h
csi-cephfsplugin-provisioner-5c65b94c8d-tljqd   0/6     Pending     0          22h
csi-cephfsplugin-provisioner-5c65b94c8d-x4kgq   6/6     Running     8          22h
csi-rbdplugin-b6jgs                             3/3     Running     0          22h
csi-rbdplugin-provisioner-569c75558-7twxh       0/6     Pending     0          22h
csi-rbdplugin-provisioner-569c75558-kx69l       6/6     Running     8          22h
rook-ceph-mds-myfs-a-6dd4596558-lt94j           1/1     Running     0          22h
rook-ceph-mds-myfs-b-77986b766d-rlc2v           1/1     Running     0          22h
rook-ceph-mgr-a-75f9c8bd4b-gwvvx                1/1     Running     2          22h
rook-ceph-mon-a-5f85c959bf-gmssg                1/1     Running     0          22h
rook-ceph-operator-6db6f67cd4-7wmkq             1/1     Running     0          22h
rook-ceph-osd-0-77b585d64-5pfcq                 1/1     Running     0          22h
rook-ceph-osd-prepare-minikube-hqqj7            0/1     Completed   0          15h
rook-ceph-tools-6c984f579-m6ccc                 1/1     Running     0          22h
rook-discover-hjj6c                             1/1     Running     0          22h
```

Check ceph filesystem  and block pool is create

```bash=
[🎩︎]mrajanna@localhost $]kubectl -n rook-ceph get cephfilesystems myfs
NAME   ACTIVEMDS   AGE
myfs   1           22h
[🎩︎]mrajanna@localhost $]kubectl -n rook-ceph get  cephblockpools replicapool
NAME          AGE
replicapool   22h
```

Let exec into the ceph toolbox pod and verify `ceph status`, pools and
filesystem  created in ceph.

```bash=
[🎩︎]mrajanna@localhost $]kubectl exec -it rook-ceph-tools-6c984f579-m6ccc sh -nrook-ceph
sh-4.4# ceph -s
  cluster:
    id:     f50de5f0-5d0a-4621-a6fb-04192efdcc79
    health: HEALTH_OK

  services:
    mon: 1 daemons, quorum a (age 22h)
    mgr: a(active, since 112m)
    mds: myfs:1 {0=myfs-b=up:active} 1 up:standby-replay
    osd: 1 osds: 1 up (since 113m), 1 in (since 22h)

  task status:
    scrub status:
        mds.myfs-a: idle
        mds.myfs-b: idle

  data:
    pools:   4 pools, 97 pgs
    objects: 35 objects, 3.2 MiB
    usage:   1.0 GiB used, 49 GiB / 50 GiB avail
    pgs:     97 active+clean

  io:
    client:   1.2 KiB/s rd, 2 op/s rd, 0 op/s wr

sh-4.4# ceph osd lspools
1 device_health_metrics
2 replicapool
3 myfs-metadata
4 myfs-data0
sh-4.4# ceph fs ls
name: myfs, metadata pool: myfs-metadata, data pools: [myfs-data0 ]
sh-4.4#
```

Now we have create ceph cluster using rook in minikube. The commands we have
exectuted is available as a
[shell-script](https://gist.github.com/Madhu-1/2f5db960884671942540f06c599e50c2)
