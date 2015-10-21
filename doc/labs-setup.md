# Hands-on Labs - 硬件安装

## 导论

这种支持的主要目的是让与会者舒适与 OSP 导演到 **便利** 您的使用情况，并帮助您掌握 **概念**。虽然这很多能布满视频和幻灯片，学习的最好方法是去 **动手** 和自己做。我们并没有多个裸金属机器的每一个人，但每个人都会有到大，访问 **双-系统CPU** **64 GB** 内存。所以虽然你会有做与裸金属主机的虚拟机上顶端-实验室，我们的用意是要使尽可能接近动手 **现实-世界/客户部署** 尽可能。我们可以成功地 **效仿** 很大一部分的"裸机"电源管理。

唯一你会需要你的笔记本电脑是其 **SSH 客户端**，和也许 **web-浏览器**。本章的其余部分将澄清的环境您会使用的并将确保您已连接。你会被分配 **连接详细信息** 和 **凭据**，请让教练知道是否你并没有必要的信息。

## 实验室体系结构

在导言中所述，每个与会者已拨款裸露的金属机双 CPU、 64 GB 内存，与 1gb 容量连接到企业网络 (带短跳到所有我们需要的软件包存储库)。这将会是 **你** 主机的一周，没人会访问它，，如果我们需要重新构建它我们可以为你这样做。这将是我们的平台，以部署我们的基础设施上的:

<center>
    <img src=./images/lab_arch1.png>
</center>

非常 *简化* 为目的的概述-此主机体系结构将用于部署数量的虚拟机，将使两个你 **undercloud** 和你 **overcloud**，与主机本身被虚拟电源管理平台，**Ironic** 将用于启动机器。我们可以很容易地使用虚拟机管理程序本身作为 undercloud，但清洁我们会部署在它自己的虚拟机 undercloud:

<center>
    <img src=./images/lab_arch2.png>
</center>

然后，我们会使用 **libvirt** 来定义将准确地表示我们会如插入我们 OpenStack 部署的客户网络的逻辑网络公共管理/内部存储，、 群集网络。虚拟机监控程序内这样做允许我们拓扑结构的完全控制权，给我们完成 **隔离** 从其他实验室与会者虽然不必担心外部开关:

<center>
    <img src=./images/lab_arch3.png>
</center>

> **注意**: VLAN 的代表以上 **任意** 用于实验室环境的解释并不确切地描绘我们会在以后的实验室中部署的环境。本文将详细彻底在适当的时候。

然后，我们将创建一组空的虚拟机的虚拟机监控程序将成为 **overcloud** 和定义他们对 **undercloud** 所以，它可以提供他们请求时。虚拟机做的非常快速和易于管理的同时不影响客户部署的现实。为了澄清下, 图显示了 undercloud deplpoying overcloud 机器;它实际上调用底层的虚拟机监控程序通电虚拟机 — — 这是 **exact** 同样的过程对于裸露的金属硬件，但功率控制将来自 **IPMI** (或供应商特定的例如戴尔DRAC) 执行:

<center>
    <img src=./images/lab_arch4.png>
</center>

## 实验室环境

实验室环境描述如下;我们有一个 **baremetal** 主机提供我们 **虚拟** 基础设施，使两个我们 **undercloud** 和 **overcloud**:

<center>
    <img src="images/osp-director-env-1.png"/>
</center>

## 连接

为这台机器已为课程的时间分配给你，我们建议您部署您 **公** 安全壳关键从您的客户端 (例如你的笔记本电脑/台式机机) 到你已经被分配的节点。让我们确保我们可以成功地访问我们的环境、 根密码应设置为 **Redhat01** '，如果这不是案例请让老师知道。用你已经被分配的节点的 IP 地址替换**"\<your given host\>"**:

~~~
client$ ssh root@<your given host>
host# disconnect
~~~

请确保你断开连接，然后让我们从您的便携式计算机客户端上载您安全壳的密钥。它会要求你再一次，指定的根密码，然后就可以到 ssh 进入宿主 **不** 警用密码:

~~~
client$ ssh-copy-id -i ~/.ssh/id_rsa.pub root@<your given host>
(enter root password)

client$ ssh root@<your given host>
host# whoami
root
host# exit
~~~

> **注意**: 如果你想要使用 **不同** 键，简单地取代内的密钥文件的位置 '**-i**' 参数;您可能需要指定此关键字的位置，试图通过 SSH，重新连接，默认情况下它查找文件时 '**~/.ssh/id_rsa.pub**'。如果你不已经 SSH 密钥，只需使用命令 '**ssh-keygen**' 和使用所有的默认值。

## Package Access

您已分配的节点将不启用任何软件包存储库，所以之前我们进展到第一个实验室，让我们确保已经这样了，你能够安装软件包。请注意，我们是 **不** 使用订阅管理器中，我们将使用内部存储库，主要是为了便于和 **性能**:

First install the **rhos-release** rpm package:
~~~
client$ ssh root@<your given host>
host# rpm -ivh http://XXX.XXX.XXX.XXX/repos/rhos-release/rhos-release-latest.noarch.rpm
(...)
~~~

然后，启用"0_Day"OSP 主任和 RHEL OSP 软件包;这些将 * * 镜像 * * 您的客户目前看到什么，即遗传算法包，再加上 0 天修复。

~~~
host# rhos-release -p 0_Day 7-director
Installed: /etc/yum.repos.d/rhos-release-7-director-rhel-7.1.repo
director requires the core repo as well, Installing...
Installed: /etc/yum.repos.d/rhos-release-7-rhel-7.1.repo
~~~

最后，添加 A1 RHEL 7 套餐，我们有最新的客户，面对 * * RHEL 7.1* * 位太。我们可以确认这通过运行一个简单的 'yum update':

~~~
host# rhos-release -p A1 7
Installed: /etc/yum.repos.d/rhos-release-7-rhel-7.1.repo

host# yum update -y
(...)
~~~

> **注意**: 你可能会看到错误，如 **rhos-release*' 已经安装 rpm。这是 *确定*，我们只在确保你底层主机的配置正确，在我们开始下一实验室之前。与 0_Day 回购相关联的核心回购就是坏了，因此为什么我们指定 A1 释放 **分开**。

下一步，使 * * 嵌套 KVM * *，我们可以加速嵌套虚拟化:

~~~
host# cat << EOF > /etc/modprobe.d/kvm_intel.conf
options kvm-intel nested=1
options kvm-intel enable_shadow_vmcs=1
options kvm-intel enable_apicv=1
options kvm-intel ept=1
EOF
~~~

我们还需要禁用 **rp_filter**，让我们的虚拟机与以后任务的底层主机进行通信:

~~~
host# cat << EOF > /etc/sysctl.d/98-rp-filter.conf
net.ipv4.conf.default.rp_filter = 0
net.ipv4.conf.all.rp_filter = 0
EOF
~~~

现在，我们已经完全 **更新** 我们机并进行了一些配置更改，让我们重新启动以确保我们开始从完全更新，和干净的状态:

~~~
host# reboot
~~~

> **注意**: 当我们在处理 **baremetal** 服务器，这可能需要几分钟来完成，并再次变得可用。

## 下一章

下一章是开始我们 **动手实验室**，将主机的配置中我们' 种子'，即我们将使用一个 **undercloud**，请点击 [这里] [lab1](./lab01.md)进行。
