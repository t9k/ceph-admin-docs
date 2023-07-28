# K8s 集成

通过 [Ceph CSI](https://github.com/ceph/ceph-csi/)，K8s 可以利用 Ceph 作为 PVC 的存储提供者（Provisioner）。


## 配置 Ceph CSI

首先，在 Ceph 集群中创建一个名为 demo-fs 的 Ceph File System 和一个名为 demo-user 的 Ceph 用户，在管理节点中执行以下命令即可：


```
$ sudo ceph fs volume create demo-fs
$ sudo ceph auth get-or-create client.demo-user mon 'allow r' \
  osd 'allow rw tag cephfs *=*' mgr 'allow rw' mds 'allow rw'
[client.demo-user]
	key = AQAhuZBjSuS9AxBD77AvVjr+vAg9zhjRK7NR+g==
```


注意上述生成的 key 值后面会用到。

然后，查看 Ceph 集群的一些相关信息，在管理节点中执行以下命令即可：


```
$ sudo ceph mon dump
epoch 2
fsid 47f20cbc-914f-24ed-93dc-9f2800951ba2
last_changed 2023-01-12T06:01:55.659954+0000
created 2023-01-11T01:30:12.337519+0000
min_mon_release 17 (quincy)
election_strategy: 1
0: [v2:10.0.0.1:3300/0,v1:10.0.0.1:6789/0] mon.ds01
1: [v2:10.0.0.2:3300/0,v1:10.0.0.2:6789/0] mon.e01
dumped monmap epoch 2
```


注意上述输出中的 fsid (47f20cbc-914f-24ed-93dc-9f2800951ba2) 和节点列表 (10.0.0.1:6789, 10.0.0.2:6789)，后面会用到。

最后，在 K8s 集群中创建相关资源，包括 ConfigMap、Secret、Deployment、DaemonSet 等等。


```
git clone https://github.com/ceph/ceph-csi.git
cd ceph-csi/examples
```


编辑 csi-config-map-sample.yaml，填入上面的 fsid 和节点列表，除此之外的其他项可以删除，示例如下：


```
apiVersion: v1
kind: ConfigMap
data:
 config.json: |-
   [
     {
       "clusterID": "47f20cbc-914f-24ed-93dc-9f2800951ba2",
       "monitors": [
         "10.0.0.1:6789",
         "10.0.0.2:6789"
       ]
     }
   ]
metadata:
 name: ceph-csi-config
```


编辑 cephfs/secret.yaml，填入上面创建的用户名和 key 值，除此之外的其他项可以删除，示例如下：


```
apiVersion: v1
kind: Secret
metadata:
 name: csi-cephfs-secret
 namespace: default
stringData:
 # Required for dynamically provisioned volumes
 adminID: demo-user
 adminKey: AQB7JL5jugJwFxAA+szhrjIi48JhJbZsI3feRg==
```


编辑 cephfs/storageclass.yaml，填入上面的 fsid 和 Ceph File System 名称，示例如下：


```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
 name: sc-cephfs
provisioner: cephfs.csi.ceph.com
parameters:
 clusterID: 47f20cbc-914f-24ed-93dc-9f2800951ba2
 fsName: demo-fs
 csi.storage.k8s.io/provisioner-secret-name: csi-cephfs-secret
 csi.storage.k8s.io/provisioner-secret-namespace: default
 csi.storage.k8s.io/controller-expand-secret-name: csi-cephfs-secret
 csi.storage.k8s.io/controller-expand-secret-namespace: default
 csi.storage.k8s.io/node-stage-secret-name: csi-cephfs-secret
 csi.storage.k8s.io/node-stage-secret-namespace: default
reclaimPolicy: Delete
allowVolumeExpansion: true
mountOptions:
 - debug
```


按照 README.md 中的指引，执行以下命令在 K8s 集群中依次创建所需资源：


```
kubectl apply -f ./ceph-conf.yaml
kubectl apply -f ./csi-config-map-sample.yaml
kubectl apply -f ./cephfs/secret.yaml
kubectl apply -f ./cephfs/storageclass.yaml
cd ../../deploy/cephfs/kubernetes
kubectl create -f ./csi-provisioner-provisioner.yaml
kubectl create -f ./csi-nodeplugin-rbac.yaml
kubectl create -f ./csi-cephfsplugin-provisioner.yaml
kubectl create -f ./csi-cephfsplugin.yaml
kubectl create -f ./csidriver.yaml
```


注意，上述资源均默认在 default namespace 中创建。

等待所创建的 Pod 成功运行后，即可创建基于 Ceph 的 PVC。


```
kubectl get pod -w
```



## 通过 Helm Chart 配置 Ceph CSI

理论上，我们可以在同一个 K8s 集群中部署两套 Ceph CSI，提供两个 Storage Class 以供用户使用。

在部署过程中：

* 对于 Ceph 集群，我们需要创建两个不同的 Ceph Filesystem 及对应的用户
* 对于 K8s 集群，我们需要创建两个不同的命名空间，并对 Storage Class、ClusterRole、ClusterRoleBinding 等集群级别资源使用不同的名称

为了方便部署，我们提供了 Helm Chart（点击[此处](./assets/k8s/cephcsi-helmchart.zip)下载），您只需要在 values.yaml 中填写参数即可。


values.yaml 示例如下：

```
ceph:
  storageClassName: sc-cephfs
  driverName: cephfs.csi.ceph.com
  clusterID: 47f20cbc-914f-24ed-93dc-9f2800951ba2
  fsName: demo-fs
  adminID: demo-user
  adminKey: AQAhuZBjSuS9AxBD77AvVjr+vAg9zhjRK7NR+g==
  metricsPort: 8681
  monitors:
    - "10.0.0.1:6789"
    - "10.0.0.2:6789"
  # erasureCode: true
  # pool: ecpool-k4-m2
  images:
    cephCSI: quay.io/cephcsi/cephcsi:v3.8.0
    csiProvisioner: registry.k8s.io/sig-storage/csi-provisioner:v3.3.0
    csiResizer: registry.k8s.io/sig-storage/csi-resizer:v1.6.0
    csiSnapshotter: registry.k8s.io/sig-storage/csi-snapshotter:v6.1.0
    csiNodeDriverRegistrar: registry.k8s.io/sig-storage/csi-node-driver-registrar:v2.6.2
```


运行以下命令在 K8s 集群中创建相关资源：


```
unzip cephcsi-helmchart.zip
helm template ./cephcsi-helmchart -n cephfs -o ./deploy.yaml
kubectl create ns cephfs
kubectl create -n cephfs -f ./deploy.yaml
kubectl get pod -n cephfs -w
```



## 创建 PVC 并绑定 Pod

在 PVC spec 中指定 StorageClass 名称即可创建基于 Ceph 存储集群的 PVC。例如，[配置 Ceph CSI](#配置-ceph-csi) 一节中创建了名为 sc-cephfs 的 Storage Class，通过如下 YAML 创建 PVC：


```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cephfs-pvc
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: sc-cephfs
```


然后创建 Pod 绑定该 PVC：


```
apiVersion: v1
kind: Pod
metadata:
 name: cephfs-demo-pod
spec:
 containers:
   - name: web-server
     image: nginx:latest
     resources:
       limits:
         cpu: 100m
         memory: 200Mi
     volumeMounts:
       - name: cephfs-pvc
         mountPath: /var/lib/www
 volumes:
   - name: cephfs-pvc
     persistentVolumeClaim:
       claimName: cephfs-pvc
```



## 查看 PVC 创建/绑定失败原因

如果创建 PVC 和 Pod 后，Pod 一直无法正常运行，请按照以下步骤查找原因：

**1. 查看 PVC 是否创建成功**

运行以下命令获取 PVC 信息：


```
kubectl get pvc cephfs-pvc
```


如果 STATUS 字段是 Bound，则转至第 2 步。

如果 STATUS 字段是 Pending，说明 PVC 没有 provision 成功，请查看 Ceph CSI Provisioner Pod 的日志寻找原因。

通过以下命令获取 Ceph CSI Provisioner Pod 名称：


```
$ kubectl get pod -n cephfs -l app=csi-cephfsplugin-provisioner
NAME                                            READY   STATUS    RESTARTS   AGE
csi-cephfsplugin-provisioner-5b77dc5fff-56mr9   6/6     Running   0          6d21h
csi-cephfsplugin-provisioner-5b77dc5fff-rcpbg   6/6     Running   0          6d21h
csi-cephfsplugin-provisioner-5b77dc5fff-tkmsh   6/6     Running   0          6d21h
```


其中只有一个 Pod 是主要的工作 Pod，另外两个处于待命状态。通过以下命令查看其日志：


```
kubectl logs -n cephfs csi-cephfsplugin-provisioner-5b77dc5fff-56mr9 csi-provisioner
```


**2. 查看 PVC 与 Pod 是否绑定成功**

运行以下命令获取 Pod 信息：


```
kubectl describe pod cephfs-demo-pod
```


输出结果中的 Events 字段会显示 PVC 绑定不成功的原因。

另外，查看 Ceph CSI Plugin Pod 日志也有助于查找原因。

首先，通过以下命令查看所创建的 Pod 位于哪个节点上：


```
$ kubectl get pod -o wide
NAME              READY   STATUS    RESTARTS   AGE     IP               NODE
cephfs-demo-pod   1/1     Running   0          6d22h   10.233.106.203   nc13
```


然后，通过以下命令找到位于同一个节点上的 Ceph CSI Plugin Pod：


```
$ kubectl get pod -n cephfs -l app=csi-cephfsplugin -o wide
NAME                     READY   STATUS    RESTARTS   AGE    IP             NODE
csi-cephfsplugin-55nc4   3/3     Running   0          3d1h   100.64.4.74    nc14
csi-cephfsplugin-dd99m   3/3     Running   0          3d1h   100.64.4.199   nuc
csi-cephfsplugin-jcxv6   3/3     Running   0          3d1h   100.64.4.71    nc11
csi-cephfsplugin-jf4mn   3/3     Running   0          3d1h   100.64.4.209   acer
csi-cephfsplugin-jhffm   3/3     Running   0          3d1h   10.0.0.2    e01
csi-cephfsplugin-mxjh2   3/3     Running   0          3d1h   100.64.4.73    nc13
csi-cephfsplugin-rdbbp   3/3     Running   0          3d1h   100.64.4.72    nc12
csi-cephfsplugin-vql6n   3/3     Running   0          3d1h   100.64.4.132   kylin
```


最后，通过以下命令查看该 Ceph CSI Plugin Pod 的日志：


```
kubectl logs -n cephfs csi-cephfsplugin-mxjh2 csi-cephfsplugin
```

如果仍然无法找到问题原因，可尝试重启该 Ceph CSI Plugin Pod：

```
kubectl delete -n cephfs csi-cephfsplugin-mxjh2
```

如果上述删除命令卡住，可通过以下命令强行删除该 Ceph CSI Plugin Pod：

```
kubectl delete -n cephfs csi-cephfsplugin-mxjh2 --force
```



## PVC 备份

为了支持 PVC 备份，首先需要[在 K8s 中启用 Volume Snapshot 功能](#在-k8s-中启用-volume-snapshot-功能)；其次，需要针对不同的 PVC Provisioner 创建 [VolumeSnapshotClass](https://kubernetes.io/docs/concepts/storage/volume-snapshot-classes/)。

Ceph CSI 提供了对应的 VolumeSnapshotClass 资源，其 YAML 如下（[来源](https://github.com/ceph/ceph-csi/blob/devel/examples/cephfs/snapshotclass.yaml)）：


```
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
 name: csi-cephfsplugin-snapclass
driver: cephfs.csi.ceph.com
parameters:
 # String representing a Ceph cluster to provision storage snapshot from.
 # Should be unique across all Ceph clusters in use for provisioning,
 # cannot be greater than 36 bytes in length, and should remain immutable for
 # the lifetime of the StorageClass in use.
 # Ensure to create an entry in the configmap named ceph-csi-config, based on
 # csi-config-map-sample.yaml, to accompany the string chosen to
 # represent the Ceph cluster in clusterID below
 clusterID: 47f20cbc-914f-24ed-93dc-9f2800951ba2

 # Prefix to use for naming CephFS snapshots.
 # If omitted, defaults to "csi-snap-".
 # snapshotNamePrefix: "foo-bar-"

 csi.storage.k8s.io/snapshotter-secret-name: csi-cephfs-secret
 csi.storage.k8s.io/snapshotter-secret-namespace: default
deletionPolicy: Delete
```


与[配置 Ceph CSI](#配置-ceph-csi) 一节类似，YAML 中需要填入 Ceph File System 的 fsid。

针对[创建 PVC 并绑定 Pod](#创建-pvc-并绑定-pod) 一节中所创建的名为 cephfs-pvc 的 PVC，我们创建一个 [VolumeSnapshot](https://kubernetes.io/docs/concepts/storage/volume-snapshots/) 来对其进行备份，YAML 如下（[来源](https://github.com/ceph/ceph-csi/blob/devel/examples/cephfs/snapshot.yaml)）：


```
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
 name: cephfs-pvc-snapshot
spec:
 volumeSnapshotClassName: csi-cephfsplugin-snapclass
 source:
   persistentVolumeClaimName: cephfs-pvc
```


查看所创建的备份：


```
kubectl describe volumesnapshot cephfs-pvc-snapshot
```


基于该备份创建一个新的 PVC：


```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
 name: cephfs-pvc-restore
spec:
 storageClassName: tsz-cephfs
 dataSource:
   name: cephfs-pvc-snapshot
   kind: VolumeSnapshot
   apiGroup: snapshot.storage.k8s.io
 accessModes:
   - ReadWriteMany
 resources:
   requests:
     storage: 1Gi
```


然后创建一个新的 Pod 绑定该 PVC：


```
apiVersion: v1
kind: Pod
metadata:
 name: cephfs-restore-demo-pod
spec:
 containers:
   - name: web-server
     image: nginx:latest
     resources:
       limits:
         cpu: 100m
         memory: 200Mi
     volumeMounts:
       - name: cephfs-pvc-restore
         mountPath: /var/lib/www
 volumes:
   - name: cephfs-pvc-restore
     persistentVolumeClaim:
       claimName: cephfs-pvc-restore
```


最后，进入 Pod 查看 PVC 中的内容是否与之前相同：


```
kubectl exec -it cephfs-restore-demo-pod -- /bin/bash
ls /var/lib/www
```

### 在 K8s 中启用 Volume Snapshot 功能

根据[官方博客](https://kubernetes.io/blog/2020/12/10/kubernetes-1.20-volume-snapshot-moves-to-ga/)，Volume Snapshot 特性从 K8s 1.20 版本开始 GA。

通过以下命令查看 K8s 版本：

```
kubectl version
```

通过以下命令查看 K8s 中是否已经安装 Volume Snapshot 相关组件：

```
kubectl get deploy -n kube-system snapshot-controller
kubectl api-resouces | grep volumesnapshot
```

如果尚未安装，可以根据[官方文档](https://github.com/kubernetes-csi/external-snapshotter#usage)，手动安装 CRD、controller、webhook 等组件：


```
git clone https://github.com/kubernetes-csi/external-snapshotter.git
cd external-snapshotter
kubectl kustomize client/config/crd | kubectl create -f -
kubectl -n kube-system kustomize deploy/kubernetes/snapshot-controller | kubectl create -f -
```


## 配额管理

PVC 能够使用的最大存储空间由其 spec 指定，K8s 集群中所有基于 Ceph 的 PVC 总共能够使用的最大存储空间可通过 Ceph File System 的底层 Pool 来限制。

在任意管理节点上运行以下命令列举所有 Pool：


```
$ sudo ceph osd pool ls
.mgr
cephfs.demo-fs.meta
cephfs.demo-fs.data
```


其中，名为 `cephfs.demo-fs.data` 的 Pool 就是名为 demo-fs 的 Ceph File System 存储数据的 Pool，通过以下命令限制该 Pool 能够使用的最大存储空间：


```
sudo ceph osd pool set-quota cephfs.demo-fs.data max_bytes 10000000
```


参考文档：



* [Ceph - Set Pool Quotas](https://docs.ceph.com/en/quincy/rados/operations/pools/#set-pool-quotas)
* [Ceph - Quotas](https://docs.ceph.com/en/quincy/cephfs/quota/#quotas)
