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

实验室环境描述如下;我们有一个 **baremetal** 主机提供我们 **虚拟** 基础设施，使两个我们 **undercloud** 和 **overcloud **:

<center>
    <img src="images/osp-director-env-1.png"/>
</center>

## Next Chapter

The next chapter is the start of our **hands-on labs** and will be the configuration of our 'seed host', i.e. the one we'll use for the **undercloud**, click [here][lab1](./lab01.md) to proceed.
