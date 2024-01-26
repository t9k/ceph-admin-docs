# 常见故障

## PVC 创建/绑定失败

参考[查看 PVC 创建/绑定失败原因](./k8s#查看-pvc-创建/绑定失败原因)。


## K8s 节点网络断开后恢复，但是运行在节点上的 Pod 仍然报错

如果 K8s 集群网络断开后恢复，但是 Pod 出现如下报错：

```
$ k describe po -n aim-yyx managed-notebook-c26a6-0
Events:
  Type     Reason       Age                    From           Message
  ----     ------       ----                   ----           -------
  Normal   Scheduled    11m                    t9k-scheduler  Successfully assigned demo/managed-notebook-c26a6-0 to node1
  Warning  FailedMount  7m38s (x2 over 9m54s)  kubelet        Unable to attach or mount volumes: unmounted volumes=[datadir], unattached volumes=[pep-config workingdir datadir managed-notebook-c26a6 kube-api-access-2wbpn]: timed out waiting for the condition
  Warning  FailedMount  3m2s (x2 over 5m20s)   kubelet        Unable to attach or mount volumes: unmounted volumes=[datadir], unattached volumes=[pep-config workingdir datadir managed-notebook-c26a6 kube-api-access-2wbpn]: timed out waiting for the condition
  Warning  FailedMount  99s (x13 over 11m)     kubelet        MountVolume.SetUp failed for volume "pvc-003a5630-b230-406f-9493-2f66653f768c" : rpc error: code = Internal desc = stat /var/lib/kubelet/plugins/kubernetes.io/csi/cephfs.csi.ceph.com/d542aa1dad943aea439969def75da7d1ded28a5fcdbae0657f2b7bd0dd98c694/globalmount: permission denied
  Warning  FailedMount  48s                    kubelet        Unable to attach or mount volumes: unmounted volumes=[datadir], unattached volumes=[pep-config workingdir datadir managed-notebook-c26a6 kube-api-access-2wbpn]: timed out waiting for the condition
```

根据 [GitHub issue](https://github.com/ceph/ceph-csi/issues/970)，原因是因为 ceph-csi 没有正确处理 mount point 由于长时间没有响应被 linux kernel 驱逐的情况。

暂时的解决办法是重启 Pod 所在的节点：

```
kubectl drain <node>
# reboot <node>
kubectl uncordon <node>
```


## mpath 设备所在节点重启后，OSD 报错

参考[集群安装 - mpath 设备](./installation.md#mpath-设备)，当 mpath 设备所在的节点重启时，lv 可能变为 unavailable 状态，导致 osd 异常，需要通过以下命令重新激活 lv：


```
sudo lvchange -ay /dev/vgmpatha/lv0
```
