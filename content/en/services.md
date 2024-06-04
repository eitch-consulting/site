---
title: Services
date: 2024-05-27T15:02:17-03:00
draft: false
description: Services we provide
---

Check here exactly what we do and how we can solve your problem! If you need something that is related to Linux and DevOps and don't see here, consider <a href="/contact/">contacting us</a> so we can see if we know it or could indicate someone else to help you!

## Index

- <a href="#linux_troubleshooting">Linux troubleshooting</a>
- <a href="#iac">Infrastructure as code</a>
- <a href="#conf_management">Configuration management</a>
- <a href="#monitoring_tool">Monitoring tools</a>
- <a href="#cicd">Continuous integration and delivery</a>
- <a href="#developer_backends">Developer backends and workflows</a>
- <a href="#containers">Containers</a>
- <a href="#public_cloud">Public cloud</a>
- <a href="#vm_management">VM management</a>
- <a href="#specific_applications">Specific applications</a>

---

<a name="linux_troubleshooting"></a>
<br/>
## Linux troubleshooting

If you want recommendations on what Linux to use, which distributions, how many resources, how to scale processes,
how to install any application, or any information about Linux systems, we work with specifically with Linux systems
for more than 25 years.

We can also troubleshoot your application: we can identify errors on processes, memory contraints, performance
issues with network, I/O or CPU usage, in various types of Linux distributions.

<div style="text-align: right"><a href="#">top</a></div>

---

<a name="iac"></a>
<br/>
## Infrastructure as code

Software we most use: terraform, opentofu, terragrunt, atlantis.

Not having the infrastructure as code is practical at the beginning, but when you begin to scale, reproducing
your current setup can be a pain. Having your infrastructure as code can help you create any infrastructure
from scratch without having to remember those details and following horrenduous big check-lists.

Some example use-cases:

- You are growing and need to rapidly create resources, but don't have any audit or infrastructure as code: we
  can study your infrastructure and convert into a code repository ready for use. We'll also import your current
  configuration and you can always use this repository as the bases for every resource deployment. We usually use
  terraform or opentofu for this.

- You currently have an enormous infrastructure and the terraform code is a bit messy. Your state files are so big
  that any plan/apply operation takes 10+ minutes. We can solve this by transforming your terraform repository into
  segregated modules and state directories using terragrunt.

- You need to easily deploy resources by many team members, even those who are not infrastructure administrators. But
  these deployments should be auditted easily and some permissions must be made to, i.e., not destroy resources.
  We can transform your infrastructure into code and use continuous delivery to deploy and log/audit these changes.

<div style="text-align: right"><a href="#">top</a></div>

---

<a name="conf_management"></a>
<br/>
## Configuration management

Software we most use: Chef, Ansible, PyInfra, Packer.

Using configuration management is a must if you have lots of virtual machines. Configuration management acts as an
infrastructure as code, but managing the configuration of a Linux system, instead of the server hardware itself. This
way you can automate and audit creation of system image files, distributions, and is a must to deploy applications
outside a container environment.

Some example use-cases:

- There's currently dozens, hundreds or thousands of virtual machines running Linux, and everytime I have to change
  any configuration, you must do this on all servers with some wizardry and scripts, and each server will have its
  own quirks depending on the environment (production, staging, development...), type of hardware, type of application
  and so on... This can be solved by configuration management, where there will be always an index of servers and what
  they are, so you can track and change configuration exactly where it's needed, with idempotency guaranteed. This is
  done using softwares like `Chef`, `Ansible`, `PyInfra`, and others.

- You need to generate system image files with pre-defined software installed and configured, without having to
  manually run a server, do the configuration, shutdown and then create the image from it. With the system configuration
  as code, you can use tools like `packer` to automate the image creation.

- Your application doesn't have a good way to configure automatically based on an environment, server type, application
  version, and maybe different secrets for each one. Configuration management could write the application configuration
  automatically based on those variables, and also grab secrets from a trusted store, keeping them out of git
  repositories (which is both common and a security nightmare).

<div style="text-align: right"><a href="#">top</a></div>

---

<a name="monitoring_tool"></a>
<br/>
## Monitoring tools

Software we most use: Prometheus, Zabbix, Grafana, AlertManager.

Monitoring allows you to know what is happening with everything. There's many monitoring layers to watch out for, and 
we specialize in tools that do this in the infrastructure layer, like monitoring basic resources: memory, cpu, disk usage,
processes and other application metrics. But more importantly, there's also the need to have proper alarms and 
integration with operators and developers to see when something goes wrong.

Some example use-cases:

- You have lots of metrics for everything: system resources, application metrics, response times, and so on, but you
  don't have good alarms and integrations to make use of it. We can help you identify what is important and what is not
  (we hate having *"normal alarms"*) and how to integrate with external services that sends notifications.

- You want to have an automated way to register applications and metrics in a distribuited monitoring system, without
  having to manually configure endpoints or anything like that. For this, a combination of configuration management and
  dynamic monitoring systems like `prometheus` could be a great solution. This is valid for bare-metal servers, virtual
  machines (on premises or on cloud) and containers.

- You want to have automated ways to do things on your monitoring stack: adding applications, importing alarms from one
  place to another, recreating one environment from scratch, and so on. We can help by developing software that
  integrates with your monitoring API and do everything you want (as the API permits it). 

- You want some fancy dashboards for your metrics and alarms, and we can help you by doing this on Grafana.

<div style="text-align: right"><a href="#">top</a></div>

---

<a name="cicd"></a>
<br/>
## Continuous integration and delivery

Software we most use: Github Actions, Gitlab-CI, Jenkins, RunDeck.
<a href="#conf_management">configuration management</a>.

Having an unified way for developers to checkout, run tests, integrate with other services, deploy resources
and applications is a must on teams that have many people and/or many applications. Automating and documenting
this developer workflow also helps a lot with onboarding and debugging between different developer teams.

Some example use-cases:

- Create a workflow for an application: check out code, run a lint and check for code style definitions, run
  tests, run security checks for dependencies and code libraries, deploy into an staging environment and
  ask for confirmation for deploying into production.

- Sometimes a remote workflow on a provider (github, gitlab, etc) has so many steps that a simple deployment
  could take 10 minutes to do everything needed on the code. This type of bloat is very annoying and developers
  usually hate it. We can help you think how to separate some of these steps to be run locally in the developers
  machines (they are usually more than capable of doing this) in some seconds instead of 10+ minutes remotely.
  The only part that we recommend keeping remote is the continuos delivery, that most of the time needs proper
  permissions and auditing.

- You need to have private workflows that don't use any service provider at all. This could be done with
  configuration managament and softwares like `jenkins` and `rundeck`.

- Developers on your company use `rsync` or `scp` to deploy applications and you want a more robust and
  auditable way to do these. We can help check if this is necessary and implement it in a good way.

<div style="text-align: right"><a href="#">top</a></div>

---

<a name="developer_backends"></a>
<br/>
## Developer backends and workflows

Software we most use: backstage, atlantis, rundeck, semaphore.

A bit of self-service operations is a must if your organization have many teams and/or applications. Nobody
wants to wait for an operator team to deploy, monitor and run maintenance tasks on applications and resources.
In this case, self-service of resource creation can be great for team agility. But yeah, you can't just
give all permissions on an infrastructure backend configuration to everyone and be done with it. Proper
limits and auditing should be prepared for these self-service operations.

Some example use-cases:

- Teams constantly need to create infrastructure resources for their applications, even on production,
  and they're responsible for them. They can use infrastructure as code with combination of a "deployer" like
  `atlantis` to do this via git pull/merge requests, and specific teams or colleagues can approve them to
  automatilly create resources, fast!

- You want to create a quick MVP using the same standards you company use for creating an application in
  a specific language. Instead of looking for examples, copy and pasting files and configurations from other
  applications, we can help you set up a developer portal using `Backstage` to give the developers ready to
  use boilerplates/templates for specific languages, along with specific infrastructure resources. In another
  words: A developer wants to create a Python project, they click a button, give the project a name, and
  Backstage would: create the git repository, create a python project boilerplate and replace the proper
  locations with the project name, and create infrastructure resources like queues and containers to run it.

- Operations team needs to automate the execution of some of their tasks. Instead of looking at an schedule
  and running them, they can configure something like `RunDeck` or `semaphore` to schedule or ad-hoc run
  these tasks, notifying as needed.

<div style="text-align: right"><a href="#">top</a></div>

---

<a name="containers"></a>
<br/>
## Containers

Software we most use: Kubernetes, OpenShift, docker

Containers simplified a lot creation of applications without having to deal with system settings and library
versions. But nothing is perfect, so when you introduce a container environment, you have to keep managing
a container cluster, container image versions, auditing, securing, and so on. This is a hard tasks that many
overlook but when you have a large deployment, it's a must!

Some example use-cases:

- You want to start using containers, but don't know where to start. We can advise what to use. Do you
  need a kubernetes distribution to handle this? Do you want a more integrated approach using OpenShift?
  Maybe for you, the best solution would be using a ready-to-use cloud provider like Amazon's EKS or ECS,
  Google's GKE, or others. And after deciding, we can do the best implementation and managing practises.

- You began to have lots of private images and it's getting a bit messy. We can help you develop an approach
  to organize, make common, re-usable and secure base images, implement private repositories and apply
  caching on them, and overral make the container image ecosystem more clean.

- You have difficulties into creating proper `Dockefiles` (or `Containerfiles`) and `helm` packages. We
  can help you building any of those and instruct how to re-use them.

<div style="text-align: right"><a href="#">top</a></div>

---

<a name="public_cloud"></a>
<br/>
## Public cloud

Public Clouds we most use: Amazon Web Services, Google Cloud, Oracle Cloud.

Using public computing clouds is one of the best ways to mitigate operational costs when you don't have
too much scale and personnel for it. But this doesn't mean that you don't need to have someone that
looks up your cloud infrastructure configuration and make it secure and scale better. Nowadays, we say
that if you don't need a classic system administrator, you'll probably need a *"CloudOps"* team, and it's
almost the same.

Some example use-cases:

- We can help you chose the best cloud for you, considering technical needs, availability and costs.

- You have some cloud account that is running some simple applications, clusters and databases that maintain
  your company running and they're doing great. But you need to scale, or get some certification, have
  more control while you grow your company... We can check your current structure and advise (or do) the work
  to get more things right, like: segregate environments into accounts/projects, create secure connections
  to access these resources (VPN or zero-trust model), review how you backup and audit, apply some
  infrastructure as code, and so on.

- You need some help implementing or using some infrastructure feature of the public cloud. A database (which
  version, what flags do I use, which size, how do I monitor?), a queue (what type of queue is better for
  my application?), a serverless function or a virtual machine (which one is better for me?). We can help you
  analyze and decide a good cloud architecture for your application.

- Migrate from one to another: one of the most daunting tasks, migrate between public clouds or regions. We do
  this a lot and we can help you with that.

<div style="text-align: right"><a href="#">top</a></div>

---

<a name="vm_management"></a>
<br/>
## VM management

Software we most use: XenServer, ProxMox, VMWare

Not using public clouds? No problem, hypervisors can be a great way to begin a company or product when you
have the experize. It can be a lot cheaper too, depending on the case. Sometimes your data can't be on another
place, or you have specific needs. We have the knowledge help you deploy and manage on-premises datacenter.

Some example use-cases:

- We can help you decide which service provider (VPS, Colocation) to use based on experience.

- You have a product running in some VPS and want to automate them using configuration management along
  with some integration with your provider's API.

- You have an on-premises datacenter with a hypervisor software like XenServer, ProxMox or VMWare and want
  to automate VM creation, or container nodes, or application deployments.

<div style="text-align: right"><a href="#">top</a></div>

---

<a name="specific_applications"></a>
<br/>
## Specific applications

Besides the other topics, we also have great experience working with this list that is always growing:

- `Single sign-on with Keycloak`: instance configuration, theme development, custom provider creation, knowledge
  with application oauth2 and saml flows, and many other things related to single sign-on services.

- `LDAP directory with OpenLDAP`: full installation and configuration of a LDAP directory, including recommendations
  about its structure and classes, and integration with other systems like SSH for centralized logins.

- `WordPress`: full installation, configuration and optimization services. We can help you with scaling,
  caching, CDN distribution, help with many plugins, the web server and php process behind it, and so on.

- `Mautic`: full instalation, configuration and optimization services. We can help you with scaling, version
  management and troubleshooting.

- `Falco/sysdig`: full integration into your container workflow/cluster and configuration to identify
  threats in running containers.

- `trivy`: full integration into your container building workflow to identify threats while building
  containers.

- `Python applications`: python programming for devops/system administrator tasks and applications, know-how
  to debug and troubleshoot running applications on a Linux system.

- `PHP applications`: know-how to debug and troubleshooting running code and applications.

- `Documentation`: mkdocs, sphinks, backstage.

<div style="text-align: right"><a href="#">top</a></div>
