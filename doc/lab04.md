# 实验四: 创建和配置的基本节点镜像

## 序言

在用OSP director 部署overcloud时 ，重要的事情是意识到我们**不**采用kickstart启动或从零开始像我们用的 RHEL OSP 安装程序 (Staypuft) 那样在每台机器上部署 RHEL。OSP Director 部署依赖于 **预构建的镜像** ，通过 PXE 部署到节点。。这些磁盘映像通常已经包含 **RHEL** 和所需的 **RHEL OSP** 包。这种机制符合OpenStack镜像/模板的部署工作负载落的**原则。

在以前的实验，我们看到Ironic利用 **PXE** 部署磁盘镜像到节点为两个步骤，首先是初始**discovery/introspection**阶段，然后是 **实际部署**。在 undercloud 上，我们需要为这两个任务提供 **三** 个不同类型的镜像:

镜像类型        | 用途
--------------- | -----------------
**Discovery**   | 在部署阶段之前, 节点调配到发现模式，更加深入地**specification**，以确定有 **问题** 的硬件，并在帮助 **角色匹配**。Ironic提供了内核和 ramdisk到每个已知的节点。这些文件包含**introspection**所有所需的二进制文件和库 。<br><br> 所需文件: **discovery-ramdisk-ironic.{initramfs,kernel}**
**Deployment**  | 当管理员开始 **overcloud 部署**时，部署机制 **预建** 的RHEL 磁盘镜像必须可用。部署镜像只是一个基本内核和 ramdisk 引导具有预构建镜像的节点通过**undercloud Glance**服务器下载他们和"*dd'ing*"镜像到该节点的磁盘上。<br><br> 所需文件: **deploy-ramdisk-ironic.{initramfs,kernel}**
**Overcloud**   | **Overcloud** 类型镜像是在系统引导过程中**deployment**到节点上的。这里的主要区别是这些镜像像包含的软件将 **持久化** 在 overcloud 节点上,而**不是**只是为特定目的ramdisk镜像。<br><br> 所需文件:**overcloud-full.{initrd,vmlinuz,qcow2}**

Red Hat 提供 " **预构建的**" 镜像可用于大多数客户overcloud部署。OSP Director 可以不需变更使用这些镜像。我们也为客户提供对模版 **建立自己的图像** 的方法，以适应特定用户的需要和要求，这能帮助我们的计划外的 **业务工具**支持 和  测试正在新修补的程序和自动升级。

> **注意**: " **预构建的**" 镜像可在以下链接找到: **https://access.redhat.com/site/products/red-hat-enterprise-linux-openstack-platform/**

这第四个实验将引导您完成上面; 生成的 overcloud 镜像将用于后续的实验，估计的完成时间约 ��� **90 分钟**:

## Next Lab

The next lab will be the creation and registration of your overcloud nodes, ready for deployment, click [here][lab5](./lab05.md) to proceed.

