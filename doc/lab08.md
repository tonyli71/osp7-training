# 实验八： 推到与重建

## 序言

目前我们已经部署了一个基本生产云并且对其基本功能进行了测试。是时候进行推到重建。本实验我们将进行高级的部署，我们会有不同的硬件描述、高级的网络（包括VLAN的隔离与绑定）以及Ceph的集成。因此我们会建立一组新的虚拟机以配合我们的实验。

我们会保留现存的undercloud,只需清理数据即可。目前本版本的OSP导演尚未能支持多生产云的部署，因此我们要对环境进行清理。

本实验预期要 **45 分钟

* 通过Heat清理现有的生产云
* 清理数据
* 定义新的环境
* 导入新的环境

## 实验目标

底层的宿主与**undercloud** VM:

<center>
<img src="images/osp-director-env-3.png">
</center>

## 清除现存的环境

~~~
undercloud$ cd /home/stack/
undercloud$ source ~/stackrc
undercloud$ heat stack-delete overcloud
+--------------------------------------+------------+--------------------+----------------------+
| id                                   | stack_name | stack_status       | creation_time        |
+--------------------------------------+------------+--------------------+----------------------+
| 1e95c401-86ec-4609-9fba-c67d078e0817 | overcloud  | DELETE_IN_PROGRESS | 2015-08-08T16:04:21Z |
+--------------------------------------+------------+--------------------+----------------------+
~~~

确认生产云堆栈已经移除

~~~
undercloud$ heat stack-list
+----+------------+--------------+---------------+
| id | stack_name | stack_status | creation_time |
+----+------------+--------------+---------------+
+----+------------+--------------+---------------+
~~~

删除Ironic的节点

~~~
undercloud$ ironic node-list
+--------------------------------------+-----------------+---------------+-------------+-----------------+-------------+^M
| UUID                                 | Name            | Instance UUID | Power State | Provision State | Maintenance |^M
+--------------------------------------+-----------------+---------------+-------------+-----------------+-------------+^M
| 93785975-0527-4501-8504-00a24d30087f | overcloud-node1 | None          | power off   | available       | False       |^M
| 4451d778-6fd2-462b-ab27-08d2c1807d60 | overcloud-node2 | None          | power off   | available       | False       |^M
| b3ba94fa-c360-40da-acb0-af7e9079af71 | overcloud-node3 | None          | power off   | available       | False       |^M
| f91db03d-cf00-4f6a-be03-66a98812fc2e | overcloud-node4 | None          | power off   | available       | False       |^M
| 789af767-51fe-417d-bd05-fc3c779e7737 | overcloud-node5 | None          | power off   | available       | False       |^M
+--------------------------------------+-----------------+---------------+-------------+-----------------+-------------+
~~~

~~~
undercloud$ for i in $(ironic node-list | grep -v UUID | awk ' { print $2 } '); \
    do ironic node-delete $i; done
~~~

清除配置文件

~~~
undercloud$ rm -f ~/overcloudrc ~/tripleo-overcloud-passwords ~/instackenv.json /tmp/nodes.txt
~~~

清除原生产云的VM

~~~
client$ ssh root@<your host ip>
host# for i in $(virsh list --all | awk ' /overcloud/ { print $2 } '); do \
    virsh destroy $i; virsh undefine $i; rm -f /var/lib/libvirt/images/$i.qcow2; \
    done
(...)
~~~

~~~
host# virsh list --all
 Id    Name                           State
----------------------------------------------------
 3     undercloud                     running
~~~

~~~
host# rm -f /tmp/nodes.txt /tmp/overcloud-node*
~~~

## 创建新的生产云虚拟机

<br><center>

Node Role        | vCPU Count  | Memory Allocation | Storage Allocation
------------| ----------- | -------------| ---------------
Controller  | <center>4</center> | <center>8GB</center> | <center>1x30GB</center>
Compute     | <center>2</center> | <center>6GB</center> | <center>1x20GB</center>
Storage     | <center>2</center> | <center>4GB</center> | <center>2x20GB</center>

</center><br>


## 下一实验

在下一实验我们将进行 **高级的** 生产云 **部署** , 我们将我已配置的机器 **导入** 到Ironic, 点击 [这里](./lab09.md) 进入。
