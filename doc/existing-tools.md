# 回顾现有的部署工具

## 介绍

红帽在不断完善Openstack的部署工具。

现今我们多种部署工具在不同的Openstack版本中。您可以看到以下名称的部署工具：

* OpenStack Foreman Installer (OFI)
* RHEL OSP installer (Codename Staypuft)
* Packstack
* Ansible-based homebrew
* SpinalStack (via acquisition of eNovance)
* Bashstack

OpenStack 是 ** 极 ** 可配置，由许多底层组件，如果我们不遵循明确部署指南，则很可能 OpenStack 环境可以成为不稳定和不可靠;它是非常重要的是客户使用我们的工具，或至少了解建议的拓扑结构和配置指南。

上游称为' * RDO Manger *'，现在被称为 ' * OSP Director *' 下游，一种机制，采取最佳技术的功能的上面，但批判地删除 '安装包' 问题，创建一个明显的选择对于 OpenStack 部署会向前升级，恢复客户的信心。

我们进入 OSP Director之前，让我们了解一下这三个 ** 主 ** 现有部署工具，了解他们的优点和其局限性。

## Packstack

Packstack 是原始的安装程序是用于部署 OpenStack 在 Red Hat 的基础系统;这包括 RHEL，CentOS，Fedora，以及科学的 Linux。Packstack 是唯一 CLI 的工具即在命令行的驱动的接口并使用Puppet驱动安装没有用户界面和配置 OpenStack 组件连接到所需主机通过 SSH。

<center>
	<img src=./images/packstack.png>
</center>

