 ---
layout: post
title: Openstack Cloud Administrator Guide 1
category: 技术
tags: OpenStack
keywords: OpenStack,Overview
description: OpenStack Cloud Administrator Guide是一个面向云管理员的手册，主要作用是manage and troubleshoot an OpenStack cloud。
---


 <a href="http://docs.openstack.org/admin-guide-cloud/content/">OpenStack Cloud Administrator Guide</a>是一个面向云管理员的手册，主要作用是manage and troubleshoot an OpenStack cloud。读它可以对各组件有一个整体上的把握。

  这里是第一章Get started with OpenStack的穷人版翻译。。

#Overview
##服务简述
OpenStack的服务从下面这张表中可以看得很清楚，包括了Heat和Ceilometer。

![OpenStack-services](/public/upload/OpenStack Cloud Administrator Guide/OpenStack-services.jpg)
 Table1 OpenStack Services

##逻辑架构
![OpenStack-logical-architecture](/public/upload/OpenStack Cloud Administrator Guide/OpenStack-logical-architecture.jpg)
Table2 OpenStack Logical Architecture

         OpenStack 的模块是由守护进程(Daomon)，脚本(Script)以及命令行接口(Command-line interface(CLI))组成。守护进程在linux平台表现为服务；脚本用于虚拟环境的安装及测试；CLI允许用户通过易用的命令向OpenStack的服务提交API请求。

 

##服务介绍
###Compute service(Nova)
计算服务是云计算的结构控制器，它构成了IaaS系统的主要部分。它的构成如下：

API
nova-api service.接受和响应终端用户的计算API请求。支持，OpenStack Compute API, the Amazon EC2 API, 和一个特殊的AdminAPI 供特权用户实现管理的操作。同时，初始化大部分 orchestration活动, 譬如启动一台虚拟机，实施某些策略。
nova-api-metadata service.从虚拟机接受元数据请求。需要注意的是，通常情况下，此API仅在安装nova-network的multi-host模式下使用。（在大便系统中，它属于nova-api 包的一部分）

Compute core
nova-compute process.一个通过hypervisor API来创建和终止虚拟机实例的工作进程。譬如，XenServer/XCP的XenAPI, KVM or QEMU的libvirt, VMware的VMwareAPI，等等。进程的执行相当复杂，但是基本原理却很简单：从消息队列中接受相应的动作(actions),然后在更新数据库中的状态的时候执行一系列的系统命令（比如启动一台KVM实例）Accept actions from the queue and perform a series of systemcommands, like launching a KVM instance, to carry them out while updating statein the database.
nova-scheduler process.从概念上来讲这是Compute中最简单的一块代码了。从队列中获取一个虚拟机的请求然后决定它应该在哪个compute server host中响应(run)。
nova-conductor module.协调nova-compute 和数据库的交互。目的在于减少nova-compute对云数据库的直接访问。由于该模块是水平的伸缩，因此不要部署在任何nova-compute运行的节点上。

Networking for VMs
nova-network work daemon.类似于nova-compute，它从队列中接受关于网络的任务（比如修改桥接的接口，或是修改iptables的规则）并执行，从而实现对网络的控制。这项功能正在被迁移到一个独立的服务OpenStack Networking(Neutron)，
nova-dhcpbridge script.通过dnsmasq的dhcp-script跟踪IP地址的租约并把它们记录在数据库中。它同样正被迁移到Neutron中。Neutron提供了一个不同的脚本。

Console interface
nova-consoleauth daemon.认证控制台代理提供给用户的令牌。这个服务必须运行在控制台代理才能工作。Many proxies of either type can be run against a singlenovaconsoleauthservice in a cluster configuration
nova-novncproxy daemon.通过VNC连接来提供一个访问运行的虚拟机的代理。支持基于浏览器的novnc 客户端。
nova-console daemon.在G版被弃用，由下面一个进程替代。
nova-xvpnvncproxy daemon.通过VNC连接来访问运行状态的虚拟机的代理。支持一种专为OpenStack设计的Java客户端。
nova-cert daemon.管理x509证书。

Image management(EC2Scenario)
nova-objectstore daemon.提供一个S3(Simple Storage Service, Amazon 存储服务)接口供Glance（镜像服务）来注册镜像。主要在支持euca2ools的情况下是使用。euca2ools 工具用S3的语法告知nova-objectstore，然后nova-objectstore把S3请求转换为Glance请求。
uca2ools client.一组用来管理云资源的命令。尽管它不是OpenStack的模块，仍然可以通过配置nova-api 来支持这个EC2的接口。

Command-line clients and otherinterfaces
nova client.允许用户以租户管理者或者终端用户的身份来提交命令。
nova-manage client.允许云管理员来提交明亮。

Other components
The queue.在守护进程中传递消息的核心。通常是由RabbitMQ来实现的，但也可以是任何AMPQ的队列，如Apache Qpid 或者 Zero MQ。
SQL database.为云基础设施存储大部分构建时以及运行时的信息，包括可以使用的虚拟机类型，正在使用的虚拟机，可用的网络和项目。理论上来说，Nova支持任何SQL类型的数据库，但是目前仅有MySQL和PostgreSQL是广泛使用的，sqlite3仅在测试和开发工作中表现正常。
 

###Storage concepts
![storage-types](/public/upload/OpenStack Cloud Administrator Guide/storage-types.jpg)
Table3 Storage types

 

###Object Storage service(Swift)
         Swift是一个有高度伸缩以及可持续性的多租户对象存储系统，它通过RESTful HTTP API为大量非结构化的数据以极低的开销来提供存储服务。

它包括以下部件：

Proxy servers(swift-proxy-server).接受Object Storage API 和原生HTTP请求，如上传文件，修改元数据和创建容器等。同时它也为web浏览器提供文件和容器列表（？）。为了提升性能，代理服务器可以任选使用哪种cache，不过通常情况下都部署在memcache下。
Account servers(swift-account-server).管理定义了对象存储服务的账户。
Container servers(swift-container-server).管理使用了swift的容器或者文件夹的映射。
Object servers(swift-object-server).管理实际的对象，例如存储节点上的文件。
A number of periodic processes.执行内部任务。主从同步服务(replication services)保证了集群中的持续性和可用性。其他一些周期性进程包括审计，更新，以及获取(auditors,updaters,reapers)。
可配置的WSGI中间件负责认证授权，通常是由Keystone负责。

 

###Block Storage Service(Cinder)
         块存储服务允许对卷(volumes)，卷快照，卷类型的管理。它包括以下组件：

cinder-api.接受API请求并根据一定的策略把它们转发(routes)到cinder-volume来执行。
cinder-volume.响应对块存储数据库的读写请求来维护状态(maintain state),通过一个消息队列和其他进程(如cinder-scheduler)进行交互，并直接基于块存储提供硬件或软件。它可以通过一个驱动架构来和大多数提供存储的服务交互。
cinder-scheduler daemon.像nova-scheduler一样，选取最合适的提供块存储的节点来创建卷。
Messaging queue.在cinder的进程中交互(routes)信息。
cinder为nova提供虚拟机的卷。

 

###Networking service overview(Neutron)
         为设备接口提供网络连接即服务(network-connectivity-as-a-service)，这类设备通常由OpenStack的服务——Compute所管理。它允许用户创建和添加网络接口。像OpenStack的其他服务一样，模块化可插拔的架构使得它可以高度的自由配置。这些插件整合了不同的网络设备和软件，因此，它的架构和部署会发生很大的变化。它包括了如下组件：

neutron-server.接受API请求并转发到合适的网络插件中去执行。
OpenStack Networking plug-ins and agents.绑定以及解绑端口，创建网络和子网，并提供IP地址。这些插件和代理(agent)通常在特定的云中依据代理商和使用的技术而定。目前Neutron支持(ships)的一些插件和代理有Cisco的虚拟和物理交换机，Nicira 的NVP产品，NEC OpenFlow 产品，Open vSwitch, Linux bridging,和Ryu网络操作系统。
通用的代理是L3(layer 3)，DHCP和插件代理。

Messaging queue.大部分的Neutron安装版本都利用了消息队列在neutron-server和各种代理间传递消息，并利用一个数据库来存储特定插件的网络状态。
 

###Dashboard(Horizon)
         dashboard是一个模块化的Django web应用，为OpenStack的服务提供图形化接口。它通常通过mod_wsgi部署在Apache下。你可以对代码进行修改使他适合不同的站点。

         从网络架构的观点来看，它必须可以被客户和OpenStack的其他服务通过公用API访问。而想要使用其他服务的管理员功能，它必须可以连接到管理员API端点(Admin API endpoints)，因为这个端点是不能被客户直接访问的。

###Identity Service concepts(Keystone)
         Identity服务进行如下操作：

用户管理。跟踪用户及他们的权限。
服务目录。提供可用的服务以及他们的API端点的目录。
想要理解Keystone，必须理解一下概念：

User
使用OpenStack云服务的人，系统或服务的数字表象(账号?)。Keystone负责验证请求是否由所需要的用户发出。用户可以登录并被授予令牌来使用资源。用户可以直接被分配到某一租户下，就像是本身就是租户的一部分一样。

Credentials
仅能被某一用户识别的表示他们身份的数据，就像Keystone中的用户名和密码，用户名和API key，由Keystone提供的认证令牌等。

Authentication
验证一个用户身份的动作。Keystone通过用户提供的一系列证书(credentials)来验证请求。

这些证书由用户名密码或者用户名和API key进行初始化。Keystone响应这些证书请求，发给用户一个认证的令牌，用户在随后的请求中使用此令牌即可。

Token
一个用来访问资源的随机生成的比特序列。每个令牌都有一个起作用的资源的范围。一个令牌可以在有限的时间内用于认证，也可以在任何时间被撤销。

尽管在当前版本中(Havana)Keystone支持基于令牌的认证，但是它的主要作用在于在未来支持更多额外的协议。它的目的在于成为一个重要的集成服务，而非一个负责存储和管理身份的解决方案。

Tenant
一组或单独的资源和(或)标识对象的容器。(A container used to group or isolate resources and/or identityobjects.)依据服务的使用者不同，一个租户可以对应一个用户(customer)，账号(account),组织(organization),或者项目(project).

Service
类似于Compute(Nova),Object Storage(Swift),Image Service(Glance)的OpenStack服务。提供一个或多个端点(endpoints)供用户使用资源或进行操作。

Endpoint
一个可以在网络上进行访问的地址，通常在使用服务的时候是以URL的形式表示。如果想在模板上进行扩展，你可以创建一个表示所有可以跨区(regions)使用的服务的端点模板，

Role
一个允许用户进行一系列指定操作的身份(personality)。一个角色包括一系列权利以及权限。一个拥有指定角色的用户等于继承了该角色的权利以及权限。

         在Keystone中，授予一个用户的令牌包括了用户拥有的角色列表。用户请求哪个服务取决于怎样翻译(interpret)用户拥有的角色以及每个角色可以访问哪些操作以及资源。

 

         下图表示了Keystone的工作流程：


![keystone-identity-manager](/public/upload/OpenStack Cloud Administrator Guide/keystone-identity-manager.jpg)
Figure1 The Keystone Identity Manager

 

###Image Service overview(Glance)
Glance包括如下组件：

Glance-api.
接受镜像API请求来搜索(discovery)，检索(retrieval)和存储(storage)镜像。

Glance-registry.
存储，处理，检索镜像的元数据，例如大小，类型等。

Database.
存储镜像元数据。你可以根据自己的喜好选择合适的数据库，大部分人使用MySQL和SQlite。

Storage repository for image files.
在逻辑结构图中，Glance是一个镜像仓库。但是你也可以配置一个不同的仓库。Glance支持普通的文件系统，RADOS块设备，Amazon S3和HTTP。一些系统仅能提供只读数据的功能。

 

         在Glance中有一些周期性的进程用来支持缓存(caching)。主从同步服务(replication services)保证了集群中的持续性和可用性。其他一些周期性进程包括审计，更新，以及获取(auditors,updaters,reapers)。

         在概念架构图上可以看到，Glance是整个IaaS层中的中心。它接受终端用户或者Nova处理镜像或镜像元数据的API请求，然后可以把磁盘文件存储到Swift中。

 

###The Telemetry Service(Ceilometer)
OpenStackTelemetry 服务：

高效的收集CPU和网络开销的计量数据。
通过监控服务发出的通知或者轮询基础设施来收集数据。
可以配置收集数据的类型来满足不同操作的需求。通过REST API来获取和插入数据。
通过额外的插件扩展框架来收集客户的使用数据。
生成有签名的计量信息，这些信息是不可否认的。(Produces signed metering messages that cannot be repudiated)
 

系统包括如下基本组件：

A compute agent(ceilometer-agent-compute).
运行在每一个compute节点上并轮询资源使用的统计数据。在未来可能有其他类型的agents，但目前我们专注于创建compute agent。

A central agent(ceilometer-agent-central).
在中枢管理服务器上运行，轮询那些与虚拟机或计算节点独立(not tied)的资源利用的统计数据。

A collector(ceilometer-collector).
运行在一个或多个中枢管理服务器上来监控消息队列(通知和来自agent的计量数据)。通知消息被处理后被转换为计量消息并使用恰当的主题发回给消息总线。Telemetry消息不做修改直接写入数据库。

An alarm notifier(ceilometer-alarm-notifier).
运行在一个或多个中枢管理服务器，并基于采集的样本进行评估所得到的一个阈值来设定警报。

A data store.
一个支持并发读写数据的数据库，其中写入数据来自一个或多个收集计量数据的虚拟机，读数据是从API server读。

An API server(ceilometer-api).
运行在一个或多个中枢管理服务器来提供数据库中的数据。

 

这些服务使用标准OpenStack消息总线进行交互。只有collector和API server有权访问数据库。

 

###Orchestration service overview(Heat)
         Heat对云应用提供了基于模板编排(orchestration……无力)管理，通过OpenStack的API调用来生成运行的云应用。它把OpenStack的其它核心组件整合到了一个单文件模板系统。这套模板允许你创建大部分OpenStack资源类型，比如虚拟机，浮动IP，卷，安全组和用户等。它同时提供了很多更高级的功能，比如虚拟机的高可用性，虚拟机动态伸缩以及嵌套栈等。通过对OpenStack核心组件的紧密整合，所有OpenStack的核心组件可以得到一个更大的用户群(a larger user base)。

         这项服务允许部署人员直接整合该服务或者可以自定义插件。

         Heat包括如下组件：

heat command-line client.
一个命令行界面，可以和heat-api交互来调用 AWS CloudFormation(就是Amazon的模板) APIs. 终端开发者同样可以直接调用Orchestration REST API。

heat-api component.
提供一个OpenStack原生(OpenStack-native)REST API来处理API请求，并使用RPC把请求转发到heat-engine。

heat-api-cfn component.
提供一个和AWS CloudFormation兼容的AWS Query API，并使用RPC把请求转发到heat-engine。

heat-engine.
整理模板的启动并把事件(events)发回给API请求者。