# 实验二: Undercloud 的配置

## 序言

到目前为止我们已经配置我们底层主机运行的虚拟机，将弥补我们 **undercloud** 和 **overcloud** 的基础设施。我们运行在虚拟机环境中，这给我们 **灵活性** 和 **隔离** 没有妥协的能力 *复制* 或 *模仿** **客户环境**。唯一真正的区别是，我们不得不使用的电源管理 — — 在我们的例子，它是 libvirt 虚拟机监控程序，但是客户的环境可能会某种形式的 IPMI 接口。

这第二个实验室是跟 **undercloud** 机; 配置我们要进行以下任务在这个实验室中，估计的完成时间的 **30 分钟**:

* 必要的 undercloud 软件包的安装
* 可用于 undercloud 的各种配置选项的探索
* Undercloud 以匹配我们部署的环境的配置

> **注意**: 此实验室开始假定您已经完成 [实验一] 和你设置的 undercloud 机器。如果这不是个案，请重温以前的实验。

## 实验重点

这个实验室将集中于 **undercloud** 虚拟机本身:

<center>
    <img src="images/osp-director-env-6.png"/>
</center>

## Undercloud 软件包的安装

假设你已经已经断开您的 undercloud 节点，让我们重新连接到它，做一个小的预安装配置到节点。首先，设置主机名正确;默认情况下这是 localhost，因为它没有料想到会有 ' 云 init' 通过设置其主机名。

~~~
host# ssh root@undercloud
undercloud# hostnamectl set-hostname undercloud.redhat.local
undercloud# systemctl restart network.service
~~~

> **注意**: 控制台语法中，我们会请参阅本机纯粹作为"**undercloud #**"为 **根** 命令，和"**undercloud$**"为 **非特权** 命令。你可能会看到略 *不同* 控制台输出。

测试包的访问并使条目 **/etc/hosts**，让我们安装在 eth1 上 '因素' 来缓解我们检索 IP 地址:

~~~
undercloud# yum install facter -y
undercloud# ipaddr=$(facter ipaddress_eth1)
undercloud# echo -e "$ipaddr\t\tundercloud.redhat.local\tundercloud" >> /etc/hosts
undercloud# grep undercloud /etc/hosts
192.168.122.253   undercloud.redhat.local undercloud
~~~

更新 undercloud 系统:

~~~
undercloud# yum -y update && reboot
~~~

现在，我们可以安装 undercloud 配置所需要的软件包。做最方便的方式，这是为包中拉 **统一 CLI**，这将依赖关系替我们管理。此外请注意，该软件包的名称取自上游 **RDO 经理** 的名字，**不是** OSP 主任。等待您的计算机以启动备份，并运行以下命令以安装软件包:

~~~
host# ssh root@undercloud
undercloud# yum install python-rdomanager-oscplugin -y
(...)
~~~

## 配置部署接口

你会记得我们在以前实验里我们的虚拟机配置两个网络接口，第一个网络接口(**eth0**) 被用来作为专用的 PXE/管理网络，我们会用来 **引导** **overcloud**，和第二个网络接口 (**eth1**) 被用于公共/外部访问。到目前为止，**eth1** 设置来在启动时，默认情况下，我们不需要做任何与此接口。第一个接口 eth0，将 undercloud 安装工具的自动配置。

> **注意**: 在客户环境中很重要的是要理解联结在部署网络的影响。如结合模式设置为资源调配使用的网络交换机上，交换机必须支持 ' * 回退 *' 模式支持 PXE 启动，显然不会配置的粘接。一些交换机 (例如**BigSwitch**) 不支持这种模式下，将简单地切断连接。这是 **强烈推荐** 非专用的接口用于部署网络。

## Undercloud 的配置

OSP 主任 CLI 在其操作过程中使用多个配置文件。对于 undercloud 部署它来读取配置参数从指定的本地目录中的文件 **undercloud.conf**。

作为安装脚本期望用户必须 **非-根** 用户，我们需要创建 **更多** 用户，我们将使用配置和最终部署的 undercloud:

~~~
undercloud# useradd stack
undercloud# echo "Redhat01" | passwd stack --stdin

undercloud# echo "stack ALL=(root) NOPASSWD:ALL" | tee -a /etc/sudoers.d/stack
undercloud# chmod 0440 /etc/sudoers.d/stack

undercloud# su - stack
undercloud$ whoami
stack
~~~

下面的配置文件当前不存在，但一个示例文件将已经部署为我们作为一部分的程序包部署;让我们将它拷贝到我们当前的工作目录:

~~~
undercloud$ cp /usr/share/instack-undercloud/undercloud.conf.sample ~/undercloud.conf
~~~

打开此配置文件 (**~/undercloud.conf**) 中你最喜欢的文本编辑器和进行一些修改，复制的环境。您需要更改这些行中，注意，注释掉条目是 **默认值**。

或者，您可以使用 **openstack-config** (或 crudini)，使其多一点可脚本化:

~~~
undercloud$ sudo yum -y install openstack-utils
undercloud$ openstack-config --set undercloud.conf DEFAULT local_ip 172.16.0.1/24
undercloud$ openstack-config --set undercloud.conf DEFAULT undercloud_public_vip  172.16.0.10
undercloud$ openstack-config --set undercloud.conf DEFAULT undercloud_admin_vip 172.16.0.11
undercloud$ openstack-config --set undercloud.conf DEFAULT local_interface eth0
undercloud$ openstack-config --set undercloud.conf DEFAULT masquerade_network 172.16.0.0/24
undercloud$ openstack-config --set undercloud.conf DEFAULT dhcp_start 172.16.0.20
undercloud$ openstack-config --set undercloud.conf DEFAULT dhcp_end 172.16.0.120
undercloud$ openstack-config --set undercloud.conf DEFAULT network_cidr 172.16.0.0/24
undercloud$ openstack-config --set undercloud.conf DEFAULT network_gateway 172.16.0.1
undercloud$ openstack-config --set undercloud.conf DEFAULT discovery_iprange 172.16.0.150,172.16.0.180
~~~

> **注意**: PXE/管理界面仅-所有 undercloud 服务 API 听你不能设置任何以下要在 192.168.122.0/24 (默认值) 子网的地址!因此这意味著，undercloud API 只是可用的如果你有一个路由到 PXE/管理网络，和 **不** 通过默认网络。

参数          | 值                | 意图
------------- | ----------------- | -------------------------------
**local_ip**  | **172.16.0.1/24** | 这是 **所需IP地址** 的 undercloud 网络上的节点 PXE/管理-我们建议的子网。
**undercloud\_public\_vip** | **172.16.0.10** | 这是由 undercloud 用于 API 请求上运行的虚拟 ip 地址 **公共** 入口。
**undercloud\_admin\_vip** | **172.16.0.11** | 在这环境中，我们有无需分离出公共和管理入口结点，但我们可以设置这无论如何是良好做法。
**local_interface** | **eth0** | 这就告诉 OSP 主任您想要您配置的网络是 **eth0**.
**masquerade_network** | **172.16.0.0/24** | 这将允许使用您的 undercloud 主机作为通往出包在安装过程中任何部署的节点。
**dhcp_start** | **172.16.0.20** | 这是一个的 IP 地址范围的起始地址我们 overcloud 节点都会收到我们通过 DHCP PXE/管理网络上的 undercloud。
**dhcp_end** | **172.16.0.120** | 作为上面，这是分配给 overcloud PXE/管理网络上的节点的 DHCP 范围的结束地址。
**network_cidr** | **172.16.0.0/24** | 这将用于在中子内定义整体的 PXE/管理网络CIDR
**network_gateway** | **172.16.0.1** | 这是将用作网关 overcloud 节点的外部使用的节点的 IP 地址访问所需的地方。
**discovery_iprange** | **172.16.0.150,172.16.0.180** | 这作为一个临时池，在发现过程中使用的 IP 地址。这应该 **与重叠不** **dhcp_start** 和 **dhcp_end** 范围先前设置。

没有加任何注析，您的文件内容应该看起来类似于以下:

~~~
undercloud$ egrep -v '^#|^$' undercloud.conf
[DEFAULT]
local_ip = 172.16.0.1/24
undercloud_public_vip = 172.16.0.10
undercloud_admin_vip = 172.16.0.11
local_interface = eth0
masquerade_network = 172.16.0.0/24
dhcp_start = 172.16.0.20
dhcp_end = 172.16.0.120
network_cidr = 172.16.0.0/24
network_gateway = 172.16.0.1
discovery_iprange = 172.16.0.150,172.16.0.180
[auth]
~~~

> **注意**: 有大量的 **认证** 配置选项，您可以选择更改应你想去的但离开他们 **空白** 会 *自动生成* **凭据** 我们和他们安装后输出为明文文件。

此配置文件现在最终敲定，我们将使用它在下一个实验室来部署我们的 undercloud!

## 快照 Undercloud 虚拟机

再次，让我们创建另一个 **快照** 的我们 **undercloud** 虚拟机的这样我们有另一个已知的工作状态，如果需要我们可以 **回复** 向回。请注意，你将需要有一个控制台到你 **这基础主机上运行 **不是 **undercloud** 机中运行:

~~~
undercloud$ exit
undercloud# exit
host# virsh snapshot-create-as undercloud undercloud-snap-lab2
~~~

## 这个实验的结果

通过完成这个实验你应该达到以下目标:
* 配置undercloud 机的**主机名**
* /etc/hosts 中输入 **持久化的主机条目**
* 配置为 undercloud 机 **包** 访问
* 安装所需的 undercloud **包**
* 修改示例 **配置** 基本 undercloud 部署文件

## 下一个实验

下一个实验室将是部署 undercloud，请单击[这里][lab03](./lab03.md).

