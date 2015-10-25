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

这第四个实验将引导您完成上面; 生成的 overcloud 镜像将用于后续的实验，估计的完成时间约 **90 分钟**:

* **创造** 所有类型的 **overcloud** 镜像
* **调查** OSP Director 如何创建 overcloud 镜像
* 让 **创建后修改** 的镜像以适合我们的需要
* **上传** 必要的转换后镜像到undercloud  **Glance** 服务器 

> **注意**: 此实验开始假定您已经完成 [第三个实验] [lab3] 和已部署了 undercloud。如果这不是，请重温以前的实验。

## 实验重点

这个实验将继续把重点放在 **undercloud** VM:

<center>
    <img src="images/osp-director-env-6.png"/>
</center>

## 镜像的创建

对于多数情况我们使用 *rhel-guest-image* 云镜像作为一个起始镜像应该够了，因此本实验 overcloud 镜像我们会用这个。然而，它也可能是一个从零开始新建立的 RHEL 操作系统，安装所需的软件包，删除任何唯一标识(例如在配置文件中的 MAC 地址)的组件的镜像 ，如运行 virt-sysprep，等等。而*rhel-guest-image* 不需要此类操作。

在实验 3 完成后我们回到我们的主机机采取 **快照** 刚安装的 **undercloud** 虚拟机，所以这个实验现在假定你仍然连接到基础 **主机**。复制您的 SSH 公钥到 *stack* 用户:

~~~
host# ssh-copy-id stack@undercloud
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already instled
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
stack@undercloud's password:

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'stack@undercloud'"
and check to make sure that only the key(s) you wanted were added.
~~~

从主机服务器复制已经 **修改** RHEL 客镜像:

~~~
host# scp /var/lib/libvirt/images/rhel7-guest.qcow2 stack@undercloud:~/
rhel7-guest.qcow2
~~~

返回到 **undercloud** 服务器:

~~~
host# ssh stack@undercloud
~~~

镜像生成器需要知道你使用哪个 **操作系统**  (这通常可以 RHEL，CentOS，Fedora)，它是这个实验用 **RHEL 7**。我们还将设置几个参数，帮助知道我们正在使用的镜像构建工具 **本地提供**的 RHEL 7 客镜像 (而不是从 Red Hat 下载他们) 和从那里找出额外/必要的**包**:

~~~
undercloud$ export NODE_DIST=rhel7
undercloud$ export DIB_LOCAL_IMAGE=rhel7-guest.qcow2
undercloud$ export USE_DELOREAN_TRUNK=0
undercloud$ export RHOS=1
undercloud$ export DIB_YUM_REPO_CONF="/etc/yum.repos.d/rhos-release-7-director.repo /etc/yum.repos.d/rhos-release-7.repo /etc/yum.repos.d/rhos-release-rhel-7.1.repo"
~~~

**验证** 目前镜象的变量正确性:

~~~
undercloud$ ls $DIB_LOCAL_IMAGE
-rw-r--r--. 1 stack stack 421M Sep  1 16:45 rhel7-guest.qcow2
~~~

建立 **overcloud** 镜像和日志审查输出:

~~~
undercloud$ source ~/stackrc
undercloud$ time openstack overcloud image build --all 2>&1 | tee openstack_image_build.log
(...)

real    33m37.235s
user    20m32.044s
sys     12m17.020s
~~~

> **注意**: 这整个过程将需要大约 **30 分钟**;你可以去喝杯咖啡，或者打开 **新的控制台**，然后按照下面的说明，探讨发生一些什么。请注意，"**cat: write error: broken pipe**" **不** 表示有错误。关于 diskimage 生成器进程的更多详细信息请参阅 [这个链接] [dib 信息]。

## 发生一些什么?

Openstack 客户端在这里调用图像生成 python 脚本:

~~~
/usr/lib/python2.7/site-packages/rdomanager_oscplugin/v1/overcloud_image.py
~~~




## Next Lab

The next lab will be the creation and registration of your overcloud nodes, ready for deployment, click [here][lab5](./lab05.md) to proceed.

