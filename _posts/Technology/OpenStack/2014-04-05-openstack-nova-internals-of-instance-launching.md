---
layout: post
title: OpenStack启动虚拟机时Nova内部工作流程
category: 技术
tags: OpenStack
keywords: OpenStack,Nova
description: 本文描述了OpenStack Nova在启动虚拟机时的内部工作机制。
---

原文地址：[这里](http://www.laurentluce.com/posts/openstack-nova-internals-of-instance-launching/)

###Overview
OpenStack Nova中与启动虚拟机相关的组件:

* API server: 处理来自用户的请求并转发到cloud controller.
* Cloud controller: 处理计算节点， the networking controllers, the * API server and the scheduler之间的通信
* Scheduler: 选择一个节点来执行命令（启动虚拟机）
* Compute worker: 管理虚拟机实例：启动/终止虚拟机，绑定，解绑磁盘卷
* Network controller: 管理网络资源：分配固定IP，管理VLANs…
Note: 本文仅关注虚拟机启动，而对身份验证，镜像存储等相关细节不做探究


首先API server收到一个来自用户的run_instances命令：

1. API server把这条消息转发给云控制器
2. 先要认证用户有需要的权限。云控制器把消息发给调度器
3. 调度器把这条消息扔给一个随机的host（计算节点），并请求启动一台新虚拟机。
4. 计算节点上的compute worker捕获这条消息
5. 6.7.8. compute worker需要一个固定IP来启动新虚拟机，因此发送一条消息给network controller。

![launch_instance_overview](/public/upload/OpenStack-nova-launch-instance-progress/launch_instance_overview.png)

###API

有两种API OpenStack API和EC2 API。这里使用EC2 API。先添加一个新的key pair，然后用它来启一台规模是m1.tiny的虚拟机


	cd /tmp/
	euca-add-keypair test > test.pem
	euca-run-instances -k test -t m1.tiny ami-tiny

在api/ec2/cloud.py中的 run_instances()使用了compute API ——位于compute/API.py下的create() 


	def run_instances(self, context, **kwargs):
	  ...
	  instances = self.compute_api.create(context,
	            instance_type=instance_types.get_by_type(
	                kwargs.get('instance_type', None)),
	            image_id=kwargs['image_id'],
	            ...

compute的API create() 会做如下这下事情 :

* 检查此类型虚拟机数量是否达到最大值
* 如果没有安全组的话创建一个
* 为新虚拟机生成MAC地址和hostnames
* 向调度器发消息来启动虚拟机


###Cast

现在看看消息是怎么发送到调度器的。在OpenStack中这种消息交付类型叫做RPC转换，在转换中使用了RabbitMQ。发布者（API）发送消息到一个交易处（topic exchange），消费者（scheduler worker）从队列中获取消息。正是因为这是一个cast而不是call，因此并没有响应发生。

![rpc_cast](/public/upload/OpenStack-nova-launch-instance-progress/rpc_cast.png)
 
下面是casting消息的代码:


	LOG.debug(_("Casting to scheduler for %(pid)s/%(uid)s's"
	        " instance %(instance_id)s") % locals())
	rpc.cast(context,
	         FLAGS.scheduler_topic,
	         {"method": "run_instance",
	          "args": {"topic": FLAGS.compute_topic,
	                   "instance_id": instance_id,
	                   "availability_zone": availability_zone}})


可以看到使用了调度器主题，并且消息参数中也表名我们想要使用调度器来delivery。这个case中，我们想让调度器使用compute主题来发送消息。

###Scheduler

调度器收到消息并把run_instance消息发送到任意计算节点上。这里用了chance scheduler。有很多调度算法，比如说zone scheduler（在一个特定可用域内pick一个随机host），simple scheduler（pick最小负载的host）。现在已经选定了一个host，下面的代码是在host上发送消息给一个compute worker。

	rpc.cast(context,
	         db.queue_get_for(context, topic, host),
	         {"method": method,
	          "args": kwargs})
	LOG.debug(_("Casting to %(topic)s %(host)s for %(method)s") % locals()) 

###Compute

compute worker收到消息，然后执行compute/manager.py中的run_instance方法，如下：


	def run_instance(self, context, instance_id, **_kwargs):
	  """Launch a new instance with specified options."""
	  ...


run_instance() 干什么呢:

* 检查虚拟机是否已经启动。
* 分配一个固定IP。
* 如果没有安装VLAN和bridge的话，执行安装操作。
* virtualization driver“造(spawn)”虚拟机

###Call to network controller

使用RPC请求来分配固定IP。一个RPC请求和RPC cast是区别在于它使用了一个topic.host exchange，这意味着它要指定目标host，因此，还要有一个请求。

![rpc_call](/public/upload/OpenStack-nova-launch-instance-progress/rpc_call.png)
 

###Spawn instance

接下来是由虚拟化驱动器执行的虚拟机生成进程。这个case中使用libvirt，下面要看的代码在virt/libvirt_conn.py

启动一个虚拟机首先要做的是创建libvirt xml文件。使用to_xml()方法来获取xml内容。

	<domain type='qemu'>
	    <name>instance-00000001</name>
	    <memory>524288</memory>
	    <os>
	        <type>hvm</type>
	        <kernel>/opt/novascript/trunk/nova/..//instances/instance-00000001/kernel</kernel>
	        <cmdline>root=/dev/vda console=ttyS0</cmdline>
	        <initrd>/opt/novascript/trunk/nova/..//instances/instance-00000001/ramdisk</initrd>
	    </os>
	    <features>
	        <acpi/>
	    </features>
	    <vcpu>1</vcpu>
	    <devices>
	        <disk type='file'>
	            <driver type='qcow2'/>
	            <source file='/opt/novascript/trunk/nova/..//instances/instance-00000001/disk'/>
	            <target dev='vda' bus='virtio'/>
	        </disk>
	        <interface type='bridge'>
	            <source bridge='br100'/>
	            <mac address='02:16:3e:17:35:39'/>
	            <!--   <model type='virtio'/>  CANT RUN virtio network right now -->
	            <filterref filter="nova-instance-instance-00000001">
	                <parameter name="IP" value="10.0.0.3" />
	                <parameter name="DHCPSERVER" value="10.0.0.1" />
	                <parameter name="RASERVER" value="fe80::1031:39ff:fe04:58f5/64" />
	                <parameter name="PROJNET" value="10.0.0.0" />
	                <parameter name="PROJMASK" value="255.255.255.224" />
	                <parameter name="PROJNETV6" value="fd00::" />
	                <parameter name="PROJMASKV6" value="64" />
	            </filterref>
	        </interface> 
	        <!-- The order is significant here.  File must be defined first -->
	        <serial type="file">
	            <source path='/opt/novascript/trunk/nova/..//instances/instance-00000001/console.log'/>
	            <target port='1'/>
	        </serial> 
	        <console type='pty' tty='/dev/pts/2'>
	            <source path='/dev/pts/2'/>
	            <target port='0'/>
	        </console>
	        <serial type='pty'>
	            <source path='/dev/pts/2'/>
	            <target port='0'/>
	        </serial>
	    </devices>
	</domain>

使用的hypervisor是qemu。为guest分配的内存是524kbytes。guest OS将会从内核启动，initrd存储在host OS上。

分配给guest OS 1个虚拟CPU。电源管理中允许ACPI。

同时定义了多种设备:

* 磁盘镜像是放在host OS上的qcow2格式的文件。qcow2是qemu磁盘镜像copy-on-write的格式。
* The network interface is a bridge visible to the guest. We define network filtering parameters like IP which means this interface will always use 10.0.0.3 as the source IP address.
* 设备的日志文件。所有发到字符设备的数据都被写到console.log中。
* Pseudo TTY: virsh console can be used to connect to the serial port locally.

接下来准备网络过滤器。默认的防火墙驱动是iptables。规则定义在apply_ruleset()中的IptablesFirewallDriver类中。

	*filter
	...
	:nova-ipv4-fallback - [0:0]
	:nova-local - [0:0]
	:nova-inst-1 - [0:0]
	:nova-sg-1 - [0:0]
	-A nova-ipv4-fallback -j DROP
	-A FORWARD -j nova-local
	-A nova-local -d 10.0.0.3 -j nova-inst-1
	-A nova-inst-1 -m state --state INVALID -j DROP
	-A nova-inst-1 -m state --state ESTABLISHED,RELATED -j ACCEPT
	-A nova-inst-1 -j nova-sg-1
	-A nova-inst-1 -s 10.1.3.254 -p udp --sport 67 --dport 68
	-A nova-inst-1 -j nova-ipv4-fallback
	-A nova-sg-1 -p tcp -s 10.0.0.0/27 -m multiport --dports 1:65535 -j ACCEPT
	-A nova-sg-1 -p udp -s 10.0.0.0/27 -m multiport --dports 1:65535 -j ACCEPT
	-A nova-sg-1 -p icmp -s 10.0.0.0/27 -m icmp --icmp-type 1/65535 -j ACCEPT
	COMMIT

首先有这些链： nova-local, nova-inst-1, nova-sg-1, nova-ipv4-fallback 和这些规则。

下面看看这些不同的链和规则:

数据包在虚拟网络中的路由由nova-local来控制

	-A FORWARD -j nova-local
如果目的地址是10.0.0.3说明使我们启动的虚拟机，因此跳转到nova-inst-1

	-A nova-local -d 10.0.0.3 -j nova-inst-1
如果包不能被识别，丢掉

	-A nova-inst-1 -m state --state INVALID -j DROP
如果包被关联到一条已经建立的连接或者启动一个新的连接但是关联到一个已经存在的连接，接受。

	-A nova-inst-1 -m state --state ESTABLISHED,RELATED -j ACCEPT
允许DHCP响应。

	-A nova-inst-1 -s 10.0.0.254 -p udp --sport 67 --dport 68
跳到安全组chain并检查包是否符合规则。

	-A nova-inst-1 -j nova-sg-1
Security group chain. 允许所有来自10.0.0.0/27（端口从1到65535）的TCP包。

	-A nova-sg-1 -p tcp -s 10.0.0.0/27 -m multiport --dports 1:65535 -j ACCEPT
允许所有来自10.0.0.0/27（端口从1到65535）的UDP包。

	-A nova-sg-1 -p udp -s 10.0.0.0/27 -m multiport --dports 1:65535 -j ACCEPT
允许所有来自10.0.0.0/27（端口从1到65535）的ICMP包。

	-A nova-sg-1 -p icmp -s 10.0.0.0/27 -m icmp --icmp-type 1/65535 -j ACCEPT
跳到fallback chain.

	-A nova-inst-1 -j nova-ipv4-fallback
这就是fallback chain中我们丢到那个包的规则。

	-A nova-ipv4-fallback -j DROP
Here is an example of a packet for a new TCP connection to 10.0.0.3:

![iptables](/public/upload/OpenStack-nova-launch-instance-progress/iptables.png)

接下来是创建镜像。在\_create\_image()中。


	def _create_image(self, inst, libvirt_xml, suffix='', disk_images=None):
	  ...

libvirt.xml基于我们之前生成的XML来创建。

ramdisk，initrd和磁盘镜像都要复制一份供hypervisor使用

如果使用flat网络管理则一份网络配置已经备注入到guest OS image中。这里我们就用这种方式。

虚拟机的SSH key被注入到镜像中。这里使用了inject_data()方法

	disk.inject_data(basepath('disk'), key, net,
	                 partition=target_partition,
	                 nbd=FLAGS.use_cow_images)

basepath('disk')是实例磁盘镜像存放在host OS的位置。key是SSH key 字符串。net没有设置因为在此例中我们没有注入网络配置。partition是None因为我们使用了一个内核镜像，否则我们可以使用一个partitioned的磁盘镜像。下面看一下inject_data()的内部。

首先要把镜像链接到一块设备上。在\_link\_device()中：

	device = _allocate_device()
	utils.execute('sudo qemu-nbd -c %s %s' % (device, image))
	# NOTE(vish): this forks into another process, so give it a chance
	#             to set up before continuuing
	for i in xrange(10):
	    if os.path.exists("/sys/block/%s/pid" % os.path.basename(device)):
	        return device
	    time.sleep(1)
	raise exception.Error(_('nbd device %s did not show up') % device)

\_allocate\_device()返回下一个可用的ndb设备：/dev/ndbx,x在0到15之间。qemu-ndb是一个QEMU磁盘网络块设备服务器。这些做好后，我们得到了设备：/dev/ndb0

我们禁止了这个设备的文件系统检查。

	out, err = utils.execute('sudo tune2fs -c 0 -i 0 %s' % mapped_device)
我们把一个文件系统挂载到一个临时目录，然后添加SSH key到ssh authorized_keys文件。

	sshdir = os.path.join(fs, 'root', '.ssh')
	utils.execute('sudo mkdir -p %s' % sshdir)  # existing dir doesn't matter
	utils.execute('sudo chown root %s' % sshdir)
	utils.execute('sudo chmod 700 %s' % sshdir)
	keyfile = os.path.join(sshdir, 'authorized_keys')
	utils.execute('sudo tee -a %s' % keyfile, '\n' + key.strip() + '\n')

在上述代码中，fs是所说的临时文件。

最后，解除文件系统的挂载和设备的连接。这包括了镜像创建和安装。

虚拟驱动器spawn()方法的下一步是使用createXML()绑定方法来启动实例。



OVEr

原作者的linkedin： [LinkedIn profile](http://www.linkedin.com/in/lluce). Twitter @laurentluce.