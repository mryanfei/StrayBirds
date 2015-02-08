---
layout: default
title: 搭建双节点的达梦数据库环境
comments: true
---


有时候我们需要多个数据库服务器协同工作,让它们看起来象单个服务器一样，即所谓的数据库集群。实现数据库集群的方式有很多种，比较容易理解的是使用中间件代理转发SQL的分库分表的方式，这种方式在互联网公司搭建MySQL集群时经常使用；其他集群方式有共享(share everything)和非共享(share nothing)两类。另外还有基于复制技术（不管是基于Redo的物理复制还是基于SQL回放的逻辑复制）的集群系统。
 
达梦的非共享多节点系统配置非常简单，我们可以在一台笔记本上配置两个达梦实例的集群，整个过程大概只需要数分钟时间。
## 准备工作
首先需要下载达梦7并安装好执行码。下载地址：http://www.dameng.com/service/download.shtml  一般每隔1-2个月都会有版本更新。下载页面上有Windows和Linux及对应的32位/64位版本。我的笔记本装的是WIN7 32版本, 所以采用了win32 版本。下面的操作示例都是基于windows环境。如果是linux环境，相应的路径需要做相应的修改。
 安装时你可能可能没有创建数据库，或者创建了数据库没有启动，也可能创建了数据库并且服务已经自动启动，这都没有关系。重要的是找到那个执行码所在的目录，如：c:\dmdbms\bin，然后确认里面有四个可执行文件：
 
```
n  Disql.exe
n  dmctlcvt.exe
n  dminit.exe
n  dmserver.exe
```

它们还需要依赖一些动态库，这些库都在同一个目录下。现在我们只需要关心这四个可执行文件，搭建一个双节点的系统只需要这四个文件，每一个都有它们特有的功能。

## 实例创建

我们使用dminit.exe来手工创建实例。打开一个命令行窗口，转到那四个文件所在的目录，然后执行dminit命令：

```
dminitpath=c:\dmdata db_name=EP01 instance_name=EP01 mal_flag=1 mpp_flag=1port_num=5000
```

这个命令将创建一个名为 EP01的数据库实例，所在的文件路径在c:\dmdata\EP01下，mal_flag和mpp_flag都设置为1表示将启用多节点间的网络通讯，port_num=5000表示该节点（数据库实例）的对外服务端口为5000。只要5000不和系统已经启动的达梦服务器的端口冲突即可。
 因为需要两个节点，因此我们再次执行dminit.exe:
 
```
dminit path=c:\dmdata db_name=EP02instance_name=EP02 mal_flag=1 mpp_flag=1 port_num=5001
```

同理，这次创建了EP02, 对外服务端口是5001，注意不能相同。

## 配置邮件通讯

前面我们已经创建了两个实例，在启动它们前，还需要做一点点配置工作。首先我们需要让这两个节点能互相通讯。达梦数据库内部有一个通讯子系统，模仿真实世界的邮件网络，称为MAIL，其配置非常简单。用文本编辑器编辑一个名为dmmal.ini的文件，插入下面几行，然后保存到实例所在的目录：

```
[inst1]
mal_inst_name=EP01
mal_host = localhost
mal_port=6000
[inst2]
mal_inst_name=EP02
mal_host=localhost
mal_port=6001
```

 这个配置文件中包括了两个节点的描述，实例名分别为EP01和EP02, 都运行在本机上，它们的邮件通讯端口分别为6000和6001。把dmmal.ini分别复制到c:\dmdata\EP01和c:\dmdata\EP02目录下，邮件配置就算完成了。
 
## 配置双机协同工作

邮件模块是一个底层子系统，配置完成使得节点间可以通讯，至于是用于主/备还是读写分离等还需要上层决定，因此还需要配置多机协同工作模式（这里是双机）。同样打开一个文本编辑器，插入下面几行：

```
[node1]
mpp_seq_no = 0
mpp_inst_name=EP01
[node2]
mpp_seq_no = 1
mpp_inst_name=EP02
```

这里也有两项，分别表示有2个节点，其序号分别为0和1，其实例名分别为EP01和EP02, 注意这里的序号必须从0开始依次编号，实例名必须和dmmal.ini中对应。我们把这个文件起个名，比如:a.txt, 然后保存到一个地方，比如: c:\dmdata\a.txt
 然后，我们使用dmctlcvt.exe, 把a.txt转换为一个二进制文件dmmpp.ctl，并把dmmpp.ctl复制到e:\dmdata\EP01和e:\dmdata\EP02两个目录下。
 
```
dmctlcvt t2c c:\dmdata\a.txt dmmpp.ctl
copy dmmpp.ctl c:\dmdata\EP01\
copy dmmpp.ctl c:\dmdata\EP02\
```

## 启动双机

至此，配置已经完成（简单吧？）我们可以分别启动EP01和EP02两个实例。打开一个命令行窗口，转到c:\dmdbms\bin 下，然后执行：

```
dmserver c:\dmdata\ep01\dm.ini
```

这样我们就启动了一个达梦7的数据库实例EP01。因为是第一次启动，系统需要做一点额外的初始化工作，最后我们会看到“SYSTEM IS READY”的输出，表示启动成功。
 打开另一个命令行窗口，转到c:\dmdbms\bin下，执行：
 
```
dmserver c:\dmdata\ep02\dm.ini
```

就可以启动另一个达梦7的数据库实例EP02

## 体验双机协同的魅力

现在集群已经启动，我们可以使用disql来连接任何一个节点。EP01和EP02是对等的，接入任何一个都能完成相同的工作。这里我们连EP02。
 打开一个命令行窗口，转到c:\dmdbms\bin下，然后执行：
 
```
Disql SYSDBA/SYSDBA@localhost:5001
```

如果一切顺利，我们就打开了一个对EP02的连接，系统会输出类似下列的信息：

```
 C:\dmdbms\bin>disqlSYSDBA/SYSDBA@localhost:5001
 服务器[localhost:5001]:处于普通打开状态
使用普通用户登录
 登录使用时间: 50.325(毫秒)
disqlV7.1.3.212-Build(2015.01.13-52915trunc)
Connected to: DM 7.1.3.212
SQL>
```

 注意在命令行窗口的标题栏，其文字为：”localhost:5001 /SYSDBA [MPP_GLOBAL]”, 表示连接的是本机5001端口，连接的用户为SYSDBA,[MPP_GLOBAL]表示本次连接是全局模式，即SQL查询的范围是所有节点。
 我们创建一个表，插入一些数据，然后做一个简单查询：
 
```SQL
SQL> create table t1(c1 int, c2 int);
SQL> insert into t1 select level, level* 10 from dual connect by  level <1000;
SQL> commit;
SQL> select count(*) from t1 a, t1 bwhere a.c1 = b.c1 and a.c1 < 10;
行号       COUNT(*)
---------- --------------------
1         9
```

 这些操作似乎没有什么特别的，完全和单节点一个样。打开另一个命令窗口，连接到EP01, 执行同样的查询，其结果完全一样。
## 本地模式查询
前面T1的数据有999行，被随机地分布到EP01和EP02两个节点。达梦7支持多种分布模式，包括随机， HASH,列表和范围等。前面我们用缺省的全局模式连接到某个节点做操作，感觉不到这些分布。下面我们用本地模式连接，本地(mpp_local)模式就只能查询到本节的数据了。为了安全起见，MPP_LOCAL模式下，不能做DDL操作，DML操作如何违反分布规则，也会被拒绝。
 使用下列命令可以实现disql的MPP_LOCAL模式连接：
 
```
disql SYSDBA/SYSDBA*LOCAL@localhost:5000
注意这里增加了*LOCAL。
SQL> select count(*) from t1 a, t1 bwhere a.c1 = b.c1 and a.c1 < 10;
行号       COUNT(*)
---------- --------------------
1                                  0
SQL> select count(*) from t1;
 行号       COUNT(*)
---------- --------------------
1                                  499
```

这两个简单查询的结果表明，在EP01上，T1有499行数据，其中c1小于10的记录一行都没有。
 通过这个小例子，我们可以看到达梦7的多机非共享集群搭建非常方便，数据被自动分布到不同节点，在多机环境下，应用层接入某一个节点即可实现SQL操作，对SQL操作几乎没有任何限制，可以很方便地把应用系统从单机扩展到多机集群环境，对应用几乎没有限制和修改要求。目前我们的不少客户已经使用这类集群，用于大量数据的业务应用。
