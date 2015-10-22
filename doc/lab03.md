# 实验三: 安装并测试Undercloud

## 序言

在前一实验我们将 RHEL 7 虚拟机变成一个平台，它将承载我们 **undercloud**。我们已经安装所需的包，并配置我们的 undercloud 来表示我们的环境;现在，我们已准备好安装我们 undercloud 本身。

这第三个实验将引导您完成部署;我们要进行以下任务在这个实验室中，估计的完成时间的 **60 分钟**:

* 部署的 **undercloud** 的基础设置的配置选项
* 探索发生在undercloud的部署  **背景进程** 
* 示范的相关 **文件** 生成供我们使用，例如 **stackrc**
* 勘探部署的 undercloud 来看看如何我们 **配置** 选项被实现
* 一般 **测试** 的 undercloud，准备配置 overcloud

> **注意**: 此实验室开始假定您已经完成 [第二个实验] [lab2] 你与所需的软件包一起安装的 undercloud 机器和有文件准备好要使用的配置。如果这不是个案，请重温以前的实验。

## 实验重点

这个实验将继续把重点放在 undercloud 虚拟机:

<center>
    <img src="images/osp-director-env-6.png"/>
</center>

## 部署Undercloud

在这一节我们要实际安装 undercloud 本身。这个实验室假定的你仍然连接到你 **作为登录机 undercloud** 和 **堆栈** 用户，如果你是 **不** 执行以下操作:

~~~
host# ssh root@undercloud
undercloud# su - stack
undercloud$
~~~

下一步，与预配置的 undercloud.conf 文件为每个第二个实验室。确保你有类似输出如下所示:

~~~
undercloud$ whoami
stack

undercloud$ pwd
/home/stack

undercloud$ ll undercloud.conf
-rw-r--r--. 1 stack stack 5532 Jul 20 13:47 undercloud.conf
~~~

> **注意**: 如果你尝试作为 root 运行 undercloud 安装，它将部署大部分是 **成功**，但作为脚本对你期待 **不** 是 root 用户，它会抛出一些 **错误** 在结束了。请确保你是 **堆栈** 之前的用户。

我们下一阶段完全是 **自动化** 。假设你已经在没有犯错你 **undercloud.conf**，你可以调用统一的 CLI 来部署我们的机器，undercloud 注意到，它将查找当前工作目录中的 undercloud 配置文件。此步骤将需要约 20 分钟;它生成 *多* 的 **输出**。不要在意，不能够继续，我们将探索到底发生了什么事安装后。执行以下命令时可喝杯**咖啡**:

> **注意**: 当您咖啡返回时，它很可能仍在运行。建议您阅读前面作为实验室的其余的解释，在幕后发生了什么。

~~~
undercloud$ openstack undercloud install
(...)

#############################################################################
instack-install-undercloud complete.

The file containing this installation's passwords is at
/home/stack/undercloud-passwords.conf.

There is also a stackrc file at /home/stack/stackrc.

These files are needed to interact with the OpenStack services, and should be
secured.

#############################################################################
~~~

你应该看到的是上面的输出，undercloud 是 **成功地部署** 和它写了 **stackrc** 对我们来说，文件包含用于访问的 undercloud API-记住凭据，他们只是 **OpenStack API** 所以我们可以使用 **本机** 命令行实用程序来沟通。作为一个快速 **测试**，让我们确保我们可以做这;让我们问问我们在哪里可以找到 Nova 的重点:

~~~
undercloud$ source ~/stackrc
undercloud$ openstack catalog show nova
+-----------+---------------------------------------------------------------------------+
| Field     | Value                                                                     |
+-----------+---------------------------------------------------------------------------+
| endpoints | regionOne                                                                 |
|           |   publicURL: http://172.16.0.1:8774/v2/74ddd233842043269955550d890953ab   |
|           |   internalURL: http://172.16.0.1:8774/v2/74ddd233842043269955550d890953ab |
|           |   adminURL: http://172.16.0.1:8774/v2/74ddd233842043269955550d890953ab    |
|           |                                                                           |
| name      | nova                                                                      |
| type      | compute                                                                   |
+-----------+---------------------------------------------------------------------------+
~~~

## 发生了什么?

作为 undercloud 的部署是完全 **自动** 对我们来说，很难了解到底发生了什么和如何实施我们的配置选项。由于 undercloud 是 OpenStack 环境本身，部署这很大程度上可以通过执行 **现有Puppet舱单**，我们已经有，并已成功应用于在现有的部署工具;这就是 **完全** OSP 主任如何安装 (**大多数的**) undercloud。然而的是 **很多** 更多比只申请的一个Puppet清单的事情。

当命令"**openstack undercloud install**"是一个专用的脚本执行，获取执行 (**/usr/lib/python2.7/site-packages/instack_undercloud/undercloud.py**) 对其进行了大量的任务可以被分解成两个不同的阶段，第一次 **instack**，和第二次被 **os-refresh-config**:

* **Instack** 是旨在允许管理员部署工具 **diskimage 生成器元素** 本地。这些元素可以让我们来描述一种组件，我们想要加上 **地方** 机，例如软件包安装或系统配置。除其他外，OSP 主任船舶描述如何部署的特定 diskimage 生成器元素 **undercloud**，包括任何必要的 **Puppet** 清单，或 **脚本** 要求进行安装。
    通过部署 diskimage 生成器元素时 **instack**，它会看起来成 **install.d** 需要执行，例如设置为部署，元素的脚本的目录复制文件到其文件系统上的地方。它然后将复制任何 **os-refresh-config** 执行脚本配置 **os-refresh-config** 在稍后阶段。

* **os-refresh-config** 提供一种机制的订购和分期系统配置的部署，我们使用它与 OSP 导演 * * 申请 * * 脚本、 配置和 * 如果必要 **puppet** 与所需关联的清单 * * diskimage 生成器元素 * * 安装 undercloud。os-刷新-配置还可以用于刷新任何配置，如果它改变 * * 部署后 * *。

    os-refresh-config 调用 **os-apply-config** 申请额外配置到 **config files** ，我们不能与傀儡单独设置，或如果是没有当前的傀儡的舱单可用来做这项工作。

最重要 * * diskimage 生成器元素 * *，我们使用作为一部分的 OSP 主任是 * * 傀儡-堆栈-配置 * *、 部署元素 * * 大多数 * * 的我们 undercloud OpenStack 通过 * * 傀儡 * *。此元素还卖国贼我们表现出流上文所述在更多的细节。元素具有以下结构:

~~~
undercloud$ ls -la /usr/share/instack-undercloud/puppet-stack-config
total 48
drwxr-xr-x.  5 root root  4096 Jul 30 03:46 .
drwxr-xr-x. 33 root root  4096 Jul 30 03:46 ..
-rw-r--r--.  1 root root    21 Apr 16 10:30 element-deps
drwxr-xr-x.  2 root root    27 Jul 30 03:46 extra-data.d
drwxr-xr-x.  2 root root    99 Jul 30 03:46 install.d
drwxr-xr-x.  4 root root    47 Jul 30 03:46 os-refresh-config
-rw-r--r--.  1 root root    52 Apr 16 10:30 package-installs.yaml
-rw-r--r--.  1 root root 12858 Jul 23 07:10 puppet-stack-config.pp
-rw-r--r--.  1 root root  8315 Jul 23 07:10 puppet-stack-config.yaml.template
-rw-r--r--.  1 root root   345 Apr 16 10:30 README.rst
~~~

在此元素内部 **install.d** 你可以看到它有大量的脚本，它将使用该元素准备部署的目录:

~~~
undercloud$ ls -la /usr/share/instack-undercloud/puppet-stack-config/install.d
total 20
drwxr-xr-x. 2 root root   99 Jul 30 03:46 .
drwxr-xr-x. 5 root root 4096 Jul 30 03:46 ..
-rwxr-xr-x. 1 root root 4238 Jul 23 07:10 02-puppet-stack-config
-rwxr-xr-x. 1 root root  158 Apr 16 10:30 10-puppet-stack-config-puppet-module
-rwxr-xr-x. 1 root root  418 Apr 16 10:30 20-ironic-user
~~~

在这个元素中，**02-puppet-stack-config** 建立了 * * 基于中设置的配置选项的文件 hieradata * * **undercloud.conf** 和硬编码值 **'puppet-stack-config.yaml.template'** (在父目录中可用)。这输出 hieradata 文件到 **/etc/puppet/hieradata/puppet-stack-config.yaml**，让木偶在部署期间使用此 hieradata。

下面你可以看到其引用 undercloud 的原始模板。conf 条目，然后实际 hieradata 文件生成的 **02-puppet-stack-config**  为我们的部署:

~~~
undercloud$ head -n4 /usr/share/instack-undercloud/puppet-stack-config/puppet-stack-config.yaml.template
debug: {{UNDERCLOUD_DEBUG}}
controller_host: {{LOCAL_IP}} #local-ipv4
controller_admin_vip: {{UNDERCLOUD_ADMIN_VIP}}
controller_public_vip: {{UNDERCLOUD_PUBLIC_VIP}}

undercloud$ sudo head -n4 /etc/puppet/hieradata/puppet-stack-config.yaml
debug: True
controller_host: 172.16.0.1 #local-ipv4
controller_admin_vip: 172.16.0.11
controller_public_vip: 172.16.0.10
~~~

这个 hieradata 文件是可用在 * * 安装 * * 和 * * 超越 * *，允许两个木偶 manifest(s)，要知道哪些参数使用，并为管理员能够快速地在任何时间访问参数。
请注意，你必须执行此命令与 * * sudo * * 作为这些文件受到保护:

~~~
undercloud$ sudo hiera keystone::roles::admin::password
4e733f782f79827c266acfa19a948bca4c2a4b2c
~~~

下一个文件，**10-puppet-stack-config-puppet-module** 只是移动静态Puppet清单 **puppet-stack-config.pp** (从父目录) 到 **/etc/puppet/manifests**;它很大程度依赖于 hieradata。此清单描述 undercloud 的部署和执行时配置我们的绝大多数。例如:

~~~
undercloud$ grep -A5 "nova::network::neutron" /etc/puppet/manifests/puppet-stack-config.pp
class { 'nova::network::neutron':
      neutron_admin_auth_url    => join(['http://', hiera('controller_host'), ':35357/v2.0']),
      neutron_url               => join(['http://', hiera('controller_host'), ':9696']),
      neutron_admin_password    => hiera('neutron::server::auth_password'),
      neutron_admin_tenant_name => hiera('neutron::server::auth_tenant'),
      neutron_region_name       => '',
~~~

> **注意**: Instack 并 **不** 直接调用此Puppet清单，这获取由 **os-refresh-config** 后调用

Diskimage 生成器的每个元素 **可选** 有一组条目对于 os 的-刷新配置，我们 **puppet-stack-config** 它带有以下脚本的示例:

~~~
undercloud$ sudo yum install tree -y
(...)

undercloud$ cd /usr/share/instack-undercloud/puppet-stack-config/os-refresh-config/
undercloud$ tree
.
├── configure.d
│   └── 50-puppet-stack-config
└── post-configure.d
     └── 10-tftp-iptables

2 directories, 2 files
~~~

然后这些文件自动复制到所需的 **os-refresh-config** 目录执行;我已经强调了我们 **puppet-stack-config** 下面的脚本:

~~~
undercloud$ tree /usr/libexec/os-refresh-config/
/usr/libexec/os-refresh-config/
├── configure.d
│   ├── 00-apply-selinux-policy
│   ├── 20-compile-and-install-selinux-policies
│   ├── 20-os-apply-config
│   ├── 20-os-net-config
│   ├── 40-hiera-datafiles
│   ├── 50-puppet-stack-config                    <------
│   ├── 70-tftpboot-dir
│   ├── 80-tuskar-db-create
│   ├── 88-httpd-vhost-port
│   ├── 90-tuskar-db-sync
│   └── ipxe-vhost.template
├── post-configure.d
│   ├── 100-tuskar-api
│   ├── 101-plan-add-roles
│   ├── 10-tftp-iptables                          <------
│   ├── 68-ironic-conductor
│   ├── 80-seedstack-masquerade
│   ├── 98-undercloud-setup
│   ├── 99-admin-swiftoperator-role
│   ├── 99-refresh-completed
│   └── 99-restart-discovery
└── pre-configure.d
    ├── 90-discoverd-iptables
    └── 97-tuskar-fedora-iptables

3 directories, 22 files
~~~

> **注意**: 其他 **os-refresh-config** 脚本来自 undercloud 部署所必需的其他 diskimage 生成器元素。

当 **os-refresh-config** 执行，它将最终得到我们 **50-puppet-stack-config** 模块，提供它已经已在阶段覆盖掉前提条件的。此特定的模块实际调用的Puppet要运行清单 (即被复制 instack) 以部署我们通过Puppet的 undercloud:

~~~
undercloud$ cat /usr/libexec/os-refresh-config/configure.d/50-puppet-stack-config
#!/bin/bash

set -eux
set -o pipefail

set +e
puppet apply --detailed-exitcodes /etc/puppet/manifests/puppet-stack-config.pp
rc=$?
set -e

echo "puppet apply exited with exit code $rc"

if [ $rc != 2 -a $rc != 0 ]; then
    exit $rc
fi
~~~

根据其他 diskimage-builder 元素，它具有  **install.d** 脚本 (**02-undercloud-stack-heat-metadata**)，获取执行，此输出额外所需的配置详细信息在一个 json 结构 **/var/lib/heat-cfntools/cfn-init-data**，以从数据 **hieradata** (和 **undercloud.conf**)。

Json 文件中的每个元素描述某些配置更改，需要应用，例如"**os\_net\_config**"，本质上发出指示，额外需要应用到主机的网络配置:

~~~
undercloud$ grep -A25 os_net_config /var/lib/heat-cfntools/cfn-init-data
   "os_net_config": {
       "network_config": [
         {
           "addresses": [
             {
               "ip_netmask": "172.16.0.1/24"
             }
           ],
           "members": [
             {
               "primary": "true",
               "name": "eth0",
               "type": "interface"
             }
           ],
           "ovs_extra": [
             "br-set-external-id br-ctlplane bridge-id br-ctlplane"
           ],
           "name": "br-ctlplane",
           "type": "ovs_bridge"
         }
       ]
     }
    }
~~~

一种称为 **os-apply-config** 工具称为 **os-refresh-config** (通过 **/usr/libexec/os-refresh-config/configure.d/20-os-apply-config**)，其中读取 json 结构和映射条目到 **/usr/libexec/os-apply-config/templates/** 写模板 **configuration** 配置文件基于数据。

在 **os\_net\_config** 的示例中，它只是强迫到文件中的文件系统位于 **/etc/os-net-config/config.json** "**os\_net\_config**" 节所载的 json:

~~~
undercloud$ cat /usr/libexec/os-apply-config/templates/etc/os-net-config/config.json
{{#os_net_config}}
{{.}}
{{/os_net_config}}

undercloud$ cat /etc/os-net-config/config.json
{"network_config": [{"ovs_extra": ["br-set-external-id br-ctlplane bridge-id br-ctlplane"], "type": "ovs_bridge", "addresses": [{"ip_netmask": "172.16.0.1/24"}], "members": [{"type": "interface", "primary": "true", "name": "eth0"}], "name": "br-ctlplane"}]}
~~~

然后获取使用 **/usr/libexec/os-refresh-config/configure.d/20-os-net-config** 通过另一种工具设置网络配置 **os-net-config**:

~~~
undercloud$ cat /usr/libexec/os-refresh-config/configure.d/20-os-net-config
#!/bin/bash
set -eux

NET_CONFIG=$(os-apply-config --key os_net_config --type raw --key-default '')

if [ -n "$NET_CONFIG" ]; then
    os-net-config -c /etc/os-net-config/config.json -v
fi
~~~

这是如何我们结束与网络当它在原始定义 **undercloud.conf**. 看看 json 文件，看看os-apply-config还有什么配置:

~~~
undercloud$ less /var/lib/heat-cfntools/cfn-init-data
(...)
~~~

## 生成的文件

应配置了 **特定** 服务密码，你本来会猜测如何沟通你的 undercloud;值得庆幸的是 OSP 主任 **生成** 我们立即使用它所需的文件。成功部署后在 '**堆栈**' 用户的主目录，你应该有大量的文件:

~~~
undercloud$ ls -l ~
total 16
-rw-------. 1 stack stack  227 Jul 30 04:04 stackrc
-rw-r--r--. 1 stack stack 5532 Jul 30 03:47 undercloud.conf
-rw-rw-r--. 1 stack stack 1398 Jul 30 03:48 undercloud-passwords.conf
~~~

**stackrc** 文件包含大量的环境变量，我们可以源沟通我们 undercloud 通过本机 OpenStack CLI 的工具，也被称为 **运行时的配置文件**。对于那些你熟悉的概念 **keystonerc_\<user\>** 文件，这是完全相同，但为 undercloud 具体:

~~~
undercloud$ cat ~/stackrc
export NOVA_VERSION=1.1
export OS_PASSWORD=$(sudo hiera admin_password)
export OS_AUTH_URL=http://172.16.0.1:5000/v2.0
export OS_USERNAME=admin
export OS_TENANT_NAME=admin
export COMPUTE_API_VERSION=1.1
export OS_NO_CACHE=True
~~~

正如你所看到的它已配置重点终结点为我们所需的凭据，记住，我们可以使用 * * hiera * * 抢了必要的数据而无需以纯文本形式存储数据，如密码。我们可以此源文件，然后使用它与我们的 undercloud 进行通信:

~~~
undercloud$ source ~/stackrc
undercloud$ openstack host list
+-------------------------+-------------+----------+
| Host Name               | Service     | Zone     |
+-------------------------+-------------+----------+
| undercloud.redhat.local | consoleauth | internal |
| undercloud.redhat.local | scheduler   | internal |
| undercloud.redhat.local | conductor   | internal |
| undercloud.redhat.local | compute     | nova     |
+-------------------------+-------------+----------+
~~~

除了 * * stackrc * * 文件，也是所有生成的密码，在 **undercloud-passwords.conf** 方便列表。请记住，我们的配置与我们问 OSP 主任为我们，生成我们的密码，因此他们肯定不是令人难忘:

~~~
undercloud$ tail -n5 undercloud-passwords.conf
undercloud_rabbit_cookie=99939ed79bc494772b030ef49c144447c8841178
undercloud_rabbit_password=e489665f96bb19e57ad94c5b87cbd92ea13117cf
undercloud_rabbit_username=24302e50c73728274952381a33ab99b304a653d4
undercloud_heat_stack_domain_admin_password=ffe928d7a192a7cd99d49d911a9db83753e7263b
undercloud_swift_hash_suffix=b3d1600ecc79c62c7fc435e7880eb9fbf1710f95
~~~

最后，原 **undercloud.conf** 应仍驻留，保持不变，从原来的 undercloud 部署。

## 网络配置

正如我们在前一节中提到，OSP 主任用途 **os-refresh-config** 在部署时，在与一个专用的网络工具，**os-net-config** 底层主机上配置网络。所需的操作系统刷新配置脚本获取执行时它将申请网络，采取从这里 json 结构主机的配置:

~~~
undercloud$ cat /etc/os-net-config/config.json | python -m json.tool
{
    "network_config": [
        {
            "addresses": [
                {
                    "ip_netmask": "172.16.0.1/24"
                }
            ],
            "members": [
                {
                    "name": "eth0",
                    "primary": "true",
                    "type": "interface"
                }
            ],
            "name": "br-ctlplane",
            "ovs_extra": [
                "br-set-external-id br-ctlplane bridge-id br-ctlplane"
            ],
            "type": "ovs_bridge"
        }
    ]
}
~~~

> **注意**: 我们使用 **os-net-config** 时部署的 undercloud * * * 和 * overcloud，所以它是重要的是熟悉它。

基于 undercloud 配置文件中指定的参数，我们已经告诉我们想要的 OSP 主任我们 * * PXE / 管理 * * 必须 * * eth0 * *，我们希望我们本地的 IP 地址是 * * 172.16.0.1**。使用此信息，操作系统网络配置创建命名 OVS 桥 * * br-ctlplane * *，将用于部署我们的 overcloud，并将其附加物理 * * eth0 * * 到它的 nic:

~~~
undercloud$ sudo ovs-vsctl show
85287db1-3efa-4c4a-b7c6-96a2d7285b50
    Bridge br-int
        fail_mode: secure
        Port int-br-ctlplane
            Interface int-br-ctlplane
                type: patch
                options: {peer=phy-br-ctlplane}
        Port br-int
            Interface br-int
                type: internal
        Port "tapee85e21b-0d"
            tag: 1
            Interface "tapee85e21b-0d"
                type: internal
    Bridge br-ctlplane
        Port "eth0"
            Interface "eth0"
        Port phy-br-ctlplane
            Interface phy-br-ctlplane
                type: patch
                options: {peer=int-br-ctlplane}
        Port br-ctlplane
            Interface br-ctlplane
                type: internal
    ovs_version: "2.3.1-git3282e51"
~~~

然后，它将对这座桥的本地计算机的 IP 地址相关联:

~~~
undercloud$ cat /etc/sysconfig/network-scripts/ifcfg-br-ctlplane
# This file is autogenerated by os-net-config
DEVICE=br-ctlplane
ONBOOT=yes
HOTPLUG=no
NM_CONTROLLED=no
DEVICETYPE=ovs
TYPE=OVSBridge
BOOTPROTO=static
IPADDR=172.16.0.1
NETMASK=255.255.255.0
OVS_EXTRA="set bridge br-ctlplane other-config:hwaddr=52:54:00:eb:ee:2b -- br-set-external-id br-ctlplane bridge-id br-ctlplane"

undercloud$ ip addr show br-ctlplane
5: br-ctlplane: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN
    link/ether 52:54:00:eb:ee:2b brd ff:ff:ff:ff:ff:ff
    inet 172.16.0.1/24 brd 172.16.0.255 scope global br-ctlplane
       valid_lft forever preferred_lft forever
    inet 172.16.0.11/32 scope global br-ctlplane
       valid_lft forever preferred_lft forever
    inet 172.16.0.10/32 scope global br-ctlplane
       valid_lft forever preferred_lft forever
    inet6 fe80::5054:ff:feeb:ee2b/64 scope link
       valid_lft forever preferred_lft forever
~~~

请注意额外的虚拟 ip 地址也-已附加到此接口 **172.16.0.10** 和 **172.16.0.11**;IPs 的不使用，但将来会当我们支持 **undercloud HA**

所有的 undercloud 服务侦听这 **local_ip** 我们指定以前，即**172.16.0.1**，与 undercloud **Neutron** 例如:

~~~
undercloud$ curl -s http://172.16.0.1:9696 | python -m json.tool
{
    "versions": [
        {
            "id": "v2.0",
            "links": [
                {
                    "href": "http://172.16.0.1:9696/v2.0",
                    "rel": "self"
                }
            ],
            "status": "CURRENT"
        }
    ]
}
~~~

## Ironic支持 PXE 启动

OSP 主任使用 PXE 启动两个用途:

1. 初始 **introspection** 和 **discovery** 的 overcloud 节点-即了解 **capabilities** 为节点的 **role matching**，和 **benchmarking** 如果必要
2. 在 **provisioning** 的具有所需的节点 **role** 在 **overcloud deployment**。

为第一宗旨，undercloud 安装过程中 **os-refresh-config**  **os-config-apply**  设置 **ironic-discoverd** 从元数据生成从原始配置 **undercloud.conf**。我们使用  **dnsmasq** 为我们 DHCP 和 PXE 能力提供ironic-discoverd:

~~~
undercloud$ cat /usr/libexec/os-apply-config/templates/etc/ironic-discoverd/dnsmasq.conf
port=0
interface={{discovery.interface}}
bind-interfaces
dhcp-range={{discovery.iprange}},29
enable-tftp
tftp-root=/tftpboot
dhcp-match=ipxe,175
dhcp-boot=tag:!ipxe,undionly.kpxe,localhost.localdomain,{{local-ip}}
dhcp-boot=tag:ipxe,http://{{local-ip}}:8088/discoverd.ipxe

undercloud$ cat /etc/ironic-discoverd/dnsmasq.conf
port=0
interface=br-ctlplane
bind-interfaces
dhcp-range=172.16.0.50,172.16.0.80,29
enable-tftp
tftp-root=/tftpboot
dhcp-match=ipxe,175
dhcp-boot=tag:!ipxe,undionly.kpxe,localhost.localdomain,172.16.0.1
dhcp-boot=tag:ipxe,http://172.16.0.1:8088/discoverd.ipxe
~~~

显然需要侦听 dnsmasq 进程 **PXE/management**，因此 "**interface=br-ctlplane**" 选项是重要以上。这 dnsmasq 服务是systemd:

~~~
undercloud$ systemctl status openstack-ironic-discoverd-dnsmasq.service
openstack-ironic-discoverd-dnsmasq.service - PXE boot dnsmasq service for ironic-discoverd
   Loaded: loaded (/usr/lib/systemd/system/openstack-ironic-discoverd-dnsmasq.service; enabled)
   Active: active (running) since Thu 2015-07-30 04:04:13 EDT; 1 day 3h ago
 Main PID: 18814 (dnsmasq)
   CGroup: /system.slice/openstack-ironic-discoverd-dnsmasq.service
           └─18814 /sbin/dnsmasq --conf-file=/etc/ironic-discoverd/dnsmasq.conf
~~~

如果我们看看这个过程 (**18814**) 你可以看到它在与关联的所有 IP 地址上侦听 **br-ctlplane** 界面:

~~~
undercloud$ sudo netstat -tunpl | grep dnsmasq
udp        0      0 0.0.0.0:67              0.0.0.0:*                           18814/dnsmasq
udp        0      0 127.0.0.1:69            0.0.0.0:*                           18814/dnsmasq
udp        0      0 172.16.0.1:69           0.0.0.0:*                           18814/dnsmasq
udp        0      0 172.16.0.11:69          0.0.0.0:*                           18814/dnsmasq
udp        0      0 172.16.0.10:69          0.0.0.0:*                           18814/dnsmasq
udp6       0      0 ::1:69                  :::*                                18814/dnsmasq
udp6       0      0 fe80::5054:ff:feeb:e:69 :::*                                18814/dnsmasq
~~~

当我们开始节点的反思时，Ironic权力节点上通过相应的电源管理界面，他们 PXE 引导通过 **ironic-discoverd** 和他们下载一个临时的形象。此图像然后送入数据回Ironic数据库，我们可以用它来 **角色匹配**。

PXE 镜像传递通过 HTTP 通过虚拟的主机端口上上文 **8088**:

~~~
undercloud$ head -n3 /etc/httpd/conf.d/10-ipxe_vhost.conf
Listen 8088
<VirtualHost *:8088>
    DocumentRoot "/httpboot"

undercloud$ cat /httpboot/discoverd.ipxe
#!ipxe

dhcp

kernel http://172.16.0.1:8088/discovery.kernel discoverd_callback_url=http://172.16.0.1:5050/v1/continue RUNBENCH=0 ip=${ip}:${next-server}:${gateway}:${netmask} BOOTIF=${mac}
initrd http://172.16.0.1:8088/discovery.ramdisk
boot
~~~

> **注意**: 鹰眼当中你会注意到 **discovery.kernel** 和 **discovery.ramdisk** 文件中当前不存在 **/httpboot**;别担心，我们将创建这些在下一个实验。

所以这只是 **discovery** 过程，哪些关于实际**provisioning** 的 overcloud 部署期间的节点?
当我们提供一个节点时，它的 MAC 地址是 **列入黑名单** 与 **ironic-discoverd** dnsmasq 处理，因此可以有没有冲突，和管理 DHCP 和 PXE 是照顾Neutron运行在 undercloud 内。

此Neutron服务器管理子网 overcloud 的管理网络，我们使用 **172.16.0.0/24** 位于同一子网，但我们早些时候在 undercloud.conf 文件 中定义的固定范围 :

~~~
undercloud$ source ~/stackrc
undercloud$ neutron subnet-list
undercloud$ subnet_id=$(neutron subnet-list | grep 172.16.0.0 | awk '{print $2;}')
undercloud$ neutron subnet-show $subnet_id
+-------------------+----------------------------------------------------------------+
| Field             | Value                                                          |
+-------------------+----------------------------------------------------------------+
| allocation_pools  | {"start": "172.16.0.20", "end": "172.16.0.120"}                |
| cidr              | 172.16.0.0/24                                                  |
| dns_nameservers   |                                                                |
| enable_dhcp       | True                                                           |
| gateway_ip        | 172.16.0.1                                                     |
| host_routes       | {"destination": "169.254.169.254/32", "nexthop": "172.16.0.1"} |
| id                | 4e457dba-2a86-4f24-bc14-09466a880ca6                           |
| ip_version        | 4                                                              |
| ipv6_address_mode |                                                                |
| ipv6_ra_mode      |                                                                |
| name              |                                                                |
| network_id        | 0b452b19-ebbb-40d2-88b8-f8924e63eac1                           |
| subnetpool_id     |                                                                |
| tenant_id         | 74ddd233842043269955550d890953ab                               |
+-------------------+----------------------------------------------------------------+
~~~

> **注意**: 此网络的配置是由照顾 **/usr/libexec/os-refresh-config/post-configure.d/98-undercloud-setup** 通过数据在 **/var/lib/heat-cfntools/cfn-init-data** 举行

所以后 overcloud 的部署，启动任何节点收到他们 **分配** IP 地址并指示到的 undercloud Neutron服务器启动 **PXE**。Neutron服务器使用 **dnsmasq** 按正常传递这些指令;并且在网络命名空间内的听:

~~~
undercloud$ sudo ip netns list
qdhcp-0b452b19-ebbb-40d2-88b8-f8924e63eac1

undercloud$ netns=$(ip netns list | grep qdhcp)
undercloud$ sudo ip netns exec $netns netstat -tunpl
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 172.16.0.20:53          0.0.0.0:*               LISTEN      18577/dnsmasq
tcp6       0      0 fe80::f816:3eff:fe4b:53 :::*                    LISTEN      18577/dnsmasq
udp        0      0 172.16.0.20:53          0.0.0.0:*                           18577/dnsmasq
udp        0      0 0.0.0.0:67              0.0.0.0:*                           18577/dnsmasq
udp6       0      0 fe80::f816:3eff:fe4b:53 :::*                                18577/dnsmasq
~~~

这 dnsmasq 进程侦听打开 vSwitch 集成桥坐着一个接口 (**br-int**) 根据其他 OpenStack 部署:

~~~
undercloud$ ps ax | egrep 'dnsmasq|18577'
18577 ?        S      0:00 dnsmasq --no-hosts --no-resolv --strict-order --bind-interfaces --interface=tapee85e21b-0d --except-interface=lo --pid-file=/var/lib/neutron/dhcp/0b452b19-ebbb-40d2-88b8-f8924e63eac1/pid --dhcp-hostsfile=/var/lib/neutron/dhcp/0b452b19-ebbb-40d2-88b8-f8924e63eac1/host --addn-hosts=/var/lib/neutron/dhcp/0b452b19-ebbb-40d2-88b8-f8924e63eac1/addn_hosts --dhcp-optsfile=/var/lib/neutron/dhcp/0b452b19-ebbb-40d2-88b8-f8924e63eac1/opts --dhcp-leasefile=/var/lib/neutron/dhcp/0b452b19-ebbb-40d2-88b8-f8924e63eac1/leases --dhcp-range=set:tag0,172.16.0.0,static,86400s --dhcp-lease-max=256 --conf-file=/etc/dnsmasq-ironic.conf
~~~

它不是完全清楚在详细输出上面，但这使用自来水设备附加到 **br-int**，这反过来 **patch** 到 **br-ctlplane**

~~~
undercloud$ sudo ovs-vsctl list-ifaces br-int
int-br-ctlplane
tapee85e21b-0d
~~~

> **注意**: 我们将探索 PXE 资源调配的 overcloud 中的节点 **后** 实验。

最后，让我们创建另一个 * * 快照 * * 的我们 * * undercloud * * 虚拟机的这样我们有另一个已知的工作状态，我们可以 * * 回复 * * 如果需要向回。请注意，你将需要有一个控制台到你 * * 上运行这基础主机 * * 不 * * undercloud * * 机:

~~~
undercloud$ exit
undercloud# exit
host# virsh snapshot-create-as undercloud undercloud-snap-lab3
~~~

## 这个实验的结果

通过完成这个实验你应该达到以下目标:

* 理解的 OSP 主任如何使用 **instack** 和 **os-refresh-config** 部署 undercloud
* 认识到 * * 输出文件 * *，OSP 主任叶与你进一步使用
* 理解 OSP 主任的配置 * * undercloud 网络 * * 通过 os 网配置
* 将熟悉 undercloud 用Ironic PXE 启动配置

## 下一个实验

下一个实验将节点图像为 overcloud 的配置，请点击 [这里][lab4](./lab04.md) 

