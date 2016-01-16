# Lab 10: Making Post-Deployment Changes

## Introduction

In the previous lab we deployed our **advanced** overcloud, making use of **Ceph** integration, advanced networking with VLAN **isolation**, and we deployed some additional configuration via the **firstboot hook**. OSP director has been designed to be an ongoing management tool for a RHEL OSP installation; enabling you to do a lot more than just **deploy** RHEL OSP.



The tools have been designed to help operators **scale**, **monitor**, **update**, **upgrade**, and **automate** a RHEL OSP environment, and whilst a lot of these functions are **evolving** and becoming more **comprehensive**, a lot of the underlying capabilities are ready for us to explore. One of the main tasks that an operator will no doubt want to carry out is making **post-deployment** changes, **without** needing to **re-deploy**.



This **tenth** lab will walk you through some of the **post-deployment** tasks that can be completed using OSP director; we're going to be carrying out the following in this lab, with an estimated completion time of **45 minutes**:



* **Scaling** our existing advanced overcloud to make use of our **second** compute node

* Changing a set of **configuration** options on the overcloud

* Running a remote **package update** call to our nodes



> **NOTE**: Starting of this lab assumes that you've completed the [ninth lab][lab9] and have a **successfully deployed** advanced overcloud environment. If this is not the case, please revisit the previous lab.



## Lab Focus



This lab will focus on the **undercloud** VM:



<center>

<img src="images/osp-director-env-6.png">

</center>



## Scaling your Environment



In the [previous lab][lab9] we deployed our advanced overcloud, using **three controllers**, **three Ceph** nodes, but we utilised a **single compute** node. This obviously left a defined compute node **unused**. The purpose of this section of the lab is to invoke that **second** compute node into the cluster without requiring a full redeployment; one of the main benefits of using **Heat** is that it calculates the **differences** between what has already been deployed, and what it needs to do to match the new requirements in the form of a **stack-update**.



We've already got our second compute node defined in **Ironic**, has a role/**profile** assigned to it from our introspection run, and is ready to go. However, when customers do want to introduce **new** and **unknown** equipment into the environment they'll have to **import** the nodes and run **introspection** before they can be used for scaling.



When we want to **scale** our environment we need only update the **quantity** of a given role in the overcloud-deploy command. The CLI tooling is **intelligent** enough to understand if an overcloud has already been **deployed** and when to just make an **update** rather than a redeployment. We can use the following command to adjust our compute count:



~~~

undercloud$ cd /home/stack/

undercloud$ source ~/stackrc

undercloud$ openstack overcloud deploy --templates ~/my_templates/ \

--ntp-server 10.5.26.10 \

--control-flavor control --compute-flavor compute --ceph-storage-flavor ceph \

--control-scale 3 --compute-scale 2 --ceph-storage-scale 3 \

--neutron-tunnel-types vxlan --neutron-network-type vxlan \

-e ~/my_templates/environments/storage-environment.yaml \

-e ~/my_templates/advanced-networking.yaml \

-e ~/my_templates/firstboot-environment.yaml

Deploying templates in the directory /home/stack/my_templates

(...)

~~~



> **NOTE**: The only difference between **this** command and the command executed in the **previous lab** is the **"--compute-scale"** count. We have to specify all of the environment files **and** the parameters from the previous run, otherwise OSP director will calculate the differences and adjust the stack accordingly - often leaving you with **undesirable** results.



This **stack-update** should only take 10-minutes or so, as the second compute node is provisioned in the exact same manor as the initial deployment. During its operation you can watch the **Heat** output in **another** console:



~~~

undercloud$ heat stack-list

+--------------------------------------+------------+--------------------+----------------------+

| id                                   | stack_name | stack_status       | creation_time        |

+--------------------------------------+------------+--------------------+----------------------+

| d4538cc4-5704-461e-876e-489f535602f4 | overcloud  | UPDATE_IN_PROGRESS | 2015-08-15T14:26:12Z |

+--------------------------------------+------------+--------------------+----------------------+

~~~



Also, verify that your **remaining** node is in a deploying or active state, noting that this may take a few minutes to update:



~~~

undercloud$ ironic node-list

+--------------------------------------+------+--------------------------------------+-------------+-----------------+-------------+

| UUID                                 | Name | Instance UUID                        | Power State | Provision State | Maintenance |

+--------------------------------------+------+--------------------------------------+-------------+-----------------+-------------+

| 1f12b975-90bb-4eb0-a0ff-ee0043b579be | None | e3f97d11-626a-4d0a-ae7a-9524e6bdd95b | power on    | active          | False       |

| 6022491e-f5f9-4b6c-a3b1-a42dbd81e733 | None | e569d3a6-9efe-432d-bf8c-a72b0672d47a | power on    | active          | False       |

| 7612ecb8-39fa-40fa-a0c3-20cffe84cc6d | None | c7b6bc03-34bb-4804-886f-a176dfb49bb3 | power on    | active          | False       |

| 07d6dfbe-6e43-4eeb-b108-aba473de5202 | None | 372f3329-e228-4756-85c3-f9b9bede8152 | power on    | active          | False       |

| 8f3556e4-0dfd-4450-908c-eae0b60bd4ea | None | 598ed1c0-3853-48bb-a781-029aed82e8c6 | power on    | deploying       | False       |

| 5df30a00-0653-4f23-8cb5-f4c35aac4cee | None | fb502ab1-dd5a-46da-8555-692b90a4a703 | power on    | active          | False       |

| 548af01e-5504-448f-8981-af04c76835a9 | None | a8b3b375-285f-4ee7-8067-1118fe4de789 | power on    | active          | False       |

| 4b7f8543-200a-4942-8377-127eac7e26f3 | None | d4314abb-9ad8-4e17-b55b-6a1c600e2a50 | power on    | active          | False       |

+--------------------------------------+------+--------------------------------------+-------------+-----------------+-------------+

~~~



Once this is **complete**, OSP director should advise you that the deployment has been **successful**:



~~~

(...)

Deploying templates in the directory /home/stack/my_templates

Overcloud Endpoint: http://192.168.122.100:5000/v2.0/

Overcloud Deployed

~~~



Now we can check that we have **two** compute nodes listed by Nova:



~~~

undercloud$ nova list

+--------------------------------------+-------------------------+--------+------------+-------------+----------------------+

| ID                                   | Name                    | Status | Task State | Power State | Networks             |

+--------------------------------------+-------------------------+--------+------------+-------------+----------------------+

| e3f97d11-626a-4d0a-ae7a-9524e6bdd95b | overcloud-cephstorage-0 | ACTIVE | -          | Running     | ctlplane=172.16.0.75 |

| c7b6bc03-34bb-4804-886f-a176dfb49bb3 | overcloud-cephstorage-1 | ACTIVE | -          | Running     | ctlplane=172.16.0.81 |

| e569d3a6-9efe-432d-bf8c-a72b0672d47a | overcloud-cephstorage-2 | ACTIVE | -          | Running     | ctlplane=172.16.0.76 |

| 372f3329-e228-4756-85c3-f9b9bede8152 | overcloud-compute-0     | ACTIVE | -          | Running     | ctlplane=172.16.0.77 |

| 598ed1c0-3853-48bb-a781-029aed82e8c6 | overcloud-compute-1     | ACTIVE | -          | Running     | ctlplane=172.16.0.82 |

| fb502ab1-dd5a-46da-8555-692b90a4a703 | overcloud-controller-0  | ACTIVE | -          | Running     | ctlplane=172.16.0.80 |

| d4314abb-9ad8-4e17-b55b-6a1c600e2a50 | overcloud-controller-1  | ACTIVE | -          | Running     | ctlplane=172.16.0.78 |

| a8b3b375-285f-4ee7-8067-1118fe4de789 | overcloud-controller-2  | ACTIVE | -          | Running     | ctlplane=172.16.0.79 |

+--------------------------------------+-------------------------+--------+------------+-------------+----------------------+

~~~



> **NOTE**: It is not currently possible to scale the number of **controller** nodes; **only** compute and storage nodes can be increased. It is also **not** possible to **safely** scale **down** the environment without major **disruption** and **downtime**.



## Making Post-Deployment Configuration Changes



In the previous lab we provided an example of how to **configure** the node at boot time, known as the **firstboot hook**, e.g. running a **script** to set up **package** access. This is a nice feature to have, but what if the deployment is already running? There are **three** major post-deployment change types that we'll look into here:



1. Making **configuration** changes to RHEL OSP.<br><br>

2. Running **scripts** to do anything you like, on deployed nodes.<br><br>

3. **Updating** or **upgrading** RHEL OSP and underlying packages on deployed nodes.



With OSP director, we can do all three of these. For (**1**) we can modify the **hieradata** in our templates directory and **"redeploy"** the overcloud, noting that we wouldn't be carrying out a full redeployment, OSP director (via Heat) would calculate the **differences** and only apply the these where necessary.



Before doing this the **proper** way, let's make a **manual** change to one of the compute nodes and see what happens after we apply the change via **OSP director**:



> **NOTE**: The point here is to show that manual changes will be **overwritten** by OSP director during the **puppet-based** software deployment stages, this is **not** a good practice. Only make this temporary change on **one** compute node.



Firstly, list your available compute nodes:



~~~

undercloud$ nova list --fields name,networks | grep compute

| 344d56e2-46a3-4887-ae06-dc3d86656d25 | overcloud-compute-0 | ctlplane=172.16.0.37 |

| 7545695f-fa15-470a-b631-929cbf286229 | overcloud-compute-1 | ctlplane=172.16.0.40 |

~~~



Select **one** of these nodes to make the temporary manual change on:



~~~

undercloud$ ssh heat-admin@172.16.37

overcloud-compute-0$ sudo -i

overcloud-compute-0# grep ram_allocation_ratio /etc/nova/nova.conf

#ram_allocation_ratio=1.5

overcloud-compute-0# openstack-config --set /etc/nova/nova.conf DEFAULT ram_allocation_ratio 3.0

overcloud-compute-0# openstack-config --get /etc/nova/nova.conf DEFAULT ram_allocation_ratio

3.0

overcloud-compute-0# openstack-service restart openstack-nova-compute

overcloud-compute-0# exit

overcloud-compute-0$ exit

~~~



Now let's carry out a configuration change to RHEL OSP for our example for (**1**); let's modify some Nova compute configuration settings via hieradata, noting that we use **two** modules here, one **nova::config** which allows us to set arbitrary data, and one module **nova::compute** which explicitly manages the *reserved\_host\_memory* parameter (so we cannot list this in *nova::config*):



~~~

undercloud$ cat << EOF >> ~/my_templates/puppet/hieradata/compute.yaml



# Increase the memory overcommit ratio from 1.5 -> 2.0

nova::config::nova_config:

  DEFAULT/ram_allocation_ratio:

    value: 2.0



# Keep 1G memory reserved for the host

nova::compute::reserved_host_memory: 1024

EOF

~~~



> **NOTE**: These changes will only be reflected on the compute nodes as we've modified the **compute.yaml** file.



To execute these changes, simply re-run the deployment command we used earlier:



~~~

undercloud$ openstack overcloud deploy --templates ~/my_templates/ \

--ntp-server 10.5.26.10 \

--control-flavor control --compute-flavor compute --ceph-storage-flavor ceph \

--control-scale 3 --compute-scale 2 --ceph-storage-scale 3 \

--neutron-tunnel-types vxlan --neutron-network-type vxlan \

-e ~/my_templates/environments/storage-environment.yaml \

-e ~/my_templates/advanced-networking.yaml \

-e ~/my_templates/firstboot-environment.yaml

Deploying templates in the directory /home/stack/my_templates

(...)

~~~



As the vast **majority** of components are already **deployed**, and the changes to the stack are **small**, this shouldn't take more than **15 minutes** to complete. Next, let's give an example of how we can run post-deployment scripts via Heat, as described in (**2**) above.



Just like the **firstboot** hook, we can have Heat automatically instruct our nodes to execute a script via the **NodeExtraConfigPost** hook. This is a special resource that adds an extra SoftwareDeploy step in once all other resources are deployed.



Verify the change was made on both compute nodes and note that it overwrote the manual change made earlier:



~~~

undercloud$ ssh heat-admin@172.16.37

overcloud-compute-0$ sudo openstack-config --get /etc/nova/nova.conf DEFAULT ram_allocation_ratio

2.0

undercloud$ ssh heat-admin@172.16.40

overcloud-compute-1$ sudo openstack-config --get /etc/nova/nova.conf DEFAULT ram_allocation_ratio

2.0

overcloud-compute-1$ exit

~~~



Each role has a post-deployment resource listed in the top-level overcloud stack; if we look at the **compute** node for example:



~~~

undercloud$ grep -A5 ComputeNodesPostDeployment \

~/my_templates/overcloud-without-mergepy.yaml

  ComputeNodesPostDeployment:

    type: OS::TripleO::ComputePostDeployment

    depends_on: [ComputeAllNodesDeployment, ComputeCephDeployment]

    properties:

      servers: {get_attr: [Compute, attributes, nova_server_resource]}

      NodeConfigIdentifiers: {get_attr: [Compute, attributes, config_identifier]}

~~~



The resource type is "**OS::TripleO::ComputePostDeployment**", which is mapped to another resource definition in the resource registry:



~~~

undercloud$ grep OS::TripleO::ComputePostDeployment \

~/my_templates/overcloud-resource-registry-puppet.yaml

   OS::TripleO::ComputePostDeployment: puppet/compute-post-puppet.yaml

~~~



This file calls another resource called **NodeExtraConfigPost**:



~~~

undercloud$ grep -A5 ExtraConfig ~/my_templates/puppet/compute-post-puppet.yaml

  ExtraConfig:

    depends_on: ComputePuppetDeployment

    type: OS::TripleO::NodeExtraConfigPost

    properties:

        servers: {get_param: servers}

~~~



This itself is mapped in the **resource registry** to the exact file we can either **modify**, or **override**:



~~~

undercloud$ grep OS::TripleO::NodeExtraConfigPost \

~/my_templates/overcloud-resource-registry-puppet.yaml

OS::TripleO::NodeExtraConfigPost: extraconfig/post_deploy/default.yaml

~~~



Let's feed a **script** into our environment; as before, let's **override** this resource registry entry with our **own** file:



~~~

undercloud$ cat << EOF > ~/my_templates/post-deployment.yaml

resource_registry:

  OS::TripleO::NodeExtraConfigPost: /home/stack/my_templates/node-setup.yaml

EOF

~~~



Now let's populate that file with our instructions. Let's ensure that our post-deploy script carries out the following:



* Installs a set of useful **packages** and **utilities** for our overcloud nodes

* Enables **root login** over SSH

* Sets a custom **root password** for us to use

* Deploys a **system identifier** httpd server



First we're going to need to specify the resource definition:



~~~

undercloud$ cat << EOF > ~/my_templates/node-setup.yaml

heat_template_version: 2014-10-16



description: >

  Example file to install a HTTP server on each node



parameters:

  servers:

    type: json



resources:



  ExtraConfig:

    type: OS::Heat::SoftwareConfig

    properties:

      group: script

      config: {get_file: /home/stack/my_templates/node-setup.sh}





  ExtraDeployments:

    type: OS::Heat::SoftwareDeployments

    properties:

      servers:  {get_param: servers}

      config: {get_resource: ExtraConfig}

      actions: ['CREATE', 'UPDATE']

EOF

~~~



If you notice, the **group** is **script** - so the **os-refresh-config** tool on the nodes understands what to do with the **metadata** provided by Heat - in this case it simply executes it as a **script** rather than, for example, pushing it into **puppet** for execution. The resource definition requires an extra script; this is where we'll put our bash script that carries out our required configuration:



~~~

undercloud$ cat << EOF > ~/my_templates/node-setup.sh

#!/bin/bash



# Install some packages

yum install tcpdump wget strace screen ftp mlocate -y



# Update the locate database

updatedb



# Permit root login over SSH

sed -i 's/.*ssh-rsa/ssh-rsa/' /root/.ssh/authorized_keys

sed -i 's/PasswordAuthentication.*/PasswordAuthentication yes/g' /etc/ssh/sshd_config

sed -i 's/ChallengeResponseAuthentication.*/ChallengeResponseAuthentication yes/g' /etc/ssh/sshd_config



systemctl restart sshd



# Update the root password to something we know

echo redhat | sudo passwd root --stdin



# Configure a system identifier httpd virtual host

yum install httpd -y

mkdir -p /var/www/ident/



cat << FOE > /etc/httpd/conf.d/ident.conf

Listen 8088

NameVirtualHost *:8088



<VirtualHost *:8088>

DocumentRoot /var/www/ident/

</VirtualHost>

FOE



restorecon /etc/httpd/conf.d/ident.conf



cat << FOE > /var/www/ident/index.html

Hello from \$(hostname) :-)

FOE



restorecon -R /var/www/ident/

chown -R apache:apache /var/www/ident/



iptables -A INPUT -p tcp -m tcp --dport 8088 -j ACCEPT

semanage port -a -t http_port_t -p tcp 8088

systemctl start httpd

systemctl reload httpd

systemctl enable httpd



EOF

~~~



> **NOTE**: These are just **examples** of what you can put in here - customers will likely want to use this to run one off scripts when required.



Now we simply **re-run** our OSP director deployment, specifying our new **environment file** (make sure you include all of the **previous** environment files and parameters also):



~~~

undercloud$ cd /home/stack

undercloud$ source ~/stackrc

undercloud$ openstack overcloud deploy --templates ~/my_templates/ \

--ntp-server 10.5.26.10 \

--control-flavor control --compute-flavor compute --ceph-storage-flavor ceph \

--control-scale 3 --compute-scale 2 --ceph-storage-scale 3 \

--neutron-tunnel-types vxlan --neutron-network-type vxlan \

-e ~/my_templates/environments/storage-environment.yaml \

-e ~/my_templates/advanced-networking.yaml \

-e ~/my_templates/firstboot-environment.yaml \

-e ~/my_templates/post-deployment.yaml

Deploying templates in the directory /home/stack/my_templates

(...)

~~~



Once this has completed **successfully**, you can run a quick **test** to see if the script has **successfully executed**:



~~~

undercloud$ compute_ip=$(nova list | grep compute-0 |\

awk '{print $12;}' | cut -d "=" -f2)



undercloud$ curl http://$compute_ip:8088

Hello from overcloud-compute-0.localdomain :-)

~~~



You should now be able to also **login** to your nodes directly as **root**:



~~~

undercloud$ ssh root@$compute_ip

[root@overcloud-compute-0 ~]# openstack-service status

neutron-openvswitch-agent (pid 16935) is active

openstack-ceilometer-compute (pid 23010) is active

openstack-nova-compute (pid 23034) is active

[root@overcloud-compute-0 ~]# exit

logout

Connection to 172.16.0.77 closed.

~~~



> **NOTE**: Now that we've executed this **post-deployment script**, we do **not** need to specify the related **environment** file again during deployment updates, unless the script is **explicitly** required to be executed **again**.



## Updating your Nodes



Continuing on with the theme of **operational** tooling, one significant task that we briefly talked about in the previous section is **updates** and **upgrades**. Currently, OSP director has **very basic** support for updates and upgrades. Typically we differentiate the two as follows-



* **Updates** - Minor bug fixes, security fixes, and feature enhancements **during** the lifecycle of a major release, e.g. RHEL OSP 7.0 (**Kilo**). These are shipped in the existing package channels as **synchronous** and **asynchronous** packages with associated **errata**, and can be deployed on-top of existing packages with very minimal (if any) disruption.<br><br>

* **Upgrades** - This is the process of moving between one **major** release of RHEL OSP to the next, e.g. RHEL OSP 7.0 (**Kilo**) to RHEL OSP 8.0 (**Liberty**). Packages are typically shipped in an entirely **new** package channel and often require **downtime** and **disruption** to services to complete the deployment-wide upgrade.



OSP director has been designed to accomodate both scenarios, with the primary focus for now being on the automation of **updates**. The long term strategy is to automate the entire **upgrade** process, although this is a very **complex** operation to complete, especially if the expectation of the customer is to **minimise** disruption. OSP director will only support **upgrades** at the launch of RHEL OSP 8.0, and therefore is **not** something we can test.



What we can test, is the mechanism it uses for rolling out package **updates** to nodes in the overcloud. In a **distributed** platform such as OpenStack, updating all systems simultaneously can have undesirable side effects, e.g. services being restarted during the **RPM** post-script, and therefore OSP director uses **breakpoints** to stage the rollout of package updates on a **node-by-node** basis. This ensures that packages are rolled out across the deployment, but not in a way that would take down the **entire** cluster. As we've deployed a **highly-available** RHEL OSP environment, other nodes in the cluster can take over the responsibility for service availability in the event of an **outage**.



The method it uses is to simply call a "**yum update -y**" - it's not selective, it's an approach to updating **all nodes** with **all available packages**. Many customers will want to have selective updates, e.g. based on the **severity** of the patch, but our response to this should be for them to use **Satellite**. In the future it will likely be possible for us to be more **selective** via OSP director though. So, **how** does it do this?



In the **same** way that we provided the stack with some **ExtraConfig** in the last section to run a script on all nodes, OSP director **wraps** the execution of a dedicated **SoftwareDeploy** via **ExtraConfig** when we pass a specific call into the OSP director CLI. The resource definition that it uses can be found here:



~~~

undercloud$ cat ~/my_templates/extraconfig/tasks/yum_update.yaml

heat_template_version: 2014-10-16



description: >

  Software-config for performing package updates using yum



resources:



  config:

    type: OS::Heat::SoftwareConfig

    properties:

      group: script

      config: {get_file: yum_update.sh}

      inputs:

      - name: update_identifier

        description: yum will only run for previously unused values of update_identifier

        default: ''

      - name: command

        description: yum sub-command to run, defaults to "update"

        default: update

      - name: command_arguments

        description: yum command arguments, defaults to ""

        default: ''



outputs:

  OS::stack_id:

    value: {get_resource: config}

~~~



It submits a script (**yum_update.sh**) to each node via the Heat metadata. Inside of this script you'll find that it does just call a full '**yum -y update**':



~~~

undercloud$ grep -A2 command= ~/my_templates/extraconfig/tasks/yum_update.sh

command=${command:-update}

full_command="yum -y $command $command_arguments"

echo "Running: $full_command"

~~~



The difference here is that OSP director implements **breakpoints**, so that the **operator** has full **control** over the roll out of the deployment; in this scenario the operator is expected to **validate** the package **updates** on a **per-node** basis, and advise OSP director when it should **continue** on with the next system to update. The environment file that OSP director calls to enable breakpoints is here:



~~~

undercloud$ cat ~/my_templates/environments/overcloud-steps.yaml

# Specifies hooks/breakpoints where overcloud deployment should stop

# Allows operator validation between steps, and/or more granular control.

# Note: the wildcards relate to naming convention for some resource suffixes,

# e.g see puppet/*-post-puppet.yaml, enabling this will mean we wait for

# a user signal on every *Deployment_StepN resource defined in those files.

resource_registry:

  resources:

    "*NodesPostDeployment":

      "*Deployment_Step*":

          hooks: [pre-create, pre-update]

~~~



As it explains above; for every deployment step, Heat puts in a **breakpoint** and waits for the **user signal** to **remove** the breakpoint before proceeding with the next step, i.e. the next node. OSP director **wraps** this process for us so that we do not have to **manually** update Heat.



Let's walk through the **update** procedure with our existing deployment. The **syntax** for the command is slightly different; you **don't** need to specify the **CLI parameters** but you do need to remember to specify all of the **environment** files that you provisioned your environment with up to this point. Recall that we do **NOT** need to specify our "**post-deployment.yaml**" environment file from the last section as the script it calls has been successfully executed already - we do not need for it to run again:



~~~

undercloud$ cd /home/stack

undercloud$ source ~/stackrc

undercloud$ openstack overcloud update stack -i overcloud \

--templates ~/my_templates/ \

-e ~/my_templates/environments/storage-environment.yaml \

-e ~/my_templates/advanced-networking.yaml \

-e ~/my_templates/firstboot-environment.yaml



starting package update on stack overcloud

IN_PROGRESS

IN_PROGRESS

IN_PROGRESS

IN_PROGRESS

IN_PROGRESS

WAITING

not_started: [u'overcloud-controller-2', u'overcloud-compute-0', u'overcloud-controller-0', u'overcloud-cephstorage-1', u'overcloud-compute-1', u'overcloud-cephstorage-0', u'overcloud-controller-1']

on_breakpoint: [u'overcloud-cephstorage-2']

Breakpoint reached, continue? Regexp or Enter=proceed, no=cancel update, C-c=quit interactive mode:

removing breakpoint on overcloud-cephstorage-2

IN_PROGRESS

(...)

~~~



You'll need to watch this happen so you can see how the **breakpoints** work, and allow the process to **continue** with the next node under **your** control. Simply press **return** to move onto the next node.



> **NOTE**: OSP director assumes that **package access** has been taken care of **prior** to executing this command, and takes **no responsibility** for ensuring this. In a previous lab we established package repositories on all of our systems so this **shouldn't** be a problem in our demonstration environment.



Eventually, the update on all systems should be **completed**:



~~~

(...)

COMPLETE

update finished with status COMPLETE



undercloud$ heat stack-list

heat stack-list

+--------------------------------------+------------+-----------------+----------------------+

| id                                   | stack_name | stack_status    | creation_time        |

+--------------------------------------+------------+-----------------+----------------------+

| d4538cc4-5704-461e-876e-489f535602f4 | overcloud  | UPDATE_COMPLETE | 2015-08-15T14:26:12Z |

+--------------------------------------+------------+-----------------+----------------------+

~~~



> **NOTE**: Occasionally, the update process will report an ERROR when it has infact completed successfully. Ensure that the output from '**heat stack-list**' shows "**UPDATE_COMPLETE**" after a few minutes of the command finishing. If it does not, you may want to repeat the update process.



You can **verify** the successful package update process by logging into one of your systems and ensuring there are **no updates available**. In this example we'll use our first **compute** node, but you can **replace** this with some of the other nodes if you'd prefer:



~~~

undercloud$ compute_ip=$(nova list | grep compute-0 |\

awk '{print $12;}' | cut -d "=" -f2)



undercloud$ ssh root@$compute_ip yum list updates | wc -l

0

~~~



> **NOTE**: If the number returned here is **greater than zero**, the update did **not** work properly.



## Outcome of this Lab



By completing this lab you should have achieved the following objectives:



* An understanding of how you can **scale** your RHEL OSP deployment with OSP director

* An understanding of how you can make post-deployment **configuration** changes

* An understanding of how OSP director can be used to **automate** the package **updates**



## Next Lab

The next lab will be dedicated to thoroughly **testing** the advanced overcloud deployment, ensuring that everything **works** as expected, including service failover, click [here][lab11](./lab11.md) to proceed.
