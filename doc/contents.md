# OSP director （ Openstack 导演 ）

## 介绍

简而言之, **OSP director** 是红帽的新RHEL OSP部署工具，它包涵了安装、配置和监控等工具集。 OSP Director 是收敛性多年的上游工程工作，建立的模具、创建点播、和采用我们通过并购最佳部署工具的总集成部署工具。 我们还对包装多年值得的经验和专门知识在部署 OpenStack 在规模和最佳做法到这个新的产品，确保它经得起时间的考验。 它将替换现有的工具，如 RHEL OSP 安装程序为 * * 默认 * * 部署工具作为附带我们 OpenStack 发布。

OSP director 倡议来自 * 三 * 独特的欲望-

1. 对需要**删除**多安装工具包的问题;有一个全面、 灵活、 稳定和可支持的部署工具，迎合了绝大多数客户的配置和要求。传达客户的信任和字段执行能力。<br><br>
2. 对 (最后) 采用 **上游** 部署组件,使得他们变得更 *成熟*，利用强度的社区发展和紧**集成**与其余 OpenStack 组件。消除了我们对我们自己工作需要*自制*工具-用有限的预算为工程和质量保证。<br><br>
3. 有一种工具，去 **超越** 简单地安装 OpenStack 环境;一旦它安装的情况下将 '火和忘记'-以前的工具，工具完成了它的工作。但正在进行什么 **业务** 任务吗?
升级、 修补、 监测的环境、 容量规划、 利用度量等。我们需要不仅仅是安装工具，我们需要超越的东西。

OSP director 被设计为更**综合**，能够满足操作的要求团队并提供功能 **API**-驱动的接口，用于配置、 自动化和最终的部署。采取的最好的特点和概念从 Red Hat 有在社区和家酿的解决办法，从其处置入的现有工具 **融合** 部署

<center>
    <img src=./images/installer_roadmap.png>
</center>

Red Hat 将聚焦主任向前发展，其发展，但很可能会保留另一个释放或两，取决于导演的功能集是如何演变 Packstack。

## 架构

OSP director 由很多不同组件组成，涵盖 upstream OpenStack 部署项目 (namely, **TripleO** and associated **Ironic**) ，这些组件已经成熟，并可用于生产。 并且含有红帽的增强工具集。它的架构如下:<br><br>

<center>
    <img src=./images/converged_installer_components.png>
</center>

和大多数OpenStack部署工具一样, OSP director 有以下机制：

1. *安装安装器*
2. 鉴定 **目标宿主主机** - 我们要安装的主机
3. 要部署的**软件** **内容**
4. 定义部署的 **拓扑** 与 **配置** 
5. 通过自动硬件控制对**裸机进行部署**
6. 分发**软件** 和 **配置** 管理
6. 对已通过*director部署的*进行**微调**

OSP director uses a variety of OpenStack components to achieve it's goal of deployment, more specifically, **TripleO** for the creation of images and environment templates, **Ironic** for baremetal control, **Heat** for component definition, ordering, and deployment, and then **Puppet** for post-instantiation configuration. Unlike existing deployment tools, OSP director includes tools that help with testing of hardware, and clears the path for future **operational** tasks such as automated OpenStack **upgrades**, **patch** management, centralised **log** collection, and identification of **problems**. Below, we'll look at each of the individual building blocks of OSP director in more detail, and explain why they're required.

>**NOTE** -我们会进入更多的细节，在未来的章节和实验室，帮助集成这些组件之间的理解，给你当前的功能集的概述。

## TripleO

The notion of **TripleO** or "**O**penStack-**o**n-**O**penStack" is the most important concept to understand. OSP director utilises TripleO heavily for deployment, configuration, and automation. Therefore anyone that deploys OpenStack using this toolset must understand the concept of **undercloud** vs **overcloud**.

TripleO is a project that aims to allow administrators to deploy a **production** cloud (where *workloads* will run) via an existing **deployment** OpenStack environment (utilising a subset of OpenStack components). The **production** cloud is known as the "**overcloud**" and the underlying OpenStack **deployment** cloud is known as the "**undercloud**". Before any **overcloud** can be deployed, an **undercloud** must be configured. Previous installation mechanisms did not have the concept of an undercloud, they just used their own platform to deploy directly to an "overcloud".

> **NOTE**: The **undercloud** is often thought of as the "**baremetal** cloud", where the "**workload**" of that cloud is the **overcloud** nodes themselves, e.g. controller nodes and compute/hypervisor nodes.

<br><center>
    <img src=./images/logical_view.png>
</center>

## Tuskar

## Ironic

<br><center>
    <img src=./images/discovery_diagram.png>
</center>

## Heat

<center>
    <img src=./images/resource-list.png>
</center>

## Puppet

## Unified CLI

## Components taken from SpinalStack

## Support and Lifecycle

<center>
    <img src=./images/ospd-lifecycle.png>
</center>

## Next Chapter

The next chapter is an overview of the main features of director and the current integration situation, click [here][director-features](./director-features.md) to proceed.

