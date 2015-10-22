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

## Next Lab

The next lab will be the deployment of the undercloud, click [here][lab03](./lab03.md) to proceed.

