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

OpenStack 业务流程，代号 **Heat** 中，最初被设计为应用程序堆栈业务流程引擎，允许组织以预定义哪些元素要为给定的应用程序部署。
只需定义一个堆栈 **模板**，组成的基础设施数目 **资源** (例如实例、 网络、 存储卷) 以及一套 **参数** 配置和热将部署资源基于给定 **依赖** 链，在必要时监视他们的可用性。这些模板启用应用程序堆栈，成为便携式，实现 **重复性** 与预期的结果。

使用热 **广泛** 内 OSP 主任向 *提供* 和 *管理* 与部署基于 OpenStack overcloud 关联的资源。考虑到我们使用本机 OpenStack 热 API，我们已经要求我们能够向 **重新使用** 热功能 overcloud 资源;从 OpenStack 组件配置到网络和存储设置，然后确保他们来按正确的顺序 — — 例如，我们可以描述环境的亲密的细节，控制器 *之前* 计算节点。
我们也可以通过使用热 **解决** 减轻部署，和 **使变化** 后部署。

~~~
# 几行热模板，定义控制器节点参数

NeutronExternalNetworkBridge:
    description: Name of bridge used for external network traffic.
    type: string
    default: 'br-ex'
NeutronBridgeMappings:
    description: >
      The OVS logical->physical bridge mappings to use. See the Neutron
      documentation for details. Defaults to mapping br-ex - the external
      bridge on hosts - to a physical name 'datacentre' which can be used
      to create provider networks (and we use this for the default floating
      network) - if changing this either use different post-install network
      scripts or be sure to keep 'datacentre' as a mapping network name.
    type: string
    default: "datacentre:br-ex"
~~~

热需要在模板中生成的 TripleO OpenStack，包括调用部署揭开序幕的援助 **Ironic** 到电源节点上。使用标准工具我们可以查看当前部署的资源 overcloud 的热和它们的当前状态:

<center>
    <img src=./images/resource-list.png>
</center>

虽然热提供 **综合** 和 **强大** 声明要部署环境的语法，它将需要一些事先了解，尤其是如果愿望是部署一个有点不正规的环境。我们会通过一些实验室指南，将运行 *介绍* 你的热模板文件和如何使用他们对默认部署在适当的时候进行修改。

## Puppet

Puppet最初被设计为 **配置管理** 和 **执法工具**;作为一种机制用来描述机器的最终状态，并保持这种方式。Puppet本质上支持两种模式，**独立** 在哪些指令的形式模式 **体现** 跑本地，和 **服务器** 在哪里它从中央服务器中检索及其清单的模式。管理员可以使 **变化** 通过将新的清单上传到一个节点并执行他们本地，或通过修改在客户端/服务器模型中 **puppet-master**。

Puppet是 *高度可扩展*，Red Hat 已经推使用它到机制。今天，我们在director许多领域用Puppet，确定如下:

1.作为一部分的 **undercloud** 安装，我们利用的Puppet根据**undercloud.conf**中列出的配置安装和配置本地包 。
2.Director 是 **基于镜像的** 解决方案，我们 **注入** *openstack-Puppet-模块* 成最终将部署在镜像 **overcloud** 建设过程中。这些Puppet模块然后将准备 **后实例化配置** 在部署时间。默认情况下，我们创建"*full fat*"包含所有 OpenStack 服务并将它用于所有节点的图像。
3.当 **overcloud** 正在部署，我们提供额外 **体现** 和 **参数** 到通过节点 **热**，和应用 (如由管理员指定) 如与Puppet，所需的配置其中 **服务** 启用/启动和 **OpenStack 配置** 申请-这些将 **节点依赖**。

当清单的推出到节点时，它们常常免费从站点或特定于节点的参数，以保证清单一致，这样，我们还分发了元数据称为 ' **hieradata**' 到的节点，所以那个Puppet只需要请求时所需的特定于节点的信息。
例如，若要引用清单内的 MySQL 密码，您可以将此信息保存到 hiera，和 **引用** 它从内向清单;你只需要改变 hieradata 在每个节点的基础上:

~~~
(As taken from the Hiera data)
# grep mysql_root_password hieradata.yaml
openstack::controller::mysql_root_password: ‘redhat123’

(Now referenced in the Puppet manifest)
# grep mysql_root_password example.pp
mysql_root_password  => hiera(‘openstack::controller::mysql_root_password’)
~~~

## 统一的命令行

统一的 CLI 是重要和 **积分** 的 OSP Director组件。所有我们上面提到过的组件有自己的界面，无论是 API 或命令直接调用。无论哪种方式，它将很难和 **费力** 来直接调用每个组件和管理交互来实现所需的部署。OSP Director提供统一的 CLI，**简化了** 的使用工具，如安装 undercloud 和 overcloud 与自己各自的单个命令。

我们提供 OSP 主任统一的 CLI 也是 **上游** 组件，**插件** 到上游 **OpenStackClient** 工具集 (http://docs.openstack.org/developer/python-openstackclient) 。它添加在 undercloud 和 overcloud 的扩展来允许我们对接口主任工具和实用程序与本机 OpenStack API 正在使用。

**示例** 的统一的 cli **用法** 如下所示:

~~~
(复制示例 undercloud 配置文件，并自定义它)
# cp /usr/share/instack-undercloud/undercloud.conf.sample ~/undercloud.conf
# vi undercloud.conf

(使用统一的 CLI  OSP 主任插件部署)
# openstack undercloud install
...

(复制本地，一些预建的镜像并将其上载到Glance的部署)
# cp -r /tmp/images/* ~/overcloud-images/
# cd ~/overcloud-images/
# openstack overcloud image upload
~~~

这使我们能够符合收敛到主流用法要替换的 'openstack' 命令的 **弃用** 由每个组件提供的个别 CLI 工具。它还允许我们可以利用现有的 rc 文件被用来定义凭据，例如' * keystonerc_admin *'。

## 从SpinalStack 引进的组件

在获取后的 eNovance，Red Hat 获得宝贵和生产测试技术 OpenStack 部署在 **eDeploy** 和 **SpinalStack**。虽然我们继续投资于此类组件的客户，我们选择采取最好的想法和重用功能从这些技术和 **集成** 他们入 OSP 主任不同。SpinalStack 启发了很多发展为 OSP 的主任，和我们继续对齐的上游的工具和我们下游的 OSP 主任提供由 eNovance 规定的原则。

我们从 SpinalStack 运来的主要特点是 **高级配置文件匹配** 功能。我们将部署我们的 overcloud 之前，我们通常要检查的机器为他们 **配置**，如果它们匹配预期的理解 **规范**，和发现不符合预期的任何节点 **的基准性能** （黑羊检测）。我们称这个过程 **反思**-它使我们能够启动与机器 **ramdisk** 收集主机的详细的信息。信息由称为组件提供给我们 **ironic-discoverd**，填补了 **Ironic** 与节点细节/事实数据库。

我们使用 '**ahc-工具**' 包采取举行具有讽刺意味的数据库中的信息，然后匹配 **角色** 与 **节点** 基于其 **特点**。匹配规则设置使用 AHC 配置文件;作为一个例子，如果我们想要确保已经大于 (或等于) 32g 内存的任何节点被分配计算节点的作用我们可以使用以下配置:

~~~
# cat /etc/ahc-tools/edeploy/compute.specs
[
 ('memory', 'total', 'size', 'ge(34359738368)'),
]
~~~

与最小值不匹配的任何节点 *4000 bogomips* 会 **排除** 从上面的例子中的控制器节点。

> **注意**: 标杆管理是一个非常详尽的过程，并且需要时间来操作。如果你做与虚拟机作为您的基础架构部署要避免使用该选项。

## 支持和生命周期

第一，我们已经分开的潜在生命周期 **核心**，即我们 OpenStack 分布，与 **部署工具**。根据以前版本的 RHEL OSP，每个主要 **核心** (同时，安装程序) 收到三年支持生命周期，该生命周期为核心将继续是三年，但 **OSP 主任** 会 **两年**。鉴于 **工程开销** 在提供向后兼容性和自动的升级到较新的版本，尤其是在定制部署，整体生命周期 *有* 将减少。

在这两年，整个 Red Hat 将确保向后兼容性维持为至少 **两个** 以前基本核心版本，允许 OSP 主任平台向 overcloud 通过介绍新功能 **自动升级**。例如，当 RHEL OSP **9.0 (Mitaka)** 是公布，最新的 OSP 导演将能够管理 OSP ** 7.0** **8.0** 和 **9.0** 环境，以及运行自动的升级。


<center>
    <img src=./images/ospd-lifecycle.png>
</center>

> **注意**: 生产阶段的详细信息，可以发现 [这里] [Production Support]（https://access.redhat.com/support/policy/updates/openstack/platform）。

OSP 主任每个主要版本还将点的版本中，每个 **两个月**，引入 **新功能**，例如支持新 OpenStack **功能** 如以前在技术预览版状态，提供技术 **bug 修复** 和 **安全修补程序**，并使第三方驱动程序 (例如，新认证Neutron或Cinder插件) 外对齐方式与基本的核心。这些点的版本将 **不** 引入任何破坏或更改底层核心功能。

## 下一章

下一章是概述主任和整合现状的主要特点，点击 [这里] [主任-功能] (./director-features.md) 继续。


