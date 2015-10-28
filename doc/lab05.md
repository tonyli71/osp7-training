# 实验五：注册和检视您的目标节点

## 序言

进行生产云的部署前最后一项工作就是注册和检视您的目标节点。与OSP 6 的  RHEL OSP installer (Staypuft) 不同，OSP 导演不会自动发现新硬件。您必须告知 OSP 导演哪些节点通过 **IPMI** 与它相连。

我们通过一个JSON结构文件导入这些配置。

本实验我们可能需要 **90** 分钟。

## 实验对象

the underlying **host** (hypervisor) and **undercloud** VM:

<center>
    <img src="images/osp-director-env-3.png"/>
</center>

## 创建您的 overcloud 节点

~~~
host# cd /var/lib/libvirt/images/
host# for i in {1..5}; do qemu-img create -f qcow2 \
    -o preallocation=metadata overcloud-node$i.qcow2 60G; done
Formatting 'overcloud-node1.qcow2', fmt=qcow2 size=64424509440 encryption=off cluster_size=65536 preallocation='metadata' lazy_refcounts=off
Formatting 'overcloud-node2.qcow2', fmt=qcow2 size=64424509440 encryption=off cluster_size=65536 preallocation='metadata' lazy_refcounts=off
Formatting 'overcloud-node3.qcow2', fmt=qcow2 size=64424509440 encryption=off cluster_size=65536 preallocation='metadata' lazy_refcounts=off
Formatting 'overcloud-node4.qcow2', fmt=qcow2 size=64424509440 encryption=off cluster_size=65536 preallocation='metadata' lazy_refcounts=off
Formatting 'overcloud-node5.qcow2', fmt=qcow2 size=64424509440 encryption=off cluster_size=65536 preallocation='metadata' lazy_refcounts=off
~~~

~~~
host# for i in {1..5}; do \
    virt-install --ram 8192 --vcpus 4 --os-variant rhel7 \
    --disk path=/var/lib/libvirt/images/overcloud-node$i.qcow2,device=disk,bus=virtio,format=qcow2 \
    --noautoconsole --vnc --network network:provisioning \
    --network network:default --name overcloud-node$i \
    --cpu SandyBridge,+vmx \
    --dry-run --print-xml > /tmp/overcloud-node$i.xml; \
    virsh define --file /tmp/overcloud-node$i.xml; done
~~~

~~~
host# virsh list --all
 Id    Name                           State
----------------------------------------------------
 1     undercloud                     running
 -     overcloud-node1                shut off
 -     overcloud-node2                shut off
 -     overcloud-node3                shut off
 -     overcloud-node4                shut off
 -     overcloud-node5                shut off
~~~

## 创建仿真IPMI访问

~~~
host# useradd stack
host# echo "Redhat01" | passwd stack --stdin
~~~

~~~
host# cat << EOF > /etc/polkit-1/localauthority/50-local.d/50-libvirt-user-stack.pkla
[libvirt Management Access]
Identity=unix-user:stack
Action=org.libvirt.unix.manage
ResultAny=yes
ResultInactive=yes
ResultActive=yes
EOF
~~~

~~~
host# virsh --connect qemu+ssh://stack@192.168.122.1/system list --all
stack@192.168.122.1's password: ******
 Id    Name                           State
----------------------------------------------------
 1     undercloud                     running
 -     overcloud-node1                shut off
 -     overcloud-node2                shut off
 -     overcloud-node3                shut off
 -     overcloud-node4                shut off
 -     overcloud-node5                shut off
~~~

~~~
host# ssh stack@undercloud
undercloud$ ssh-copy-id -i ~/.ssh/id_rsa.pub stack@192.168.122.1
~~~

~~~
undercloud$ virsh --connect qemu+ssh://stack@192.168.122.1/system list --all
 Id    Name                           State
----------------------------------------------------
 1     undercloud                     running
 -     overcloud-node1                shut off
 -     overcloud-node2                shut off
 -     overcloud-node3                shut off
 -     overcloud-node4                shut off
 -     overcloud-node5                shut off
~~~

## 创建您的Overcloud环境文件

~~~
undercloud$ for i in {1..5}; do \
    virsh -c qemu+ssh://stack@192.168.122.1/system domiflist overcloud-node$i | awk '$3 == "provisioning" {print $5};'; \
    done > /tmp/nodes.txt
~~~

~~~
undercloud$ cat /tmp/nodes.txt
52:54:00:99:51:60
52:54:00:1d:00:04
52:54:00:e0:db:5e
52:54:00:b6:99:5c
52:54:00:b6:47:f4
~~~

~~~
undercloud$ jq . << EOF > ~/instackenv.json
{
  "ssh-user": "stack",
  "ssh-key": "$(cat ~/.ssh/id_rsa)",
  "power_manager": "nova.virt.baremetal.virtual_power_driver.VirtualPowerManager",
  "host-ip": "192.168.122.1",
  "arch": "x86_64",
  "nodes": [
    {
      "name": "overcloud-node1",
      "pm_addr": "192.168.122.1",
      "pm_password": "$(cat ~/.ssh/id_rsa)",
      "pm_type": "pxe_ssh",
      "mac": [
        "$(sed -n 1p /tmp/nodes.txt)"
      ],
      "cpu": "4",
      "memory": "8192",
      "disk": "60",
      "arch": "x86_64",
      "pm_user": "stack"
    },
    {
      "name": "overcloud-node2",
      "pm_addr": "192.168.122.1",
      "pm_password": "$(cat ~/.ssh/id_rsa)",
      "pm_type": "pxe_ssh",
      "mac": [
        "$(sed -n 2p /tmp/nodes.txt)"
      ],
      "cpu": "4",
      "memory": "8192",
      "disk": "60",
      "arch": "x86_64",
      "pm_user": "stack"
    },
    {
      "name": "overcloud-node3",
      "pm_addr": "192.168.122.1",
      "pm_password": "$(cat ~/.ssh/id_rsa)",
      "pm_type": "pxe_ssh",
      "mac": [
        "$(sed -n 3p /tmp/nodes.txt)"
      ],
      "cpu": "4",
      "memory": "8192",
      "disk": "60",
      "arch": "x86_64",
      "pm_user": "stack"
    },
    {
      "name": "overcloud-node4",
      "pm_addr": "192.168.122.1",
      "pm_password": "$(cat ~/.ssh/id_rsa)",
      "pm_type": "pxe_ssh",
      "mac": [
        "$(sed -n 4p /tmp/nodes.txt)"
      ],
      "cpu": "4",
      "memory": "8192",
      "disk": "60",
      "arch": "x86_64",
      "pm_user": "stack"
    },
    {
      "name": "overcloud-node5",
      "pm_addr": "192.168.122.1",
      "pm_password": "$(cat ~/.ssh/id_rsa)",
      "pm_type": "pxe_ssh",
      "mac": [
        "$(sed -n 5p /tmp/nodes.txt)"
      ],
      "cpu": "4",
      "memory": "8192",
      "disk": "60",
      "arch": "x86_64",
      "pm_user": "stack"
    }
  ]
}
EOF
~~~

## 导入您的JSON文件

先对JSON文件进行验证

~~~
undercloud$ curl -O https://raw.githubusercontent.com/rthallisey/clapper/master/instackenv-validator.py
(...)

undercloud$ python instackenv-validator.py -f instackenv.json
INFO:__main__:Checking node 192.168.122.1
DEBUG:__main__:Identified virtual node
INFO:__main__:Checking node 192.168.122.1
DEBUG:__main__:Identified virtual node
INFO:__main__:Checking node 192.168.122.1
DEBUG:__main__:Identified virtual node
INFO:__main__:Checking node 192.168.122.1
DEBUG:__main__:Identified virtual node
INFO:__main__:Checking node 192.168.122.1
DEBUG:__main__:Identified virtual node
DEBUG:__main__:Baremetal IPs are all unique.
DEBUG:__main__:MAC addresses are all unique.

--------------------
SUCCESS: instackenv validator found 0 errors
~~~

导入

~~~
undercloud$ source ~/stackrc
undercloud$ openstack baremetal import --json instackenv.json
~~~

~~~
undercloud$ ironic node-list
+--------------------------------------+-----------------+---------------+-------------+-----------------+-------------+
| UUID                                 | Name            | Instance UUID | Power State | Provision State | Maintenance |
+--------------------------------------+-----------------+---------------+-------------+-----------------+-------------+
| 93785975-0527-4501-8504-00a24d30087f | overcloud-node1 | None          | power off   | available       | False       |
| 4451d778-6fd2-462b-ab27-08d2c1807d60 | overcloud-node2 | None          | power off   | available       | False       |
| b3ba94fa-c360-40da-acb0-af7e9079af71 | overcloud-node3 | None          | power off   | available       | False       |
| f91db03d-cf00-4f6a-be03-66a98812fc2e | overcloud-node4 | None          | power off   | available       | False       |
| 789af767-51fe-417d-bd05-fc3c779e7737 | overcloud-node5 | None          | power off   | available       | False       |
+--------------------------------------+-----------------+---------------+-------------+-----------------+-------------+
~~~

## 检视您的节点

~~~
undercloud$ openstack baremetal configure boot
~~~

~~~
undercloud$ ironic node-show 93785975-0527-4501-8504-00a24d30087f | grep -A1 deploy
| driver_info            | {u'ssh_username': u'stack', u'deploy_kernel':                         |
|                        | u'806c72d2-9629-4e58-ad42-917c923786a3', u'deploy_ramdisk':           |
|                        | u'b7e70339-2e66-4975-aaf8-4187ca2947ff', u'ssh_key_contents': u'----- |
~~~

~~~
undercloud$ openstack image show 806c72d2-9629-4e58-ad42-917c923786a3
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | 1b46154c65910ee3e70796eb93f3c490     |
| container_format | aki                                  |
| created_at       | 2015-08-03T12:31:29.000000           |
| deleted          | False                                |
| disk_format      | aki                                  |
| id               | 806c72d2-9629-4e58-ad42-917c923786a3 |
| is_public        | True                                 |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | bm-deploy-kernel                     |
| owner            | 74ddd233842043269955550d890953ab     |
| protected        | False                                |
| size             | 5026624                              |
| status           | active                               |
| updated_at       | 2015-08-03T12:31:29.000000           |
+------------------+--------------------------------------+
~~~

## fix boot 问题

~~~
undercloud$ sudo su -
undercloud# cat << EOF > /usr/bin/bootif-fix
#!/usr/bin/env bash

while true;
        do find /httpboot/ -type f ! -iname "kernel" ! -iname "ramdisk" ! -iname "*.kernel" ! -iname "*.ramdisk" -exec sed -i 's|{mac|{net0/mac|g' {} +;
done
EOF

undercloud# chmod a+x /usr/bin/bootif-fix
undercloud# cat << EOF > /usr/lib/systemd/system/bootif-fix.service
[Unit]
Description=Automated fix for incorrect iPXE BOOFIF

[Service]
Type=simple
ExecStart=/usr/bin/bootif-fix

[Install]
WantedBy=multi-user.target
EOF

undercloud# systemctl daemon-reload
undercloud# systemctl enable bootif-fix
ln -s '/usr/lib/systemd/system/bootif-fix.service' '/etc/systemd/system/multi-user.target.wants/bootif-fix.service'
undercloud# systemctl start bootif-fix
~~~

~~~
undercloud# exit
undercloud$ openstack baremetal introspection bulk start
Setting available nodes to manageable...
Starting introspection of node: 93785975-0527-4501-8504-00a24d30087f
Starting introspection of node: 4451d778-6fd2-462b-ab27-08d2c1807d60
Starting introspection of node: b3ba94fa-c360-40da-acb0-af7e9079af71
Starting introspection of node: f91db03d-cf00-4f6a-be03-66a98812fc2e
Starting introspection of node: 789af767-51fe-417d-bd05-fc3c779e7737

Waiting for discovery to finish...
(...)
~~~

通过**virt-manager**监控这一过程

~~~
client$ ssh root@<your host ip> -Y
~~~

~~~
host# virt-manager &
~~~

~~~
client$ virt-manager --connect qemu+ssh://root@<your given host>
~~~

或在另一终端运行

~~~
undercloud$ sudo journalctl -l -u openstack-ironic-discoverd \
	-u openstack-ironic-discoverd-dnsmasq -f
~~~

最终您会看到您的VM被检视完成

~~~
(...)

Waiting for discovery to finish...
Discovery for UUID 93785975-0527-4501-8504-00a24d30087f finished successfully.
Discovery for UUID b3ba94fa-c360-40da-acb0-af7e9079af71 finished successfully.
Discovery for UUID 789af767-51fe-417d-bd05-fc3c779e7737 finished successfully.
Discovery for UUID 4451d778-6fd2-462b-ab27-08d2c1807d60 finished successfully.
Discovery for UUID f91db03d-cf00-4f6a-be03-66a98812fc2e finished successfully.
Setting manageable nodes to available...
Node 93785975-0527-4501-8504-00a24d30087f has been set to available.
Node 4451d778-6fd2-462b-ab27-08d2c1807d60 has been set to available.
Node b3ba94fa-c360-40da-acb0-af7e9079af71 has been set to available.
Node f91db03d-cf00-4f6a-be03-66a98812fc2e has been set to available.
Node 789af767-51fe-417d-bd05-fc3c779e7737 has been set to available.
Discovery completed.
~~~

## 检验发现的数据

~~~
undercloud$ ironic node-show 93785975-0527-4501-8504-00a24d30087f | grep extra
| extra      | {u'newly_discovered': u'true', u'hardware_swift_object':
             | u'extra_hardware-93785975-0527-4501-8504-00a24d30087f'}
~~~

~~~
undercloud$ OS_TENANT_NAME=service swift download \
	ironic-discoverd extra_hardware-93785975-0527-4501-8504-00a24d30087f
extra_hardware-93785975-0527-4501-8504-00a24d30087f [auth 0.760s, headers 0.914s, total 0.915s, 0.062 MB/s]
~~~

~~~
undercloud$ cat extra_hardware-93785975-0527-4501-8504-00a24d30087f | python -m json.tool
(...)
~~~

## 实验成果

* **instackenv.json**
* **discovery**

## 下一实验

下一实验将是部署Overcloud, 点击[这里 lab6](./lab06.md) 进入。
