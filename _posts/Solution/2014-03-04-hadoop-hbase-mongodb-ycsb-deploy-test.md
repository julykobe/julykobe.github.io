突然发现很多同学对搭环境很敬畏，一部分原因在于没有这种经历，另一部分原因便是对linux的不熟悉。

解决办法就两点：

1、熟悉linux基本操作 

2、google

我认为主要在锻炼解决问题的能力，而且在搭建环境的过程中会对所搭建的工程有更进一步的理解。



我本来准备把hadoop，hbase，mongodb以及ycsb的测试从头到尾写一个入门级的文档，结果过了个寒假发现都忘了差不多了。。于是把之前的草稿找出来草草的写一下大致流程吧，并且每个部分都提供了比较全面的文档。



文章分为5个部分，分别为系统安装配置，hadoop安装配置，hbase安装配置，mongodb安装配置，ycsb配置以及测试hbase和mongodb的性能。



环境：使用3台VMware的虚拟机，单核，内存1G，系统镜像为ubuntu-12.04.4-server-i386（最好是干净的系统）



软件准备：

关于版本的问题下面会说到。

jdk-6u24-linux-i586.bin

hadoop-1.1.2.tar.gz

hbase-0.96.0-hadoop1-bin.tar.gz

mongodb-linux-i686-2.4.9.tgz

apache-maven-3.1.1-bin.tar.gz

YCSB-master.zip

打包下载： <a href="http://pan.baidu.com/s/1sjwFCE5">点这里</a>)



1.   系统安装配置
  a)       安装虚拟化软件，VMware workstation 或者 virtual box，或者申请云主机。。

  b)       启动3台虚拟机，载入ubuntu的镜像，可以是用傻瓜安装一路next（只装ssh），一些配置装好后在改

  c)       配置网络

三个节点的ip和hostname如下：

    IP:10.10.82.225 hostname:zfhmaster

    IP:10.10.82.199 hostname:zfhslave

    IP:10.10.82.221 hostname:hwxslave

以下配置以zfhmaster为例说明

  sudo vi /etc/network/interfaces #网络接口配置，包括网络接口说明、IP地址、子网掩码、网关等

      auto eth0

              iface eth inetstatic

              address10.10.82.225

              gateway10.10.82.1

              netmask 255.255.255.0

 

      sudo vi/etc/resolv.conf             # DNS服务器设置

              nameserver 202.120.224.6

      sudo vi /etc/hostname               # 主机名设置

              zfhmaster

      sudo vi /etc/hosts                      #域名解析映射

          10.10.82.225 zfhmaster
          10.10.82.199 zfhslave
          10.10.82.221 hwxslave

2.   hadoop安装配置
需要注意：

  a)，开始前需要创建hadoop用户来操作，在下面提供的文档中说明了哪一步用hadoop用户哪一步用root。
创建用户用命令：
adduser [username]
比如想创建hadoop用户：
adduser hadoop
然后就会提示你输入密码等，比useradd更入门级。

  b)，在设置hadoop无密码使用root权限的时候，需要修改/erc/sudoers文件，但是有时会发现没有写的权限，无法修改。
使用root用户给这个文件增加写的权限:
chmod u+w /etc/sudoers
即可解决。


参考文档：
Hadoop完全分布式模式的安装和配置（主要）
在三台虚拟机上部署多节点Hadoop（辅助）


3.   hbase安装配置
需要注意：

1，由于hbase版本需要和hadoop版本兼容，所以建议使用我前面提供的文件版本，或者去官网查一下对应版本号。

参考文档：
HBASE完全分布式模式的安装（主要）
Hbase官方文档（辅助）


4.   mongodb安装配置
需要注意：

1，在32位机器下，需要创建/data/mongodb/log目录:
mkdir /data/mongodb/log -p
即可。

参考文档：
Mongodb 集群搭建--Replica Set（主要）
Mongodb集群搭建配置（辅助）

5.   ycsb配置以及测试hbase和mongodb的性能

参考文档：

利用ycsb测试redis性能（主要参考ycsb安装配置部分）



用这个版本会遇到的问题：

How to compile YCSB for Hbase 0.96.0?

即针对pom.xml需要修改两个地方，

<dependency>
  <groupId>org.apache.hbase</groupId>
  <artifactId>hbase</artifactId>
  <version>0.96.0-hadoop2</version>
</dependency>
需要改为：
<dependency>
  <groupId>org.apache.hbase</groupId>
  <artifactId>hbase-client</artifactId>
  <version>0.96.0-hadoop1</version>
</dependency>

测试性能的话，看一下workloads下不同的负载文件，然后跑负载，处理数据就行了（一笔带过。。。）