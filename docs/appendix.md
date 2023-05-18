# 附录


## Enable Volume Snapshot in K8s

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
