# 实验三: 安装并测试Undercloud

## 序言

在以前实验室里我们将我们通用的 RHEL 7 虚拟机变成一个平台，将承载我们 **undercloud**。我们已经安装所需的包，并配置我们的 undercloud 来表示我们的环境;现在，我们已准备好安装我们 undercloud 本身。

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

This then gets used by **/usr/libexec/os-refresh-config/configure.d/20-os-net-config** to setup the network configuration by another tool named **os-net-config**:

~~~
undercloud$ cat /usr/libexec/os-refresh-config/configure.d/20-os-net-config
#!/bin/bash
set -eux

NET_CONFIG=$(os-apply-config --key os_net_config --type raw --key-default '')

if [ -n "$NET_CONFIG" ]; then
    os-net-config -c /etc/os-net-config/config.json -v
fi
~~~

This is how we end up with the networking as it was defined in the original **undercloud.conf**. Take a look at the json file to see what else gets configured by os-apply-config:

~~~
undercloud$ less /var/lib/heat-cfntools/cfn-init-data
(...)
~~~


The next lab will be the configuration of node images for the overcloud, click [here][lab4](./lab04.md) to proceed.

