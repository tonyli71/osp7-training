# OSP director （ Openstack 导演 ）

## 介绍

简而言之, **OSP director** 是红帽的新RHEL OSP部署工具，它包涵了安装、配置和监控等工具集。 OSP Director 是收敛性多年的上游工程工作，建立的模具、创建点播、和采用我们通过并购最佳部署工具的总集成部署工具。 我们还对包装多年值得的经验和专门知识在部署 OpenStack 在规模和最佳做法到这个新的产品，确保它经得起时间的考验。 它将替换现有的工具，如 RHEL OSP 安装程序为 **默认** 部署工具作为附带我们 OpenStack 发布。

OSP director 倡议来自 *三* 独特的欲望-

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

OSP director使用各种 OpenStack 组件来实现它的目标的部署, 更具体地说, **TripleO** 用来制作镜像和环境模板, **Ironic** 用来对裸机控制, **Heat** 用来对组件定义、关系及部署, **Puppet** 用来对安装后的配置. 不同于现有的部署工具, OSP director 包含的工具可以帮助测试硬件, 为未来 **业务** 任务，如自动 OpenStack **升级** 扫清了道路, **补丁** 管理, 集中 **日志** 收集, 定位 **问题** . 下面，我们会看看每个构建基块的 OSP director更多的细节，并解释为什么需要他们。

> **NOTE** -我们会进入更多的细节，在未来的章节和实验室，帮助集成这些组件之间的理解，给你当前的功能集的概述。

## TripleO

理解 **TripleO** 或 " **O** penStack- **o** n- **O** penStack" 的概念是非常重要的。 OSP director 非常深入采用TripleO于部署、配置、自动化。 因此部署 OpenStack 使用此工具集的任何人都必须了解 **undercloud** 与 **overcloud** 的概念.

TripleO 是一个项目，目的是让管理员可以通过现有的 **部署** OpenStack 环境 (利用 OpenStack 组件的子集)部署，**生产** 云 (在那里 *工作负载* 将运行) 。 **生产** 云被称为" **overcloud** "和底层 OpenStack **部署** 云被称为" **undercloud** "。 在任何 **overcloud** 可以部署前，**undercloud** 必须进行配置。以前的安装机制并没有 undercloud 的概念，他们只是用他们自己的平台直接部署一个"overcloud"。

> **注意** : **undercloud** 通常被认为是" **baremetal** 云"，其中" **工作** "的那朵云是 **overcloud** 节点本身，例如控制器节点和计算/虚拟机监控程序节点。

<br><center>
    <img src=./images/logical_view.png>
</center>

<br>进一步澄清，TripleO 主张使用 **自身** OpenStack API 来配置、 部署和管理 OpenStack 环境本身。 利用这种 API 与 OSP 导演的主要好处是他们好 **记录**，去通过广泛 **集成测试** 上游，是成熟的对于那些已经熟悉 OpenStack 的工作方式，它是很容易理解 TripleO (director) 的工作原理。 因此，社区的功能增强，安全修补程序、 和 bug 修复是自动 **继承** 到 OSP Director中。

除了使用 OpenStack 部署 OpenStack **概念** ，TripleO 项目实际上是什么? 可以认为 TripleO **胶合** 驻留在组件之间 **undercloud** ，允许管理员定义什么 **overcloud** 环境看起来就像 — — 它了解 **设计模式** 与部署 OpenStack 环境相关联，并且可以进行定制部署种类繁多的定制配置。 TripleO 附带工具、 实用工具以及用于创建示例模板 **模板** 定义环境和相关 **镜像** 支持实例化此类配置。

TripleO **不** 实际上执行部署，它只是产生如转嫁到组件所需的数据 **Ironic** 和 **Heat** 来执行安装。

## Tuskar

每个 OpenStack 部署，无论哪种分布或安装工具正在使用，需要的定义 **角色**。这些角色确定一个给定的系统去做;它有什么职责。虽然 OpenStack 是高度可定制，提供在每个系统什么的灵活性，有常见的角色，我们将分配给节点的数目。这些 **通常** 角色是，如下所示

角色            | 职责
--------------- | ----------------------------------
控制节点        | 运行 RESTful API **endpoints**, **数据库** 服务, 和 **AMQP** 服务
网络节点        | 运行 inbound/outbound 网络 **routing**, **DHCP**, **metadata**, 等服务。（通常会被合并到控制节点）
计算节点        | 运行 **hypervisor** 服务, 实例运行的地方
存储节点        | 运行实例的 **backend storage** 例如运行 **Ceph**

当规划部署的这种作用，是重要的是了解所需 **系统规范** 为每个角色，节点 **数量** 的要求，每个角色和 **软件** 的要求为每个，包括任何用户选定 **参数**。
**Tuskar** 提供 API 和管理的平台，用于定义和链接所有这些内 TripleO 的部署，通过' **计划** '的概念 。

计划详情 **角色** 所界定，**口味** (即与每个角色相关联所需的系统规范作用匹配)，**数量** 的要求每个角色，每个节点 **图像** 这个特定的角色，和一组关联的 **heat** 模板 (与 **参数**)，配置环境后实例化。当部署 OpenStack 环境，你可以引用此计划通过它的唯一标识符和 OSP 主任知道到底哪个节点使用哪种软件要安装和哪些配置设置。

> **注意** : Tuskar 是 **可选** 单元到部署与 OSP 的导演;Tuskar 经常使部署细节容易定义通过提供的 API 和数据库用于存储配置，但可以输入此信息 **手动** 通过在部署过程中提供的模板和其他细节到 undercloud。
如果部署需要大量特定于站点的增强功能，或者如果您想要修改在部署之后它可能更适合于 **避免** Tuskar 在这个时候。

> **IMPORTANT**: 我们将 **不** 会使用 Tuskar 在这次培训

## Ironic

Ironic 用 ** Nova** "baremetal 驱动程序"使管理员能够提供 ' **baremetal-到-租客** '，换句话说 *专用* baremetal 主机向最终用户通过 *自助* 资源调配。最终用户将被保证 **隔离** 但也裸露的金属 **性能**，不需要 **共享** 与其他用户或其他租户的资源。今天，Ironic的两个目的，首先为客户提供，*做* 想要为他们的客户，提供 baremetal 向租客用例，其次是作为一种机制的 **控制** 我们使用 openstack baremetal 硬件 **部署** 在我们的 overcloud。

Ironic 有它自己的本机 API 用于定义 * * 节点 * *;机器可以设置为上述两个用例。要提供通过 OSP 主任 OpenStack 环境需要直接与Ironic，注册它们的节点的管理员指定他们 * * IPMI * * (或相当于特定供应商，例如HP iLO，思科 UCS，戴尔 DRAC;哪里有可用驱动程序) 凭据和地址。Ironic负责 **电源管理** 的节点收集 **硬件信息** 或 '**事实**' 的机制通过 **发现**。

在部署到期间使用达到通过发现过程的信息 **匹配** 的各种 **角色**，OpenStack 环境要求，例如，一个具有 10 个磁盘节点可能是一个存储节点。典型的流程看起来有点类似于以下内容:

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

