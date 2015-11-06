# 实验六: 部署 Overcloud

## 序言

到这一步，您已经完成了部署环境的准备。下面我们将进行 **生产云** 的部署，**生产云** 也就是 **OverCloud**
本实验预计耗时 **120** 分钟：
* 部署一个五节点带高可用的 **基础** OverCloud
* 了解部署的后台进程

## 实验对象

**undercloud** VM 和 **overcloud** VMs

<center>
<img src="images/osp-director-env-4.png">
</center>

## 部署前的准备

~~~
undercloud$ sudo su -
undercloud# openstack-config --set /etc/nova/nova.conf DEFAULT rpc_response_timeout 600
undercloud# openstack-config --set /etc/ironic/ironic.conf DEFAULT rpc_response_timeout 600
undercloud# openstack-service restart nova
undercloud# openstack-service restart ironic
undercloud# exit
undercloud$
~~~

指定DNS

~~~
undercloud$ source ~/stackrc
undercloud$ neutron subnet-list
undercloud$ subnet_id=$(neutron subnet-list | awk '/172.16.0.0/ {print $2;}')
undercloud$ neutron subnet-update $subnet_id --dns-nameserver 192.168.122.1
Updated subnet: 4e457dba-2a86-4f24-bc14-09466a880ca6
~~~

## 部署Overcloud

创建对应的flavor

~~~
undercloud$ openstack flavor create --id auto --ram 8192 --disk 58 --vcpus 4 baremetal
+----------------------------+--------------------------------------+
| Field                      | Value                                |
+----------------------------+--------------------------------------+
| OS-FLV-DISABLED:disabled   | False                                |
| OS-FLV-EXT-DATA:ephemeral  | 0                                    |
| disk                       | 58                                   |
| id                         | 88c20cf4-9f42-41e8-b133-3d42697d6238 |
| name                       | baremetal                            |
| os-flavor-access:is_public | True                                 |
| ram                        | 8192                                 |
| rxtx_factor                | 1.0                                  |
| swap                       |                                      |
| vcpus                      | 4                                    |
+----------------------------+--------------------------------------+
~~~

~~~
undercloud$ openstack flavor set --property "cpu_arch"="x86_64" --property "capabilities:boot_option"="local" baremetal
+----------------------------+-----------------------------------------------------+
| Field                      | Value                                               |
+----------------------------+-----------------------------------------------------+
| OS-FLV-DISABLED:disabled   | False                                               |
| OS-FLV-EXT-DATA:ephemeral  | 0                                                   |
| disk                       | 58                                                  |
| id                         | 88c20cf4-9f42-41e8-b133-3d42697d6238                |
| name                       | baremetal                                           |
| os-flavor-access:is_public | True                                                |
| properties                 | capabilities:boot_option='local', cpu_arch='x86_64' |
| ram                        | 8192                                                |
| rxtx_factor                | 1.0                                                 |
| swap                       |                                                     |
| vcpus                      | 4                                                   |
+----------------------------+-----------------------------------------------------+
~~~

确认我们的目标节点状态是 '**available**'

~~~
undercloud$ ironic node-list
+--------------------------------------+------+---------------+-------------+-----------------+-------------+
| UUID                                 | Name | Instance UUID | Power State | Provision State | Maintenance |
+--------------------------------------+------+---------------+-------------+-----------------+-------------+
| b6ef49e3-6e7a-4298-9838-71542a1d173c | None | None          | power off   | available       | False       |
| 5886e6fc-4bb1-42b3-a9ba-cbd673a71db9 | None | None          | power off   | available       | False       |
| 030e5c56-5158-4d4c-b6ff-50e459a2fde3 | None | None          | power off   | available       | False       |
| 0596a845-59b4-42ba-bf99-7f159aba75b2 | None | None          | power off   | available       | False       |
| de46c4a4-05a0-4c4e-8677-a38916436df2 | None | None          | power off   | available       | False       |
+--------------------------------------+------+---------------+-------------+-----------------+-------------+
~~~

启动部署

~~~
undercloud$ openstack overcloud deploy --templates \
    --ntp-server 10.5.26.10 --control-scale 3 --compute-scale 2 \
    --neutron-tunnel-types vxlan --neutron-network-type vxlan
Deploying templates in the directory /usr/share/openstack-tripleo-heat-templates
(...)
~~~

在另一窗口监控部署

~~~
undercloud$ heat resource-list -n5 overcloud
(...)
~~~

## 部署过程做了什么？

部署主要做了两件事

### 模板处理

告诉OSP导演采用 **/usr/share/openstack-tripleo-heat-templates/**. 去建立我们的stack。

* **/usr/share/openstack-tripleo-heat-templates/overcloud-without-mergepy.yaml**
* **/usr/share/openstack-tripleo-heat-templates/overcloud-resource-registry-puppet.yaml**

~~~
undercloud$ cd /usr/share/openstack-tripleo-heat-templates/
undercloud$ grep -A20 parameters overcloud-without-mergepy.yaml | head -n20
parameters:

  # Common parameters (not specific to a role)
  AdminPassword:
    default: unset
    description: The password for the keystone admin account, used for monitoring, querying neutron etc.
    type: string
    hidden: true
  CeilometerBackend:
    default: 'mongodb'
    description: The ceilometer backend type.
    type: string
  CeilometerMeteringSecret:
    default: unset
    description: Secret shared by the ceilometer services.
    type: string
    hidden: true
  CeilometerPassword:
    default: unset
    description: The password for the ceilometer service account.
~~~

~~~
undercloud$ grep -A20 "Controller:" overcloud-without-mergepy.yaml | head -n15
  Controller:
    type: OS::Heat::ResourceGroup
    depends_on: Networks
    properties:
      count: {get_param: ControllerCount}
      removal_policies: {get_param: ControllerRemovalPolicies}
      resource_def:
        type: OS::TripleO::Controller
        properties:
          AdminPassword: {get_param: AdminPassword}
          AdminToken: {get_param: AdminToken}
          CeilometerBackend: {get_param: CeilometerBackend}
          CeilometerMeteringSecret: {get_param: CeilometerMeteringSecret}
          CeilometerPassword: {get_param: CeilometerPassword}
          CinderLVMLoopDeviceSize: {get_param: CinderLVMLoopDeviceSize}
~~~

~~~
undercloud$ grep "OS::TripleO::Controller: " overcloud-resource-registry-puppet.yaml
  OS::TripleO::Controller: puppet/controller-puppet.yaml
~~~

~~~

undercloud$ grep -A10 "OS::Nova::Server" puppet/controller-puppet.yaml
    type: OS::Nova::Server
    properties:
      image: {get_param: Image}
      image_update_policy: {get_param: ImageUpdatePolicy}
      flavor: {get_param: Flavor}
      key_name: {get_param: KeyName}
      networks:
        - network: ctlplane
      user_data_format: SOFTWARE_CONFIG
      user_data: {get_resource: NodeUserData}
      name: {get_param: Hostname}

~~~


~~~
undercloud$ grep -A9 NetworkConfig puppet/controller-puppet.yaml | head -n9
  NetworkConfig:
    type: OS::TripleO::Controller::Net::SoftwareConfig
    properties:
      ExternalIpSubnet: {get_attr: [ExternalPort, ip_subnet]}
      InternalApiIpSubnet: {get_attr: [InternalApiPort, ip_subnet]}
      StorageIpSubnet: {get_attr: [StoragePort, ip_subnet]}
      StorageMgmtIpSubnet: {get_attr: [StorageMgmtPort, ip_subnet]}
      TenantIpSubnet: {get_attr: [TenantPort, ip_subnet]}
~~~

~~~
undercloud$ grep -A9 NetworkDeployment puppet/controller-puppet.yaml | head -n9
  NetworkDeployment:
    type: OS::TripleO::SoftwareDeployment
    properties:
      config: {get_resource: NetworkConfig}
      server: {get_resource: Controller}
      input_values:
        bridge_name: br-ex
        interface_name: {get_param: NeutronPublicInterface}
~~~

### 部署节点

~~~
undercloud$ source ~/stackrc
undercloud$ neutron net-list
undercloud$ net_id=$(neutron net-list | grep 172.16.0.0 | awk '{print $2;}')
undercloud$ cat /var/lib/neutron/dhcp/$net_id/opts
tag:tag0,option:dns-server,192.168.122.1
tag:tag0,option:classless-static-route,169.254.169.254/32,172.16.0.1,0.0.0.0/0,172.16.0.1
tag:tag0,249,169.254.169.254/32,172.16.0.1,0.0.0.0/0,172.16.0.1
tag:tag0,option:router,172.16.0.1
tag:0ac4912b-23fd-4fb5-8236-7079edb776f8,option:tftp-server,172.16.0.1
tag:0ac4912b-23fd-4fb5-8236-7079edb776f8,tag:ipxe,option:bootfile-name,http://172.16.0.1:8088/boot.ipxe
tag:0ac4912b-23fd-4fb5-8236-7079edb776f8,tag:!ipxe,option:bootfile-name,undionly.kpxe
tag:0ac4912b-23fd-4fb5-8236-7079edb776f8,option:server-ip-address,172.16.0.1
tag:e6b1c89e-e231-4475-9564-225c9ccf3201,tag:!ipxe,option:bootfile-name,undionly.kpxe
tag:e6b1c89e-e231-4475-9564-225c9ccf3201,option:server-ip-address,172.16.0.1
tag:e6b1c89e-e231-4475-9564-225c9ccf3201,option:tftp-server,172.16.0.1
tag:e6b1c89e-e231-4475-9564-225c9ccf3201,tag:ipxe,option:bootfile-name,http://172.16.0.1:8088/boot.ipxe
tag:ac18d149-3bab-4302-ad18-f31614cb28c0,option:server-ip-address,172.16.0.1
tag:ac18d149-3bab-4302-ad18-f31614cb28c0,tag:!ipxe,option:bootfile-name,undionly.kpxe
tag:ac18d149-3bab-4302-ad18-f31614cb28c0,option:tftp-server,172.16.0.1
tag:ac18d149-3bab-4302-ad18-f31614cb28c0,tag:ipxe,option:bootfile-name,http://172.16.0.1:8088/boot.ipxe
tag:684c031e-1428-4a58-afbd-884ca69ef5b7,option:server-ip-address,172.16.0.1
tag:684c031e-1428-4a58-afbd-884ca69ef5b7,tag:!ipxe,option:bootfile-name,undionly.kpxe
tag:684c031e-1428-4a58-afbd-884ca69ef5b7,tag:ipxe,option:bootfile-name,http://172.16.0.1:8088/boot.ipxe
tag:684c031e-1428-4a58-afbd-884ca69ef5b7,option:tftp-server,172.16.0.1
tag:4c792d00-b38a-4bdf-94da-3704626f0822,option:server-ip-address,172.16.0.1
tag:4c792d00-b38a-4bdf-94da-3704626f0822,tag:!ipxe,option:bootfile-name,undionly.kpxe
tag:4c792d00-b38a-4bdf-94da-3704626f0822,tag:ipxe,option:bootfile-name,http://172.16.0.1:8088/boot.ipxe
tag:4c792d00-b38a-4bdf-94da-3704626f0822,option:tftp-server,172.16.0.1
~~~

~~~
undercloud$ neutron port-show 4c792d00-b38a-4bdf-94da-3704626f0822
neutron port-show 4c792d00-b38a-4bdf-94da-3704626f0822
+-----------------------+----------------------------------------------------------------------------------------------------------+
| Field                 | Value                                                                                                    |
+-----------------------+----------------------------------------------------------------------------------------------------------+
| admin_state_up        | True                                                                                                     |
| allowed_address_pairs |                                                                                                          |
| binding:host_id       | undercloud.redhat.local                                                                                  |
| binding:profile       | {}                                                                                                       |
| binding:vif_details   | {"port_filter": true, "ovs_hybrid_plug": true}                                                           |
| binding:vif_type      | ovs                                                                                                      |
| binding:vnic_type     | normal                                                                                                   |
| device_id             | be020e9c-a7ad-4e0d-94b5-91aa663d8170                                                                     |
| device_owner          | compute:None                                                                                             |
| extra_dhcp_opts       | {"opt_value": "172.16.0.1", "ip_version": 4, "opt_name": "server-ip-address"}                            |
|                       | {"opt_value": "undionly.kpxe", "ip_version": 4, "opt_name": "tag:!ipxe,bootfile-name"}                   |
|                       | {"opt_value": "http://172.16.0.1:8088/boot.ipxe", "ip_version": 4, "opt_name": "tag:ipxe,bootfile-name"} |
|                       | {"opt_value": "172.16.0.1", "ip_version": 4, "opt_name": "tftp-server"}                                  |
| fixed_ips             | {"subnet_id": "4e457dba-2a86-4f24-bc14-09466a880ca6", "ip_address": "172.16.0.167"}                      |
| id                    | 4c792d00-b38a-4bdf-94da-3704626f0822                                                                     |
| mac_address           | 52:54:00:a3:d8:c8                                                                                        |
| name                  |                                                                                                          |
| network_id            | 0b452b19-ebbb-40d2-88b8-f8924e63eac1                                                                     |
| security_groups       | 369c4cf2-3889-4eeb-9199-b3b937d86d38                                                                     |
| status                | DOWN                                                                                                     |
| tenant_id             | 74ddd233842043269955550d890953ab                                                                         |
+-----------------------+----------------------------------------------------------------------------------------------------------+
~~~

~~~
undercloud$ cat /httpboot/boot.ipxe
#!ipxe

# load the MAC-specific file or fail if it's not found
chain --autofree pxelinux.cfg/${net0/mac:hexhyp} || goto error_no_config

:error_no_config
echo PXE boot failed. No configuration found for MAC ${net0/mac}
echo Press any key to reboot...
prompt --timeout 180
reboot
~~~

~~~
undercloud$ cat /httpboot/pxelinux.cfg/525400a3d8c8
#!ipxe

dhcp

goto deploy

:deploy
kernel http://172.16.0.1:8088/0596a845-59b4-42ba-bf99-7f159aba75b2/deploy_kernel selinux=0 disk=cciss/c0d0,sda,hda,vda iscsi_target_iqn=iqn-0596a845-59b4-42ba-bf99-7f159aba75b2 deployment_id=0596a845-59b4-42ba-bf99-7f159aba75b2 deployment_key=6SFB1ZQGDFGZ9BDS6KU54VRD3KRVMMYI ironic_api_url=http://172.16.0.1:6385 troubleshoot=0 text nofb nomodeset vga=normal boot_option=local ip=${ip}:${next-server}:${gateway}:${netmask} BOOTIF=${net0/mac}  ipa-api-url=http://172.16.0.1:6385 ipa-driver-name=pxe_ssh coreos.configdrive=0

initrd http://172.16.0.1:8088/0596a845-59b4-42ba-bf99-7f159aba75b2/deploy_ramdisk
boot

:boot_partition
kernel http://172.16.0.1:8088/0596a845-59b4-42ba-bf99-7f159aba75b2/kernel root={{ ROOT }} ro text nofb nomodeset vga=normal
initrd http://172.16.0.1:8088/0596a845-59b4-42ba-bf99-7f159aba75b2/ramdisk
boot

:boot_whole_disk
kernel chain.c32
append mbr:{{ DISK_IDENTIFIER }}

~~~

~~~
undercloud$ sudo grep -m 1 "Still waiting for ironic node" /var/log/nova/nova-compute.log
2015-08-07 10:25:44.350 341 DEBUG nova.virt.ironic.driver [-] [instance: 6df9cb36-8555-427c-9c39-aecf2e142a26] Still waiting for ironic node 0596a845-59b4-42ba-bf99-7f159aba75b2 to become ACTIVE: power_state="power on", target_power_state="power on", provision_state="deploying", target_provision_state="active" _log_ironic_polling /usr/lib/python2.7/site-packages/nova/virt/ironic/driver.py:159
~~~

### 对节点进行配置

* **os-collect-config**

**/var/lib/heat-cfntools/cfn-init-data**
**/etc/os-collect-config.conf**

~~~
undercloud$ nova list
+--------------------------------------+------------------------+--------+------------+-------------+----------------------+
| ID                                   | Name                   | Status | Task State | Power State | Networks             |
+--------------------------------------+------------------------+--------+------------+-------------+----------------------+
| 506c41bb-9e4d-4edc-9d96-af44f1c3dd5a | overcloud-compute-0    | ACTIVE | -          | Running     | ctlplane=172.16.0.24 |
| e1a42221-eb36-4890-ae96-d5e0a6820708 | overcloud-compute-1    | ACTIVE | -          | Running     | ctlplane=172.16.0.25 |
| aa0db61d-f033-4c5c-9328-9e23fedaca7e | overcloud-controller-0 | ACTIVE | -          | Running     | ctlplane=172.16.0.27 |
| d23422f8-c510-4df5-8bae-632f53e01ddb | overcloud-controller-1 | ACTIVE | -          | Running     | ctlplane=172.16.0.26 |
| 64c2c1aa-461a-4bed-b443-f9e69deaf60b | overcloud-controller-2 | ACTIVE | -          | Running     | ctlplane=172.16.0.28 |
+--------------------------------------+------------------------+--------+------------+-------------+----------------------+
~~~

~~~
undercloud$ ssh heat-admin@172.16.0.27
The authenticity of host '172.16.0.27 (172.16.0.27)' can't be established.
ECDSA key fingerprint is d9:20:db:eb:68:b6:16:1f:5c:f3:8b:5e:9c:da:11:2e.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '172.16.0.27' (ECDSA) to the list of known hosts.
Last login: Sun Aug 16 01:12:24 2015 from 172.16.0.1
overcloud$
~~~

~~~
overcloud$ sudo cat /etc/os-collect-config.conf
    [DEFAULT]
    command = os-refresh-config

    [cfn]
    metadata_url = http://172.16.0.1:8000/v1/
    stack_name = overcloud-Controller-7wz2m5lz25zb-0-lephavupw6vp
    secret_access_key = 98a1c79fb11543f1a3c11c71d2de0429
    access_key_id = cf85e7595d9a4168b8755af1e24818f0
    path = Controller.Metadata
~~~

**/var/lib/os-collect-config/.**

* **os-refresh-config**

**/usr/libexec/os-refresh-config/**
**pre-configure.d/** ==> **configure.d/** ==> **post-configure.d/**

os-refresh-config 主要进行了以下事务-

    1. Apply **systemctl** configurables
    2. Run **os-apply-config** (see below)
    3. Configure the **networking** for the host
    4. Download and set the **hieradata** files for puppet parameters
    5. Configure **/etc/hosts**
    6. Deploy **software configuration** with Heat

~~~
    overcloud$ ll /usr/libexec/os-refresh-config/configure.d/
    total 32
    -rwxr-xr-x. 1 root root  396 Aug  5 07:31 10-sysctl-apply-config
    -rwxr-xr-x. 1 root root   42 Aug  5 07:31 20-os-apply-config
    -rwxr-xr-x. 1 root root  189 Aug  5 07:31 20-os-net-config
    -rwxr-xr-x. 1 root root  629 Aug  5 07:31 25-set-network-gateway
    -rwxr-xr-x. 1 root root 2265 Aug  5 07:31 40-hiera-datafiles
    -rwxr-xr-x. 1 root root 1387 Aug  5 07:31 51-hosts
    -rwxr-xr-x. 1 root root 5784 Aug  5 07:31 55-heat-config
~~~

* **os-apply-config**

~~~
overcloud$ sudo sh /usr/libexec/os-refresh-config/configure.d/20-os-apply-config
    [2015/08/07 01:17:40 PM] [INFO] writing /etc/os-net-config/config.json
    [2015/08/07 01:17:40 PM] [INFO] writing /var/run/heat-config/heat-config
    [2015/08/07 01:17:40 PM] [INFO] writing /etc/puppet/hiera.yaml
    [2015/08/07 01:17:40 PM] [INFO] writing /etc/os-collect-config.conf
    [2015/08/07 01:17:40 PM] [INFO] success
~~~

* **heat-config-notify**

'**/usr/libexec/os-refresh-config/configure.d/55-heat-config**'
'**/var/lib/heat-config/hooks**'
'**/var/lib/heat-config/heat-config-puppet**'

~~~
overcloud$ sudo grep -v '^#' /var/lib/heat-config/heat-config-puppet/1f86e251-7e5e-4ee6-95c6-76dd691ef279.pp | grep neutron | head -n20
    $neutron_dsn = split(hiera('neutron::server::database_connection'), '[@:/?]')
    class { 'neutron::db::mysql':
      user          => $neutron_dsn[3],
      password      => $neutron_dsn[4],
      host          => $neutron_dsn[5],
      dbname        => $neutron_dsn[6],
  include ::nova::network::neutron
  include ::neutron
  class { '::neutron::server' :
  class { '::neutron::agents::dhcp' :
  class { '::neutron::agents::l3' :
  class { 'neutron::agents::metadata':
  file { '/etc/neutron/dnsmasq-neutron.conf':
    content => hiera('neutron_dnsmasq_options'),
    owner   => 'neutron',
    group   => 'neutron',
    notify  => Service['neutron-dhcp-service'],
    require => Package['neutron'],
  class { 'neutron::plugins::ml2':
    flat_networks   => split(hiera('neutron_flat_networks'), ','),
~~~

### 成功完成

~~~
(...)
Overcloud Endpoint: http://172.16.0.172:5000/v2.0/
Overcloud Deployed
~~~

~~~
undercloud$ cat ~/overcloudrc
export OS_NO_CACHE=True
export COMPUTE_API_VERSION=1.1
export OS_USERNAME=admin
export no_proxy=,172.16.0.172
export OS_TENANT_NAME=admin
export OS_CLOUDNAME=overcloud
export OS_AUTH_URL=http://172.16.0.172:5000/v2.0/
export NOVA_VERSION=1.1
export OS_PASSWORD=6aedc059f1e5e7c83b7bffffb863e9e3f4593db2
~~~

~~~
undercloud$ cat tripleo-overcloud-passwords
OVERCLOUD_SWIFT_HASH=60879e305baf6368e99b10ed853ffaae247c62f1
OVERCLOUD_HEAT_STACK_DOMAIN_PASSWORD=abf86f88ec63072bdbfa00f2affb1fc8ad75a82e
OVERCLOUD_CEILOMETER_PASSWORD=55a48f6e108aa0c6b83d3c15b74064a0f65d944a
OVERCLOUD_ADMIN_TOKEN=d6ab3e132d45e0e95ba601adccd7756afeed78a1
OVERCLOUD_CEILOMETER_SECRET=05ef79286f3a99ee3fc5ee0113b25f7c523ca758
OVERCLOUD_GLANCE_PASSWORD=4882dd1f337b2baec39eeaae6b69336e50c62a75
OVERCLOUD_CINDER_PASSWORD=4432dcf808765cc0b448148fdde18425357978fa
OVERCLOUD_NOVA_PASSWORD=cab9562cb5a0af2687015851352177e0d7505e59
OVERCLOUD_ADMIN_PASSWORD=6aedc059f1e5e7c83b7bffffb863e9e3f4593db2
OVERCLOUD_NEUTRON_PASSWORD=6e7870d5f37218b1cc7628d08abbe27574613bd6
OVERCLOUD_SWIFT_PASSWORD=6e5fada86d952d3b304441098649a7bcd82bf9f8
OVERCLOUD_DEMO_PASSWORD=4cd61de53913c01421d7910680902f948fe491c2
OVERCLOUD_HEAT_PASSWORD=fb8c1ba89a3f1cc95f947e6f9322cfab1427bffb
~~~

~~~
undercloud$ cat instackenv.json | python -m json.tool | grep -A3 overcloud
    "overcloud": {
        "endpoint": "http://172.16.0.172:5000/v2.0/",
        "password": "6aedc059f1e5e7c83b7bffffb863e9e3f4593db2"
    },
~~~

恭喜您完成了基本生产云的部署。

## 下一实验

下一实验将是测试我们新部署的overcloud。点击 [这里](./lab07.md) 进入。
