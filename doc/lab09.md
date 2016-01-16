# Lab 9: Advanced OpenStack Deployment

## Introduction

It's likely that the environment we **previously** deployed would be useful for a **basic PoC**, but likely nothing further. The purpose of that deployment was primarily to get you **comfortable** with how OSP director **deploys** and **configures** RHEL OSP onto a given set of nodes. It was also the intention of that lab to give you a general understanding of how OSP director **seeds** data for execution into each node, e.g. setting up the **network configuration**. However, what it didn't do was explain how we can perform a more **customised** deployment, one that would more likely **match** a customers requirements and expectations. The purpose of this **ninth** lab is to do just that.



Whilst OSP director has its **flaws** today, it has been built in a way that allows us to create a deployment tool with ultimate **flexibility**. OSP director ships with **templates** that allow us to deploy a RHEL OSP environment with a **default** configuration, but via the notion of **overriding** we can apply our own **modifications**, tailoring it to suit the deployment requirements. Every customer deployment is **different**, each customer has their own **expectations** of feature functionality, networking configuration, storage utilisation, etc, and we have to have a tool worthy of being used to **satisfy** such expectations.



This **ninth** lab will walk you through the deployment of an advanced overcloud; we're going to be carrying out the following tasks in this lab, with an estimated completion time of **180 minutes**:



* **Introspection** and **benchmarking** of our overcloud infrastructure

* **Advanced role matching** to automatically assign flavors to nodes based on their **specification**

* Creation of dedicated network configuration **templates**, to allow for **bonding** and **network isolation**

* Deployment and integration of **Ceph** storage for our overcloud infrastructure

* Exploration of the **CLI-based overrides** that can be performed

* Making **configuration-specific changes** to the overcloud to match **example** requirements



> **NOTE**: Starting of this lab assumes that you've completed the [eighth lab][lab8] and have created a **new** set of overcloud infrastructure nodes, all of which now **registered** in the undercloud's **Ironic** service. If this is not the case, please revisit the previous lab.



## Lab Focus



This lab will focus on the underlying **host** (hypervisor) and **undercloud** VM:



<center>

<img src="images/osp-director-env-3.png">

</center>



## Introspection and Benchmarking



In [Lab 5][lab5] we carried out an initial **introspection**, and demonstrated how OSP directors' **discovery** process permits us to gain valuable insight into the system **specification** of nodes that are available to us. This data is typically used to perform **role matching**, ensuring that the **correct** nodes get allocated the most **appropriate** roles. This only goes part of the way; OSP directors' discovery process permits us to perform **benchmarking** of our overcloud nodes before deployment, ensuring that all systems are **performing** as expected, and that we do not have any **misconfigured** nodes. Both the system **specification**, and **benchmarking** data, can be fed into the **advanced role matching** process to ensure that only nodes truly **capable** get assigned the roles for our deployment. The benchmarking process can gather **CPU**, **memory**, and **disk** performance metrics.



Firstly, install the package '**ahc-tools**', a set of tools that allows us to perform benchmarking data **analysis** and will help us with our **advanced role matching** later:



~~~

undercloud$ sudo yum install ahc-tools -y

(...)

~~~



We need to allow the **ahc-tools** to **interrogate** the introspection and benchmarking **data**, recall this is held in **Swift** on the undercloud by **ironic-discoverd**. The ahc-tools package uses an **almost identical** configuration file syntax, so we can **copy** the configuration file between these two tools with a minor modification:



~~~

undercloud$ sudo su -

undercloud# sed 's/\[discoverd/\[ironic/' /etc/ironic-discoverd/discoverd.conf \

> /etc/ahc-tools/ahc-tools.conf



undercloud# cat /etc/ahc-tools/ahc-tools.conf

[ironic]

debug = false

os_auth_url = http://172.16.0.1:5000/v2.0

identity_uri = http://172.16.0.1:35357

os_username = ironic

os_password = b2593fa523b6a388a78da8b9eb30508858926443

os_tenant_name = service

dnsmasq_interface = br-ctlplane

database = /var/lib/ironic-discoverd/discoverd.sqlite

ramdisk_logs_dir = /var/log/ironic-discoverd/ramdisk/

processing_hooks = ramdisk_error,root_device_hint,scheduler,validate_interfaces,edeploy

enable_setting_ipmi_credentials = true

keep_ports = added

ironic_retry_attempts = 6

ironic_retry_period = 10



[swift]

username = ironic

password = b2593fa523b6a388a78da8b9eb30508858926443

tenant_name = service

os_auth_url = http://172.16.0.1:5000/v2.0



undercloud# exit

undercloud$

~~~



Initially we didn't configure OSP director to run benchmarks on our overcloud nodes during introspection, we'll need to enable it. The method for doing this is simply to update the **undercloud.conf** file, and re-run the installation. Ensure that you're connected to your undercloud machine as the **stack** user:



~~~

undercloud$ unset DIB_YUM_REPO_CONF

undercloud$ unset DIB_LOCAL_IMAGE

undercloud$ sudo systemctl stop bootif-fix

undercloud$ source ~/stackrc

undercloud$ openstack-config --set ~/undercloud.conf DEFAULT discovery_runbench true

undercloud$ openstack undercloud install

(...)

~~~



> **NOTE**: During this process, an automated **'yum update -y'** is performed, so don't be **alarmed** if new packages are downloaded and installed. The deployment runs through the **entire** installation script again, and takes some time; unfortunately this is currently a necessary evil, but is **overkill** just to make a simple configuration change.



Now that our undercloud is reconfigured to support **benchmarking** of our overcloud nodes, let's kick off the **introspection** process. As **benchmarking** has been enabled, this takes a little while longer to complete as opposed to the simple **discovery** process, a typical benchmark of 8 virtual nodes on a single hypervisor can take up to **45 minutes**.



As before, configure your freshly registered nodes to boot and make sure that the **bootif-fix** script is running:



~~~

undercloud$ sudo systemctl start bootif-fix

undercloud$ source ~/stackrc

undercloud$ openstack baremetal configure boot

~~~



As we're benchmarking with virtual hardware, it has been known to timeout (the default timeout is **60 minutes**) and cause the benchmarking/introspection process to fail. Therefore, instead of doing a **bulk** introspection, we're going to do it manually in sets of two or three nodes. The **bulk** command takes care of a few things for us as follows-



1. Sets each nodes **provision** state to **manage** to ensure it's not used

2. **Powers** on each node via Ironic

3. Upon **completion**, returns the node back to the **available** state



Unfortunately, when running through the introspection and benchmarking process manually, the steps above need to also be carried out **manually**. So first, choose the first three nodes from your node list, and take their id's:



~~~

undercloud$ ironic node-list

+--------------------------------------+------+---------------+-------------+-----------------+-------------+

| UUID                                 | Name | Instance UUID | Power State | Provision State | Maintenance |

+--------------------------------------+------+---------------+-------------+-----------------+-------------+

| d8719680-5029-4302-90dc-e273ebf0ecb3 | None | None          | power off   | available       | False       |

| 63e3bba9-f002-430e-a8eb-5cf2b2fb2698 | None | None          | power off   | available       | False       |

| 6bfebafe-8525-47d2-a2e0-2b5bddda6afb | None | None          | power off   | available       | False       |

| dd6fb9a9-b8eb-4991-92a0-87d4159c8beb | None | None          | power off   | available       | False       |

| e794d4db-6331-4509-99f0-4ad576edf256 | None | None          | power off   | available       | False       |

| d6e059c4-59c9-4ad1-9b5d-a01afd552524 | None | None          | power off   | available       | False       |

| af812435-8c40-4f2b-a312-358941430317 | None | None          | power off   | available       | False       |

| 92a03eec-408c-4360-a51f-81d2445ef8d8 | None | None          | power off   | available       | False       |

+--------------------------------------+------+---------------+-------------+-----------------+-------------+



undercloud$ ironic node-list | grep -v UUID \

| awk '{print $2;}' | grep -v -e '^$' | \

awk 'NR==1||NR==2||NR==3'



d8719680-5029-4302-90dc-e273ebf0ecb3

63e3bba9-f002-430e-a8eb-5cf2b2fb2698

6bfebafe-8525-47d2-a2e0-2b5bddda6afb

~~~



We first need to shift these into the '**manage**' provision state so that we can put them into introspection:



~~~

undercloud$ for i in d8719680-5029-4302-90dc-e273ebf0ecb3 \

63e3bba9-f002-430e-a8eb-5cf2b2fb2698 6bfebafe-8525-47d2-a2e0-2b5bddda6afb; \

do ironic node-set-provision-state $i manage; done

~~~



> **NOTE**: Make sure you replace the ID's listed above with the ones that you had in your output.



Next, send these nodes off to introspection:



~~~

undercloud$ openstack baremetal introspection \

start d8719680-5029-4302-90dc-e273ebf0ecb3 

undercloud$ openstack baremetal introspection \

start 63e3bba9-f002-430e-a8eb-5cf2b2fb2698

undercloud$ openstack baremetal introspection \

start 6bfebafe-8525-47d2-a2e0-2b5bddda6afb

~~~



You can check that their power has been turned on automatically:



~~~

undercloud$ ironic node-list | grep "power on" | \

awk '{print $2" ("$8" "$9")";}'

d8719680-5029-4302-90dc-e273ebf0ecb3 (power on)

63e3bba9-f002-430e-a8eb-5cf2b2fb2698 (power on)

6bfebafe-8525-47d2-a2e0-2b5bddda6afb (power on)

~~~



Occasionally, the **benchmarking** process will **fail silently**, this is a current bug that is being worked on. The result of this is that the introspection process **times out** and reports that the node **failed** to be introspected. It's also possible that the nodes will fail to boot via PXE, so we should watch the nodes via **virt-manager** and observe for failures as before.



~~~

client$ ssh root@<your host ip> -Y

host# virt-manager &

~~~



Thankfully, a simple **reboot** of the affected virtual machine will typically **resolve** the problem. To perform the reboot, simply execute the following command for the failed node, for example if the affected node is '*overcloud-controller1*':



~~~

host# virsh destroy overcloud-controller1



Domain overcloud-controller1 destroyed



host# virsh start overcloud-controller1



Domain overcloud-controller1 started

~~~



> **NOTE**: You may have to **repeat** the **reboot** process if this continues.



The status of the benchmarking and introspection can be found by tailing the log files for **ironic-discoverd**, it will advise of a success or failure in real-time:



~~~

undercloud$ sudo journalctl -u openstack-ironic-discoverd -f

(...)

~~~



Or alternatively, requesting a status via the CLI:



~~~

undercloud$ openstack baremetal introspection \

status d8719680-5029-4302-90dc-e273ebf0ecb3

+----------+-------+

| Field    | Value |

+----------+-------+

| error    | None  |

| finished | False |

+----------+-------+

~~~



Eventually, the **introspection** and **benchmarking** process should complete successfully for the three nodes. We should now repeat the process for the next three, and then the final two nodes listed in '*ironic node-list*'.



Recall that the following command can be used to get the node UUID's for specific nodes:



~~~

undercloud$ ironic node-list | grep -v UUID \

| awk '{print $2;}' | grep -v -e '^$' | \

awk 'NR==4||NR==5||NR==6'

~~~



Remember to put the nodes into the '**manage**' state, and send them off to **introspection**.



> **NOTE**: Do not proceed without completing the introspection and benchmarking process for the rest of the nodes unless you want to **skip** the analysis of the benchmarking data. If you **want** to skip benchmarking after starting, you'll need to clean up your environment and disable benchmarking. Firstly, ensure that your nodes are cleaned up:



> ~~~

> undercloud$ for i in $(ironic node-list | grep -v UUID | \

> awk ' { print $2 } '); do ironic node-set-power-state $i off; done



> undercloud$ for i in $(ironic node-list | grep -v UUID | \

awk ' { print $2 } '); do ironic node-set-provision-state $i provide; done

> ~~~

> 

> Then, disable benchmarking:

> 

> ~~~

> undercloud$ openstack-config --set ~/undercloud.conf DEFAULT \

> discovery_runbench false

> 

> undercloud$ sudo systemctl stop bootif-fix

> undercloud$ openstack undercloud install

> (...)

> 

> undercloud$ sudo systemctl start bootif-fix

> ~~~

> 

> Finally, run through the basic introspection to gather system specific information:

> 

> ~~~

> undercloud$ source ~/stackrc

> undercloud$ openstack baremetal configure boot

> undercloud$ openstack baremetal introspection bulk start

> ~~~

> 

> Once introspection has completed, jump straight to the section called "**Advanced Role Matching**"; noting that you may have to reboot nodes if they fail to PXE into the introspection image.



<br>

Once benchmarking has been completed on all nodes; i.e. you've persisted through the benchmarking and not given up, we should return their status to the '**available**' state so that they can be used for provisioning later; note that when using the bulk command this is automated for us:



~~~

undercloud$ for i in $(ironic node-list | grep -v UUID \

| awk ' { print $2 } '); do ironic node-set-provision-state $i provide; done

~~~



Now we can take a look at the benchmarking data that we have; as part of the ahc-tools package we installed earlier is a tool called **ahc-report**. This can be used to generate a report of the benchmarking and introspection data that was collected. To **print** the **entire** data, you can use the following command, although note that it is very **verbose**:



~~~

undercloud$ ahc-report --full

(...)

~~~



If we take a look at a **subset** of the data, we can see that it has correctly **grouped** the systems together based on the **hardware profiles**, for example, the **three** identical systems that have 4 vCPU's (our Ceph nodes):



~~~

undercloud$ ahc-report --categories | grep -A3 "3 identical systems"

3 identical systems :

[u'A35843D4-88CF-4F58-85CC-A6787AC5AD4C', u'9DBC302B-ED70-486B-99CA-1597FF589CCD', u'91E4B16D-6C96-4FC0-B446-A5203158D99C']

[(u'cpu', u'logical', u'number', u'4'),

 (u'cpu', u'physical', u'number', u'4'),

~~~



We can also view areas that our systems are not **consistent** with, e.g. disk write speed. In the **virtual** environment where the systems are all being **benchmarked simultaneously**, it's likely to provide a **lot** of inconsistent results:



~~~

undercloud$ ahc-report --outliers | grep -i inconsistent | head -n5

standalone_read_1M_IOps           : ERROR   : vda          : Group performance : UNSTABLE

standalone_read_1M_IOps           : ERROR   : vdb          : Group performance : UNSTABLE

standalone_randread_4k_KBps       : ERROR   : vda          : Group performance : UNSTABLE

standalone_randread_4k_KBps       : ERROR   : vdb          : Group performance : UNSTABLE

standalone_read_1M_KBps           : ERROR   : vda          : Group performance : UNSTABLE

~~~



On **real** hardware, **inconsistent** results can demonstrate hardware that is not perfoming correctly. The report can output the node ID's of those that are either **overperforming**, or **underperforming** based on the group average. For example:



~~~

undercloud$ ahc-report --outliers | grep -i underperformance | head -n5

simultaneous_read_1M_IOps         : WARNING : vdb          : 8E21AF4B-AFF3-477C-BA1A-FBBEF8B5099F : Curious underperformance 2631.00 : min_allow_group = 2675.10, mean_group = 3159.67 max_allow_group = 3644.23

simultaneous_read_1M_IOps         : WARNING : vdb          : CC091BCB-78F8-4E70-9733-4AF505976356 : Curious underperformance 2388.50 : min_allow_group = 2675.10, mean_group = 3159.67 max_allow_group = 3644.23

simultaneous_read_1M_KBps         : WARNING : vdb          : 8E21AF4B-AFF3-477C-BA1A-FBBEF8B5099F : Curious underperformance 2698035.00 : min_allow_group = 2742554.54, mean_group = 3239253.00 max_allow_group = 3735951.46

simultaneous_read_1M_KBps         : WARNING : vdb          : CC091BCB-78F8-4E70-9733-4AF505976356 : Curious underperformance 2449151.50 : min_allow_group = 2742554.54, mean_group = 3239253.00 max_allow_group = 3735951.46

loops_per_sec                     : WARNING : logical_1    : CC091BCB-78F8-4E70-9733-4AF505976356 : Curious underperformance  490.00 : min_allow_group = 496.81, mean_group = 543.38 max_allow_group = 589.94

~~~



> **NOTE**: The results and specifics that are retrieved inside of your environment will likely be **different** to the ones above; do not expect **identical** output.



## Advanced Role Matching



In the initial **basic** deployment, we had five **identically** configured overcloud nodes, at deploy time OSP director did not have any **role matching** information to go by, and therefore the nodes were allocated **randomly**; any node could be a controller, and any node could be a compute node. Now, with the advanced overcloud deployment having nodes with **varying** system specifications, we'll want to ensure that the correct roles are **assigned** to the most **appropriate** systems. It may seem **simple** to assign roles on a node by node basis, and arguably for our deployment, it is, but when you've got a deployment with tens, or hundreds of nodes, these tools become really **useful**.



We can use the **ahc** tools shipped with OSP director to perform the role assignments for us **before** our overcloud deployment. We need only tell the tools what **specification** we expect for our roles, be that **CPU count**, **memory capacity**, **disk space**, or even a given **performance** metric, then the tool will scan the introspection and benchmarking data for applicable nodes, and tag them with a **profile** accordingly. These **profiles** get assigned inside of the Ironic database and get matched to overcloud **roles** and associated **flavors** at deployment time.



> **NOTE**: We're **not** going to use the **benchmarking data** that we retrieved in the previous step due to it being **non-realistic** and **non-representative** of true bare metal systems where you could rely on such data. Including it as part of the role matching at this stage would not be a prudent exercise.



We define the necessary specifications for each role inside of '**/etc/ahc-tools/edeploy/**' with each role having its own individual file, for **example**:



~~~

undercloud$ cat /etc/ahc-tools/edeploy/control.specs

[

 ('disk', '$disk', 'size', 'gt(4)'),

 ('network', '$eth', 'ipv4', 'network(192.0.2.0/24)'),

 ('memory', 'total', 'size', 'ge(4294967296)'),

]

~~~



The above file shows that the tooling should search for nodes with **greater** than 4GB disk space, a network interface that's **present** on the network 192.168.2.0/24, and **greater than or equal to** 4GB memory. We can match on any metric captured from the introspection process, the example above is **not comprehensive**. A full **reference** for these files can be found [here][ahc-reference].



Next, the ahc tools need to know how many of each role it needs to satisfy during its run, this is listed in the file **/etc/ahc-tools/edeploy/state**:



~~~

undercloud$ cat /etc/ahc-tools/edeploy/state

[('control', '1'), ('compute', '*')]

~~~



What this suggests is that the tooling must at least match **one controller** and **any** number of compute nodes from the **remaining** available nodes. Let's modify these files (and others) to match our environment, and our expectations to satisfy the roles we want to fill. **Recall** that our nodes have been defined with the following **specification**:



<center>



Node Role        | vCPU Count  | Memory Allocation | Storage Allocation

------------- | ----------- | -------------| ---------------

Controller    | <center>4</center> | <center>8GB</center> | <center>1x30GB</center>

Compute       | <center>2</center> | <center>6GB</center> | <center>1x20GB</center>

Ceph Storage  | <center>2</center> | <center>4GB</center> | <center>2x20GB</center>



</center>



Let's first modify our **controller** specification file to require greater than or equal to **4 logical CPU's**, and in excess of **25GB** disk space. You'll need to be the root user to change these files:



~~~

undercloud$ sudo su -

undercloud# cat << EOF > /etc/ahc-tools/edeploy/control.specs

[

 ('cpu', 'logical', 'number', 'ge(4)'),

 ('disk', 'vda', 'size', 'gt(25)'),

]

EOF

~~~



Next, let's configure our compute node specification, for this we'll require at least **2 logical CPU's**, and 6GB memory:



~~~

undercloud# cat << EOF > /etc/ahc-tools/edeploy/compute.specs

[

 ('cpu', 'logical', 'number', 'ge(2)'),

 ('memory', 'total', 'size', 'ge(6000000)'),

]

EOF

~~~



Next, configure the specification required for our **Ceph storage** nodes. We'll make sure that we match based on the fact that our nodes have **two hard drives**:



~~~

undercloud# cat << EOF > /etc/ahc-tools/edeploy/ceph.specs

[

 ('cpu', 'logical', 'number', 'ge(2)'),

 ('disk', 'vda', 'size', 'ge(20)'),

 ('disk', 'vdb', 'size', 'ge(20)'),

]

EOF

~~~



Ensure that we specify our required **node counts**, noting that we can use the **wildcard** for the compute nodes, this is typically **common practice** - ensure that you cover your controller and storage node requirements (specifically, **three** controllers and **three** Ceph nodes), and the remainder can be considered compute nodes:



~~~

undercloud# cat << EOF > /etc/ahc-tools/edeploy/state

[('control', '3'), ('ceph', '3'), ('compute', '*')]

EOF

~~~



Finally, we can run the **ahc-match** tool to assign these tags to our overcloud nodes:



~~~

undercloud# ahc-match

~~~



> **NOTE**: This command is **silent** unless there are any **errors**.



The tagging can now be verified, for example:



~~~

undercloud# exit

undercloud$ source ~/stackrc

undercloud$ first_node=$(ironic node-list | grep -v UUID \

| awk '{print $2;}' | sed -e /^$/d  | head -n1)



undercloud$ ironic node-show $first_node | grep -A1 properties

properties   | {u'memory_mb': u'4096', u'cpu_arch': u'x86_64', u'local_gb': u'29',

             | u'cpus': u'2', u'capabilities': u'profile:ceph,boot_option:local'}



~~~



As you can see, in the **capabilities** section, this specific node has the **Ceph** storage profile assigned to it. We should now create **flavors** in Nova that map to these profiles, ensuring that when OSP director informs Nova of the request to boot up bare metal servers, it uses a **flavor** to select the **correct** nodes for each role.



For a **complete** output to verify that all nodes got allocated the **correct** **profile**, run the following command:



~~~

undercloud$ for i in $(ironic node-list | grep -v UUID | awk '{print $2;}' \

| sed -e /^$/d); do ironic node-show $i | grep -A1 properties; \

echo "======="; done;



| properties             | {u'memory_mb': u'4096', u'cpu_arch': u'x86_64', u'local_gb': u'29',      |

|                        | u'cpus': u'2', u'capabilities': u'profile:ceph,boot_option:local'}       |

| instance_uuid          | None                                                                     |

=======

| properties             | {u'memory_mb': u'4096', u'cpu_arch': u'x86_64', u'local_gb': u'29',      |

|                        | u'cpus': u'2', u'capabilities': u'profile:ceph,boot_option:local'}       |

| instance_uuid          | None                                                                     |

=======

| properties             | {u'memory_mb': u'4096', u'cpu_arch': u'x86_64', u'local_gb': u'29',      |

|                        | u'cpus': u'2', u'capabilities': u'profile:ceph,boot_option:local'}       |

| instance_uuid          | None                                                                     |

=======

| properties             | {u'memory_mb': u'6144', u'cpu_arch': u'x86_64', u'local_gb': u'19',      |

|                        | u'cpus': u'2', u'capabilities': u'profile:compute,boot_option:local'}    |

| instance_uuid          | None                                                                     |

=======

| properties             | {u'memory_mb': u'6144', u'cpu_arch': u'x86_64', u'local_gb': u'19',      |

|                        | u'cpus': u'2', u'capabilities': u'profile:compute,boot_option:local'}    |

| instance_uuid          | None                                                                     |

=======

| properties             | {u'memory_mb': u'8192', u'cpu_arch': u'x86_64', u'local_gb': u'29',      |

|                        | u'cpus': u'4', u'capabilities': u'profile:control,boot_option:local'}    |

| instance_uuid          | None                                                                     |

=======

| properties             | {u'memory_mb': u'8192', u'cpu_arch': u'x86_64', u'local_gb': u'29',      |

|                        | u'cpus': u'4', u'capabilities': u'profile:control,boot_option:local'}    |

| instance_uuid          | None                                                                     |

=======

| properties             | {u'memory_mb': u'8192', u'cpu_arch': u'x86_64', u'local_gb': u'29',      |

|                        | u'cpus': u'4', u'capabilities': u'profile:control,boot_option:local'}    |

| instance_uuid          | None                                                                     |

=======

~~~



> **NOTE**: If you do not have the profile allocations **perfectly assigned**, deployment can **fail** as Nova will not be able to match the requested node **count**.



<br>



The role assignment in OSP director works as follows-



1. Each node registered in Ironic has a **profile** attached to it, applied either **manually**, or via **advanced role matching** (with ahc-tools).<br><br>

2. **Flavors** are created in **Nova** that reference a **profile** as a flavor **property**.<br><br>

3. OSP director has to satisfy specific **roles**, and it does so by being told which **flavor** to use for each **role**. If no flavor is specified, by default it uses any nodes that fulfill the "**baremetal**" flavor specification, but the flavors can be **overridden** at deploy time.



Therefore, let's create appropriately sized flavors, noting that we need to specify the disk size slightly **smaller** than our nodes have so **partitioning** works properly:



~~~

undercloud$ source ~/stackrc

undercloud$ openstack flavor create --id auto --ram 8192 --disk 26 --vcpus 4 control

+----------------------------+--------------------------------------+

| Field                      | Value                                |

+----------------------------+--------------------------------------+

| OS-FLV-DISABLED:disabled   | False                                |

| OS-FLV-EXT-DATA:ephemeral  | 0                                    |

| disk                       | 26                                   |

| id                         | 7ffab934-05b1-4ae9-8df8-c2fd84f89a60 |

| name                       | control                              |

| os-flavor-access:is_public | True                                 |

| ram                        | 8192                                 |

| rxtx_factor                | 1.0                                  |

| swap                       |                                      |

| vcpus                      | 4                                    |

+----------------------------+--------------------------------------+



undercloud$ openstack flavor create --id auto --ram 6144 --disk 18 --vcpus 2 compute

+----------------------------+--------------------------------------+

| Field                      | Value                                |

+----------------------------+--------------------------------------+

| OS-FLV-DISABLED:disabled   | False                                |

| OS-FLV-EXT-DATA:ephemeral  | 0                                    |

| disk                       | 18                                   |

| id                         | 8330c272-2e4b-45c9-ab69-1873fece0ed6 |

| name                       | compute                              |

| os-flavor-access:is_public | True                                 |

| ram                        | 6144                                 |

| rxtx_factor                | 1.0                                  |

| swap                       |                                      |

| vcpus                      | 2                                    |

+----------------------------+--------------------------------------+



undercloud$ openstack flavor create --id auto --ram 4096 --disk 18 --vcpus 2 ceph

+----------------------------+--------------------------------------+

| Field                      | Value                                |

+----------------------------+--------------------------------------+

| OS-FLV-DISABLED:disabled   | False                                |

| OS-FLV-EXT-DATA:ephemeral  | 0                                    |

| disk                       | 18                                   |

| id                         | 2d54ea41-c4e1-424d-8717-c27a6a04c860 |

| name                       | ceph                                 |

| os-flavor-access:is_public | True                                 |

| ram                        | 4096                                 |

| rxtx_factor                | 1.0                                  |

| swap                       |                                      |

| vcpus                      | 2                                    |

+----------------------------+--------------------------------------+

~~~



> **NOTE**: Recall that we have to specify the **size** of the machine when creating flavors because of the way that **Nova** works, even though we've already accounted for system specifications when we did the advanced role matching process.



Next, we need to map the **Ironic profiles** to the **flavors** we just created, meaning that when we provision, Nova searches for hosts that have the correct **profile** assigned to them:



~~~

undercloud$ openstack flavor set --property "cpu_arch"="x86_64" \

--property "capabilities:boot_option"="local" \

--property "capabilities:profile"="control" control

+----------------------------+-------------------------------------------------------------------------------------+

| Field                      | Value                                                                               |

+----------------------------+-------------------------------------------------------------------------------------+

| OS-FLV-DISABLED:disabled   | False                                                                               |

| OS-FLV-EXT-DATA:ephemeral  | 0                                                                                   |

| disk                       | 26                                                                                  |

| id                         | 7ffab934-05b1-4ae9-8df8-c2fd84f89a60                                                |

| name                       | control                                                                             |

| os-flavor-access:is_public | True                                                                                |

| properties                 | capabilities:boot_option='local', capabilities:profile='control', cpu_arch='x86_64' |

| ram                        | 8192                                                                                |

| rxtx_factor                | 1.0                                                                                 |

| swap                       |                                                                                     |

| vcpus                      | 4                                                                                   |

+----------------------------+-------------------------------------------------------------------------------------+



undercloud$ openstack flavor set --property "cpu_arch"="x86_64" \

--property "capabilities:boot_option"="local" \

--property "capabilities:profile"="compute" compute

+----------------------------+-------------------------------------------------------------------------------------+

| Field                      | Value                                                                               |

+----------------------------+-------------------------------------------------------------------------------------+

| OS-FLV-DISABLED:disabled   | False                                                                               |

| OS-FLV-EXT-DATA:ephemeral  | 0                                                                                   |

| disk                       | 18                                                                                  |

| id                         | 8330c272-2e4b-45c9-ab69-1873fece0ed6                                                |

| name                       | compute                                                                             |

| os-flavor-access:is_public | True                                                                                |

| properties                 | capabilities:boot_option='local', capabilities:profile='compute', cpu_arch='x86_64' |

| ram                        | 6144                                                                                |

| rxtx_factor                | 1.0                                                                                 |

| swap                       |                                                                                     |

| vcpus                      | 2                                                                                   |

+----------------------------+-------------------------------------------------------------------------------------+



undercloud$ openstack flavor set --property "cpu_arch"="x86_64" \

    --property "capabilities:boot_option"="local" \

    --property "capabilities:profile"="ceph" ceph

+----------------------------+----------------------------------------------------------------------------------+

| Field                      | Value                                                                            |

+----------------------------+----------------------------------------------------------------------------------+

| OS-FLV-DISABLED:disabled   | False                                                                            |

| OS-FLV-EXT-DATA:ephemeral  | 0                                                                                |

| disk                       | 18                                                                               |

| id                         | 2d54ea41-c4e1-424d-8717-c27a6a04c860                                             |

| name                       | ceph                                                                             |

| os-flavor-access:is_public | True                                                                             |

| properties                 | capabilities:boot_option='local', capabilities:profile='ceph', cpu_arch='x86_64' |

| ram                        | 4096                                                                             |

| rxtx_factor                | 1.0                                                                              |

| swap                       |                                                                                  |

| vcpus                      | 2                                                                                |

+----------------------------+----------------------------------------------------------------------------------+

~~~



Upon **deployment**, we'll ensure that we **override** the **role to flavor matching** with the ones that we've created above.



## Making Changes to Templates



This section aims to **explain** exactly how we can make **changes** to the default templates and **override** values to match our expected overcloud deployment. Recall that when OSP director launches a deployment, by **default**, it passes the following data into Heat:



1. A **top-level** Heat stack **template**, defining an **abstract** representation of the overcloud:



**/usr/share/openstack-tripleo-heat-templates/overcloud-without-mergepy.yaml**



2. An **environment** file, containing a **resource registry**, advising Heat of how it can satisfy the requested resources from the **top-level** stack template with **nested** stacks:



**/usr/share/openstack-tripleo-heat-templates/overcloud-resource-registry-puppet.yaml**



3. A set of **parameters** that define the environment to be deployed. The overcloud deployment script has a large number of parameters already defined as **defaults**, but can be **overridden** by the CLI tooling. Depending on the parameter, these options may be submitted into **hieradata** for the overcloud puppet run, or will set options that **Heat** needs, e.g. number of nodes to deploy.



If we want to build up an environment that's more **advanced** or more **customised** than the **default** templates (and the CLI parameter overrides that we support), we can push **modified** or **customised** templates to OSP director for deployment. You do not have to push a complete template, simply sections that **override** the bits that you want to change. The exact process for this is to override sections in the **resource registry** by specifying an additional **environment** file.



This additional **environment** file can contain either **parameter** overrides, or resource registry overrides (sending OSP director to an alternative Heat **template** for the resources that you want to **override**). These alternative Heat templates will define the **modifications** we're trying to make. To make more fundamental changes to the RHEL OSP deployment, modification of the top-level template is also possible, but **not recommended**.



It is, however, recommended to make a **copy** of the **heat templates directory**, allowing for **modification** of local files, rather than updating files in **/usr/share/**. We can specify an alternative directory for OSP director to use at deploy time; so let's do that before we start to modify these files for our advanced overcloud:



~~~

undercloud$ cp -rf /usr/share/openstack-tripleo-heat-templates ~/my_templates/

~~~



> **NOTE**: In the labs we'll copy this directory, but **better practice** would be to invoke a version control system, e.g. creation of a **local git repo** of these files rather than copying them. In larger, and more complex configurations this would assist in **troubleshooting** by ensuring that changes are **tracked**. Administrators can see what changes were made, easily allowing changes to be **reverted**, or to isolate problematic changes more quickly.



We now have the **entire** directory structure of the OSP director supplied Heat templates. We can **modify** these to suit our overcloud deployment as we see fit. This also has the added benefit of customers being able to use **version control** for the directory should they want to control the templates being used.



When we want to **override** parameters or even **resource registry** entries, we can simply pass the relevant file(s) into the OSP director overcloud deployment (with the '**-e**' flag), for example:



**(Do not run this command, it's an example only)**



~~~

$ openstack overcloud deploy \

--templates ~/my_templates/ -e ~/my_templates/my_params.yaml

~~~



For example, to force Nova to use a Ceph for its ephemeral storage we could specify the following parameter via an environment file; our '**my_params.yaml**' could look like the following:



~~~

# My Custom Overcloud Parameters

parameters:

   NovaEnableRbdBackend: true

~~~



This parameter will find its way into **hieradata**, and when **puppet** is executed during the **SoftwareDeployment** step, Nova would be set to point to Ceph on the overcloud for its ephemeral storage requirements.



> **NOTE**: **CLI provided** values, or **default** parameters (if a CLI override has **not** been provided) **cannot be overridden** via an **environment** file. In the future, it's intended that all parameters can be set via an environment file, providing functionality akin to an **overcloud.conf**. So, when setting parameters here, your results may vary. See [this bug][config-override] entry for more details.



Aside from **parameter** overrides, it's possible to specify alternative **resource definition** templates by overriding entries in the **resource registry** via an environment file. By default, OSP director pulls in the file '**overcloud-resource-registry-puppet.yaml**' which tells Heat where it can go to satisfy resource types that it does not understand, for example-



~~~

undercloud$ head -n3 /usr/share/openstack-tripleo-heat-templates/overcloud-resource-registry-puppet.yaml

resource_registry:

  OS::TripleO::BlockStorage: puppet/cinder-storage-puppet.yaml

  OS::TripleO::BlockStorage::Net::SoftwareConfig: net-config-noop.yaml

~~~



This is telling heat that for any stacks that request "**OS::TripleO::BlockStorage**", the definition can be found at "**puppet/cinder-storage-puppet.yaml**", creating a **nested stack**. So, if we wanted to specify an alternative location to Heat, i.e. to use a different template for the **OS::TripleO::BlockStorage**" resource, we could simply **override** it via an environment file. For example:



**(Do not run this command, it's an example only)**



~~~

$ cat ~/my_templates/my_cinder_changes.yaml

# Environment file to specify alternative block storage template

resource_registry:

   OS::TripleO::BlockStorage: ~/my_templates/my_cinder.yaml

~~~



We **only** need to specify the resources that we want to override, the ones we want to remain **default** will be taken from the **top-level environment file** that OSP director uses. Note that it is possible to **chain** environment files together, allowing a single deployment call to pull in **multiple** overrides; we could pull in the previous two examples-



**(Do not run this command, it's an example only)**



~~~

$ openstack overcloud deploy --templates ~/my_templates/ \

-e ~/my_templates/my_params.yaml \

-e ~/my_templates/my_cinder_changes.yaml

~~~



> **NOTE**: If you specify **multiple** environment files that have **conflicting** parameters or overrides, the **last** environment file specified takes priority.



## Integration of Ceph



Many of our customers will want to deploy Ceph **alongside** RHEL OSP; in previous deployment tools this was a **two-step process** where one would have to deploy Ceph nodes outside of the RHEL OSP installation and tie them together **afterwards**. With OSP director we have a single platform that can do both for us, creating a **unified** approach to OSP + Ceph deployment. As per the existing reference architectures, the Ceph **monitors** run on the **controller** nodes, and only the **OSD** functionality runs on the designated Ceph servers.



Ceph **isn't** configured by default when deploying RHEL OSP with OSP director and therefore we have to enable it. Enablement of this functionality is **easy**; we need to carry out the following:



1. If you want Ceph to be provisioned on **specific** hosts (highly likely) then you'll need to create and assign a node profile using **introspection** data, or via manual assignment, to **identify** those hosts that are **capable** of being OSD nodes.<br><br>

2. Configure the **hieradata** within the **templates** directory that you push to OSP director; this is used during the Ceph configuration to choose the **OSD** drives, and set other **parameters**. The **hieradata** is requested here; the Heat stack for a Ceph storage node:



~~~

undercloud$ grep -A20 StructuredConfig ~/my_templates/puppet/ceph-storage-puppet.yaml

    type: OS::Heat::StructuredConfig

    properties:

      group: os-apply-config

      config:

        hiera:

          hierarchy:

            - heat_config_%{::deploy_config_name}

            - ceph_cluster # provided by CephClusterConfig

            - ceph

            - RedHat

            - common

          datafiles:

            common:

              raw_data: {get_file: hieradata/common.yaml}

            ceph:

              raw_data: {get_file: hieradata/ceph.yaml}

              mapped_data:

                ntp::servers: {get_input: ntp_servers}

                enable_package_install: {get_input: enable_package_install}

                ceph::profile::params::cluster_network: {get_input: ceph_cluster_network}

                ceph::profile::params::public_network: {get_input: ceph_public_network}

~~~



The **content** of the hieradata is as follows; we can **edit** this data to suit the Ceph deployment as per the requirements of the overcloud, e.g. specifying the **disks** to use for OSD's, and perhaps the default number of object **replicas** we'd like in the cluster:



~~~

undercloud$ cat ~/my_templates/puppet/hieradata/ceph.yaml

ceph::profile::params::osd_journal_size: 1024

ceph::profile::params::osd_pool_default_pg_num: 128

ceph::profile::params::osd_pool_default_pgp_num: 128

ceph::profile::params::osd_pool_default_size: 3

ceph::profile::params::osd_pool_default_min_size: 1

ceph::profile::params::osds: {/srv/data: {}}

ceph::profile::params::manage_repo: false

ceph::profile::params::authentication_type: cephx



ceph_pools:

- volumes

- vms

- images



ceph_osd_selinux_permissive: true

~~~



3. At **provision** time, specify to OSP director the **number** of Ceph **OSD** nodes that you would like, reference the **flavor** that you've configured (that maps to the node **property**), and pass an **environment** file that overrides certain Ceph-specific parameters. OSP director ships with an example for us to use:



> **NOTE**: If the following file *storage-environment.yaml* is not present, you likely have GA bits and need to check your repo and possibly run a **yum update**.



~~~

undercloud$ cat ~/my_templates/environments/storage-environment.yaml | egrep -v '^  #|^$'



## A Heat environment file which can be used to set up storage

## backends. Defaults to Ceph used as a backend for Cinder, Glance and

## Nova ephemeral storage.

parameters:

  CinderEnableIscsiBackend: false

  CinderEnableRbdBackend: true

  NovaEnableRbdBackend: true

  GlanceBackend: rbd

~~~



As in the previous section, we can **override** a wide variety of **parameters** so long as they're **not hard coded** by the OSP director unified CLI. Setting the Ceph storage is a prime **example** of the parameters that we can change, giving customers the ability to use Ceph for **part**, **or all**, of their OpenStack storage needs.



As we've **already** got this storage environment file preconfigured to enable Ceph integration (should we call it), the only thing we need to do is configure the **hieradata** to point towards our **OSD volumes**. Recall that we provided our Ceph nodes a second disk - this will come up as **/dev/vdb** when using VirtIO:



~~~

undercloud$ cat << EOF > ~/my_templates/puppet/hieradata/ceph.yaml

ceph::profile::params::osd_journal_size: 1024

ceph::profile::params::osd_pool_default_pg_num: 128

ceph::profile::params::osd_pool_default_pgp_num: 128

ceph::profile::params::osd_pool_default_size: 3

ceph::profile::params::osd_pool_default_min_size: 1

ceph::profile::params::osds:

     '/dev/vdb':

       journal: ''

ceph::profile::params::manage_repo: false

ceph::profile::params::authentication_type: cephx



ceph_pools:

- volumes

- vms

- images



ceph_osd_selinux_permissive: true

EOF

~~~



> **NOTE**: Given the constraints of the virtual environment, we're **not** opting to use a **journal** disk, so we'll leave this blank in our deployment.



We'll **hold off** from testing the Ceph integration at this stage; we'll build up an advanced networking configuration first and build our advanced, ceph-integrated overcloud in **one step**.



## Advanced Networking Configuration



When using the OSP director defaults, the **PXE/management** network is used for **all** network traffic; we assume that our nodes only have **one** usable network interface, therefore all API endpoints, database synchronisation, message replication, storage replication, and tenant network traffic all flows over the single network interface. This may work just fine for **PoC's**, but production clouds make use of **multiple** network interfaces, network traffic **isolation**, and where possible, **bonding** for **resiliency**.



As we've previously explored, each node's networking configuration is setup at deploy time via **os-net-config** with data seeded to the node via Heat. **os-collect-config** pulls down the metadata, **os-refresh-config** gets executed , which, in turn, calls **os-apply-config** to set the networking config at **/etc/os-net-config/config.json** and finally calls **os-net-config** to apply it.



Just like the Ceph example where we **overrode** the default storage configuration, we can do the same thing with the **networking** configuration. The key thing to understand is **how** networks are defined in OSP director, and what the default network **topology** looks like. Out of the box, OSP director assumes that you only have a **single** flat network interface (the one it uses for provisioning) with all network **types** running on-top; no isolation, no resilience. However, OSP director does provide extensions to allow you to **modify** this.



The **network** creation is called here:



~~~

undercloud$ grep -A2 "creates the network" ~/my_templates/overcloud-without-mergepy.yaml

  # creates the network architecture

  Networks:

    type: OS::TripleO::Network

~~~



This **type** is defined in the **resource registry** as below:



~~~

undercloud$ grep "OS::TripleO::Network" \

~/my_templates/overcloud-resource-registry-puppet.yaml | grep -v "Port"

  OS::TripleO::Network: network/networks.yaml

  OS::TripleO::Network::External: network/noop.yaml

  OS::TripleO::Network::InternalApi: network/noop.yaml

  OS::TripleO::Network::StorageMgmt: network/noop.yaml

  OS::TripleO::Network::Storage: network/noop.yaml

  OS::TripleO::Network::Tenant: network/noop.yaml

~~~



The **networks.yaml** file defines the network types identified above, e.g. "InternalAPI" or "StorageMgmt". Note that these available network **types** are all linked to a **noop.yaml** file, which tells it to take **no action** with the specific network definitions, i.e. just place all services on the PXE/management network. If these were specified, **Neutron** networks on the undercloud would be created for us (more on this later).



The default resource registry also defines each **port** template per **role**, allowing nodes to link into the relevant networks. These also point to **noop.yaml** as you can't have a port without a network, with the exception of the **external** port for the controller, where in a default deployment it uses **ctlplane_vip.yaml** to setup **virtual IP addresses** on the network being used for the PXE/management network. If we use the **controller** as an example:



~~~

undercloud$ grep "OS::TripleO::Controller::Ports" \

~/my_templates/overcloud-resource-registry-puppet.yaml

  OS::TripleO::Controller::Ports::ExternalPort: network/ports/ctlplane_vip.yaml

  OS::TripleO::Controller::Ports::InternalApiPort: network/ports/noop.yaml

  OS::TripleO::Controller::Ports::StoragePort: network/ports/noop.yaml

  OS::TripleO::Controller::Ports::StorageMgmtPort: network/ports/noop.yaml

  OS::TripleO::Controller::Ports::TenantPort: network/ports/noop.yaml

  OS::TripleO::Controller::Ports::RedisVipPort: network/ports/ctlplane_vip.yaml

~~~



Recall that each **role** also has an associated **template** that gets called to satisfy the **requirements** for that role. This template dictates the networking configuration, it sets a number of **port** resources for example:



~~~

undercloud$ grep -A22 ExternalPort \

~/my_templates/puppet/controller-puppet.yaml | head -n24



  ExternalPort:

    type: OS::TripleO::Controller::Ports::ExternalPort

    properties:

      ControlPlaneIP: {get_attr: [Controller, networks, ctlplane, 0]}



  InternalApiPort:

    type: OS::TripleO::Controller::Ports::InternalApiPort

    properties:

      ControlPlaneIP: {get_attr: [Controller, networks, ctlplane, 0]}



  StoragePort:

    type: OS::TripleO::Controller::Ports::StoragePort

    properties:

      ControlPlaneIP: {get_attr: [Controller, networks, ctlplane, 0]}



  StorageMgmtPort:

    type: OS::TripleO::Controller::Ports::StorageMgmtPort

    properties:

      ControlPlaneIP: {get_attr: [Controller, networks, ctlplane, 0]}



  TenantPort:

    type: OS::TripleO::Controller::Ports::TenantPort

    properties:

      ControlPlaneIP: {get_attr: [Controller, networks, ctlplane, 0]}

~~~



As you can see, all of the networks get hard coded by default to **ctlplane**. The details for these ports get put into a SoftwareConfig and SoftwareDeployment resource for application on the node at deploy time with **os-net-config**:



~~~

undercloud$ grep -A12 NetworkConfig ~/my_templates/puppet/controller-puppet.yaml

  NetworkConfig:

    type: OS::TripleO::Controller::Net::SoftwareConfig

    properties:

      ExternalIpSubnet: {get_attr: [ExternalPort, ip_subnet]}

      InternalApiIpSubnet: {get_attr: [InternalApiPort, ip_subnet]}

      StorageIpSubnet: {get_attr: [StoragePort, ip_subnet]}

      StorageMgmtIpSubnet: {get_attr: [StorageMgmtPort, ip_subnet]}

      TenantIpSubnet: {get_attr: [TenantPort, ip_subnet]}



  NetworkDeployment:

    type: OS::TripleO::SoftwareDeployment

    properties:

      config: {get_resource: NetworkConfig}

      server: {get_resource: Controller}

      input_values:

        bridge_name: br-ex

        interface_name: {get_param: NeutronPublicInterface}



  ControllerDeployment:

    type: OS::TripleO::SoftwareDeployment

    depends_on: NetworkDeployment

    properties:

      config: {get_resource: ControllerConfig}

      server: {get_resource: Controller}

      input_values:

~~~



The **NetworkConfig** (type being "**OS::TripleO::Controller::Net::SoftwareConfig**") is mapped in the resource registry:



~~~

undercloud$ grep OS::TripleO::Controller::Net::SoftwareConfig \

~/my_templates/overcloud-resource-registry-puppet.yaml



  OS::TripleO::Controller::Net::SoftwareConfig: net-config-bridge.yaml

~~~



This **net-config-bridge** file is where the deployment actually tells Heat to put the metadata in place for **os-apply-config**, so that **os-net-config** can use it:



~~~

undercloud$ grep -A26 resources ~/my_templates/net-config-bridge.yaml

resources:

  OsNetConfigImpl:

    type: OS::Heat::StructuredConfig

    properties:

      group: os-apply-config

      config:

        os_net_config:

          network_config:

            -

              type: ovs_bridge

              name: {get_input: bridge_name}

              use_dhcp: true

              # Can't do this yet: https://bugs.launchpad.net/heat/+bug/1344284

              #ovs_extra:

              #  - list_join:

              #    - ' '

              #    - - br-set-external-id

              #      - {get_input: bridge_name}

              #      - bridge-id

              #      - {get_input: bridge_name}

              members:

                -

                  type: interface

                  name: {get_input: interface_name}

                  # force the MAC address of the bridge to this interface

                  primary: true

~~~



The only thing that this will ask **os-net-config** to do is create **br-ex** on the controller nodes, and add the "**NeutronPublicInterface**" as an interface - a very **simple** network configuration but arguably just fine for a **PoC**.



## Building an Advanced Networking Environment File



So, now we know **how** it sets up the networks, how do we configure an advanced networking configuration; one that takes into consideration **multiple** network interfaces, **bonding**, and networking **isolation**? Unfortunately the virtual infrastructure does **not** allow for us to replicate **bonding**, but we'll certainly configure **multiple** network interfaces for our nodes and implement **network isolation** for our multiple **traffic types**. We will need to set the following components up:



1. An environment file that **overrides** the *necessary* overcloud **network** definition resource registry entries; we override the base template from a **noop** configuration to one that has a full network definition, including any necessary subnets, so that Heat will look to **create** the networks at deploy time.<br><br>

2. In the same, or **another**, environment file we override the *necessary* **port definitions** for each **role** in the resource registry. To use a network the node **needs** to have a port on such network, and we'll need to have a resource definition in the resource registry. Note that not every node will need a port on every network, e.g. **storage** nodes don't necessarily need to see the **tenant** networks.<br><br>

3. Again, in the same, or **another**, environment file we override the per-role definitions of **network specification data** or **nic data** that gets fed into **os-net-config** at deploy time, this allows the node to attach to the networks and ports we've listed above; creating and utilising multiple interfaces and optionally creating bonds.<br><br>

4. Finally, in the same, or **another** environment file we provide a set of **network parameter overrides** that we want to use (overriding any of the defaults found in the templates), including any necessary **subnet** details, **VLAN** id's (if required), and if required, an external network bridge.



<br>To explore all of the possible configuration options, let's put together an environment file that configures the following for our **advanced** overcloud deployment:



* All of our overcloud nodes use **eth0** as their **PXE/management** network (**ctlplane**)

* We'll use our second interface (**eth1**) to run a number of VLANs for **network isolation** for internal networks.

* We configure our third NIC (**eth2**) to have a flat interface for **external access** this will include the **public API access** and **floating IP access**. This will allow us to have access to the overcloud **without** using the PXE network. This interface will only configured for the **controller** node(s).



As a **suggested** set of VLAN's and subnets we can use the following:<center>



Interface | Purpose | VLAN | Subnet

----------|---------|------|-------

eth0 | **PXE/Management** | None | 172.16.0.0/24

eth1 | **Internal API** | 101 | 172.17.1.0/24

eth1 | **Tenant Traffic** | 201 | 172.17.2.0/24

eth1 | **Storage** | 301 | 172.17.3.0/24

eth1 | **Storage Clustering** | 401 | 172.16.4.0/24

eth2 | **External Access** | None | 192.168.122.0/24



</center>



Firstly, let's take a copy of the example **network-isolation.yaml** file that already overrides the **noop** with the creation of individual networks for the above configurations, as well as assigns the ports that are necessary for each role:



~~~

undercloud$ cp ~/my_templates/environments/network-isolation.yaml ~/my_templates/advanced-networking.yaml



undercloud$ cat ~/my_templates/advanced-networking.yaml

# Enable the creation of Neutron networks for isolated Overcloud

# traffic and configure each role to assign ports (related

# to that role) on these networks.

resource_registry:

  OS::TripleO::Network::External: ../network/external.yaml

  OS::TripleO::Network::InternalApi: ../network/internal_api.yaml

  OS::TripleO::Network::StorageMgmt: ../network/storage_mgmt.yaml

  OS::TripleO::Network::Storage: ../network/storage.yaml

  OS::TripleO::Network::Tenant: ../network/tenant.yaml



  # Port assignments for the controller role

  OS::TripleO::Controller::Ports::ExternalPort: ../network/ports/external.yaml

  OS::TripleO::Controller::Ports::InternalApiPort: ../network/ports/internal_api.yaml

  OS::TripleO::Controller::Ports::StoragePort: ../network/ports/storage.yaml

  OS::TripleO::Controller::Ports::StorageMgmtPort: ../network/ports/storage_mgmt.yaml

  OS::TripleO::Controller::Ports::TenantPort: ../network/ports/tenant.yaml



  # Port assignments for the compute role

  OS::TripleO::Compute::Ports::InternalApiPort: ../network/ports/internal_api.yaml

  OS::TripleO::Compute::Ports::StoragePort: ../network/ports/storage.yaml

  OS::TripleO::Compute::Ports::TenantPort: ../network/ports/tenant.yaml



  # Port assignments for the ceph storage role

  OS::TripleO::CephStorage::Ports::StoragePort: ../network/ports/storage.yaml

  OS::TripleO::CephStorage::Ports::StorageMgmtPort: ../network/ports/storage_mgmt.yaml



  # Port assignments for the swift storage role

  OS::TripleO::SwiftStorage::Ports::InternalApiPort: ../network/ports/internal_api.yaml

  OS::TripleO::SwiftStorage::Ports::StoragePort: ../network/ports/storage.yaml

  OS::TripleO::SwiftStorage::Ports::StorageMgmtPort: ../network/ports/storage_mgmt.yaml



  # Port assignments for the block storage role

  OS::TripleO::BlockStorage::Ports::InternalApiPort: ../network/ports/internal_api.yaml

  OS::TripleO::BlockStorage::Ports::StoragePort: ../network/ports/storage.yaml

  OS::TripleO::BlockStorage::Ports::StorageMgmtPort: ../network/ports/storage_mgmt.yaml



  # Port assignments for service virtual IPs for the controller role

  OS::TripleO::Controller::Ports::RedisVipPort: ../network/ports/vip.yaml

~~~



As we've moved this file out of its directory structure, let's ensure that the resource registry entries link to an **absolute** path:



~~~

undercloud$ sed -i 's|../network/|/home/stack/my_templates/network/|g' \

~/my_templates/advanced-networking.yaml

~~~



As our intention is to deploy our environment with **Ceph**, we can **remove** the Swift and the BlockStorage ports; it won't affect the deployment to leave them in, but for **cleanliness** we should remove them:



~~~

undercloud$ sed -i '/Swift/Id' ~/my_templates/advanced-networking.yaml

undercloud$ sed -i '/Block/Id' ~/my_templates/advanced-networking.yaml

~~~



Next, we need to add some overrides for our **per-role nic configuration** that gets executed on each node as part of the SoftwareConfig and executed by **os-net-config**, we can add this directly to our existing environment file:



~~~

undercloud$ cat << EOF >> ~/my_templates/advanced-networking.yaml

  # NIC Configs for our roles

  OS::TripleO::Compute::Net::SoftwareConfig: /home/stack/my_templates/nic-configs/compute.yaml

  OS::TripleO::Controller::Net::SoftwareConfig: /home/stack/my_templates/nic-configs/controller.yaml

  OS::TripleO::CephStorage::Net::SoftwareConfig: /home/stack/my_templates/nic-configs/ceph-storage.yaml

EOF

~~~



These NIC configurations do **not** currently **exist**, so we'll need to **create** them. First create the relevant directory:



~~~

undercloud$ mkdir -p ~/my_templates/nic-configs/

~~~



Now, download the (**hopefully**) soon to be **merged** upstream network definitions for our roles:



~~~

undercloud$ curl -o ~/my_templates/nic-configs/multiple-nic-vlans.tar.bz2 \

http://people.redhat.com/roxenham/multiple-nic-vlans.tar.bz2

~~~



> **NOTE**: This merge request can be found [here][multi-nic-merge].



Next, extract the files from this tarball:



~~~

undercloud$ cd ~/my_templates/nic-configs/

undercloud$ tar -xvjpf multiple-nic-vlans.tar.bz2

ceph-storage.yaml

cinder-storage.yaml

compute.yaml

controller.yaml

README.md

swift-storage.yaml

~~~



These files define the **per-role network configuration** that gets applied **locally** on the node. Each **role** has it's own configuration file that we've already specified in the resource registry so at deploy time it knows which **configuration** to apply. This is the example for a **controller** node:



~~~

undercloud$ grep -A41 resources ~/my_templates/nic-configs/controller.yaml

resources:

  OsNetConfigImpl:

    type: OS::Heat::StructuredConfig

    properties:

      group: os-apply-config

      config:

        os_net_config:

          network_config:

            -

              type: ovs_bridge

              # Assuming you want to keep br-ex as external bridge name

              name: {get_input: bridge_name}

              use_dhcp: true

              addresses:

                -

                  ip_netmask: {get_param: ExternalIpSubnet}

                  routes:

                    -

                      ip_netmask: 0.0.0.0/0

                      next_hop: {get_param: ExternalInterfaceDefaultRoute}

              members:

                -

                  type: interface

                  name: nic3

                  # force the MAC address of the bridge to this interface

                  primary: true

            -

              type: ovs_bridge

              name: br-isolated

              use_dhcp: false

              members:

                -

                  type: interface

                  name: nic2

                  primary: true

                -

                  type: vlan

                  vlan_id: {get_param: InternalApiNetworkVlanID}

                  addresses:

                  -

                    ip_netmask: {get_param: InternalApiIpSubnet}

(...)

~~~



> **NOTE**: **os-net-config** uses **nic1** for **eth0**, **nic2** for **eth1**, and so on.



So, you can see that on the controller it will create **two OVS bridges**, one that defaults to "**br-ex**" from the "**bridge_name**" parameter; it will add the external network IP address to this so that it can be contacted on the relevant subnet for **nic3** (or eth2), which maps to our external default network (**192.168.122.0/24**). It then creates another OVS bridge, called **br-isolated** and attaches a number of VLANs to this that correspond to the network types we've established.



The **other** definitions are slightly **different** based on the **required** network access each role has; for example - the compute and storage nodes do **not** need to see the external network, etc. Each definition has an OVS bridge called **br-isolated** that maps to the **VLANs** that are required for the role in the deployment. To see other roles you can simply view them in the directory structure:



~~~

undercloud$ less ~/my_templates/nic-configs/compute.yaml

(...)

~~~



> **NOTE**: Whilst the compute and storage nodes have a third network interface (**eth2**), this will not be used. Only the **controller** nodes utilise the third interface for **external** connectivity. If the desire is to use **provider networks**, the third interface could also be used for the compute nodes to bridge network traffic directly, **without** the need for floating IPs.



These network definitions rely on a set of parameters that we'll want to override from their defaults. So let's add a few more lines to our **advanced-networking.yaml** environment file to represent our desired configuration. These will set the VLAN tags and the subnet ranges that we highlighted in the table above.



~~~

undercloud$ cat << EOF >> ~/my_templates/advanced-networking.yaml

parameter_defaults:

  # Internal API used for private OpenStack Traffic

  InternalApiNetCidr: 172.17.1.0/24

  InternalApiAllocationPools: [{'start': '172.17.1.10', 'end': '172.17.1.200'}]

  InternalApiNetworkVlanID: 101



  # Tenant Network Traffic - will be used for VXLAN over VLAN

  TenantNetCidr: 172.17.2.0/24

  TenantAllocationPools: [{'start': '172.17.2.10', 'end': '172.17.2.200'}]

  TenantNetworkVlanID: 201



  # Public Storage Access - e.g. Nova/Glance <--> Ceph

  StorageNetCidr: 172.17.3.0/24

  StorageAllocationPools: [{'start': '172.17.3.10', 'end': '172.17.3.200'}]

  StorageNetworkVlanID: 301



  # Private Storage Access - i.e. Ceph background cluster/replication

  StorageMgmtNetCidr: 172.17.4.0/24

  StorageMgmtAllocationPools: [{'start': '172.17.4.10', 'end': '172.17.4.200'}]

  StorageMgmtNetworkVlanID: 401



  # External Networking Access - Public API Access

  ExternalNetCidr: 192.168.122.0/24

  # Leave room for floating IPs in the External allocation pool (if required)

  ExternalAllocationPools: [{'start': '192.168.122.100', 'end': '192.168.122.129'}]

  # Set to the router gateway on the external network

  ExternalInterfaceDefaultRoute: 192.168.122.1

EOF

~~~



Before assuming that this network configuration is correct, we can run through a **validation** script that checks the syntax and outputs the configuration for us to verify. Let's download this and execute it on our advanced-networking.yaml file:



~~~

undercloud$ cd ~

undercloud$ curl -O https://raw.githubusercontent.com/rthallisey/clapper/master/network-environment-validator.py

(...)



undercloud$ python network-environment-validator.py -n ~/my_templates/advanced-networking.yaml

INFO:__main__:Validating /home/stack/my_templates/nic-configs/controller.yaml

DEBUG:__main__:There are 0 bonds for bridge {'get_input': 'bridge_name'} of resource OsNetConfigImpl in /home/stack/my_templates/nic-configs/controller.yaml

DEBUG:__main__:There are 0 bonds for bridge br-isolated of resource OsNetConfigImpl in /home/stack/my_templates/nic-configs/controller.yaml

INFO:__main__:Validating /home/stack/my_templates/nic-configs/ceph-storage.yaml

DEBUG:__main__:There are 0 bonds for bridge br-isolated of resource OsNetConfigImpl in /home/stack/my_templates/nic-configs/ceph-storage.yaml

INFO:__main__:Validating /home/stack/my_templates/nic-configs/compute.yaml

DEBUG:__main__:There are 0 bonds for bridge br-isolated of resource OsNetConfigImpl in /home/stack/my_templates/nic-configs/compute.yaml

INFO:__main__:Checking allocation pool TenantAllocationPools

INFO:__main__:Checking allocation pool ExternalAllocationPools

INFO:__main__:Checking allocation pool StorageAllocationPools

INFO:__main__:Checking allocation pool StorageMgmtAllocationPools

INFO:__main__:Checking allocation pool InternalApiAllocationPools

INFO:__main__:Checking Vlan ID TenantNetworkVlanID

INFO:__main__:Checking Vlan ID StorageMgmtNetworkVlanID

INFO:__main__:Checking Vlan ID StorageNetworkVlanID

INFO:__main__:Checking Vlan ID InternalApiNetworkVlanID



 ----------SUMMARY----------

SUCCESSFUL Validation with 0 error(s)

~~~



Finally, it's important to understand how OSP director maps **services** to the **networks** that have been created; this all takes place in the **network map** (resource name **ServiceNetMap**):



~~~

undercloud$ grep -A30 ServiceNetMap \

~/my_templates/overcloud-without-mergepy.yaml | head -n30

  ServiceNetMap:

    default:

      NeutronTenantNetwork: tenant

      CeilometerApiNetwork: internal_api

      MongoDbNetwork: internal_api

      CinderApiNetwork: internal_api

      CinderIscsiNetwork: storage

      GlanceApiNetwork: storage

      GlanceRegistryNetwork: internal_api

      KeystoneAdminApiNetwork: internal_api

      KeystonePublicApiNetwork: internal_api

      NeutronApiNetwork: internal_api

      HeatApiNetwork: internal_api

      NovaApiNetwork: internal_api

      NovaMetadataNetwork: internal_api

      NovaVncProxyNetwork: internal_api

      SwiftMgmtNetwork: storage_mgmt

      SwiftProxyNetwork: storage

      HorizonNetwork: internal_api

      MemcachedNetwork: internal_api

      RabbitMqNetwork: internal_api

      RedisNetwork: internal_api

      MysqlNetwork: internal_api

      CephClusterNetwork: storage_mgmt

      CephPublicNetwork: storage

      ControllerHostnameResolveNetwork: internal_api

      ComputeHostnameResolveNetwork: internal_api

      BlockStorageHostnameResolveNetwork: internal_api

      ObjectStorageHostnameResolveNetwork: internal_api

      CephStorageHostnameResolveNetwork: storage

~~~



These could, of course, be overridden inside of your **advanced-networking.yaml** file should you want to adjust which services run on the respective networks that are defined, but for the purposes of the training we'll leave this for now.



We'll **not** use our advanced networking configuration yet, we'll keep this ready for a **later** step.



## Making Additional Changes



OSP director has a number of **hooks** that allow us to feed **extra** configuration into the nodes at boot time, e.g. **scripts**/commands. This is particularly useful if we want to change the configuration of the **overcloud-full** images created by the image build process, for example, inserting **package repositories**, registering the node to **Satellite**, or perhaps updating a configuration file. In this example we'll attach our nodes to the package repositories supplied by the **rhos-release** package.



One of the **easiest** ways to get scripts executed on our overcloud nodes is to **inject** it into the **user data**, so that it gets executed by **cloud-init** at boot time. Each node definition injects user data by calling a resource called **NodeUserData**:



~~~

undercloud$ grep -A16 Controller: \

~/my_templates/puppet/controller-puppet.yaml | head -n16

  Controller:

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



  NodeUserData:

    type: OS::TripleO::NodeUserData

~~~



This resource type (**"OS::TripleO::NodeUserData"**) is mapped in the resource registry as follows:



~~~

undercloud$ grep OS::TripleO::NodeUserData \

~/my_templates/overcloud-resource-registry-puppet.yaml

  OS::TripleO::NodeUserData: firstboot/userdata_default.yaml

~~~



So, by **default**, the file firstboot/userdata_default.yaml gets executed:



~~~

undercloud$ cat ~/my_templates/firstboot/userdata_default.yaml

heat_template_version: 2014-10-16



description: >

  This is a default no-op template which provides empty user-data

  which can be passed to the OS::Nova::Server resources.

  This template can be replaced with a different implementation via

  the resource registry, such that deployers may customise their

  first-boot configuration.



resources:

  userdata:

    type: OS::Heat::MultipartMime



outputs:

  # This means get_resource from the parent template will get the userdata, see:

  # http://docs.openstack.org/developer/heat/template_guide/composition.html#making-your-template-resource-more-transparent

  # Note this is new-for-kilo, an alternative is returning a value then using

  # get_attr in the parent template instead.

  OS::stack_id:

    value: {get_resource: userdata}

~~~



As the comments in the resource definition highlight, this is a **noop** file. It essentially does nothing.



Let's create a new first boot configuration to set our nodes to install the necessary repositories. As before, let's create a new **environment** file to **override** this, and point it towards a **new file** that we're going to create in the next step:



~~~

undercloud$ cat << EOF > ~/my_templates/firstboot-environment.yaml

resource_registry:

  OS::TripleO::NodeUserData: /home/stack/my_templates/firstboot-config.yaml

EOF

~~~



Next, let's create that **firstboot-config.yaml** file that will be called during deployment:



~~~

undercloud$ cat << EOF > ~/my_templates/firstboot-config.yaml

heat_template_version: 2014-10-16



resources:

  userdata:

    type: OS::Heat::MultipartMime

    properties:

      parts:

      - config: {get_resource: repo_config}



  repo_config:

    type: OS::Heat::SoftwareConfig

    properties:

      config: |

        #!/bin/bash

        rpm -ivh http://rhos-release.virt.bos.redhat.com/repos/rhos-release/rhos-release-latest.noarch.rpm

        rhos-release -p 0_Day 7-director

        rhos-release -p A1 7



outputs:

  OS::stack_id:

    value: {get_resource: userdata}

EOF

~~~



We'll just need to specify this environment file when we deploy our advanced overcloud.



## Deployment of our Advanced Overcloud



The time has come to put all of the **modifications** we've made into action and deploy our **advanced** overcloud. To recap, we've made the following changes:



1. We're opting to integrate **Ceph** into our environment for Glance (**image** storage), Cinder (**block** storage), and Nova (instance **ephemeral** storage). We do this by providing OSP director with a **quantity** of Ceph OSD servers, the flavor to match, the override **parameters** required to configure Ceph (via the storage-environment file), and an updated **hieradata** file to advise OSP director of our OSD disks.<br><br>

2. We've overhauled the overcloud networking to make use of **multiple NICs** and **network isolation** for our various networking traffic types. We've created an **advanced-networking.yaml** environment file that overrides the networking resources for our roles, and specifies **parameters** to define our subnets and VLAN's. We've also specified a custom **nic-config** for each role, so that each node gets configured to use these networks.<br><br>

3. We've also included a **firstboot** script via the **userdata hook** - allowing us to run a **script** at first boot of our overcloud nodes; in our case, this sets the nodes to use the **package respositories** provided by **rhos-release**.



Recall that OSP director takes **ownership** of setting a large number of **parameters** that **cannot be overridden** in the current release, and therefore we have to specify them on the command line. The specific parameters that we're concerned with here are the scale numbers, the flavors we want to use, and the overlay networking configuration.



Use the following command to deploy our **advanced** overcloud, referencing the above **parameters**, and linking in the necessary **environment** files that will apply the changes we requested:



~~~

undercloud$ cd /home/stack/

undercloud$ source ~/stackrc

undercloud$ openstack overcloud deploy --templates ~/my_templates/ \

   --ntp-server 10.5.26.10 \

--control-flavor control --compute-flavor compute --ceph-storage-flavor ceph \

--control-scale 3 --compute-scale 1 --ceph-storage-scale 3 \

--neutron-tunnel-types vxlan --neutron-network-type vxlan \

-e ~/my_templates/environments/storage-environment.yaml \

-e ~/my_templates/advanced-networking.yaml \

-e ~/my_templates/firstboot-environment.yaml

Deploying templates in the directory /home/stack/my_templates

(...)

~~~



> **NOTE**: We're specifically deploying with **one compute node** for now, as in a later lab we'll show how to make **deployment changes**, one of the tasks will be to **scale** your compute nodes.



As before, some nodes can occasionally **fail** to boot first time, reporting **DHCP timeout** errors, or no config available for a given MAC. The simple fix for this is to **reboot** the overcloud node in question until they pick up their PXE entries and begin the **deployment** process. So whilst we wait, open up a virt-manager console to your underlying hypervisor via a **new terminal**:



~~~

client$ ssh root@<your host ip> -Y

host# virt-manager &

~~~



Meanwhile, you can usually spot the node(s) that are getting stuck if they display the '**wait callback**' provision state for **more than a couple of minutes**, or seem to have **zero activity** in the virt-manager console:



~~~

undercloud$ ironic node-list

+--------------------------------------+------+--------------------------------------+-------------+-----------------+-------------+

| UUID                                 | Name | Instance UUID                        | Power State | Provision State | Maintenance |

+--------------------------------------+------+--------------------------------------+-------------+-----------------+-------------+

| 1f12b975-90bb-4eb0-a0ff-ee0043b579be | None | 9a51469a-f641-4e34-bf1d-ea05205dd988 | power on    | deploying       | False       |

| 6022491e-f5f9-4b6c-a3b1-a42dbd81e733 | None | 68962198-3231-47da-b800-cda2cf3002a7 | power on    | wait call-back  | False       |

| 7612ecb8-39fa-40fa-a0c3-20cffe84cc6d | None | 3313cafc-7271-417b-9de9-1d960109433d | power on    | deploying       | False       |

| 07d6dfbe-6e43-4eeb-b108-aba473de5202 | None | None                                 | power off   | available       | False       |

| 8f3556e4-0dfd-4450-908c-eae0b60bd4ea | None | a56fff0f-c83e-4830-8b37-3bb809977827 | power on    | deploying       | False       |

| 5df30a00-0653-4f23-8cb5-f4c35aac4cee | None | 40d4c6e4-82ae-4730-b574-a98604ead2e9 | power off   | deploying       | False       |

| 548af01e-5504-448f-8981-af04c76835a9 | None | 3b63f591-69db-468e-b07a-430231bcea27 | power off   | deploying       | False       |

| 4b7f8543-200a-4942-8377-127eac7e26f3 | None | 7fbb99bc-5a77-4bb1-8188-90c04f3f11ea | power on    | deploying       | False       |

+--------------------------------------+------+--------------------------------------+-------------+-----------------+-------------+

~~~



> **NOTE**: When we initially **uploaded** our nodes into Ironic via **instackenv.json**, we did it in the order that they appear in libvirt (typically arranged alphabetically) and therefore if you have a node that has appeared in the **wait callback** state for a few minutes, its order in the output for 'ironic node-list' is the same order it would appear in libvirt. In the example above, the second node has got stuck, which is the **overcloud-ceph2** node. Simply power this down, and back on again until it boots successfully.



> **WARNING**: Do **not** reboot a node unless you're **sure** that it has failed to DHCP/PXE. It's possible that during a build, a node may **appear** to be doing nothing, yet it has successfully **provisioned**. View the console output **before** rebooting.



When the deployment of the **overcloud-full image** has been successful, Ironic should report that all hosts (with the exception of the second compute node) are in the **active** state, and have an instance associated:



~~~

undercloud$ ironic node-list

+--------------------------------------+------+--------------------------------------+-------------+-----------------+-------------+

| UUID                                 | Name | Instance UUID                        | Power State | Provision State | Maintenance |

+--------------------------------------+------+--------------------------------------+-------------+-----------------+-------------+

| 1f12b975-90bb-4eb0-a0ff-ee0043b579be | None | 9a51469a-f641-4e34-bf1d-ea05205dd988 | power on    | active          | False       |

| 6022491e-f5f9-4b6c-a3b1-a42dbd81e733 | None | 68962198-3231-47da-b800-cda2cf3002a7 | power on    | active          | False       |

| 7612ecb8-39fa-40fa-a0c3-20cffe84cc6d | None | 3313cafc-7271-417b-9de9-1d960109433d | power on    | active          | False       |

| 07d6dfbe-6e43-4eeb-b108-aba473de5202 | None | None                                 | power off   | available       | False       |

| 8f3556e4-0dfd-4450-908c-eae0b60bd4ea | None | a56fff0f-c83e-4830-8b37-3bb809977827 | power on    | active          | False       |

| 5df30a00-0653-4f23-8cb5-f4c35aac4cee | None | 40d4c6e4-82ae-4730-b574-a98604ead2e9 | power on    | active          | False       |

| 548af01e-5504-448f-8981-af04c76835a9 | None | 3b63f591-69db-468e-b07a-430231bcea27 | power on    | active          | False       |

| 4b7f8543-200a-4942-8377-127eac7e26f3 | None | 7fbb99bc-5a77-4bb1-8188-90c04f3f11ea | power on    | active          | False       |

+--------------------------------------+------+--------------------------------------+-------------+-----------------+-------------+

~~~



The same status should be reported in Nova, that all of the instances are **'ACTIVE'** and have an IP address on the PXE/management network assigned:



~~~

undercloud$ nova list

+--------------------------------------+-------------------------+--------+------------+-------------+----------------------+

| ID                                   | Name                    | Status | Task State | Power State | Networks             |

+--------------------------------------+-------------------------+--------+------------+-------------+----------------------+

| 9a51469a-f641-4e34-bf1d-ea05205dd988 | overcloud-cephstorage-0 | ACTIVE | -          | Running     | ctlplane=172.16.0.66 |

| 3313cafc-7271-417b-9de9-1d960109433d | overcloud-cephstorage-1 | ACTIVE | -          | Running     | ctlplane=172.16.0.65 |

| 68962198-3231-47da-b800-cda2cf3002a7 | overcloud-cephstorage-2 | ACTIVE | -          | Running     | ctlplane=172.16.0.67 |

| a56fff0f-c83e-4830-8b37-3bb809977827 | overcloud-compute-0     | ACTIVE | -          | Running     | ctlplane=172.16.0.69 |

| 40d4c6e4-82ae-4730-b574-a98604ead2e9 | overcloud-controller-0  | ACTIVE | -          | Running     | ctlplane=172.16.0.72 |

| 7fbb99bc-5a77-4bb1-8188-90c04f3f11ea | overcloud-controller-1  | ACTIVE | -          | Running     | ctlplane=172.16.0.70 |

| 3b63f591-69db-468e-b07a-430231bcea27 | overcloud-controller-2  | ACTIVE | -          | Running     | ctlplane=172.16.0.71 |

+--------------------------------------+-------------------------+--------+------------+-------------+----------------------+

~~~



The deployment should now proceed further with the Heat SoftwareDeployment **steps**, taking perhaps **20-30 minutes**, eventually printing:



~~~

(...)

PKI initialization in init-keystone is deprecated and will be removed.

Warning: Permanently added '192.168.122.100' (ECDSA) to the list of known hosts.

Connection to 192.168.122.100 closed.

Overcloud Endpoint: http://192.168.122.100:5000/v2.0/

Overcloud Deployed

~~~



If you have any **problems** with deployment and it reports that there has been a **failure**, discovering the problem can be found by calling the following:



~~~

undercloud$ heat resource-list -n5 overcloud | grep -i fail

(...)

~~~



## Outcome of this Lab



By completing this lab you should have achieved the following objectives:



* A clear understanding of how to **modify** the OSP director templates.

* An appreciation for the **range** of modifications that can be achieved.

* A new, **advanced** overcloud running with example modifications for **networking** and **storage**.


## Next Lab

The next lab will be dedicated to explaing how to make **post-deployment changes** to an already running overcloud platform, e.g. **configuration** changes, package updates, and **scaling** the environment, click [here][lab10](./lab10.md) to proceed.

