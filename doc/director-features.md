# OSP director 功能 & 路线图

## 序言

## 下一章

这一节概述，并详细介绍了当前的 OSP 主任 * * 功能 * * 时的第一次一般可用性 (GA) 与 * * RHEL OSP 7.0 (Kilo) * *，它也进入了当前已知的限制以及如何减轻对他们的一些。我们也会进入某些路线图项添加额外的功能。请注意，路线图是不断在基于 **客户要求** 而改变，所以请您将 **建议** 和 **请求**反馈到 [PM 团队] (virt-field-pm@redhat.com) 。

正如所有的 Red Hat 技术，是有相对应的 **上游** 项目，"**RDO 经理**"我们专注于 **上游第一** 发展模式;所有功能都推向上游 RDO 经理和各自的上游 OpenStack 存储库拉下来之前我们 **下游** RHEL OSP 主任提供。详细信息可以在这里找到:

* [Project Overview] (https://www.rdoproject.org/RDO-Manager)
* [Git Repository] (https://www.github.com/rdo-management)
* 如果你想要 [贡献] (https://repos.fedorapeople.org/repos/openstack-m/docs/master/contributions/contributions.html#contributing-code) ，我们欢迎您的参与！<br><br>

## OSP director 工作流程

一般 **流** OSP 导演时部署与 **虚拟硬件** 是，如下所示:

<center>
<img src="./images/osp-director-workflow.png"/>
</center>

## 一般可用性功能

这个 enablement 的 OSP 主任在 GA RHEL OSP 7.0 状态附近集中，是 **第一** 迭代的工具。与 *快速* 上游释放节奏的底层核心 OpenStack 组件，我们已经 **优先** 某些功能进入遗传算法与其他项目留下的路线图中 **每2月的发放节奏** 的 OSP 主任。GA 释放的共同主题已经 **一致性** 我们现有的部署与 **最佳-做法**;因此，OSP 导演的部署环境，应为 **相同** 的 RHEL OSP 安装程序部署环境 (**Staypuft**)。

下一章我们开始 **hands-on labs**; we'll go through a lot of detail in the content and by putting the theory into practice. Let's ensure we have adequate connectivity to the dedicated lab that's been set up, click [here][labs-setup](./labs-setup.md) to proceed.
