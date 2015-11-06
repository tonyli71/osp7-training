# 实验七: 测试已部署的Overcloud

## 序言

本实验大约需要 **30** 分钟：
* 连通性验证
* 试验使用
* 能力探索

## 实验对象

**undercloud** VM 与 **overcloud** VMs:

<center>
<img src="images/osp-director-env-4.png">
</center>

### 连通性

~~~
undercloud$ source ~/overcloudrc
undercloud$ openstack compute service list
+------------------+------------------------------------+----------+---------+-------+----------------------------+
| Binary           | Host                               | Zone     | Status  | State | Updated At                 |
+------------------+------------------------------------+----------+---------+-------+----------------------------+
| nova-scheduler   | overcloud-controller-1.localdomain | internal | enabled | up    | 2015-08-08T00:48:47.000000 |
| nova-scheduler   | overcloud-controller-0.localdomain | internal | enabled | up    | 2015-08-08T00:48:48.000000 |
| nova-scheduler   | overcloud-controller-2.localdomain | internal | enabled | up    | 2015-08-08T00:48:47.000000 |
| nova-consoleauth | overcloud-controller-1.localdomain | internal | enabled | up    | 2015-08-08T00:48:43.000000 |
| nova-consoleauth | overcloud-controller-0.localdomain | internal | enabled | up    | 2015-08-08T00:48:43.000000 |
| nova-consoleauth | overcloud-controller-2.localdomain | internal | enabled | up    | 2015-08-08T00:48:42.000000 |
| nova-conductor   | overcloud-controller-0.localdomain | internal | enabled | up    | 2015-08-08T00:48:41.000000 |
| nova-conductor   | overcloud-controller-2.localdomain | internal | enabled | up    | 2015-08-08T00:48:41.000000 |
| nova-conductor   | overcloud-controller-1.localdomain | internal | enabled | up    | 2015-08-08T00:48:42.000000 |
| nova-compute     | overcloud-compute-0.localdomain    | nova     | enabled | up    | 2015-08-08T00:48:40.000000 |
| nova-compute     | overcloud-compute-1.localdomain    | nova     | enabled | up    | 2015-08-08T00:48:41.000000 |
+------------------+------------------------------------+----------+---------+-------+----------------------------+
~~~

~~~
undercloud$ source ~/stackrc
undercloud$ nova list --fields name,networks
+--------------------------------------+------------------------+-----------------------+
| ID                                   | Name                   | Networks              |
+--------------------------------------+------------------------+-----------------------+
| 60acb13f-e32e-42b8-b8ba-055d791d6314 | overcloud-compute-0    | ctlplane=172.16.0.177 |
| 6bbb7af6-9d45-4e46-ad78-d0afdea3a422 | overcloud-compute-1    | ctlplane=172.16.0.175 |
| ca165353-32bb-45ce-a54f-ef28b7080f95 | overcloud-controller-0 | ctlplane=172.16.0.174 |
| 03bdd281-e080-4337-98ff-37a979b4faff | overcloud-controller-1 | ctlplane=172.16.0.176 |
| 6df9cb36-8555-427c-9c39-aecf2e142a26 | overcloud-controller-2 | ctlplane=172.16.0.178 |
+--------------------------------------+------------------------+-----------------------+
~~~

~~~
undercloud$ ssh heat-admin@172.16.0.174
The authenticity of host '172.16.0.174 (172.16.0.174)' can't be established.
ECDSA key fingerprint is 53:91:14:b1:b5:09:69:dd:1e:7d:2e:ce:ab:24:d0:88.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '172.16.0.174' (ECDSA) to the list of known hosts.
[heat-admin@overcloud-controller-0 ~]$ exit
undercloud$
~~~

~~~
undercloud$ for i in 174 176 178; \
    do scp -oStrictHostKeyChecking=no ~/overcloudrc heat-admin@172.16.0.$i:~; done
~~~

## 部署后事务

~~~
undercloud$ ssh heat-admin@172.16.0.174
overcloud$ source ~/overcloudrc
~~~

~~~
overcloud$ sudo ovs-vsctl show | grep -A10 br-ex
    Bridge br-ex
        Port "eth0"
            Interface "eth0"
        Port br-ex
            Interface br-ex
                type: internal
        Port phy-br-ex
            Interface phy-br-ex
                type: patch
                options: {peer=int-br-ex}
    Bridge br-int
        fail_mode: secure
        Port int-br-ex
            Interface int-br-ex
                type: patch
                options: {peer=phy-br-ex}
        Port patch-tun
            Interface patch-tun
                type: patch
                options: {peer=patch-int}
        Port br-int
            Interface br-int
                type: internal
~~~

~~~
overcloud$ ip addr show dev br-ex
5: br-ex: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN
    link/ether 52:54:00:2b:86:75 brd ff:ff:ff:ff:ff:ff
    inet 172.16.0.81/24 brd 172.16.0.255 scope global dynamic br-ex
       valid_lft 75864sec preferred_lft 75864sec
    inet 172.16.0.77/32 brd 172.16.0.255 scope global br-ex
       valid_lft forever preferred_lft forever
    inet6 fe80::5054:ff:fe2b:8675/64 scope link
       valid_lft forever preferred_lft forever
~~~

~~~
overcloud$ cat /etc/os-net-config/config.json | python -m json.tool
{
    "network_config": [
        {
            "members": [
                {
                    "name": "nic1",
                    "primary": true,
                    "type": "interface"
                }
            ],
            "name": "br-ex",
            "type": "ovs_bridge",
            "use_dhcp": true
        }
    ]
}
~~~

~~~
overcloud$ sudo openstack-config --get /etc/neutron/l3_agent.ini DEFAULT external_network_bridge
br-ex

overcloud$ neutron net-create management --router:external
Created a new network:
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | True                                 |
| id                        | c6c8fa6e-92c6-45b0-83bb-2bb2fe8daf37 |
| mtu                       | 0                                    |
| name                      | management                           |
| provider:network_type     | vxlan                                |
| provider:physical_network |                                      |
| provider:segmentation_id  | 1                                    |
| router:external           | True                                 |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tenant_id                 | 45dc9f87cf3e40a29f272fbe53420065     |
+---------------------------+--------------------------------------+
~~~

~~~
overcloud$ neutron subnet-create management 172.16.0.0/24 \
    --name management_subnet --enable-dhcp=False --allocation-pool \
    start=172.16.0.210,end=172.16.0.230 --dns-nameserver 192.168.122.1
Created a new subnet:
+-------------------+--------------------------------------------------+
| Field             | Value                                            |
+-------------------+--------------------------------------------------+
| allocation_pools  | {"start": "172.16.0.210", "end": "172.16.0.230"} |
| cidr              | 172.16.0.0/24                                    |
| dns_nameservers   | 192.168.122.1                                    |
| enable_dhcp       | False                                            |
| gateway_ip        | 172.16.0.1                                       |
| host_routes       |                                                  |
| id                | 2f38b59f-7ba8-4176-acb2-116c61a8edb1             |
| ip_version        | 4                                                |
| ipv6_address_mode |                                                  |
| ipv6_ra_mode      |                                                  |
| name              | management_subnet                                |
| network_id        | c6c8fa6e-92c6-45b0-83bb-2bb2fe8daf37             |
| subnetpool_id     |                                                  |
| tenant_id         | 45dc9f87cf3e40a29f272fbe53420065                 |
+-------------------+--------------------------------------------------+
~~~

~~~
overcloud$ neutron net-create internal
Created a new network:
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | True                                 |
| id                        | 151adf71-525c-425b-8e97-8dd79419cd45 |
| mtu                       | 0                                    |
| name                      | internal                             |
| provider:network_type     | vxlan                                |
| provider:physical_network |                                      |
| provider:segmentation_id  | 2                                    |
| router:external           | False                                |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tenant_id                 | 45dc9f87cf3e40a29f272fbe53420065     |
+---------------------------+--------------------------------------+

overcloud$ neutron subnet-create internal 192.168.0.0/24 --name internal_subnet
Created a new subnet:
+-------------------+--------------------------------------------------+
| Field             | Value                                            |
+-------------------+--------------------------------------------------+
| allocation_pools  | {"start": "192.168.0.2", "end": "192.168.0.254"} |
| cidr              | 192.168.0.0/24                                   |
| dns_nameservers   |                                                  |
| enable_dhcp       | True                                             |
| gateway_ip        | 192.168.0.1                                      |
| host_routes       |                                                  |
| id                | 472d4333-5bc9-46c1-98e0-8f1dd9c1f020             |
| ip_version        | 4                                                |
| ipv6_address_mode |                                                  |
| ipv6_ra_mode      |                                                  |
| name              | internal_subnet                                  |
| network_id        | 151adf71-525c-425b-8e97-8dd79419cd45             |
| subnetpool_id     |                                                  |
| tenant_id         | 45dc9f87cf3e40a29f272fbe53420065                 |
+-------------------+--------------------------------------------------+
~~~

~~~
overcloud$ neutron router-create internal_router
Created a new router:
+-----------------------+--------------------------------------+
| Field                 | Value                                |
+-----------------------+--------------------------------------+
| admin_state_up        | True                                 |
| distributed           | False                                |
| external_gateway_info |                                      |
| ha                    | True                                 |
| id                    | bfba9274-27aa-45ad-9608-2caf667f1d05 |
| name                  | internal_router                      |
| routes                |                                      |
| status                | ACTIVE                               |
| tenant_id             | 45dc9f87cf3e40a29f272fbe53420065     |
+-----------------------+--------------------------------------+

overcloud$ neutron router-gateway-set internal_router management
Set gateway for router internal_router

overcloud$ neutron router-interface-add internal_router internal_subnet
Added interface c9cdaf7e-7308-404e-a9a6-55798fab7f0a to router internal_router.
~~~

~~~
overcloud$ curl -o /tmp/cirros.qcow2 \
    http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img
(...)

overcloud$ glance image-create --name cirros --disk-format qcow2 \
    --container-format bare --is-public true --file /tmp/cirros.qcow2 --progress
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | ee1eca47dc88f4879d8a229cc70a07c6     |
| container_format | bare                                 |
| created_at       | 2015-08-08T18:48:23.000000           |
| deleted          | False                                |
| deleted_at       | None                                 |
| disk_format      | qcow2                                |
| id               | 25f93f94-dcfd-4d03-b0e3-d717e5c7e5df |
| is_public        | True                                 |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | cirros                               |
| owner            | 45dc9f87cf3e40a29f272fbe53420065     |
| protected        | False                                |
| size             | 13287936                             |
| status           | active                               |
| updated_at       | 2015-08-08T18:48:26.000000           |
| virtual_size     | None                                 |
+------------------+--------------------------------------+
~~~

~~~
overcloud$ internal_net=$(neutron net-list | awk ' /internal/ {print $2;}')
overcloud$ nova boot --flavor m1.tiny --nic net-id=$internal_net \
    --image cirros overcloud-test
+--------------------------------------+-----------------------------------------------+
| Property                             | Value                                         |
+--------------------------------------+-----------------------------------------------+
| OS-DCF:diskConfig                    | MANUAL                                        |
| OS-EXT-AZ:availability_zone          |                                               |
| OS-EXT-SRV-ATTR:host                 | -                                             |
| OS-EXT-SRV-ATTR:hypervisor_hostname  | -                                             |
| OS-EXT-SRV-ATTR:instance_name        | instance-00000003                             |
| OS-EXT-STS:power_state               | 0                                             |
| OS-EXT-STS:task_state                | scheduling                                    |
| OS-EXT-STS:vm_state                  | building                                      |
| OS-SRV-USG:launched_at               | -                                             |
| OS-SRV-USG:terminated_at             | -                                             |
| accessIPv4                           |                                               |
| accessIPv6                           |                                               |
| adminPass                            | b8FtDcEHr4DL                                  |
| config_drive                         |                                               |
| created                              | 2015-08-08T19:40:33Z                          |
| flavor                               | m1.tiny (1)                                   |
| hostId                               |                                               |
| id                                   | d100aae8-aa95-4d30-8edc-56c8edf6c274          |
| image                                | cirros (25f93f94-dcfd-4d03-b0e3-d717e5c7e5df) |
| key_name                             | -                                             |
| metadata                             | {}                                            |
| name                                 | overcloud-test                                |
| os-extended-volumes:volumes_attached | []                                            |
| progress                             | 0                                             |
| security_groups                      | default                                       |
| status                               | BUILD                                         |
| tenant_id                            | 45dc9f87cf3e40a29f272fbe53420065              |
| updated                              | 2015-08-08T19:40:33Z                          |
| user_id                              | ffec88bbfd004bc6924a60ea7e3e66a6              |
+--------------------------------------+-----------------------------------------------+
~~~

~~~
overcloud$ nova list
+--------------------------------------+----------------+--------+------------+-------------+----------------------+
| ID                                   | Name           | Status | Task State | Power State | Networks             |
+--------------------------------------+----------------+--------+------------+-------------+----------------------+
| d100aae8-aa95-4d30-8edc-56c8edf6c274 | overcloud-test | ACTIVE | -          | Running     | internal=192.168.0.5 |
+--------------------------------------+----------------+--------+------------+-------------+----------------------+

overcloud$ neutron floatingip-create management
Created a new floatingip:
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| fixed_ip_address    |                                      |
| floating_ip_address | 172.16.0.210                         |
| floating_network_id | c6c8fa6e-92c6-45b0-83bb-2bb2fe8daf37 |
| id                  | 5173d288-6bfd-4848-be39-4274268e6fc6 |
| port_id             |                                      |
| router_id           |                                      |
| status              | DOWN                                 |
| tenant_id           | 45dc9f87cf3e40a29f272fbe53420065     |
+---------------------+--------------------------------------+

overcloud$ nova add-floating-ip overcloud-test 172.16.0.210
~~~

~~~
overcloud$ nova secgroup-add-rule default tcp 22 22 0.0.0.0/0
+-------------+-----------+---------+-----------+--------------+
| IP Protocol | From Port | To Port | IP Range  | Source Group |
+-------------+-----------+---------+-----------+--------------+
| tcp         | 22        | 22      | 0.0.0.0/0 |              |
+-------------+-----------+---------+-----------+--------------+

overcloud$ nova secgroup-add-rule default icmp -1 -1 0.0.0.0/0
+-------------+-----------+---------+-----------+--------------+
| IP Protocol | From Port | To Port | IP Range  | Source Group |
+-------------+-----------+---------+-----------+--------------+
| icmp        | -1        | -1      | 0.0.0.0/0 |              |
+-------------+-----------+---------+-----------+--------------+
~~~

~~~
overcloud$ ping -c4 172.16.0.210
PING 172.16.0.210 (172.16.0.210) 56(84) bytes of data.
64 bytes from 172.16.0.210: icmp_seq=1 ttl=63 time=2.79 ms
64 bytes from 172.16.0.210: icmp_seq=2 ttl=63 time=1.61 ms
64 bytes from 172.16.0.210: icmp_seq=3 ttl=63 time=1.72 ms
64 bytes from 172.16.0.210: icmp_seq=4 ttl=63 time=1.51 ms

--- 172.16.0.210 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 1.518/1.912/2.797/0.516 ms

overcloud$ ssh cirros@172.16.0.210
The authenticity of host '172.16.0.210 (172.16.0.210)' can't be established.
RSA key fingerprint is 9e:26:e1:42:17:40:21:af:ec:be:a5:0a:43:d3:f1:bf.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '172.16.0.210' (RSA) to the list of known hosts.
cirros@172.16.0.210's password:
$ uname -a
Linux cirros 3.2.0-80-virtual #116-Ubuntu SMP Mon Mar 23 17:28:52 UTC 2015 x86_64 GNU/Linux
$ exit
Connection to 172.16.0.210 closed.
~~~

~~~
overcloud$ nova delete overcloud-test
Request to delete server overcloud-test has been accepted.

overcloud$ exit
Connection to 172.16.0.93 closed.
undercloud$
~~~

至此您的overcloud是可用工作的

## 验证部署

~~~
undercloud$ source ~/overcloudrc
undercloud$ openstack overcloud validate \
    --overcloud-auth-url $OS_AUTH_URL \
    --overcloud-admin-password $OS_PASSWORD \
    --tempest-args '.*smoke'
(...)
~~~

## 下一实验

下一实验将是 **推倒** 我们现有的环境进行 **重建** 一个 **大一点的** **多种配置的** **高级** 部署, 点击 [这里](./lab08.md) 继续。
