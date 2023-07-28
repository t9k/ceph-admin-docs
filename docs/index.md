# Ceph 存储集群管理员手册

Ceph 是一个高可靠性、可扩展的分布式存储服务，能够存储 PB 级别的海量数据，并对外提供文件系统、对象存储、块存储接口，其架构见[官方文档](https://docs.ceph.com/en/latest/architecture/)。

T9k AI 平台支持使用 Ceph 作为集群存储服务，并提供 PVC、S3 等方式使用。

<figure class="architecture">
    <img alt="Ceph in TensorStack AI Platform" src="./assets/architecture.png"/>
<figcaption align="center">图 1：如何在 Kubernetes 中使用 Ceph：Kubernetes 可以通过 CSI 机制以 PVC 的形式访问 Ceph 的 File System 接口，也可以通过 S3 协议访问 Ceph 的 Object Storage 接口。</figcaption>
</figure>



本手册在 [Ceph 官方文档](https://docs.ceph.com/en/quincy/)的基础上提供更加具有针对性的指导，方便管理员对 TensorStackk AI 平台中部署的 Ceph 存储集群进行日常管理、故障排查等工作。

