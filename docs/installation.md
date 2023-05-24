# 集群安装


## 安装 Ceph

Ceph 集群的安装从 bootstrap 一个节点开始，然后逐步加入其它节点、添加存储设备。

首先，确保节点上装有 Docker，如果没有，请按照 [Docker 官方文档](https://docs.docker.com/engine/install/ubuntu/)安装 Docker。

然后，在第一个节点上安装 Ceph：


```
curl --silent --remote-name --location https://github.com/ceph/ceph/raw/quincy/src/cephadm/cephadm
chmod +x cephadm
sudo ./cephadm add-repo --release quincy
sudo ./cephadm install

sudo cephadm bootstrap --mon-ip <ip-address-of-first-node>

sudo cephadm add-repo --release quincy
sudo cephadm install ceph-common
```


最后，您可以继续[加入其他节点](#添加节点)、[添加存储设备](#添加存储设备)。

参考文档：



* [Ceph - Cephadm Install](https://docs.ceph.com/en/quincy/cephadm/install/) 


## 添加节点

假设新加入的节点的 IP 为 10.1.2.3，名称为 host1。

首先，将 Ceph 集群的 SSH 公钥添加到新节点的信任列表中。

如果有新节点的 root 密码，在任意管理节点上运行：


```
sudo ssh-copy-id -f -i /etc/ceph/ceph.pub root@10.1.2.3
```


如果有新节点的某个具有 sudo 权限的用户密码，假设其用户名为 superuser。

在任意管理节点上运行：


```
sudo scp /etc/ceph/ceph.pub superuser@10.1.2.3:~/ceph.pub
```


在新节点上运行：


```
sudo su
cat ~/ceph.pub >> /root/.ssh/authorized_keys
```


然后，确保新节点上安装了 Docker：


```
sudo docker version
```


如果没有，请按照 [Docker 官方文档](https://docs.docker.com/engine/install/ubuntu/)安装 Docker。

最后，在管理节点上运行以下命令添加新节点到 Ceph 集群：


```
sudo ceph orch host add host1 10.1.2.3
sudo ceph orch host add host1 10.1.2.3  --labels _admin
```


并在新节点中安装 Ceph 命令行：


```
curl --silent --remote-name --location https://github.com/ceph/ceph/raw/quincy/src/cephadm/cephadm
chmod +x cephadm
sudo ./cephadm add-repo --release quincy
sudo ./cephadm install ceph-common
```


参考文档：



* [Ceph - Adding Hosts](https://docs.ceph.com/en/quincy/cephadm/host-management/#cephadm-adding-hosts) 


## 添加存储设备

用于 Ceph 存储的存储设备需要满足下列要求：



* 没有分区
* 没有 LVM 状态
* 没有被挂载
* 没有文件系统
* 大于 5GB

!!! info "什么是 LVM？"

    Logical Volume Manager 可以将 Linux 中的物理硬盘虚拟化为逻辑卷，以便更灵活地管理和使用存储。参考文档：

    * [https://en.wikipedia.org/wiki/Logical_Volume_Manager_(Linux)](https://en.wikipedia.org/wiki/Logical_Volume_Manager_(Linux))
    * [https://wiki.archlinux.org/title/LVM](https://wiki.archlinux.org/title/LVM)


如果存储设备曾被用于其他 Ceph 集群，可通过以下命令清除设备状态：


```
sudo ceph orch device zap <hostname> <devicepath>
```


例如：


```
sudo ceph orch device zap host1 /dev/sdb
```

如果存储设备曾被用作其他用途，可通过以下命令删除设备中的分区：


```
# 删除分区
sudo fdisk /dev/sdX

# 清除残余的分区表信息
sudo sgdisk --zap-all /dev/sdX
```


如果设备中仍然有残留的 LVM 状态，可尝试：



* 重启节点
* 利用 [LVM 相关命令](https://tylersguides.com/guides/remove-an-lvm-disk/)清除

在任意管理节点上运行以下命令，即可自动化地发现所有节点上可用的存储设备，并为其创建 OSD Daemon、添加到 Ceph 集群中：


```
sudo ceph orch apply osd --all-available-devices
```


或者，停止自动发现：


```
sudo ceph orch apply osd --all-available-devices --unmanaged=true
```


然后手动为某个存储设备创建 OSD：


```
sudo ceph orch daemon add osd <hostname>:<device-path>
```


例如：


```
sudo ceph orch daemon add osd host1:/dev/sdb
```


参考文档：



* [Ceph - Deploy OSDs](https://docs.ceph.com/en/quincy/cephadm/services/osd/#deploy-osds) 


## mpath 设备

如果存储设备不是常见的 hdd、ssd 硬盘，而是 mpath 设备，您需要手动创建 osd。

!!! info "什么是 mpath 设备？"
    
    Multipath 是一种为主机和存储硬件之间的网络通信提供容错的技术。参考文档：
    
    * [https://en.wikipedia.org/wiki/Linux_DM_Multipath](https://en.wikipedia.org/wiki/Linux_DM_Multipath)
    * [https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/dm_multipath/mpath_devices](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/dm_multipath/mpath_devices)
    * [https://ubuntu.com/server/docs/device-mapper-multipathing-introduction](https://ubuntu.com/server/docs/device-mapper-multipathing-introduction)

假设 mpath 设备位于 /dev/mapper/mpatha。首先，创建 lvm 相关的 pv、vg、lv 等：


```
sudo pvcreate --metadatasize 250k -y -ff /dev/mapper/mpatha
sudo vgcreate vgmpatha /dev/mapper/mpatha
sudo lvcreate -n lv0 -l 100%FREE vgmpatha
```


然后，使用 ceph 命令行工具准备 lv：


```
sudo ceph-volume lvm prepare --data /dev/vgmpatha/lv0
```


最后，激活 osd：


```
sudo ceph-volume lvm activate --all
```


激活时可能需要指定具体的 osd：


```
sudo ceph-volume lvm activate <osd-id> <osd-fsid>
```


例如：


```
sudo ceph-volume lvm activate 3 249e9103-5fec-4acc-b545-82c644f8756f
```


其中，`<osd-id>` 和 `<osd-fsid>` 在使用 ceph 命令行工具准备 lv 步骤的输出中可以找到，也可以通过以下命令查看：


```
sudo ceph-volume lvm list 
```


通过以下命令可以查看 lvm 相关的 pv、vg、lv：


```
sudo pvs
sudo pvdisplay
sudo vgs
sudo vgdisplay
sudo lvs
sudo lvdisplay
```


当 mpath 设备所在的节点重启时，lv 可能变为 unavailable 状态，导致 osd 异常，需要通过以下命令重新激活 lv：


```
sudo lvchange -ay /dev/vgmpatha/lv0
```


参考文档：



* [https://forum.proxmox.com/threads/ceph-with-multipath.70813/post-317961](https://forum.proxmox.com/threads/ceph-with-multipath.70813/post-317961)


## 配置 dashboard

Ceph 提供可视化 dashboard 来管理集群。

启用 dashboard：


```
sudo ceph mgr module enable dashboard
```


设置 http 访问： \



```
sudo ceph config set mgr mgr/dashboard/ssl false
```


重启 dashboard： \



```
sudo ceph mgr module disable dashboard
sudo ceph mgr module enable dashboard
```


设置 dashboard 密码：


```
sudo ceph dashboard ac-user-set-password admin -i <file-containing-password>
```


参考[查看 daemon 状态](./operation.md#查看-daemon-状态)，找到 mgr daemon 所在的节点，dashboard 的地址即为 `http://<mgr-host-ip>:8080`。

为了配置 grafana 使用 http，通过 SSH 登录 grafana 所在的节点，修改 grafana 的配置文件，将 https 改为 http，然后重启 grafana 容器：


```
sudo vim /var/lib/ceph/<cluster-id>/grafana.<hostname>/etc/grafana/grafana.ini
docker ps | grep grafana
docker restart <grafana-container>
```


配置 grafana、alertmanager、prometheus 等监控组件，以便在 dashboard 中查看可视化统计图表：


```
sudo ceph dashboard set-grafana-frontend-api-url http://<host-ip>:3000
sudo ceph dashboard set-grafana-api-ssl-verify False
sudo ceph dashboard set-grafana-api-url http://<host-ip>:3000
sudo ceph dashboard set-alertmanager-api-host http://<host-ip>:9093
sudo ceph dashboard set-alertmanager-api-ssl-verify false
sudo ceph dashboard set-prometheus-api-host http://<host-ip>:9095
sudo ceph dashboard set-prometheus-api-ssl-verify false
```

参考文档：

* [Ceph - Dashboard](https://docs.ceph.com/en/quincy/mgr/dashboard/)


## 配置 crushmap

Ceph 的默认存储策略是在不同的节点之间将数据复制 3 份，以提供高可用性。如果我们只有少于 3 个节点，根据[这个博客](https://balderscape.medium.com/setting-up-a-virtual-single-node-ceph-storage-cluster-d86d6a6c658e)，需要配置 Ceph 在 OSD 之间复制数据。

查看当前的 crushmap：


```
sudo ceph osd crush rule dump
```


导出当前的 crushmap：


```
sudo ceph osd getcrushmap -o comp_crush_map.cm
```


将导出的 curshmap 解析为人类可读的文件： 



```
crushtool -d comp_crush_map.cm -o crush_map.cm
```


编辑 crushmap，将其中的 host 改为 osd： 



```
vim crush_map.cm
```


例如，寻找 crushmap 中的下列内容：


```
# rules
rule replicated_rule {
	id 0
	type replicated
	step take default
	step chooseleaf firstn 0 type host
	step emit
}
```


将其修改为：


```
# rules
rule replicated_rule {
	id 0
	type replicated
	step take default
	step chooseleaf firstn 0 type osd
	step emit
}
```


编译新的 crushmap：


```
crushtool -c crush_map.cm -o new_crush_map.cm
```


应用新的 curshmap： \



```
sudo ceph osd setcrushmap -i new_crush_map.cm
```

参考文档：

* [Ceph - CRUSH Maps](https://docs.ceph.com/en/quincy/rados/operations/crush-map/)
* [Ceph - Manaully editing a CRUSH map](https://docs.ceph.com/en/latest/rados/operations/crush-map-edits/)


## 配置时钟同步

我们推荐使用 chrony 来同步所有节点的时钟，推荐采用一个主节点向外部时钟服务器同步时钟、其他节点向该主节点同步时钟的方案。

通过以下命令安装 chrony：


```
sudo apt update
sudo apt install chrony
```


chrony 配置文件位于 /etc/chrony/chrony.conf

主节点配置如下：


```
# Welcome to the chrony configuration file. See chrony.conf(5) for more
# information about usuable directives.

# This will use (up to):
# - 4 sources from ntp.ubuntu.com which some are ipv6 enabled
# - 2 sources from 2.ubuntu.pool.ntp.org which is ipv6 enabled as well
# - 1 source from [01].ubuntu.pool.ntp.org each (ipv4 only atm)
# This means by default, up to 6 dual-stack and up to 2 additional IPv4-only
# sources will be used.
# At the same time it retains some protection against one of the entries being
# down (compare to just using one of the lines). See (LP: #1754358) for the
# discussion.
#
# About using servers from the NTP Pool Project in general see (LP: #104525).
# Approved by Ubuntu Technical Board on 2011-02-08.
# See http://www.pool.ntp.org/join.html for more information.
pool ntp.ubuntu.com        iburst maxsources 4
pool 0.ubuntu.pool.ntp.org iburst maxsources 1
pool 1.ubuntu.pool.ntp.org iburst maxsources 1
pool 2.ubuntu.pool.ntp.org iburst maxsources 2

allow 100.64.4.0/8

# This directive specify the location of the file containing ID/key pairs for
# NTP authentication.
keyfile /etc/chrony/chrony.keys

# This directive specify the file into which chronyd will store the rate
# information.
driftfile /var/lib/chrony/chrony.drift

# Uncomment the following line to turn logging on.
#log tracking measurements statistics

# Log files location.
logdir /var/log/chrony

# Stop bad estimates upsetting machine clock.
maxupdateskew 100.0

# This directive enables kernel synchronisation (every 11 minutes) of the
# real-time clock. Note that it can't be used along with the 'rtcfile' directive.
rtcsync

# Step the system clock instead of slewing it if the adjustment is larger than
# one second, but only in the first three clock updates.
makestep 1 3
```


注意，其中 `allow 100.64.4.0/8` 需填写其他节点的 ip 网段。

其他节点配置如下：


```
# Welcome to the chrony configuration file. See chrony.conf(5) for more
# information about usuable directives.

# This will use (up to):
# - 4 sources from ntp.ubuntu.com which some are ipv6 enabled
# - 2 sources from 2.ubuntu.pool.ntp.org which is ipv6 enabled as well
# - 1 source from [01].ubuntu.pool.ntp.org each (ipv4 only atm)
# This means by default, up to 6 dual-stack and up to 2 additional IPv4-only
# sources will be used.
# At the same time it retains some protection against one of the entries being
# down (compare to just using one of the lines). See (LP: #1754358) for the
# discussion.
#
# About using servers from the NTP Pool Project in general see (LP: #104525).
# Approved by Ubuntu Technical Board on 2011-02-08.
# See http://www.pool.ntp.org/join.html for more information.
server 100.64.4.104

# This directive specify the location of the file containing ID/key pairs for
# NTP authentication.
keyfile /etc/chrony/chrony.keys

# This directive specify the file into which chronyd will store the rate
# information.
driftfile /var/lib/chrony/chrony.drift

# Uncomment the following line to turn logging on.
#log tracking measurements statistics

# Log files location.
logdir /var/log/chrony

# Stop bad estimates upsetting machine clock.
maxupdateskew 100.0

# This directive enables kernel synchronisation (every 11 minutes) of the
# real-time clock. Note that it can't be used along with the 'rtcfile' directive.
rtcsync

# Step the system clock instead of slewing it if the adjustment is larger than
# one second, but only in the first three clock updates.
makestep 1 3
```


注意，其中 `server 100.64.4.104` 需填写主节点的 ip。

修改配置文件后，通过以下命令重启 chrony 使配置生效：


```
sudo systemctl restart chronyd
```


在主节点上查看 chrony 运行状态：


```
sudo chronyc activity
sudo chronyc tracking
sudo chronyc sources -v
sudo chronyc clients
```


查看 Ceph 时钟同步状态：


```
sudo ceph time-sync-status
```

参考文档：

* [https://ubuntu.com/blog/ubuntu-bionic-using-chrony-to-configure-ntp](https://ubuntu.com/blog/ubuntu-bionic-using-chrony-to-configure-ntp)


## 配置警告通知

Ceph 自带一整套监控系统，包括 prometheus/grafana/alert mamanger。为了能够及时收到 alert manager 发出的警告，您可以配置 alert manager 通过[邮箱](https://prometheus.io/docs/alerting/latest/configuration/#email_config)和[企业微信](https://prometheus.io/docs/alerting/latest/configuration/#wechat_config)来发送警告信息。

通过 SSH 登录 alert manager 所在的节点，将位于 `/var/lib/ceph/&lt;cluster-id>/alertmanager.&lt;hostname>/etc/alertmanager` 的配置文件 alertmanager.yml 修改为如下内容（注意替换其中邮箱和企业微信相关的具体配置）：


```
# This file is generated by cephadm.
# See https://prometheus.io/docs/alerting/configuration/ for documentation.

global:
  resolve_timeout: 5m
  http_config:
    tls_config:
      insecure_skip_verify: true

route:
  receiver: 'ceph-dashboard'
  routes:
    - group_by: ['alertname']
      group_wait: 10s
      group_interval: 10s
      repeat_interval: 1h
      receiver: 'ceph-dashboard'
      continue: true
    - group_by: ['alertname']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 6h
      receiver: 't9k-monitoring/email/t9k-sre'
      continue: true
    - group_by: ['alertname']
      group_wait: 10s
      group_interval: 5m
      repeat_interval: 12h
      receiver: 't9k-monitoring/wechat/alerts'
      continue: true

receivers:
- name: 'ceph-dashboard'
  webhook_configs:
  - url: 'http://<hostname>:8080/api/prometheus_receiver'
- name: t9k-monitoring/email/t9k-sre
  email_configs:
  - to: <to>
    from: <from>
    smarthost: <smarthost>
    auth_username: <auth_username>
    auth_password: <auth_password>
    headers:
      Subject: '{{ template "email.subject" . }}'
- name: t9k-monitoring/wechat/alerts
  wechat_configs:
  - api_secret: <api_secret>
    corp_id: <corp_id>
    agent_id: <agent_id>
    to_user: '@all'
    message: '{{ template "wechat.message" . }}'
templates:
- /etc/alertmanager/custom.tmpl
```



在同一文件夹下创建 custom.tmpl，内容如下：


```
{{ define "email.subject" }}[ceph]{{ template "__subject" . }}{{ end }}

{{ define "__custom_text_alert_list" }}{{ range . }}
--
{{with .Annotations.description}}Description:  {{.}} {{else -}} {{end}}

Labels:
{{- range .Labels.SortedPairs }}
- {{ .Name }}:
   {{ .Value }}
{{- end }}
{{- end }}{{- end -}}

{{ define "wechat.message" }}[ceph]{{ template "__subject" . }}

Summary: {{ .CommonAnnotations.summary }}
{{ if gt (len .Alerts.Firing) 0 }}
Firing Alerts:
{{ template "__custom_text_alert_list" .Alerts.Firing }}
{{- end }}
{{- if gt (len .Alerts.Resolved) 0}}
Resolved Alerts:
{{ template "__custom_text_alert_list" .Alerts.Resolved }}
{{- end }}{{- end -}}
```


然后重启 alert manager 容器：


```
docker ps | grep alertmanager
docker restart <alert-manager-container>
```



## 创建 cephfs

运行以下命令创建 cephfs：


```
sudo ceph fs volume create <fs_name> --placement=<placement>
```


其中 `<placement> `是指 cephfs 的 mds daemon 部署在哪些节点上。为了高可用，一个 cephfs 一般有两个 mds daemon：一个正常运行，一个随时准备接替。

例如：


```
sudo ceph fs volume create k8s --placement="host1 host2"
```

参考文档：

* [Ceph - Deploy CephFS](https://docs.ceph.com/en/quincy/cephadm/services/mds/#orchestrator-cli-cephfs)


## 配置 erasure code

erasure code 是一种纠错码，在原始数据的基础上冗余存储一些经过编码的数据块，以便在一些数据块损坏时仍然能够计算出原始数据。

erasure code 最重要的参数是 k 和 m，其中



* k 表示原始数据将被分为 k 个数据块（data chunks）
* m 表示在原始数据的基础上计算出 m 个编码块（encoding chunks）

上述 k + m 个块将会被分别存储在不同的地方，根据配置的不同，可以是不同的 host、不同的 osd 等。

erasure code 的优势在于



* 最多能承受 k + m 个块中任意 m 个块的损坏
* 存储 1 个单位的原始数据，需要 (k + m) / k 个单位的空间

作为参考，Red Hat 支持以下 k/m 值：



* k=8 m=3
* k=8 m=4
* k=4 m=2

首先创建一个 erasure code profile（以 k=4 m=2 为例）：


```
sudo ceph osd erasure-code-profile set ecprofile-k4-m2 k=4 m=2 crush-failure-domain=osd
sudo ceph osd erasure-code-profile ls
sudo ceph osd erasure-code-profile get ecprofile-k4-m2
```


然后创建一个 erasure coded pool：


```
sudo ceph osd pool create ecpool-k4-m2 erasure ecprofile-k4-m2
sudo ceph osd pool set ecpool allow_ec_overwrites true
sudo ceph osd pool application enable ecpool cephfs
```


最后将 erasure coded pool 作为第二个 data pool 加入到 CephFS 中：


```
sudo ceph fs add_data_pool <fs-name> ecpool-k4-m2
sudo ceph fs ls
```

参考文档：

* [Ceph - Erasure code](https://docs.ceph.com/en/latest/rados/operations/erasure-code/)
* [Ceph - Using erasure coded pools with CephFS](https://docs.ceph.com/en/quincy/cephfs/createfs/#using-erasure-coded-pools-with-cephfs)
* [Ceph - Adding a data pool to the file system](https://docs.ceph.com/en/quincy/cephfs/file-layouts/#adding-a-data-pool-to-the-file-system)
