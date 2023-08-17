# Ceph-csi

## Ceph

#### Connect to ceph

Refer https://docs.ceph.com/en/quincy/cephadm/client-setup/#config-file-setup 

We can exec into the toolbox pod, and check file in `/etc/ceph/ceph.conf`. The content looks like this:

```
$ cat /etc/ceph/ceph.conf
[global]
mon_host = 10.100.13.5:6789,10.100.13.2:6789,10.100.13.4:6789

[client.admin]
keyring = /etc/ceph/keyring

$ cat /etc/ceph/keyring
[client.admin]
key = AQBZk95khq7oLBAAZvpbe2rLkeq6PLqL2KdJPQ==

$ ls -lrt /etc/ceph/
total 8
-rw-r--r-- 1 2016 2016  62 Aug 17 21:39 keyring
-rw-r--r-- 1 2016 2016 115 Aug 17 21:40 ceph.conf
```

Then we can put them in the same location, and install ceph client by installing ceph-common package.

After ceph client is installed and `/etc/ceph/ceph.conf` is ready, we can use `ceph status` to validate the connection.

#### Get user key

```
ceph auth ls

ceph auth get client.admin
```

Refer: https://docs.ceph.com/en/reef/rados/operations/user-management/#managing-users 

If using rook install ceph, by default below users are already created:

```
client.csi-cephfs-node
        key: AQDQk95kUGRDKhAA1KXAF07IfmRx0GuuFuYKbg==
        caps: [mds] allow rw
        caps: [mgr] allow rw
        caps: [mon] allow r
        caps: [osd] allow rw tag cephfs *=*
client.csi-cephfs-provisioner
        key: AQDQk95kRwXiFhAAFm2OLx9V4nVVCgq5haNXtQ==
        caps: [mgr] allow rw
        caps: [mon] allow r
        caps: [osd] allow rw tag cephfs metadata=*
client.csi-rbd-node
        key: AQDQk95ky7qHBRAAqcHVrS+AluJgia4U1Obg6A==
        caps: [mgr] allow rw
        caps: [mon] profile rbd
        caps: [osd] profile rbd
client.csi-rbd-provisioner
        key: AQDPk95kRQiSLRAAO2Wt8mn29/5urbJCeKpDLQ==
        caps: [mgr] allow rw
        caps: [mon] profile rbd, allow command 'osd blocklist'
        caps: [osd] profile rbd
```

Related secret:

 * rbd

```
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph # namespace:cluster
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph # namespace:cluster
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph # namespace:cluster
```

 * cephfs

```
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph # namespace:cluster
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph # namespace:cluster
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-cephfs-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph # namespace:cluster
```

## Rook

As of writing on 08/17/2023, latest rook version is `1.12`.

Rook supports 4 kinds of clusters. Host, PVC, Stretch and External. 

When using Host storage cluster, usually in bare metal scenario, we should not format the disk. Ironic can help clean the disk in deprovision process.

When using PVC storage cluster, usually in cloud scenario, we need to use `local-storage`:

```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

Refer to https://kubernetes.io/docs/concepts/storage/storage-classes/#local understand what it means by `kubernetes.io/no-provisioner`.

We need to create PV like this:

```
---
kind: PersistentVolume
apiVersion: v1
metadata:
  name: local0-0
spec:
  storageClassName: local-storage
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  # PV for mon must be a filesystem volume.
  volumeMode: Filesystem
  local:
    # If you want to use dm devices like logical volume, please replace `/dev/sdb` with their device names like `/dev/vg-name/lv-name`.
    path: /dev/sdb
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - host0
```

#### Deploy Rook and Ceph in the same cluster

```
$ git clone --single-branch --branch v1.12.2 https://github.com/rook/rook.git
cd rook/deploy/examples
kubectl create -f crds.yaml -f common.yaml -f operator.yaml
kubectl create -f cluster.yaml
```
From https://rook.io/docs/rook/v1.12/Getting-Started/quickstart/#tldr 

By default, Rook deploy both rbd and cephfs csi plugins.

Some changes we might want to make for the `cluster.yaml`:

* `dataDirHostPath`
* Enable rook in mgr module like below:
```
  mgr:
    count: 2
    allowMultiplePerNode: false
    modules:
      - name: pg_autoscaler
        enabled: true
      # Added by me to fix a `ceph status` warning issue
      - name: rook
        enabled: true
```
* Use host network for ceph monitor, because I want to connect to this ceph cluster from outside of the K8s cluster, using host network seems simpler.
```
  network:
    # enable host networking
    provider: host
```


#### Use ceph-csi directly connect to external ceph cluster

As of writing, the latest ceph-csi version is `v3.9.0`.

Ceph-csi supports 2 plugins: `rbd` and `cephfs`.

##### rbd

Follow link: https://github.com/ceph/ceph-csi/blob/release-v3.9/docs/deploy-rbd.md#deployment-with-kubernetes 

Special tips:

* Do remember to checkout to the exact tag before apply any manifest so that the code is using correct image tag.
* It is safe to blindly apply all those yaml first, then replace csi-config-map when we have the ceph cluster detail.
* `csidriver.yaml`, `csi-provisioner-rbac.yaml`, `csi-nodeplugin-rbac.yaml`, `csi-config-map.yaml`, `csi-rbdplugin-provisioner.yaml` and `csi-rbdplugin.yaml` are in the same directory, only `ceph-conf.yaml` is in the upper directory.
* We don't have to change anything to `ceph-conf.yaml`, we can replace csi-config-map like mentioned in https://github.com/ceph/ceph-csi/blob/release-v3.9/examples/README.md#creating-csi-configuration . My csi-config-map looks like this (In my experiment, to use cephfs plugin, we have to use v1 protocol, the `6789` port, which is also what rook using by default):

```
$ kubectl get cm ceph-csi-config -oyaml
apiVersion: v1
data:
  cluster-mapping.json: '[]'
  config.json: |-
    [
      {
        "clusterID": "rook-ceph",
        "monitors": [
          "10.100.13.4:6789",
          "10.100.13.5:6789",
          "10.100.13.2:6789"
        ]
      }
    ]
kind: ConfigMap
metadata:
  name: ceph-csi-config
  namespace: default
```

* Do remember to create a `ceph-csi-encryption-kms-config` configmap like below:

```
---
apiVersion: v1
kind: ConfigMap
data:
  config.json: |-
    {}
metadata:
  name: ceph-csi-encryption-kms-config
```

After the rbdplugin pod is running, we can start to create secret, storageclass, pvc and pod. Note that, those pods are running in default namespace by default.

* In secret, we need to provide `userID` and `userKey`. Get from `ceph auth ls`, or create new one.
* In StorageClass, we need to provide `clusterID` and `pool`. In rook provisioned cluster, they are `rook-ceph` and `replicapool`.

##### cephfs

Similar steps apply to `cephfs`.

* Note that cephfs use slight different rbac to run plugin pods, so it applies another 2 rbac yamls.
* We can compare 4 different secrets in rook provisioned ceph cluster in below:

```
$ kubectl -n rook-ceph get secret rook-csi-rbd-provisioner -ojsonpath='{.data}' | jq
{
  "userID": "Y3NpLXJiZC1wcm92aXNpb25lcg==",
  "userKey": "QVFEUGs5NWtSUWlTTFJBQU8yV3Q4bW4yOS81dXJiSkNlS3BETFE9PQ=="
}
$ kubectl -n rook-ceph get secret rook-csi-rbd-provisioner -ojsonpath='{.data.userID}' | base64 -d
csi-rbd-provisioner

$ kubectl -n rook-ceph get secret rook-csi-rbd-node -ojsonpath='{.data}' |
 jq
{
  "userID": "Y3NpLXJiZC1ub2Rl",
  "userKey": "QVFEUWs5NWt5N3FIQlJBQXFjSFZyUytBbHVKZ2lhNFUxT2JnNkE9PQ=="
}
$ kubectl -n rook-ceph get secret rook-csi-rbd-node  -ojsonpath='{.data.userID}' | base64 -d
csi-rbd-node

$ kubectl -n rook-ceph get secret rook-csi-cephfs-provisioner -ojsonpath='
{.data}' | jq
{
  "adminID": "Y3NpLWNlcGhmcy1wcm92aXNpb25lcg==",
  "adminKey": "QVFEUWs5NWtSd1hpRmhBQUZtMk9MeDlWNG5WVkNncTVoYU5YdFE9PQ=="
}
$ kubectl -n rook-ceph get secret rook-csi-cephfs-provisioner  -ojsonpath=
'{.data.adminID}' | base64 -d
csi-cephfs-provisioner

$ kubectl -n rook-ceph get secret rook-csi-cephfs-node -ojsonpath='{.data}
' | jq
{
  "adminID": "Y3NpLWNlcGhmcy1ub2Rl",
  "adminKey": "QVFEUWs5NWtVR1JES2hBQTFLWEFGMDdJZm1SeDBHdXVGdVlLYmc9PQ=="
}
$ kubectl -n rook-ceph get secret rook-csi-cephfs-node  -ojsonpath='{.data.adminID}' | base64 -d
csi-cephfs-node
```

* In secret, we need to provide `adminID` and `adminKey`. Get from `ceph auth ls`, or create new one.
* In StorageClass, we need to provide `clusterID`, `fsName` and `pool`. In rook provisioned cluster, they are `rook-ceph`, `myfs` and `myfs-replicated`.

## Snapshot

* Snapshot is better than filesystem copy unless you can freeze application
* Snopshot is crash consistent but not necessary application consistent unless you can flush (ie. Kafka virtual memory)

* CSI let you create snapshot with K8s API
* Any application that want to manage backup don't need anymore secrets to your storage layer
* Some precautions must be taken because the deletion of this objects are easy and results in the deletion of data

Limitations:

* Snap is often not enough for application consistency
* Snap are not durable backup
* We need more than backup but backup policies
* Configuration management, we want to capture the whole namespace not only the data
* API are good but not for everybody


## External Ceph cluster


1. Create users and keys

```
toolbox=$(kubectl get pod -l app=rook-ceph-tools -n rook-ceph -o jsonpath='{.items[*].metadata.name}')
kubectl -n rook-ceph cp create-external-cluster-resources.py $toolbox:/e
tc/ceph/

bash-4.4$ python3 /etc/ceph/create-external-cluster-resources.py --rbd-data-pool-name replicapool --cephfs-filesystem-name myfs --namespace rook-ceph-external --format bash
export NAMESPACE=rook-ceph-external
export ROOK_EXTERNAL_FSID=777e8f8a-5fbe-407d-8b0f-cbabb2f0b8cb
export ROOK_EXTERNAL_USERNAME=client.healthchecker
export ROOK_EXTERNAL_CEPH_MON_DATA=a=10.100.13.5:6789
export ROOK_EXTERNAL_USER_SECRET=AQCe2+NkU/2QBRAA+Tv2bEZzP2BObNjsBWUyEQ==
export ROOK_EXTERNAL_DASHBOARD_LINK=https://10.100.13.4:8443/
export CSI_RBD_NODE_SECRET=AQDQk95ky7qHBRAAqcHVrS+AluJgia4U1Obg6A==
export CSI_RBD_NODE_SECRET_NAME=csi-rbd-node
export CSI_RBD_PROVISIONER_SECRET=AQDPk95kRQiSLRAAO2Wt8mn29/5urbJCeKpDLQ==
export CSI_RBD_PROVISIONER_SECRET_NAME=csi-rbd-provisioner
export CEPHFS_POOL_NAME=myfs-replicated
export CEPHFS_METADATA_POOL_NAME=myfs-metadata
export CEPHFS_FS_NAME=myfs
export CSI_CEPHFS_NODE_SECRET=AQDQk95kUGRDKhAA1KXAF07IfmRx0GuuFuYKbg==
export CSI_CEPHFS_PROVISIONER_SECRET=AQDQk95kRwXiFhAAFm2OLx9V4nVVCgq5haNXtQ==
export CSI_CEPHFS_NODE_SECRET_NAME=csi-cephfs-node
export CSI_CEPHFS_PROVISIONER_SECRET_NAME=csi-cephfs-provisioner
export MONITORING_ENDPOINT=10.100.13.4
export MONITORING_ENDPOINT_PORT=9283
export RBD_POOL_NAME=replicapool
export RGW_POOL_PREFIX=default
```


2. Apply crds, operator, etc.

```
kubectl create -f common.yaml -f crds.yaml -f operator.yaml

kubectl create -f common-external.yaml -f cluster-external.yaml
```

3. Import source data

```
$ cat import.sh
#!/bin/bash

export NAMESPACE=rook-ceph-external
export ROOK_EXTERNAL_FSID=777e8f8a-5fbe-407d-8b0f-cbabb2f0b8cb
export ROOK_EXTERNAL_USERNAME=client.healthchecker
export ROOK_EXTERNAL_CEPH_MON_DATA=a=10.100.13.5:6789
export ROOK_EXTERNAL_USER_SECRET=AQCe2+NkU/2QBRAA+Tv2bEZzP2BObNjsBWUyEQ==
export ROOK_EXTERNAL_DASHBOARD_LINK=https://10.100.13.4:8443/
export CSI_RBD_NODE_SECRET=AQDQk95ky7qHBRAAqcHVrS+AluJgia4U1Obg6A==
export CSI_RBD_NODE_SECRET_NAME=csi-rbd-node
export CSI_RBD_PROVISIONER_SECRET=AQDPk95kRQiSLRAAO2Wt8mn29/5urbJCeKpDLQ==
export CSI_RBD_PROVISIONER_SECRET_NAME=csi-rbd-provisioner
export CEPHFS_POOL_NAME=myfs-replicated
export CEPHFS_METADATA_POOL_NAME=myfs-metadata
export CEPHFS_FS_NAME=myfs
export CSI_CEPHFS_NODE_SECRET=AQDQk95kUGRDKhAA1KXAF07IfmRx0GuuFuYKbg==
export CSI_CEPHFS_PROVISIONER_SECRET=AQDQk95kRwXiFhAAFm2OLx9V4nVVCgq5haNXtQ==
export CSI_CEPHFS_NODE_SECRET_NAME=csi-cephfs-node
export CSI_CEPHFS_PROVISIONER_SECRET_NAME=csi-cephfs-provisioner
export MONITORING_ENDPOINT=10.100.13.4
export MONITORING_ENDPOINT_PORT=9283
export RBD_POOL_NAME=replicapool
export RGW_POOL_PREFIX=default

. import-external-cluster.sh
```


4. Check cluster status

```
$ kubectl -n rook-ceph-external  get CephCluster
NAME                 DATADIRHOSTPATH   MONCOUNT   AGE   PHASE       MESSAGE                          HEALTH      EXTERNAL   FSID
rook-ceph-external                                14m   Connected   Cluster connected successfully   HEALTH_OK   true       777e8f8a-5fbe-407d-8b0f-cbabb2f0b8cb
```

#### How to generate secret for external installed ceph cluster

```
#!/bin/bash

fsid=$(ceph fsid)
name=$(ceph quorum_status --format json | jq -r '.quorum_leader_name')
addr=$(ceph quorum_status --format json | jq -r --arg name "${name}" '.monmap.mons[] | select(.name==$name)' | jq -r '.public_addr' | cut -d'/' -f1)
dashboard_link=$(ceph mgr services --format json | jq -r '.dashboard')
prometheus_host=$(ceph mgr services --format json | jq -r '.prometheus' | cut -d'/' -f3 | cut -d':' -f1)
prometheus_port=$(ceph mgr services --format json | jq -r '.prometheus' | cut -d'/' -f3 | cut -d':' -f2)
healthchecker_secret=$(ceph auth get-key client.healthchecker)
rbd_node_secret=$(ceph auth get-key client.csi-rbd-node)
rbd_provisioner_secret=$(ceph auth get-key client.csi-rbd-provisioner)

echo "export NAMESPACE=rook-ceph-external"
echo "export ROOK_EXTERNAL_FSID=${fsid}"
echo "export ROOK_EXTERNAL_CEPH_MON_DATA=$name=$addr"
echo "export ROOK_EXTERNAL_DASHBOARD_LINK=${dashboard_link}"
echo "export MONITORING_ENDPOINT=${prometheus_host}"
echo "export MONITORING_ENDPOINT_PORT=${prometheus_port}"
echo "export RBD_POOL_NAME=fortirbd"
echo "export ROOK_EXTERNAL_USERNAME=client.healthchecker"
echo "export ROOK_EXTERNAL_USER_SECRET=${healthchecker_secret}"
echo "export CSI_RBD_NODE_SECRET_NAME=csi-rbd-node"
echo "export CSI_RBD_NODE_SECRET=${rbd_node_secret}"
echo "export CSI_RBD_PROVISIONER_SECRET_NAME=csi-rbd-provisioner"
echo "export CSI_RBD_PROVISIONER_SECRET=${rbd_provisioner_secret}"
```

The output:

```
export NAMESPACE=rook-ceph-external
export ROOK_EXTERNAL_FSID=c2a079a0-41d8-11ee-9adc-5f107cccb698
export ROOK_EXTERNAL_CEPH_MON_DATA=node1=10.106.58.1:6789
export ROOK_EXTERNAL_DASHBOARD_LINK=https://node1:8443/
export MONITORING_ENDPOINT=node1
export MONITORING_ENDPOINT_PORT=9283
export RBD_POOL_NAME=replicapool
export ROOK_EXTERNAL_USERNAME=client.healthchecker
export ROOK_EXTERNAL_USER_SECRET=AQDKDe1k6oM8FhAAdiTwtru1fq/OAkuZHFyP+w==
export CSI_RBD_NODE_SECRET_NAME=csi-rbd-node
export CSI_RBD_NODE_SECRET=AQAkTelkpcZfHRAA76JysGsLArXqm0VXas2ONg==
export CSI_RBD_PROVISIONER_SECRET_NAME=csi-rbd-provisioner
export CSI_RBD_PROVISIONER_SECRET=AQBQTelkSA6aGxAA8Oe0fQ9y4zR4oInI0rXP7g==
```

* You may need to replace the hostname `node1` with a real IP.

## E2E


For go-ceph to work:

```
sudo apt-get install -y libcephfs-dev librbd-dev librados-dev
```


## Troubleshooting

###### cephfs: pod pvc mount error - unable to get monitor info from DNS SRV with service name: ceph-mon

Fix: Use v1 protocol `6789` instead of v2 protocol `3300`


###### HEALTH_WARN clock skew detected on mon.b

```
### Install NTP server on 10.100.1.21
sudo apt install ntp
### Use below server from https://www.ntppool.org/zone/us 
pool 0.us.pool.ntp.org iburst
pool 1.us.pool.ntp.org iburst
pool 2.us.pool.ntp.org iburst
pool 3.us.pool.ntp.org iburst
### Restart NTP server
sudo systemctl restart ntp

### Install client
sudo apt install ntpdate
### Sync time
sudo ntpdate 10.100.1.21
```

