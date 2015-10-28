# 实验四: 创建和配置的基本节点镜像

## 序言

在用OSP director 部署overcloud时 ，重要的事情是意识到我们**不**采用kickstart启动或从零开始像我们用的 RHEL OSP 安装程序 (Staypuft) 那样在每台机器上部署 RHEL。OSP Director 部署依赖于 **预构建的镜像** ，通过 PXE 部署到节点。。这些磁盘映像通常已经包含 **RHEL** 和所需的 **RHEL OSP** 包。这种机制符合OpenStack镜像/模板的部署工作负载落的**原则。

在以前的实验，我们看到Ironic利用 **PXE** 部署磁盘镜像到节点为两个步骤，首先是初始**discovery/introspection**阶段，然后是 **实际部署**。在 undercloud 上，我们需要为这两个任务提供 **三** 个不同类型的镜像:

镜像类型        | 用途
--------------- | -----------------
**Discovery**   | 在部署阶段之前, 节点调配到发现模式，更加深入地**specification**，以确定有 **问题** 的硬件，并在帮助 **角色匹配**。Ironic提供了内核和 ramdisk到每个已知的节点。这些文件包含**introspection**所有所需的二进制文件和库 。<br><br> 所需文件: **discovery-ramdisk-ironic.{initramfs,kernel}**
**Deployment**  | 当管理员开始 **overcloud 部署**时，部署机制 **预建** 的RHEL 磁盘镜像必须可用。部署镜像只是一个基本内核和 ramdisk 引导具有预构建镜像的节点通过**undercloud Glance**服务器下载他们和"*dd'ing*"镜像到该节点的磁盘上。<br><br> 所需文件: **deploy-ramdisk-ironic.{initramfs,kernel}**
**Overcloud**   | **Overcloud** 类型镜像是在系统引导过程中**deployment**到节点上的。这里的主要区别是这些镜像像包含的软件将 **持久化** 在 overcloud 节点上,而**不是**只是为特定目的ramdisk镜像。<br><br> 所需文件:**overcloud-full.{initrd,vmlinuz,qcow2}**

Red Hat 提供 " **预构建的**" 镜像可用于大多数客户overcloud部署。OSP Director 可以不需变更使用这些镜像。我们也为客户提供对模版 **建立自己的图像** 的方法，以适应特定用户的需要和要求，这能帮助我们的计划外的 **业务工具**支持 和  测试正在新修补的程序和自动升级。

> **注意**: " **预构建的**" 镜像可在以下链接找到: **https://access.redhat.com/site/products/red-hat-enterprise-linux-openstack-platform/**

这第四个实验将引导您完成上面; 生成的 overcloud 镜像将用于后续的实验，估计的完成时间约 **90 分钟**:

* **创造** 所有类型的 **overcloud** 镜像
* **调查** OSP Director 如何创建 overcloud 镜像
* 让 **创建后修改** 的镜像以适合我们的需要
* **上传** 必要的转换后镜像到undercloud  **Glance** 服务器 

> **注意**: 此实验开始假定您已经完成 [第三个实验] [lab3] 和已部署了 undercloud。如果这不是，请重温以前的实验。

## 实验重点

这个实验将继续把重点放在 **undercloud** VM:

<center>
    <img src="images/osp-director-env-6.png"/>
</center>

## 镜像的创建

对于多数情况我们使用 *rhel-guest-image* 云镜像作为一个起始镜像应该够了，因此本实验 overcloud 镜像我们会用这个。然而，它也可能是一个从零开始新建立的 RHEL 操作系统，安装所需的软件包，删除任何唯一标识(例如在配置文件中的 MAC 地址)的组件的镜像 ，如运行 virt-sysprep，等等。而*rhel-guest-image* 不需要此类操作。

在实验 3 完成后我们回到我们的主机机采取 **快照** 刚安装的 **undercloud** 虚拟机，所以这个实验现在假定你仍然连接到基础 **主机**。复制您的 SSH 公钥到 *stack* 用户:

~~~
host# ssh-copy-id stack@undercloud
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already instled
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
stack@undercloud's password:

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'stack@undercloud'"
and check to make sure that only the key(s) you wanted were added.
~~~

从主机服务器复制已经 **修改** RHEL 客镜像:

~~~
host# scp /var/lib/libvirt/images/rhel7-guest.qcow2 stack@undercloud:~/
rhel7-guest.qcow2
~~~

返回到 **undercloud** 服务器:

~~~
host# ssh stack@undercloud
~~~

镜像生成器需要知道你使用哪个 **操作系统**  (这通常可以 RHEL，CentOS，Fedora)，它是这个实验用 **RHEL 7**。我们还将设置几个参数，帮助知道我们正在使用的镜像构建工具 **本地提供**的 RHEL 7 客镜像 (而不是从 Red Hat 下载他们) 和从那里找出额外/必要的**包**:

~~~
undercloud$ export NODE_DIST=rhel7
undercloud$ export DIB_LOCAL_IMAGE=rhel7-guest.qcow2
undercloud$ export USE_DELOREAN_TRUNK=0
undercloud$ export RHOS=1
undercloud$ export DIB_YUM_REPO_CONF="/etc/yum.repos.d/rhos-release-7-director.repo /etc/yum.repos.d/rhos-release-7.repo /etc/yum.repos.d/rhos-release-rhel-7.1.repo"
~~~

**验证** 目前镜象的变量正确性:

~~~
undercloud$ ls $DIB_LOCAL_IMAGE
-rw-r--r--. 1 stack stack 421M Sep  1 16:45 rhel7-guest.qcow2
~~~

建立 **overcloud** 镜像和日志审查输出:

~~~
undercloud$ source ~/stackrc
undercloud$ time openstack overcloud image build --all 2>&1 | tee openstack_image_build.log
(...)

real    33m37.235s
user    20m32.044s
sys     12m17.020s
~~~

> **注意**: 这整个过程将需要大约 **30 分钟**;你可以去喝杯咖啡，或者打开 **新的控制台**，然后按照下面的说明，探讨发生一些什么。请注意，"**cat: write error: broken pipe**" **不** 表示有错误。关于 diskimage 生成器进程的更多详细信息请参阅 [这个链接] [dib 信息]。

## 发生一些什么?

Openstack 客户端在这里调用镜像生成 python 脚本:

~~~
/usr/lib/python2.7/site-packages/rdomanager_oscplugin/v1/overcloud_image.py
~~~

瞥一眼脚本揭示了几个**disk-image-create**调用子进程

~~~
undercloud$ grep subprocess.call /usr/lib/python2.7/site-packages/rdomanager_oscplugin/v1/overcloud_image.py
        subprocess.call('disk-image-create {0}'.format(args), shell=True)
        subprocess.call('ramdisk-image-create {0}'.format(args), shell=True)
        subprocess.call('sudo cp -f "{0}" "{1}"'.format(src, dest), shell=True)
~~~

这些都是**diskimage-builder**核心实用程序提供的包。更多详细信息可参考 http://docs.openstack.org/developer/diskimage-builder/

~~~
undercloud$ rpm -qf $(which disk-image-create)
diskimage-builder-0.1.46-3.el7ost.noarch
undercloud$ rpm -qf $(which ramdisk-image-create)
diskimage-builder-0.1.46-3.el7ost.noarch
~~~

用mlocate可以找到它们

~~~
undercloud$ sudo yum -y install mlocate
undercloud$ sudo updatedb
undercloud$ locate 10-rhel7-cloud-image
/usr/share/diskimage-builder/elements/rhel7/root.d/10-rhel7-cloud-image
~~~

~~~
undercloud$ tree /usr/share/diskimage-builder/elements/base/
/usr/share/diskimage-builder/elements/base/
├── cleanup.d
│   ├── 01-ccache
│   └── 99-tidy-logs
├── element-deps
├── environment.d
│   └── 10-ccache.bash
├── extra-data.d
│   └── 50-store-build-settings
├── install.d
│   ├── 00-baseline-environment
│   ├── 00-up-to-date
│   ├── 10-cloud-init
│   ├── 50-store-build-settings
│   └── 99-dkms
├── package-installs.yaml
├── pkg-map
├── pre-install.d
│   └── 03-baseline-tools
├── README.rst
└── root.d
    └── 01-ccache

6 directories, 15 files
~~~

> **注意** : 这一构建过程需要大量的内存，建议您的 **Undercloud** 有16GB内存

~~~
undercloud$ grep -v "^+" openstack_image_build.log | grep -v ^$ | less
~~~

~~~
Target: root.d

Script                                     Seconds
---------------------------------------  ----------

01-ccache                                     0.017
10-rhel7-cloud-image                         93.202
50-yum-cache                                  0.045
90-base-dib-run-parts                         0.037
~~~

~~~
undercloud$ locate 01-ccache
/usr/share/diskimage-builder/elements/base/cleanup.d/01-ccache
/usr/share/diskimage-builder/elements/base/root.d/01-ccache
/usr/share/diskimage-builder/elements/opensuse/install.d/01-ccache-symlinks
~~~

~~~
Target: extra-data.d

Script                                     Seconds
---------------------------------------  ----------

01-inject-ramdisk-build-files                 0.031
10-create-pkg-map-dir                         0.114
20-manifest-dir                               0.021
50-add-targetcli-module                       0.038
50-store-build-settings                       0.006
75-inject-element-manifest                    0.040
98-source-repositories                        0.041
99-enable-install-types                       0.023
99-squash-package-install                     0.221
99-yum-repo-conf                              0.039

Target: pre-install.d

Script                                     Seconds
---------------------------------------  ----------

00-fix-requiretty                             0.015
00-rhel-registration                          0.015
00-rhsm                                       0.006
00-usr-local-bin-secure-path                  0.009
01-override-yum-arch                          0.006
01-yum-install-bin                            0.013
01-yum-keepcache                              0.012
02-package-installs                          11.042
03-baseline-tools                             0.005
04-dib-init-system                            0.013
10-rhel-blacklist                             0.005
15-remove-grub                                2.596
99-package-uninstalls                         0.099

Running install-packages install. Package list: iscsi-initiator-utils

Target: install.d

Script                                     Seconds
---------------------------------------  ----------

00-baseline-environment                       2.552
00-up-to-date                               192.938
01-package-installs                          11.800
10-cloud-init                                 0.005
10-openstack-selinux-rhel                    31.081
20-install-dracut-deps                        2.755
50-store-build-settings                       0.010
99-dkms                                       0.063
99-package-uninstalls                         0.078

Target: post-install.d

Script                                     Seconds
---------------------------------------  ----------

00-package-installs                           0.070
01-ensure-binaries                            0.049
01-ensure-drivers                             0.011
05-fstab-rootfs-label                         0.006
06-network-config-nonzeroconf                 0.007
99-build-dracut-ramdisk                     149.093
99-package-uninstalls                         0.103
99-reset-yum-conf                             0.011

Running install-packages install. Package list: yum-utils

Target: finalise.d

Script                                     Seconds
---------------------------------------  ----------

01-clean-old-kernels                          6.348
11-selinux-fixfiles-restore                   7.977
99-cleanup-tmp-grub                           0.006
99-unregister                                 0.003

Target: cleanup.d

Script                                     Seconds
---------------------------------------  ----------

01-ccache                                     0.010
01-copy-manifests-dir                         0.018
99-extract-ramdisk-files                      0.073
99-remove-yum-repo-conf                       0.017
99-tidy-logs                                  0.041
(...)
~~~

~~~
...
Converting image using qemu-img convert
~~~

观察您的成果

~~~
undercloud$ ls -lh
total 1.7G
drwxrwxr-x.  3 stack stack   26 Aug  3 19:08 deploy-ramdisk-ironic.d
-rw-r--r--.  1 stack stack  56M Aug  3 19:08 deploy-ramdisk-ironic.initramfs
-rwxr-xr-x.  1 stack stack 4.8M Aug  3 19:08 deploy-ramdisk-ironic.kernel
-rw-rw-r--.  1 stack stack 213K Aug  3 19:08 dib-deploy.log
-rw-rw-r--.  1 stack stack  15M Aug  3 19:23 dib-discovery.log
-rw-rw-r--.  1 stack stack 602K Aug  3 19:32 dib-overcloud-full.log
drwxrwxr-x.  3 stack stack   26 Aug  3 19:23 discovery-ramdisk.d
-rw-r--r--.  1 stack stack 147M Aug  3 19:23 discovery-ramdisk.initramfs
-rwxr-xr-x.  1 stack stack 4.8M Aug  3 19:23 discovery-ramdisk.kernel
-rw-r--r--.  1 stack stack 152M Aug  3 18:48 fedora-user.qcow2
-rw-rw-r--.  1 stack stack  16M Aug  3 19:32 openstack_image_build.log
drwxrwxr-x.  3 stack stack   26 Aug  3 19:30 overcloud-full.d
-rw-r--r--.  1 root  root   33M Aug  3 19:30 overcloud-full.initrd
-rw-r--r--.  1 stack stack 875M Aug  3 19:32 overcloud-full.qcow2
-rwxr-xr-x.  1 root  root  4.8M Aug  3 19:30 overcloud-full.vmlinuz
-rw-r--r--.  1 stack stack 419M Aug  3 18:47 rhel7-guest.qcow2
-rw-------.  1 stack stack  227 Aug  3 18:44 stackrc
-rw-r--r--.  1 stack stack  321 Aug  3 18:15 undercloud.conf
-rw-rw-r--.  1 stack stack 1.4K Aug  3 18:29 undercloud-passwords.conf
~~~

上载您的作品

~~~
undercloud$ openstack image list

undercloud$ openstack overcloud image upload
undercloud$ openstack image list
+--------------------------------------+------------------------+
| ID                                   | Name                   |
+--------------------------------------+------------------------+
| 9ceac7b5-b2d9-444d-ab33-8776f7a487e1 | bm-deploy-ramdisk      |
| 34f1bf3a-e295-48dd-ae93-cfe44b817116 | bm-deploy-kernel       |
| a03a230d-050e-412e-b524-8f0aabe3f41c | overcloud-full         |
| 5c7b6109-8a7e-4f11-894d-83ffad6bb54c | overcloud-full-initrd  |
| c6acf675-5232-4578-915c-3d019a01523f | overcloud-full-vmlinuz |
+--------------------------------------+------------------------+
~~~

清除不要的的文件以节省Undercloud的硬盘空间

~~~
undercloud$ rm -f ~/overcloud-full.qcow2
~~~

对您的作品进行再定制实验

~~~
undercloud$ for image in $(openstack image list | awk '{print $2}' | grep -v ID); \
    do openstack image delete $image ; done
~~~

~~~
undercloud$ sudo mkdir /usr/share/diskimage-builder/elements/rhel7/post-install.d

undercloud$ cat << EOF | sudo tee \
    /usr/share/diskimage-builder/elements/rhel7/post-install.d/01-hello-world
#!/bin/bash
echo "hello world" >> /root/hello-world.txt
EOF

undercloud$ sudo chmod +x \
    /usr/share/diskimage-builder/elements/rhel7/post-install.d/01-hello-world
~~~

~~~
undercloud$ export NODE_DIST=rhel7
undercloud$ export DIB_LOCAL_IMAGE=rhel7-guest.qcow2
undercloud$ export USE_DELOREAN_TRUNK=0
undercloud$ export RHOS=1
undercloud$ export DIB_YUM_REPO_CONF="/etc/yum.repos.d/rhos-release-7-director.repo /etc/yum.repos.d/rhos-release-7.repo /etc/yum.repos.d/rhos-release-rhel-7.1.repo"
undercloud$ time openstack overcloud image build --all 2>&1 | tee openstack_image_build2.log
~~~

验证定制结果

~~~
undercloud$ sudo yum -y install libguestfs-tools
~~~

~~~
undercloud$ guestfish -a overcloud-full.qcow2

Welcome to guestfish, the guest filesystem shell for
editing virtual machine filesystems and disk images.

Type: 'help' for help on commands
      'man' to read the manual
      'quit' to quit the shell

><fs>
Start the image with the run command

><fs> run
⟦▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒⟧
><fs>list-filesystems
/dev/sda: xfs
><fs> mount /dev/sda /
><fs> cat /root/hello-world.txt
hello world
><fs> exit
~~~

再次吧作品上载到undercloud的Glance

~~~
undercloud$ cd ~
undercloud$ openstack overcloud image upload
undercloud$ openstack image list
+--------------------------------------+------------------------+
| ID                                   | Name                   |
+--------------------------------------+------------------------+
| 66f85183-d065-41e2-8e6f-77fffc02c228 | bm-deploy-kernel       |
| 212d17fc-37fb-4a7d-817b-6ac80ac19298 | bm-deploy-ramdisk      |
| 0c31fb3e-1d22-4cc1-af81-ae61abce0b34 | overcloud-full         |
| 3b5dd3a1-c876-4491-9927-4061ca5b8437 | overcloud-full-initrd  |
| b0562fb7-8df3-4be7-b285-3cd2fe279a20 | overcloud-full-vmlinuz |
+--------------------------------------+------------------------+
~~~

## 对undercloud进行快照以备万一

~~~
undercloud$ exit
host# virsh snapshot-create-as undercloud undercloud-snap-lab4
~~~

## 本实验的成果

* 理解 OSP 导演 如何制作 **overcloud images**
* 了解 **diskimage-builder** 的调用
* 进行了定制实验
* 上载 **overcloud images** 准备部署


## 下一个实验

下一个实验将是创建并注册您的 **overcloud** 节点， click [here][lab5](./lab05.md) 

