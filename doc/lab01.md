# 实验一 :  准备部署主机

## 序言

欢迎来到了动手实验!毫无疑问，这将是最最 **重要** 部分的支持;确保你可以把 **理论变成实践** 和建立在您现有的 OpenStack 专门知识与一些 OSP 主任经验!

在这一章中，我们要去执行以下任务来准备你潜在的裸金属机 (**主机**) 运行 underclound 和 overcloud。我们假定你在追随这些材料以提供 **专用** 中的硬件 **讲师** 环境;如果你使用的这些材料用于 **自学** 学习，指令的设计执行后用干净的单个裸金属主机 **RHEL 7.1 最小** 安装。

我们要进行以下任务在这个实验室中，估计的完成时间的 **45 分钟**:
* 配置主机提供所需的 **包** 和 **服务**
* 配置所需的 **逻辑虚拟网络** 来表示现实世界网络
* 配置 **虚拟机**，将成为您 **undercloud** 节点
* 确保 undercloud 节点的连通性和网络验证

> **注意**: 此实验室开始假定您已经完成非常小的先决条件才能连接到您底层的裸露金属主机，[上一页] [实验室安装] 章中详细说明。

## Lab Focus

这个实验室将集中于 **底层主机** (hypervisor) 和初始的 VM 将成为 undercloud 机:

<center>
    <img src="images/osp-director-env-3.png"/>
</center>

## Required Packages

我们要使用这个主机作为一个管理程序，我们会在运行所有的虚拟机和网络需要模仿真实的客户环境，我们需要安装和配置了多个包，具体 **libvirt** 库， **KVM 虚拟机监控程序** 和相关的工具和实用程序，我们可以使用一套。首先，连接回您的计算机:

~~~
client$ ssh root@<your given host>
~~~

> **注意**: 你可能要为未来易用性，创建您专用计算机'/etc/hosts'的条目。

让我们确保系统配置为 * * 硬件辅助/加速虚拟化 * *。简单来说，输出中应显示数量大于 '0' 如果 CPU 支持它;实际数是你 CPU(s) 的核心计数:

~~~
host# egrep -c '(vmx|svm)' /proc/cpuinfo
12
~~~

现在请确保加载 KVM 内核模块。在这种环境中我们使用的英特尔基于的 CPU 的但是如果你在追随这通过自学和你有基于 AMD 的系统，你可以简单地取代 '**kvm_intel**' 与 ' **kvm_amd** 在下面的命令:

~~~
host# modprobe kvm && modprobe kvm_intel
~~~

> **注意**: 如果您有任何错误与上面的命令，或者你已经无法加载 KVM 模块，请让老师知道，我们可以使它通过远程管理软件。

下一步，安装所需的 libvirt 和 qemu kvm 的软件包，并开始/启用 libvirt 守护进程:

~~~
host# yum install libvirt qemu-kvm virt-manager virt-install libguestfs-tools -y
(...)

host# systemctl enable libvirtd && systemctl start libvirtd
~~~

最后，添加在 * * xauth * * 和 * * virt-查看器 * * 包，我们可以在以后的实验室使用远程控制台:

~~~
host# yum install xorg-x11-apps xauth virt-viewer -y
(...)
~~~

现在你应该有所有继续实验室所需的程序包。

## Defining Networks

作为这些实验室的一部分，我们是 **第一** 要创建 **基本** 部署 RHEL OSP 使用 OSP 导演要传达的概念，然后重新部署与更多 **先进** 以后配置。因此，我们会保持网络简单的第一次迭代。

Overcloud 节点将通过 undercloud 节点中的通过部署 **PXE**-undercloud 节点仅仅告诉基础 **虚拟机监控程序**，libvirt 通过请求的虚拟机电源上 (或在 **裸-金属** 环境调用通过 IPMI 电源)。我们把这个叫做 **资源调配** 网络，并应该是专用的接口，不是已经用于管理或直接连接。此外，每个系统可以有例如 OpenStack 用于定义的一个或多个网络群集管理、 存储、 公众和外部连接。有关基本配置中，我们将配置每个节点有两个接口、 PXE/管理网络和公共/外部接入网:

<center>
    <img src=./images/basic_networks.png>
</center>

> **注意**: 如上所示的节点计数例如纯粹是目的，我们将部署中的 HA 配置要比这更多的控制器节点。

**公共 / 外部** 网络已经为我们内创建 **libvirt**，并且已经有 **DHCP** 默认情况下启用了，我们会需要这个来帮助我们引导:

~~~
host# virsh net-dumpxml default
<network>
  <name>default</name>
  <uuid>55e6e167-ee39-4747-a158-6aa80c0a38d6</uuid>
  <forward mode='nat'>
    <nat>
      <port start='1024' end='65535'/>
    </nat>
  </forward>
  <bridge name='virbr0' stp='on' delay='0'/>
  <mac address='52:54:00:13:4f:ae'/>
  <ip address='192.168.122.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.122.2' end='192.168.122.254'/>
    </dhcp>
  </ip>
</network>
~~~

## Next Lab

The next lab will be the configuration of the undercloud, click [here][lab02](./lab02.md) to proceed.


