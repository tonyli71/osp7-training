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

主要优点是如下:
* 允许 (但要求) 的用法 ** 预安装 ** 主机，即有 RHEL 已经安装的主机
* ** 回答-文件 ** 驱动，允许可移植性和配置的备份
* 一台安装主机可以驱动 ** 多 ** 部署，只需要 SSH 访问
* ** Puppet ** 基于上游原则和通用代码
* 允许进行对现有环境的更改 ** 重新运行 ** 工具
* 很 * * 简单 * * 的使用，和理解事情到底错在哪里与日志文件
* 不断地 ** 更新 ** 支持最新 OpenStack 特点，为了实现自动化测试环境中使用

其明显的局限性:
* 不支持多个控制器，并因此 ** 没有高可用性 **
* 无法设置系统从零开始，即** 没有 baremetal **
* 正如 CLI 是一种优势，还有 ** 没有 WebUI **
* ** 没有 API ** 自动化或定制
* 很 ** 刚性 ** 在其配置能力;复杂的配置很难
* 没有一种模式来实现 ** 自定义角色 **，预计控制器或计算节点
* 没有执行的能力 ** 隔离 API 端点 **-期望一个接口
* Fire and forget implementation; no ongoing **monitoring** or final **validation**

今天，它已被认为更多的"* PoC 安装程序 *"，用来得到一个非常有限的 OpenStack 环境启动和运行速度非常快。今天使用 Packstack 的客户数目会极为有限，和它的未来是未知的;很可能当主任时更多综合它可能接手负责 Packstack，并且它获取 * 弃用 *。

## RHEL OSP Installer (Staypuft)

基于初始 OpenStack 工头安装程序 (* * OFI * *)，RHEL OSP 安装程序，内部代号为"* * Staypuft * *"
RHEL OSP 安装程序，有漂亮的 WebUI、 基于向导的配置和裸露金属资源调配。

<center>
	<img src=./images/staypuft.png>
</center>

主要优点是如下:

* Re-use of **existing tools**, i.e. Foreman (from Satellite) and Puppet
* Rich **WebUI interface** for interrogation and configuration
* Supports **modification** of previously deployed OSP Installer environments
* Can **discover** newly installed hosts and offer them up for installation
* Supports deployment of **highly available** environments
* Provisions hosts from **bare metal**, including RHEL installation
* **Asset tracking** of controlled hosts with strict **configuration enforcement**
* Allows administrators to track installation **metrics** and absent hosts over time
* Controlled **deployment ordering** and timing mechanism via DynFlow
* Allows **grouping** of configuration into host groups for bulk assignment

<center>
	<img src=./images/staypuft_config.png border=2>
</center>

其明显的局限性:

* Pre-provisioned hosts **cannot** be used - no customer '*gold builds*' allowed!
* **No flexibility** of roles, monolithic controller or compute node only
* No on-going **monitoring** of environment post-deployment - fire and forget
* No supported **API** for interrogation or automation
* No supported **CLI** for administrative tasks

The RHEL OSP installer is tied to the lifecycle of RHEL OSP 5.0 and 6.0; it will receive no further major enhancements and is on maintenance mode only - RHEL OSP 7.0 will **not** ship with support for the RHEL OSP installer, director (and Packstack) only.

## SpinalStack

Through acquisition of **eNovance**, Red Hat attained a wealth of experience and expertise in deploying OpenStack at scale, and in production. In addition to the personnel, Red Hat gained access to high quality tools and utilities that were built up from scratch for automating the deployment of OpenStack, across a variety of distributions for eNovance's customers worldwide. SpinalStack was one of these tools; a highly customisable deployment tool, based on **Puppet** and **Jenkins**.

SpinalStack takes a novel approach to deploying OpenStack infrastructure; rather than deploying the underlying operating system using PXE/kickstart like Red Hat's existing tooling, a **pre-built**, **pre-tested**, static image is rolled out to each host depending on the assigned role. Before this image is rolled out, automated system health checking takes place, identifying any nodes that aren't performing as expected or are misconfigured, saving on administrative overhead.

The automation of the image creation and validation of the configuration is driven by Jenkins, with Puppet and **Ansible** helping with configuration and the detailing of the deployment. Once the testing has passed validation, initial deployment and consecutive updates are performed by another tool, known as **eDeploy**, to deploy the image via PXE after identifying the correct nodes - based on their specification.

<center>
	<img src=./images/spinalstack.png>
</center>

主要优点是如下:

* Hardware conformity checking and benchmarking
* Completely **open-source** tooling, and based on upstream Puppet modules
* **Automated testing** framework based on Jenkins
* Creates and deploys well-tested and validated golden images

其明显的局限性:

* Very complex setup
* Really is just a very large and complex bash script
* As images are used, there is no access to yum for adding new packages
* The deployed architecture does not use pacemaker but rather keepalived and systemd

## Next Chapter

The next chapter is an introduction to the OSP director, click [here][director-intro] to proceed.
