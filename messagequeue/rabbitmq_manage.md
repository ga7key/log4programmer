> RabbitMQ 版本为3.6.10

## RabbitMQ 安装

##### 安装Erlang
1. 将erlang安装到/opt/erlang 目录下：

```bash
[root@hidden ~]# tar zxvf otp_src_19.3.tar.gz
[root@hidden ~]# cd otp_src_19.3
[root@hidden otp_src_19.3]# ./configure --prefix=/opt/erlang
```

2. 如果在安装的过程中出现类似“No ***** found”的提示，可根据提示信息安装相应的包，例如出现“No curses library functions found”报错：

```bash
[root@hidden otp_src_19.3]# yum install ncurses-devel
```

3. 安装Erlang：

```bash
[root@hidden otp_src_19.3]# make
[root@hidden otp_src_19.3]# make install
```

4. 修改/etc/profile 配置文件，添加下面的环境变量：

```bash
ERLANG_HOME=/opt/erlang
export PATH=$PATH:$ERLANG_HOME/bin
export ERLANG_HOME
```

最后执行如下命令让配置文件生效：

```bash
[root@hidden otp_src_19.3]# source /etc/profile
```

5. 输入erl 命令来验证Erlang 是否安装成功：

```bash
[root@hidden ~]# erl
Erlang/OTP 19 [erts-8.1] [source] [64-bit] [smp:4:4] [async-threads:10] [hipe] [kernel-poll:false]
Eshell V8.1 (abort with ^G)
```

##### RabbitMQ 的安装与运行

```bash
[root@hidden ~]# tar zvxf rabbitmq-server-generic-unix-3.6.10.tar.gz -C /opt
[root@hidden ~]# cd /opt
[root@hidden ~]# mv rabbitmq_server-3.6.10 rabbitmq
```

修改/etc/profile 文件，添加下面的环境变量：

```bash
export PATH=$PATH:/opt/rabbitmq/sbin
export RABBITMQ_HOME=/opt/rabbitmq
```

之后执行 source /etc/profile 命令让配置文件生效。

运行RabbitMQ，“-detached”参数是为了能够让RabbitMQ服务以守护进程的方式在后台运行：

```bash
rabbitmq-server –detached
```

默认情况下，访问RabbitMQ 服务的用户名和密码都是“guest”，这个账户有限制，默认只能通过本地网络（如localhost）访问，远程网络访问受限。

添加新用户，用户名为“root”，密码为“root123”：

```bash
[root@hidden ~]# rabbitmqctl add_user root root123
```

为root 用户设置所有权限：

```bash
[root@hidden ~]# rabbitmqctl set_permissions -p / root ".*" ".*" ".*"
```

设置root 用户为管理员角色：

```bash
[root@hidden ~]# rabbitmqctl set_user_tags root administrator
```


## RabbitMQ 管理
### rabbitmqctl
**rabbitmqctl** 工具是用来管理RabbitMQ 中间件的命令行工具，它通过连接各个RabbitMQ 节点来执行所有操作。  
rabbitmqctl 工具的标准语法如下（[]表示可选参数，{}表示必选参数）：

```bash
rabbitmqctl [-n node] [-t timeout] [-q] {command} [command options...]
```

- [-n node]：默认节点是“rabbit@hostname”，此处的hostname 是主机名称。在一个名为“node.hidden.com”的主机上，RabbitMQ 节点的名称通常是rabbit@node（除非RABBITMQ_NODENAME 参数在启
动时被设置成了非默认值）。hostname -s 命令的输出通常是“@”标志后的东西。
- [-q]：使用-q 标志来启用quiet 模式，这样可以屏蔽一些消息的输出。 默认不开启quiet 模式。
- [-t timeout]：操作超时时间（秒为单位），只适用于“list_xxx”类型的命令，默认是无穷大。

操作命令示例：

```bash
[root@node1 ~]# rabbitmqctl list_vhosts
Listing vhosts
/
[root@node1 ~]# rabbitmqctl list_vhosts -q
/
[root@node1 ~]# rabbitmqctl list_vhosts -q -t 1
/
[root@node1 ~]# rabbitmqctl list_vhosts -q -t 0
Error: {timeout,0.0}
```

### 多租户与权限
每一个RabbitMQ 服务器都能创建虚拟的消息服务器，我们称之为虚拟主机（virtual host），简称为vhost。  
每一个vhost 本质上是一个独立的小型RabbitMQ 服务器，拥有自己独立的队列、交换器及绑定关系等，并且它拥有自己独立的权限。vhost 就像是虚拟机与物理服务器一样，它们在各个实例间提供逻辑上的分离，为不同程序安全保密地运行数据，它既能将同一个RabbitMQ 中的众多客户区分开，又可以避免队列和交换器等命名冲突。vhost 之间是绝对隔离的，无法将vhost1 中的交换器与vhost2 中的队列进行绑定，这样既保证了安全性，又可以确保可移植性。  
vhost 是AMQP 概念的基础，客户端在连接的时候需要制定一个vhost。RabbitMQ 默认创建的vhost 为“/”  

**创建一个新的vhost**，大括号里的参数表示vhost 的名称：  
rabbitmqctl add_vhost {vhost}
```bash
[root@node1 ~]# rabbitmqctl add_vhost vhost1
Creating vhost "vhost1"
```

**罗列当前vhost 的相关信息**，目前vhostinfoitem 的取值有2 个：  
rabbitmqctl list_vhosts [vhostinfoitem...]

> name：表示vhost 的名称。  
tracing：表示是否使用了RabbitMQ 的trace 功能。

```bash
[root@node1 ~]# rabbitmqctl list_vhosts name tracing
Listing vhosts
vhost1 false
/ false
[root@node1 ~]# rabbitmqctl trace_on
Starting tracing for vhost "/"
[root@node1 ~]# rabbitmqctl list_vhosts name tracing
Listing vhosts
vhost1 false
/ true
```

**删除vhost** ，大括号里面的参数表示vhost 的名称。同时也会删除其下所有的队列、交换器、绑定关系、用户权限、参数和策略等信息：  
rabbitmqctl delete_vhost {vhost}

```bash
[root@node1 ~]# rabbitmqctl delete_vhost vhost1
Deleting vhost "vhost1"
```

AMQP 协议中并没有指定权限在vhost 级别还是在服务器级别实现，由具体的应用自定义。  
在RabbitMQ 中，权限控制则是以vhost 为单位的。当创建一个用户时，用户通常会被指派给至少一个vhost，并且只能访问被指派的vhost 内的队列、交换器和绑定关系等。因此，RabbitMQ中的授予权限是指在vhost 级别对用户而言的权限授予。  
**授予权限命令**为：  
rabbitmqctl set_permissions [-p vhost] {user} {conf} {write} {read}
> vhost：授予用户访问权限的vhost 名称，可以设置为默认值，即vhost 为“/”。  
user：可以访问指定vhost 的用户名。  
conf：一个用于匹配用户在哪些资源上拥有可配置权限的正则表达式，可配置指的是队列和交换器的创建及删除之类的操作。  
write：一个用于匹配用户在哪些资源上拥有可写权限的正则表达式，可写指的是发布消息。  
read：一个用于匹配用户在哪些资源上拥有可读权限的正则表达式，可读指与消息有关的操作，包括读取消息及清空整个队列等。  

AMQP 命令与权限的映射关系：

AMQP 命令 | 可 配 置 | 可 写 | 可 读
:---- | :---- | :---- | :----
Exchange.Declare | exchange | |
Exchange.Declare(with AE) | exchange | exchange(AE) | exchange
Exchange.Delete | exchange | |
Queue.Declare | queue | |
Queue.Declare(with DLX) | queue | exchange (DLX) | queue
Queue.Delete | queue | |
Exchange.Bind | | exchange (destination) | exchange (source)
Exchange.Unbind | | exchange (destination) | exchange (source)
Queue.Bind | | queue | exchange
Queue.Unbind | | queue | exchange
Basic.Publish | | exchange |
Basic.Get | | | queue
Basic.Consume | | | queue
Queue.Purge | | | queue

```bash
# 授予root 用户可访问虚拟主机vhost1，并在所有资源上都具备可配置、可写及可读的权限
[root@node1 ~]# rabbitmqctl set_permissions -p vhost1 root ".*" ".*" ".*"
Setting permissions for user "root" in vhost "vhost1"
# 授予root 用户可访问虚拟主机vhost2，在以“queue”开头的资源上具备可配置权限，并在所有资源上拥有可写、可读的权限
[root@node1 ~]# rabbitmqctl set_permissions -p vhost2 root "^queue.*" ".*" ".*"
Setting permissions for user "root" in vhost "vhost2"
```

**清除权限的命令**为：  
rabbitmqctl clear_permissions [-p vhost] {username}
> vhost ：设置禁止用户访问的虚拟主机的名称，默认为“/”。  
username ：表示禁止访问特定虚拟主机的用户名称。

```bash
[root@node1 ~]# rabbitmqctl clear_permissions -p vhost1 root
Clearing permissions for user "root" in vhost "vhost1"
```

**显示虚拟主机上的权限**：  
rabbitmqctl list_permissions [-p vhost]

**显示用户的权限**：  
rabbitmqctl list_user_permissions {username}

```bash
[root@node1 ~]# rabbitmqctl list_permissions -p vhost1
Listing permissions in vhost "vhost1"
root .* .* .*
[root@node1 ~]# rabbitmqctl list_user_permissions root
Listing permissions for user "root"
/ .* .* .*
vhost1 .* .* .*
```

### 用户管理
用户是访问控制（Access Control）的基本单元，且单个用户可以跨越多个vhost 进行授权。针对一至多个vhost，用户可以被赋予不同级别的访问权限，并使用标准的用户名和密码来认证用户。

**创建用户**：  
rabbitmqctl add_user {username} {password}  
> username ：表示要创建的用户名称  
password ：表示创建用户登录的密码  

```bash
# 创建一个用户名为root、密码为root123 的用户
[root@node1 ~]# rabbitmqctl add_user root root123
Creating user "root"
```

**更改指定用户的密码**：  
rabbitmqctl change_password {username} {newpassword}
> username ：表示要变更密码的用户名称  
newpassword ：表示要变更的新的密码

```bash
# 将root 用户的密码变更为root321
[root@node1 ~]# rabbitmqctl change_password root root321
Changing password for user "root"
```

**清除用户的密码**：  
rabbitmqctl clear_password {username}
> username ：表示要清除密码的用户名称

**通过密码来验证用户**：  
rabbitmqctl authenticate_user {username} {password}
> username ：表示需要被验证的用户名称  
password ：表示密码

```bash
# 采用密码root321 来验证root 用户
[root@node1 ~]# rabbitmqctl authenticate_user root root321
Authenticating user "root"
Success
# 采用密码root322 来验证root 用户
[root@node1 ~]# rabbitmqctl authenticate_user root root322
Authenticating user "root"
Error: failed to authenticate user "root"
```

**删除用户**：  
rabbitmqctl delete_user {username}
> username ：表示要删除的用户名称

```bash
[root@node1 ~]# rabbitmqctl delete_user root
Deleting user "root"
```

**罗列当前的所有用户**：  
rabbitmqctl list_users

```bash
[root@node1 ~]# rabbitmqctl list_users
Listing users
guest [administrator]
root []
```

每个结果行都包含用户名称，其后紧跟用户的角色（tags）。用户的角色分为5 种类型：
> none：无任何角色。新创建的用户的角色默认为none。  
management：可以访问Web 管理页面。  
policymaker：包含management 的所有权限，并且可以管理策略（Policy）和参数（Parameter）  
monitoring：包含management 的所有权限，并且可以看到所有连接、信道及节点相关的信息。  
administartor：包含monitoring 的所有权限，并且可以管理用户、虚拟主机、权限、策略、参数等。administator 代表了最高的权限。

**设置用户的角色**：  
rabbitmqctl set_user_tags {username} {tag ...}
> username ：参数表示需要设置角色的用户名称  
tag ：参数用于设置0 个、1 个或者多个的角色，设置之后任何之前现有的身份都会被删除

```bash
[root@node1 ~]# rabbitmqctl set_user_tags root
Setting tags for user "root" to []
[root@node1 ~]# rabbitmqctl list_users -q
guest [administrator]
root []
[root@node1 ~]# rabbitmqctl set_user_tags root policymaker,management
Setting tags for user "root" to ['policymaker,management']
[root@node1 ~]# rabbitmqctl list_users -q
guest [administrator]
root [policymaker,management]
```

### Web 端管理
为了运行rabbitmqctl 工具，当前的用户需要拥有访问Erlang cookie 的权限，由于服务器可能是以guest 或者root 用户身份来运行的，因此需要获得这些文件的访问权限，这样就引申出来一些权限管理的问题。  
RabbitMQ management 插件可以提供Web 管理界面用来管理如前面所述的虚拟主机、用户等，也可以用来管理队列、交换器、绑定关系、策略、参数等，还可以用来监控RabbitMQ服务的状态及一些数据统计类信息，基本上能够涵盖所有RabbitMQ 管理的功能。

先启用RabbitMQ management 插件。RabbitMQ 提供了很多的插件，默认存放在$RABBITMQ_HOME/plugins 目录下：

```bash
[root@node1 plugins]# ls -al
-rw-r--r-- 1 root root 270985 Oct 25 19:45 amqp_client-3.6.10.ez
-rw-r--r-- 1 root root 225671 Oct 25 19:45 cowboy-1.0.4.ez
(省略若干项…)
```

启动rabbitmq_management-3.6.10.ez 插件，使用rabbitmq-plugins 命令，其语法格式为：  
rabbitmq-plugins [-n node] {command} [command options...]

**启动插件**：  
rabbitmq-plugins enable [plugin-name]

**关闭插件**：  
rabbitmq-plugins disable [plugin-name]

**查看当前插件的使用情况**：  
rabbitmq-plugins list

> 标记为[E*]的为显式启动，而[e*]为隐式启动，如显式启动rabbitmq_management 插件会同时隐式启动amqp_client、cowboy、cowlib 等另外5 个插件。

```bash
[root@node1 ~]# rabbitmq-plugins list
Configured: E = explicitly enabled; e = implicitly enabled
| Status: * = running on rabbit@node1
|/
[e*] amqp_client 3.6.10
[e*] cowboy 1.0.4
[e*] cowlib 1.0.2
[ ] rabbitmq_amqp1_0 3.6.10
[E*] rabbitmq_management 3.6.10
[e*] rabbitmq_management_agent 3.6.10
[e*] rabbitmq_web_dispatch 3.6.10
[ ] rabbitmq_management_visualiser 3.6.10
(省略若干项…)
```

开启rabbitmq_management 插件之后还需要重启RabbitMQ 服务才能使其正式生效。之后就可以通过浏览器访问http://localhost:15672/ ，这样会出现一个认证登录的界面，可以通过默认的guest/guest 的用户名和密码来登录。如果访问的IP 地址不是本地地址，比如在192.168.0.2的主机上访问http://192.168.0.3:15672 的Web 管理页面，使用默认的guest 账户是访问不了的。需要使用一个具有非none 的用户角色的非guest 账户来访问Web 管理页面。

某些情况下，用户能够正确登录，但是除了页面头部和尾部没有任何其它内容呈现。此时清空下浏览器的缓存即可。

### 应用与集群管理
- rabbitmqctl stop [pid_file]  
用于停止运行RabbitMQ 的Erlang 虚拟机和RabbitMQ 服务应用。  
如果指定了pid_file，还需要等待指定进程的结束。其中pid_file 是通过调用rabbitmq-server 命令启动RabbitMQ 服务时创建的，默认情况下存放于Mnesia 目录中，可以通过RABBITMQ_PID_FILE 这个环境变量来改变存放路径。注意，如果使用rabbitmq-server –detach 这个带有-detach 后缀的命令来启动RabbitMQ 服务则不会生成pid_file 文件。

```bash
[root@node1 ~]# rabbitmqctl stop /opt/rabbitmq/var/lib/rabbitmq/mnesia/rabbit\@node1.pid
Stopping and halting node rabbit@node1
[root@node1 ~]# rabbitmqctl stop
Stopping and halting node rabbit@node1
```

- rabbitmqctl shutdown  
用于停止运行RabbitMQ 的Erlang 虚拟机和RabbitMQ 服务应用。  
执行这个命令会阻塞直到Erlang 虚拟机进程退出。如果RabbitMQ 没有成功关闭，则会返回一个非零值。  
这个命令和rabbitmqctl stop 不同的是，它不需要指定pid_file 而可以阻塞等待指定进程的关闭。

```bash
[root@node1 ~]# rabbitmqctl shutdown
Shutting down RabbitMQ node rabbit@node1 running at PID 1706
Waiting for PID 1706 to terminate
RabbitMQ node rabbit@node1 running at PID 1706 successfully shut down
```

- rabbitmqctl stop_app  
停止RabbitMQ 服务应用，但是Erlang 虚拟机还是处于运行状态。  
此命令的执行优先于其他管理操作（这些管理操作需要先停止RabbitMQ 应用），比如rabbitmqctl reset。

```bash
[root@node1 ~]# rabbitmqctl stop_app
Stopping rabbit application on node rabbit@node1
```

- rabbitmqctl start_app  
启动RabbitMQ 应用。  
此命令典型的用途是在执行了其他管理操作之后，重新启动之前停止的RabbitMQ 应用，比如rabbitmqctl reset。

```bash
[root@node1 ~]# rabbitmqctl start_app
Starting node rabbit@node1
```

- rabbitmqctl wait [pid_file]  
等待RabbitMQ 应用的启动。  
它会等到pid_file 的创建，然后等待pid_file 中所代表的进程启动。当指定的进程没有启动RabbitMQ 应用而关闭时将会返回失败。

```bash
[root@node1 ~]# rabbitmqctl wait /opt/rabbitmq/var/lib/rabbitmq/mnesia/rabbit\@node1.pid
Waiting for rabbit@node1
pid is 3468
```

- rabbitmqctl reset  
将RabbitMQ 节点重置还原到最初状态。包括从原来所在的集群中删除此节点，从管理数据库中删除所有的配置数据，如已配置的用户、vhost 等，以及删除所有的持久化消息。  
执行rabbitmqctl reset 命令前必须停止RabbitMQ 应用（比如先执行rabbitmqctl stop_app）。

```bash
[root@node1 ~]# rabbitmqctl stop_app
Stopping rabbit application on node rabbit@node1
[root@node1 ~]# rabbitmqctl reset
Resetting node rabbit@node
```

- rabbitmqctl force_reset  
强制将RabbitMQ 节点重置还原到最初状态。  
rabbitmqctl force_reset 命令不论当前管理数据库的状态和集群配置是什么，都会无条件地重置节点。它只能在数据库或集群配置已损坏的情况下使用。  
执行rabbitmqctl force_reset 命令前必须先停止RabbitMQ 应用。

```bash
[root@node1 ~]# rabbitmqctl stop_app
Stopping rabbit application on node rabbit@node1
[root@node1 ~]# rabbitmqctl force_reset
Forcefully resetting node rabbit@node1
```

- rabbitmqctl rotate_logs {suffix}  
指示RabbitMQ 节点轮换日志文件。RabbitMQ节点会将原来的日志文件中的内容追加到“原始名称+后缀”的日志文件中，然后再将新的日志内容记录到新创建的日志中（与原日志文件同名）。  
当目标文件不存在时，会重新创建。如果不指定后缀suffix，则日志文件只是重新打开而不会进行轮换。

```bash
# 原日志文件为rabbit@node1.log 和rabbit@node1-sasl.log
# 轮换日志之后，原日志文件中的内容就被追加到rabbit@node1.log.bak 和 rabbit@node1-sasl.log.bak 日志中
# 之后重新建立rabbit@node1.log 和rabbit@node1-sasl.log 文件用来接收新的日志
[root@node1 rabbitmq]# ll
-rw-r--r-- 1 root root 1024127 Oct 18 11:56 rabbit@node1.log
-rw-r--r-- 1 root root 720553 Oct 17 19:16 rabbit@node1-sasl.log
[root@node1 rabbitmq]# rabbitmqctl rotate_logs .bak
Rotating logs to files with suffix ".bak"
[root@node1 rabbitmq]# ll
-rw-r--r-- 1 root root 0 Oct 18 12:05 rabbit@node1.log
-rw-r--r-- 1 root root 1024202 Oct 18 12:05 rabbit@node1.log.bak
-rw-r--r-- 1 root root 0 Oct 18 12:05 rabbit@node1-sasl.log
-rw-r--r-- 1 root root 720553 Oct 18 12:05 rabbit@node1-sasl.log.bak
```

- rabbitmqctl hipe_compile {directory}  
将部分RabbitMQ 代码用HiPE（HiPE 是指High Performance Erlang，是Erlang 版的JIT）编译，并且将编译后的.beam 文件（beam 文件是Erlang 编译器生成的文件格式，可以直接加载到Erlang 虚拟机中运行的文件格式）保存到指定的文件目录中。如果这个目录不存在则会自行创建。如果这个目录中原本有任何.beam 文件，则会在执行编译前被删除。  
如果要使用预编译的这些文件，则需要设置RABBITMQ_SERVER_CODE_PATH 这个环境变量来指定hipe_compile 调用的路径。

```bash
[root@node1 rabbitmq]# rabbitmqctl hipe_compile
/opt/rabbitmq/tmp/rabbit-hipe/ebin
HiPE compiling: |-----------------------------------------------|
|###############################################|
Compiled 57 modules in 55s
```

- rabbitmqctl join_cluster {cluster_node} [--ram]  
将节点加入指定集群中。在这个命令执行前需要停止RabbitMQ 应用并重置节点

- rabbitmqctl cluster_status  
显示集群的状态

- rabbitmqctl change_cluster_node_type {disc|ram}  
修改集群节点的类型。在这个命令执行前需要停止RabbitMQ 应用

- rabbitmqctl forget_cluster_node [--offline]  
将节点从集群中删除，允许离线执行

- rabbitmqctl update_cluster_nodes {clusternode}  
在集群中的节点应用启动前咨询clusternode 节点的最新信息，并更新相应的集群信息。  
这个和join_cluster 不同，它不加入集群。考虑这样一种情况，节点A 和节点B 都在集群中，当节点A 离线了，节点C 又和节点B 组成了一个集群，然后节点B 又离开了集群，当A 醒来的时候，它会尝试联系节点B，但是这样会失败，因为节点B 已经不在集群中了。

```bash
##假设已有node1 和node 组成的集群
##1.初始状态
[root@node1 ~]# rabbitmqctl cluster_status
Cluster status of node rabbit@node1
[{nodes,[{disc,[rabbit@node1,rabbit@node2]}]}, {running_nodes,[rabbit@node2,rabbit@node1]},
{cluster_name,<<"rabbit@node1">>}, {partitions,[]}, {alarms,[{rabbit@node2,[]},{rabbit@node1,[]}]}]
##2.关闭node1 节点的应用
[root@node1 ~]# rabbitmqctl stop_app
Stopping rabbit application on node rabbit@node1
##3.之后将node3 加入到集群中（rabbitmqctl join_cluster rabbit@node3）
##4.再将node2 节点的应用关闭
##5.最后启动node1 节点的应用，此时会报错
[root@node1 ~]# rabbitmqctl start_app
Starting node rabbit@node1
BOOT FAILED
===========
Timeout contacting cluster nodes: [rabbit@node2].
……(省略)
##6.如果在启动node1 节点的应用之前咨询node3 并更新相关集群信息则可以解决这个问题
[root@node1 ~]# rabbitmqctl update_cluster_nodes rabbit@node3
Updating cluster nodes for rabbit@node1 from rabbit@node3
[root@node1 ~]# rabbitmqctl start_app
Starting node rabbit@node1
##7.最终集群状态
[root@node1 ~]# rabbitmqctl cluster_status
Cluster status of node rabbit@node1
[{nodes,[{disc,[rabbit@node1,rabbit@node3]}]}, {running_nodes,[rabbit@node3,rabbit@node1]},
{cluster_name,<<"rabbit@node1">>}, {partitions,[]}, {alarms,[{rabbit@node3,[]},{rabbit@node1,[]}]}]
```

- rabbitmqctl force_boot  
确保节点可以启动，即使它不是最后一个关闭的节点。  
通常情况下，当关闭整个RabbitMQ 集群时，重启的第一个节点应该是最后关闭的节点，因为它可以看到其他节点所看不到的事情。但是有时会有一些异常情况出现，比如整个集群都掉电而所有节点都认为它不是最后一个关闭的。在这种情况下，可以调用rabbitmqctl force_boot 命令，这就告诉节点可以无条件地启动节点。  
在此节点关闭后，集群的任何变化，它都会丢失。如果最后一个关闭的节点永久丢失了，那么需要优先使用rabbitmqctl forget_cluster_node --offline 命令，因为它可以确保镜像队列的正常运转。

```bash
[root@node2 ~]# rabbitmqctl force_boot
Forcing boot for Mnesia dir /opt/rabbitmq/var/lib/rabbitmq/mnesia/rabbit@node2
[root@node2 ~]# rabbitmq-server –detached
```

- rabbitmqctl sync_queue [-p vhost] {queue}  
指示未同步队列queue 的slave 镜像可以同步master 镜像行的内容。  
同步期间此队列会被阻塞（所有此队列的生产消费者都会被阻塞），直到同步完成。此条命令执行成功的前提是队列queue 配置了镜像。  
注意，未同步队列中的消息被耗尽后，最终也会变成同步，此命令主要用于未耗尽的队列。

```bash
[root@node1 ~]# rabbitmqctl sync_queue queue
Synchronising queue 'queue' in vhost '/'
```

- rabbitmqctl cancel_sync_queue [-p vhost] {queue}  
取消队列queue 同步镜像的操作

```bash
[root@node1 ~]# rabbitmqctl cancel_sync_queue queue
Stopping synchronising queue 'queue' in vhost '/'
```

- rabbitmqctl set_cluster_name {name}  
设置集群名称。集群名称在客户端连接时会通报给客户端。  
Federation 和Shovel 插件也会有用到集群名称的地方。集群名称默认是集群中第一个节点的名称。  
在Web 管理界面的右上角有个“（change）”的地方，点击也可以修改集群名称。

```bash
[root@node1 ~]# rabbitmqctl set_cluster_name cluster_hidden
Setting cluster name to cluster_hidden
[root@node1 ~]# rabbitmqctl cluster_status
Cluster status of node rabbit@node1
[{nodes,[{disc,[rabbit@node1,rabbit@node2]}]},{running_nodes,[rabbit@node2,rabbit@node1]},
{cluster_name,<<"cluster_hidden">>},{partitions,[]},{alarms,[{rabbit@node2,[]},{rabbit@node1,[]}]}]
```

### 服务端状态
服务器状态的查询会返回一个以制表符（Tab键）分隔的列表。  
该命令接受一个可选的vhost 参数以显示其结果，默认值为“/”。

- rabbitmqctl list_queues [-p vhost] [queueinfoitem ...]  
此命令返回队列的详细信息，如果无[-p vhost]参数，将显示默认的vhost 为“/”中的队列详情。  
queueinfoitem 参数用于指示哪些队列的信息项会包含在结果集中，结果集的列顺序将匹配参数的顺序。  
queueinfoitem 可以是下面列表中的任何值。

> name：队列名称。  
durable：队列是否持久化。  
auto_delete：队列是否自动删除。  
arguments：队列的参数。  
policy：应用到队列上的策略名称。  
pid：队列关联的Erlang 进程的ID。  
owner_pid：处理排他队列连接的Erlang 进程ID。如果此队列是非排他的，此值将为空。  
exclusive：队列是否是排他的。  
exclusive_consumer_pid：订阅到此排他队列的消费者相关的信道关联的Erlang进程ID。如果此队列是非排他的，此值将为空。  
exclusive_consumer_tag：订阅到此排他队列的消费者的consumerTag。如果此队列是非排他的，此值将为空。  
messages_ready：准备发送给客户端的消息个数。  
messages_unacknowledged：发送给客户端但尚未应答的消息个数。  
messages：准备发送给客户端和未应答消息的总和。  
messages_ready_ram：驻留在内存中messages_ready 的消息个数。  
messages_unacknowledged_ram：驻留在内存中messages_unacknowledged的消息个数。  
messages_ram：驻留在内存中的消息总数。  
messages_persistent：队列中持久化消息的个数。对于非持久化队列来说总是0。  
messages_bytes：队列中所有消息的大小总和。这里不包括消息属性或者任何其他开销。
messages_bytes_ready：准备发送给客户端的消息的大小总和。
messages_bytes_unacknowledged：发送给客户端但尚未应答的消息的大小总和。  
messages_bytes_ram：驻留在内存中的messages_bytes。  
messages_bytes_persistent：队列中持久化的messages_bytes。  
disk_reads：从队列启动开始，已从磁盘中读取该队列的消息总次数。  
disk_writes：从队列启动开始，已向磁盘队列写消息的总次数。  
consumer：消费者数目。  
consumer_utilisation：队列中的消息能够立刻投递给消费者的比率，介于0 和1之间。这个受网络拥塞或者Basic.Qos 的影响而小于1。  
memory：与队列相关的Erlang 进程所消耗的内存字节数，包括栈、堆及内部结构。  
slave_pids：如果队列是镜像的，列出所有slave 镜像的pid。  
synchronised_slave_pids：如果队列是镜像的，列出所有已经同步的slave 镜像的pid。  
state：队列状态。正常情况下是running；如果队列正常同步数据可能会有“{syncing, MsgCount}”的状态；如果队列所在的节点掉线了，则队列显示状态为down（此时大多数的queueinfoitems 也将不可用）。  

如果没有指定queueinfoitems，那么此命令将显示队列的名称和消息的个数。

```bash
[root@node1 ~]# rabbitmqctl list_queues -p vhost1 name disk_writes disk_reads -q
queue4 3390 0
queue5 0 0
```

- rabbitmqctl list_exchanges [-p vhost] [exchangeinfoitem ...]  
返回交换器的详细细节，如果无[-p vhost]参数，将显示默认的vhost 为“/”中的交换器详情。  
exchangeinfoitem 参数用于指示哪些信息项会包含在结果集中，结果集的列顺序将匹配参数的顺序。  
exchangeinfoitem 可以是下面列表中的任何值。

> name：交换器的名称。  
type：交换器的类型。  
durable：设置是否持久化。durable 设置为true 表示持久化，反之是非持久化。持久化可以将交换器信息存盘，而在服务器重启的时候不会丢失相关信息。  
auto_delete：设置是否自动删除。  
internal：是否是内置的。  
arguments：其他一些结构化参数，比如alternate-exchange。  
policy：应用到交换器上的策略名称。  

如果没有指定exchangeinfoitem，那么此命令将显示交换器的名称和类型。

```bash
[root@node1 ~]# rabbitmqctl list_exchanges name type durable auto_delete internal arguments policy -q
amq.rabbitmq.trace  topic   true false true  []
amq.headers         headers true false false []
                    direct  true false false []
amq.match           headers true false false []
amq.topic           topic   true false false []
amq.fanout          fanout  true false false []
amq.rabbitmq.log    topic   true false true  []
amq.direct          direct  true false false []
```

- rabbitmqctl list_bindings [-p vhost] [bindinginfoitem ...]  
返回绑定关系的细节，如果无[-p vhost]参数，将显示默认的vhost 为“/”中的绑定关系详情。  
bindinginfoitem 参数用于指示哪些信息项会包含在结果集中，结果集的列顺序将匹配参数的顺序。  
bindinginfoitem 可以是下面列表中的任何值。

> source_name：绑定中消息来源的名称。  
source_kind：绑定中消息来源的类别。  
destination_name：绑定中消息目的地的名称。  
destination_kind：绑定中消息目的地的种类。  
routing_key：绑定的路由键。  
arguments：绑定的参数。  

如果没有指定bindinginfoitem，那么将显示所有的条目。

```bash
# 交换器exchange1 和队列queue1 通过rk1 进行绑定，还有一个独立的队列queue2。显示的第一行是默认的交换器与queue1 进行绑定，这个是RabbitMQ 内置的功能。
[root@node1 ~]# rabbitmqctl list_bindings -q
          exchange queue1 queue queue1 []
          exchange queue2 queue queue2 []
exchange1 exchange queue1 queue rk1    []
```

- rabbitmqctl list_connections [connectioninfoitem ...]  
返回TCP/IP 连接的统计信息。  
connectioninfoitem 参数用于指示哪些信息项会包含在结果集中，结果集的列顺序将匹配参数的顺序。  
connectioninfoitem 可以是下面列表中的任何值。

> pid：与连接相关的Erlang 进程ID。  
name：连接的名称。  
port：服务器端口。  
host：返回反向DNS 获取的服务器主机名称，或者IP 地址，或者未启用。  
peer_port：服务器对端端口。当一个客户端与服务器连接时，这个客户端的端口就是peer_port。  
peer_host：返回反向DNS 获取的对端主机名称，或者IP 地址，或者未启用。  
ssl：是否启用SSL。  
ssl_protocol：SSL 协议，如tlsv1。  
ssl_key_exchange：SSL 密钥交换算法，如rsa。  
ssl_cipher：SSL 加密算法，如aes_256_cbc。  
ssl_hash：SSL 哈希算法，如sha。  
peer_cert_subject：对端的SSL 安全证书的主题，基于RFC4514 的形式。  
peer_cert_issuer：对端SSL 安全证书的发行者，基于RFC4514 的形式。  
peer_cert_validity：对端SSL 安全证书的有效期。  
state：连接状态，包括starting、tuning、opening、running、flow、blocking、blocked、closing 和closed 这几种。  
channels：该连接中的信道个数。  
protocol：使用的AMQP 协议的版本，当前是{0,9,1}或者{0,8,0}。注意，如果客户端请求的是AMQP 0-9 的连接，RabbitMQ 也会将其视为0-9-1。  
auth_mechanism：使用的SASL 认证机制，如PLAIN、AMQPLAIN、EXTERNAL、RABBIT-CR-DEMO 等。  
user：与连接相关的用户名。  
vhost：与连接相关的vhost 的名称。  
timeout：连接超时/协商的心跳间隔，单位为秒。  
frame_max：最大传输帧的大小，单位为B。  
channel_max：此连接上信道的最大数量。如果值0，则表示无上限，但客户端一般会将0 转变为65535。  
client_properties：在建立连接期间由客户端发送的信息属性。  
recv_oct：收到的字节数。  
recv_cnt：收到的数据包个数。  
send_oct：发送的字节数。  
send_cnt：发送的数据包个数。  
send_pend：发送队列大小。  
connected_at：连接建立的时间戳。  

如果没有指定connectioninfoitem，那么会显示user、peer_host、peer_port 和state 这几项信息。

```bash
[root@node1 ~]# rabbitmqctl list_connections -q
root 192.168.0.22 57304 running
root 192.168.0.22 57316 running
```

- rabbitmqctl list_channels [channelinfoitem ...]  
返回当前所有信道的信息。  
channelinfoitem 参数用于指示哪些信息项会包含在结果集中，结果集的列顺序将匹配参数的顺序。  
channelinfoitem 可以是下面列表中的任何值。

> pid：与连接相关的Erlang 进程ID。  
connection：信道所属连接的Erlang 进程ID。  
name：信道的名称。  
number：信道的序号。  
user：与信道相关的用户名称。  
vhost：与信道相关的vhost。  
transactional：信道是否处于事务模式。  
confirm：信道是否处于publisher confirm 模式。  
consumer_count：信道中的消费者的个数。  
messages_unacknowledged：已投递但是还未被ack 的消息个数。  
messages_uncommitted：已接收但是还未提交事务的消息个数。  
acks_uncommitted：已ack 收到但是还未提交事务的消息个数。  
messages_unconfirmed：已发送但是还未确认的消息个数。如果信道不处于publisher confirm 模式下，则此值为0。  
perfetch_count：新消费者的Qos 个数限制。0 表示无上限。  
global_prefetch_count：整个信道的Qos 个数限制。0 表示无上限。  

如果没有指定channelinfoitem，那么会显示pid、user、consumer_count 和messages_unacknowledged 这几项信息。

```bash
[root@node1 ~]# rabbitmqctl list_channels -q
<rabbit@node1.1.631.0> root 0 0
<rabbit@node1.1.643.0> root 1 0
```

- rabbitmqctl list_consumers [-p vhost]  
列举消费者信息。  
每行将显示由制表符分隔的已订阅队列的名称、相关信道的进程标识、consumerTag、是否需要消费端确认、prefetch_count 及参数列表这些信息。

```bash
[root@node1 ~]# rabbitmqctl list_consumers -p default -q
queue4 <rabbit@node1.1.1628.11> consumer_zzh true 0 []
```

- rabbitmqctl status  
显示Broker 的状态，比如当前Erlang 节点上运行的应用程序、RabbitMQ/Erlang 的版本信息、OS 的名称、内存及文件描述符等统计信息。

- rabbitmqctl node_health_check  
对RabbitMQ 节点进行健康检查，确认应用是否正常运行、list_queues 和list_channels是否能够正常返回等。

```bash
[root@node1 ~]# rabbitmqctl node_health_check
Timeout: 70.0 seconds
Checking health of node rabbit@node1
Health check passed
```

- rabbitmqctl environment  
显示每个运行程序环境中每个变量的名称和值。

- rabbitmqctl report  
为所有服务器状态生成一个服务器状态报告，并将输出重定向到一个文件。

```bash
[root@node1 ~]# rabbitmqctl report > report.txt
```

- rabbitmqctl eval {expr}  
执行任意Erlang 表达式。  
用户、Parameter、vhost、权限等都可以通过rabbitmqctl 工具来完成创建（或删除）的操作，而交换器、队列及绑定关系的创建（或删除）操作可以通过rabbitmqctl eval {expr} 实现。

```bash
# 返回rabbitmqctl连接的节点名称
[root@node1 ~]# rabbitmqctl eval 'node().'
rabbit@node1
```

<span style="color: red;font-weight: bold;">Tips</span>：若要删除所有的交换器、队列及绑定关系，删除对应的vhost 就可以“一键搞定”，而不需要一个个遍历删除。

### HTTP API 接口管理
RabbitMQ Management 插件不仅提供了Web 管理界面，还提供了HTTP API 接口来方便调用。  
HTTP API 是完全基于RESTful 风格的，不同的HTTP API 接口所对应的HTTP 方法各不相同，这里一共涉及4 种HTTP 方法：GET、PUT、DELETE 和POST。

    GET 方法一般用来获取如集群、节点、队列、交换器等信息。  
    PUT 方法用来创建资源，如交换器、队列之类的。  
    DELETE 方法用来删除资源。  
    POST 方法也是用来创建资源的，与PUT 不同的是，POST 创建的是无法用具体名称的资源。比如绑定关系（bindings）和发布消息（publish）无法指定一个具体的名称。

所有的HTTP API 接口都需要HTTP 基础认证（使用标准的RabbitMQ 用户数据库），默认的是guest/guest（非localhost 的不能使用这组认证，除非特殊设置）

```bash
# 通过curl 命令创建队列queue ，“%2F”是指默认的vhost，即“/”，这类特殊字符在HTTP URL 中是需要转义的
[root@node1 ~]# curl -i -u root:root123 -H "content-type:application/json"
-XPUT -d '{"auto_delete":false,"durable":true,"node":"rabbit@node2"}'
http://192.168.0.2:15672/api/queues/%2F/queue
HTTP/1.1 201 Created
server: Cowboy
date: Fri, 25 Aug 2017 06:03:17 GMT
content-length: 0
content-type: application/json
vary: accept, accept-encoding, origin

# 通过GET 方法来获取队列queue 的信息
[root@node1 ~]# curl -i -u root:root123 –XGET http://192.168.0.2:15672/api/queues/%2F/queue

# 通过DELETE 方法来删除队列queue
[root@node1 ~]# curl -i -u root:root123 -XDELETE http://192.168.0.2:15672/api/queues/%2F/queue
```

<span style="color: red;font-weight: bold;">Tips</span>：Web 管理界面左下角的“HTTP API”可以跳转到相应的“RabbitMQ Management HTTP API”帮助页面，里面有详细的接口信息。

### rabbitmqadmin
rabbitmqadmin是RabbitMQ Management 插件提供的功能，它会包装HTTP API 接口，使其调用显得更加简洁方便。

rabbitmqadmin 是需要安装的，可以点击Web 管理页面左下角的“Command Line”跳转到“rabbitmqadmin”页面进行下载，或者通过下面的示例进行下载并添加可执行权限。在使用rabbitmqadmin前还要确保已经成功安装Python。

```bash
[root@node1 ~]# wget http://192.168.0.2:15672/cli/rabbitmqadmin
[root@node1 ~]# chmod +x rabbitmqadmin
```

<span style="color: red;font-weight: bold;">Tips</span>：通过rabbitmqadmin --help 命令可以获得相应的使用方式。

```bash
# 创建队列
[root@node1 ~]# ./rabbitmqadmin -u root -p root123 declare queue name=queue1
queue declared
# 显示队列
[root@node1 ~]# ./rabbitmqadmin list queues
+--------+----------+
| name   | messages |
+--------+----------+
| queue1 | 0        |
+--------+----------+
# 删除队列
[root@node1 ~]# ./rabbitmqadmin -u root -p root123 delete queue name=queue1
queue deleted
```


## RabbitMQ 配置

1. 环境变量（Enviroment Variables）：RabbitMQ 服务端参数可以通过环境变量进行配置，例如，节点名称、RabbitMQ 配置文件的地址、节点内部通信端口等。
2. 配置文件（Configuration File）：可以定义RabbitMQ 服务和插件设置，例如，TCP 监听端口，以及其他网络相关的设置、内存限制、磁盘限制等。
3. 运行时参数和策略（Runtime Parameters and Policies）：可以在运行时定义集群层面的服务设置。

### 环境变量
RabbitMQ 的环境变量都是以“RABBITMQ_”开头的，可以在Shell 环境中设置，  
也可以在rabbitmq-env.conf 这个RabbitMQ 环境变量的定义文件中设置。如果是在非Shell 环境中配置，则需要将“RABBITMQ_”这个前缀去除。  
优先级顺序按照Shell 环境最优先，其次rabbitmq-env.conf 配置文件，最后是默认的配置。

服务节点默认以“rabbit@”加上当前的Shell 环境的hostname（主机名）来命名，即rabbit@$HOSTNAME。  
如果需要制定节点的名称，可以在rabbitmq-server 命令前添加RABBITMQ_NODENAME 变量来设定指定的名称。  
如下所示，此时创建的节点名称为“rabbit@node2”而非“rabbit@node1”。

```bash
[root@node1 ~]# RABBITMQ_NODENAME=rabbit@node2 rabbitmq-server -detached
Warning: PID file not written; -detached was passed.
```

以RABBITMQ_NODENAME 这个变量为例，RabbitMQ 在启动服务的时候首先判断当前Shell环境中有无RABBITMQ_NODENAME 的定义，如果有则启用此值； 如果没有， 则查看rabbitmq-env.conf 中是否定义了NODENAME 这个变量，如果有则启用此值，如果没有则采用默认的取值规则，即rabbit@$HOSTNAME。

rabbitmq-env.conf 默认在$RABBITMQ_HOME/etc/rabbitmq/目录下，可以通过在启动RabbitMQ 服务时指定RABBITMQ_CONF_ENV_FILE 变量来设置此文件的路径。

对于默认的配置规则，在$RABBITMQ_HOME/sbin/rabbitmq-defaults 文件中有相关设置，不建议修改这个文件。

常见的RabbitMQ 变量（不仅限于这些变量）如下表：

变量名称 | 描 述
 :---- | :----
RABBITMQ_NODE_IP_ADDRESS | 绑定某个特定的网络接口。默认值是空字符串，即绑定到所有网络接口上。如果要绑定两个或者更多的网络接口，可以参考rabbitmq.config 中的tcp_listeners 配置
RABBITMQ_NODE_PORT | 监听客户端连接的端口号，默认为5672
RABBITMQ_DIST_PORT | RabbitMQ 节点内部通信的端口号，默认值为RABBITMQ_NODE_PORT+20000，即25672。如果设置了kernel.inet_dist_listen_min 或者kernel.inect_dist_listen_max时，此环境变量将被忽略
RABBITMQ_NODENAME | RabbitMQ 的节点名称，默认为rabbit@$HOSTNAME。在每个Erlang 节点和机器的组合中，节点名称必须唯一
RABBITMQ_CONF_ENV_FILE | RabbitMQ 环境变量的配置文件(rabbitmq-env.conf)的地址，默认值为$RABBITMQ_HOME/etc/rabbitmq/rabbitmq-env.conf（注意这里与RabbitMQ 配置文件rabbitmq.config 的区别）
RABBITMQ_USE_LONGNAME | 如果当前的hostname 为node1.longname，那么默认情况下创建的节点名称为rabbit@node1，将此参数设置为true 时，创建的节点名称就为rabbit@node1.longname，即使用了长名称命名。默认值为空
RABBITMQ_CONFIG_FILE | RabbitMQ 配置文件（rabbitmq.config）的路径，注意没有“.config”的后缀。默认值为$RABBITMQ_HOME/etc/rabbitmq/rabbitmq
RABBITMQ_MNESIA_BASE | RABBITMQ_MNESIA_DIR 的父目录。除非明确设置了RABBITMQ_MNESIA_DIR 目录，否则每个节点都应该配置这个环境变量。默认值为$RABBITMQ_HOME/var/lib/rabbitmq/mnesia（注意对于RabbitMQ 的操作用户来说，需要有对当前目录可读、可写、可创建文件及子目录的权限）
RABBITMQ_MNESIA_DIR | 包含RabbitMQ 服务节点的数据库、数据存储及集群状态等目录，默认值为$RABBITMQ_MNESIA_BASE/$RABBITMQ_NODENAME
RABBITMQ_LOG_BASE | RabbitMQ 服务日志所在基础目录。默认值为$RABBITMQ_HOME/var/log/rabbitmq
RABBITMQ_LOGS | RabbitMQ 服务与Erlang 相关的日志，默认值为$RABBITMQ_LOG_BASE/$RABBITMQ_NODENAME.log
RABBITMQ_SASL_LOGS | RabbitMQ 服务于Erlang 的SASL(System Application Support Libraries)相关的日志，默认值为$RABBITMQ_LOG_BASE/$RABBITMQ_NODENAME-sasl.log
RABBITMQ_PLUGINS_DIR | 插件所在路径。默认值为$RABBITMQ_HOME/plugins

注意，如果没有特殊的需求，不建议更改RabbitMQ 的环境变量。如果在实际生产环境中，对于配置和日志的目录有着特殊的管理目录，那么可以参考以下相应的配置：

```bash
#配置文件的地址，对于rabbitmq.config 文件来说这里不用添加“.config 后缀”
CONFIG_FILE=/apps/conf/rabbitmq/rabbitmq
#环境变量的配置文件的地址
CONF_ENV_FILE=/apps/conf/rabbitmq/rabbitmq-env.conf
#服务日志的地址
LOG_BASE=/apps/logs/rabbitmq
#Mnesia 的路径
MNESIA_BASE=/apps/dbdat/rabbitmq/mnesia
```

### 配置文件
默认的配置文件的位置取决于不同的操作系统和安装包。  
1. 最有效的方法就是检查RabbitMQ 的服务日志，在启动RabbitMQ 服务的时候会打印相关信息。如下所示，其中的“config file(s)”为目前的配置文件所在的路径。  
![config_files](../images/rabbitmq/2024-02-06_config文件路径.png ':size=50%')

在实际应用中，可能会遇到明明设置了相应的配置却没有生效的情况，也许是RabbitMQ 启动时并没有能够成功加载到相应的配置文件，可以查看日志中“config file(s)”是否有 **(not found)** 字样。  
如果看到有“not found”标识，那么可以检查“config file(s)”的路径中有没有相关的配置文件，或者检查配置文件的地址是否设置正确（ 通过RABBITMQ_CONFIG_FILE 变量或者rabbitmq-env.conf 文件设置）。  
如果rabbitmq.config 文件不存在，可以手动创建它。

2. 通过查看进程信息的方式来检查配置文件的位置。  
通过ps aux|grep rabbitmq 命令查看到RabbitMQ 进程的信息，如果rabbitmq.config 文件不处于默认的路径中，则会有 -config 选项标记正在使用的路径。如下所示：  
![config_progress](../images/rabbitmq/2024-02-06_通过进程查看config.png ':size=50%')

#### 配置项
一个极简的rabbitmq.config 文件配置如以下代码所示（注意包含尾部的点号），该配置将RabbitMQ 监听AMQP 0-9-1 客户端连接的默认端口号从5672 修改为5673：
```bash
[
    {
        rabbit, [
            {tcp_listeners, [5673]}
        ]
    }
].
```

<span style="color: red;font-weight: bold;">Tips</span>：https://github.com/rabbitmq/rabbitmq-server/blob/stable/docs/rabbitmq.config.example 中包含了大多数配置项与其相应的说明，可供参考。

下表展示了与RabbitMQ 服务相关的大部分配置项。如无特殊需要，不建议贸然修改这些默认配置。

配 置 项 | 描 述
:---- | :----
tcp_listeners |用来监听AMQP 连接（无SSL）。可以配置为端口号或者端口号与主机名组成的二元组。示例如下：<br>[{rabbit, [{tcp_listeners, [{"192.168.0.2", 5672}]}]}].或者[{rabbit, [{tcp_listeners, [{"127.0.0.1", 5672}, {"::1", 5672}]}]}]. 默认值为[5672]
num_tcp_acceptors | 用来处理TCP 连接的Erlang 进程数目，默认值为10
handshake_timeout | AMQP 0-8/0-9/0-9-1 握手（在 socket 连接和SSL 握手之后）的超时时间，单位为毫秒。默认值为10000
ssl_listeners | 同tcp_listeners，用于SSL 连接。默认值为[]
num_ssl_acceptors | 用来处理SSL 连接的Erlang 进程数目，默认值为1
ssl_options | SSL 配置。默认值为[]
ssl_handshake_timeout | SSL 的握手超时时间。默认值为5000
vm_memory_high_watermark | 触发流量控制的内存阈值。默认值为0.4
vm_memory_calculation_strategy | 内存使用的报告方式。一共有2 种，默认值为rss：<br>（1）rss：采用操作系统的RSS 的内存报告<br>（2）erlang：采用Erlang 的内存报告
vm_memory_high_watermark_paging_ratio | 内存高水位的百分比阈值，当达到阈值时，队列开始将消息持久化到磁盘以释放内存。这个需要配合vm_memory_ high_watermark 这个参数一起使用。默认值为0.5
disk_free_limit | RabbitMQ 存储数据分区的可用磁盘空间限制。当可用空间值低于阈值时，流程控制将被触发。此值可根据RAM 的相对大小来设置（如{mem_relative, 1.0}）。此值也可以设为整数（单位为B），或者使用数字+单位（如“50MB”）。默认情况下，可用磁盘空间必须超过50MB。默认值为50000000
log_levels | 控制日志的粒度。该值是日志事件类别（category）和日志级别（level）的二元组列表。<br>目前定义了4 种日志类别：<br>（1）channel：所有与AMQP 信道相关的日志<br>（2）connection：所有与连接相关的日志<br>（3）federation：所有与federation 相关的日志<br>（4）mirroring：所有与镜像相关的日志<br>其他未分类的日志也会被记录下来。<br>日志级别有5 种：<br>none 表示不记录日志事件<br>error 表示只记录错误；<br>warning 表示只记录错误和告警；<br>info 表示记录错误、告警和信息；<br>debug 表示记录错误、告警、信息和调试信息<br>默认值为[{connection, info}]
frame_max | 与客户端协商的允许最大帧大小，单位为B。设置为０表示无限制，但在某些QPid客户端会引发bug。设置较大的值可以提高吞吐量；设置一个较小的值可能会提高延迟。默认值为131072
channel_max | 与客户端协商的允许最大信道个数。设置为０表示无限制。该数值越大，则Broker 的内存使用就越高。默认值为0
channel_operation_timeout | 信道运行的超时时间，单位为毫秒（内部使用，因为消息协议的区别和限制，不暴露给客户端）。默认值为15000
heartbeat | 服务器和客户端连接的心跳延迟，单位为秒。如果设置为0，则禁用心跳。在有大量连接的情况下，禁用心跳可以提高性能，但可能会导致一些异常。默认值为60。在3.5.5 版本之前为580
default_vhost | 设置默认的vhost。交换器amq.rabbitmq.log 就在这个vhost 上默认值为<<"/">>
default_user | 设置默认的用户。默认值为<<"guest">>
default_pass | 设置默认的密码。默认值为<<"guest">>
default_user_tags | 设置默认用户的角色。默认值为[administrator]
default_permissions | 设置默认用户的权限。默认值为[<<".*">>, <<".*">>, <<".*">>]
loopback_users | 设置只能通过本地网络（如localhost）来访问Broker 的用户列表。如果希望通过默认的guest 用户能够通过远程网络访问Broker，那么需要将这项设置为[]。默认值为<<"guest">>
cluster_nodes | 可以用来配置集群。这个值是一个二元组，二元组的第一个元素是想要与其建立集群关系的节点，第二个元素是节点的类型，要么是disc，要么是ram。默认值为{[], disc}
server_properties | 连接时向客户端声明的键值对列表。默认值为[]
collect_statistics | 统计数据的收集模式，主要与RabbitMQ Management 插件相关，共有3 个值可选：<br>（1）none：不发布统计事件<br>（2）coarse：发布每个队列/信道/连接的统计事件<br>（3）fine：同时还发布每个消息的统计事件<br>默认值为none
collect_statistics_interval | 统计数据的收集时间间隔，主要与RabbitMQ Management 插件相关。默认值为5000
management_db_cache_multiplier | 设置管理插件将缓存代价较高的查询的时间。缓存将把最后一个查询的运行时间乘以这个值，并在此时间内缓存结果。默认值为5
delegate_count | 内部集群通信中，委派进程的数目。在拥有很多个内核并且是集群中的一个节点的机器上可以增加此值。默认值为16
tcp_listen_options | 默认的socket 选项。默认值为：<br>[{backlog, 128},{nodelay, true},{linger, {true,0}},{exit_on_close, false}]
hipe_compile | 将此项设置为true 就可以开启HiPE 功能，即Erlang 的即时编译器。虽然在启动时会增加时延，但是能够有20%～50%的性能提升，当然这个数字高度依赖于负载和机器硬件。在你的Erlang 安装包中可能没有包含HiPE 的支持，如果没有，则在开启这一项，并且在RabbitMQ 启动时会有相应的告警信息。HiPE 并非在所有平台都可以，尤其是Windows 操作系统。在Erlang/OTP17.5 版本之前，HiPE 有明显的问题。如果要使用HiPE 推荐使用最新版的Erlang/OTP。默认值为false
cluster_partition_handling | 如何处理网络分区。有4 种取值：<br>ignore；pause_minority；{pause_if_all_down, [nodes],ignore | autoheal}；autoheal<br>默认值为false
cluster_keepalive_interval | 向其他节点发送存活消息的频率。单位为毫秒。这个参数和net_ticktime 参数不同，丢失存活消息并不会导致节点被认为已失效。默认值为10000
queue_index_embed_msgs_below | 消息的大小小于此值时会直接嵌入到队列的索引中。单位为B。默认值为4096
msg_store_index_module | 队列索引的实现模块。默认值为rabbit_msg_store_ets_index
backing_queue_module | 队列内容的实现模块。默认值为rabbit_variable_queue。不建议修改此项
mnesia_table_loading_retry_limit |等待集群中Mnesia 数据表可用时最大的重试次数，默认值为10
mnesia_table_loading_retry_timeout | 每次重试时，等待集群中Mnesia 数据表可用时的超时时间，默认值为30000
queue_master_locator | 队列的定位策略，即创建队列时以什么策略判断坐落的Broker 节点。如果配置了镜像，则这里指master 镜像的定位策略。<br>可用的策略有：<<"min-masters">>、<<"client-local">>、<<"random">> <br>默认值为<<"client-local">>
lazy_queue_explicit_gc_run_operation_threshold | 在使用惰性队列（lazy queue）时进行内存回收动作的阈值。一个低的值会降低性能，一个高的值可以提高性能，但是会导致更高的内存消耗。默认值为1000
queue_explicit_gc_run_operation_threshold | 在使用正常队列时进行内存回收动作的阈值。一个低的值会降低性能，一个高的值可以提高性能，但是会导致更高的内存消耗。默认值为1000

#### 配置加密
配置文件中有一些敏感的配置项可以被加密，然后在RabbitMQ 启动时可以对这些项进行解密。  
加密并不是意味着系统的安全性增强了，而是遵从一些必要的规范，让一些敏感的数据不会出现在文本形式的配置文件中。
在配置文件中将加密之后的值以“{encrypted,加密的值}”形式包裹，比如下面的示例中使用口令“zzhpassphrase”将密码“guest”加密。

```bash
[{
    rabbit,[
        {default_user,<<"guest">>},
        {default_pass,
            {
                {encrypted,<<"HuVPYgSUdbogWL+2jGsgDMGZpDfiz+HurDuedpG8dQX/U+DMHcBluAl5a5jRnAbs+OviX5EmsJJ+c0XgRRcADA==">>}
            }
        },
        {loopback_users,[]},
        {config_entry_decoder,[
            {passphrase,<<"zzhpassphrase">>}
        ]}
    ]
}].
```

config_entry_decoder 项中的passphrase 配置的就是口令。  
这里将loopback_users 项配置为[]，就可以使用非本地网络访问RabbitMQ 了，如果开启了RabbitMQ Management 插件，就可以使用guest/guest 的用户及密码来访问Web 管理界面了。

passphrase 项中的内容不一定要以硬编码的形式呈现，还可以使用单独文件来赋值：

```bash
[
    {rabbit, [
        ...
        {config_entry_decoder, [
            {passphrase, {file, "/path/to/passphrase/file"}}
        ]}
    ]}
].
```

encrypted 项中加密后的值由rabbitmqctl encode 命令的来，如下所示：

```bash
# 加密过程
[root@node1 ~]# rabbitmqctl encode '<<"guest">>' zzhpassphrase
{encrypted,<<"HuVPYgSUdbogWL+2jGsgDMGZpDfiz+HurDuedpG8dQX/U+DMHcBluAl5a5jRnAbs+OviX5EmsJJ+c0XgRRcADA==">>}
# 解密过程
[root@node1 ~]# rabbitmqctl encode --decode '{encrypted,<<"HuVPYgSUdbogWL+2jGsgDMGZpDfiz+HurDuedpG8dQX/U+DMHcBluAl5a5jRnAbs+OviX5EmsJJ+c0XgRRcADA==">>}' zzhpassphrase
<<"guest">>
```

默认情况下，加密机制PBKDF2 用来从口令中派生出密钥。默认的Hash 算法是SHA512，默认的迭代次数是1000，以及默认的加密算法为AES_256_CBC。可以在配置文件中进行修改，示例如下：

```bash
[
    {rabbit, [
        ...
        {config_entry_decoder, [
            {passphrase, "zzhpassphrase"},
            {cipher, blowfish_cfb64},
            {hash, sha256},
            {iterations, 10000}
        ]}
    ]}
].
```

也可以通过rabbitmqctl encode 命令设置时指定：

```bash
rabbitmqctl encode --cipher blowfish_cfb64 --hash sha256 --iterations 10000 '<<"guest">>' zzhpassphrase
```

rabbitmqctl encode 的完整命令为：
> rabbitmqctl encode [--decode] [--list-ciphers] [--list-hashes] [--cipher \<cipher\>] [--hash \<hash\>] [--iterations \<iterations\>] [\<value\>] [\<passphrase\>]

[--list-ciphers] 、[--list-hashes]两个参数分别用来罗列当前RabbitMQ 所支持的加密算法和Hash 算法。

```bash
[root@node1 ~]# rabbitmqctl encode --list-ciphers
[des3_cbc,des_ede3,des3_cbf,des3_cfb,aes_cbc,aes_cbc128,aes_cfb8,aes_cfb128,aes_cbc256,aes_ige256,des_cbc,des_cfb,blowfish_cbc,blowfish_cfb64,blowfish_ofb64,rc2_cbc]

[root@node1 rabbitmq]# rabbitmqctl encode --list-hashes
[sha,sha224,sha256,sha384,sha512,md5]
```

#### 优化网络配置
网络是客户端和RabbitMQ 之间通信的媒介。RabbitMQ 支持的所有协议都是基于TCP 层面的。  
除了操作系统内核参数和DNS 的许多可调节参数，所有的RabbitMQ 设置都可以通过在rabbitmq.config 配置文件中配置来实现。

RabbitMQ 在等待接收客户端连接时需要绑定一个或者多个网络接口（可以理解成IP 地址），并监听特定的端口。网络接口使用rabbit.tcp_listeners 选项来配置。  
同时监听IPv4 和IPv6 上监听，示例如下：

```bash
[
    {rabbit, [
        {tcp_listeners, [
            {"127.0.0.1", 5672},
            {"::1", 5672}
        ]}
    ]}
].
```

优化网络配置的一个重要目标就是提高吞吐量，比如禁用Nagle 算法、增大TCP 缓冲区的大小。每个TCP 连接都分配了缓冲区。一般来说，缓冲区越大，吞吐量也会越高，但是每个连接上耗费的内存也就越多，从而使总体服务的内存增大，这是一个权衡的问题。在Linux 操作系统中，默认会自动调节TCP 缓冲区的大小，通常会设置为80KB 到120KB 之间。要提高吞吐量可以使用rabbit.tcp_listen_options 来加大配置。  
当连接数量到达数万或者更多时，重要的是确保服务器能够接受入站连接。未接受的TCP 连接将会放在有长度限制的队列中。这个通过rabbit.tcp_listen_options.backlog 参数来设置，默认值为128，当挂起的连接队列的长度超过此值时，连接将被操作系统拒绝。  
下面的示例中将TCP 缓冲区大小设置为192KB：

```bash
[
    {rabbit, [
        {tcp_listen_options, [
            {backlog, 128},
            {nodelay, true},
            {linger, {true,0}},
            {exit_on_close, false},
            {sndbuf, 196608},
            {recbuf, 196608}
        ]}
    ]}
].
```

Erlang 在运行时使用线程池来异步执行I/O 操作。线程池的大小可以通过RABBITMQ_SERVER_ADDITIONAL_ERL_ARGS 这个环境变量来调节。RabbitMQ 3.6.x 版本的默认值为128。当机器的内核个数大于等于8 时，建议将此值设置为大于等于96，这样可以确保每个内核上可以运行大于等于12 个I/O 线程。注意这个值并不是越高越能提高吞吐量。示例如下：

```bash
RABBITMQ_SERVER_ADDITIONAL_ERL_ARGS="+A 128"
```

大部分操作系统都限制了同一时间可以打开的文件句柄数。在优化并发连接数的时候，需确保系统有足够的文件句柄数来支撑客户端和Broker 的交互。可以用每个节点上连接的数目乘以1.5 来粗略的估算限制。例如，要支撑10 万个TCP 连接，需要设置文件句柄数为15 万。增加文件句柄数会增加闲置机器内存的使用量，但这需要合理权衡。

如上所述，增大TCP 缓冲区的大小可以提高吞吐量，如果减小TCP 缓冲区的大小，这样就可以减小每个连接上的内存使用量。

禁用Nagle 算法可以提高吞吐量，并减少延迟。RabbitMQ 内部节点交互时可以在kernel.inet_default_connect_options 、kernel.inet_default_listen_options 和rabbit.tcp_listen_options 配置项中配置{nodelay, true} 来禁用Nagle 算法。

```bash
[
    {kernel, [
        {inet_default_connect_options, [{nodelay, true}]},
        {inet_default_listen_options, [{nodelay, true}]}
    ]},
    {rabbit, [
        {tcp_listen_options, [
            {backlog, 4096},
            {nodelay, true},
            {linger, {true,0}},
            {exit_on_close, false}
        ]}
    ]}
].
```

通用的TCP 套接字选项：

参 数 项 | 描 述
:---- | :----
rabbit.tcp_listen_options.nodelay | 当设置为true，可禁用Nagle 算法。默认为true。对于大多数用户而言，推荐设置为true
rabbit.tcp_listen_options.sndbuf |参考前面讨论的TCP 缓冲区。一般取值范围在88KB 至128KB 之间。增大缓冲区可以提高消费者的吞吐量，同时也会加大每个连接上的内存使用量。减小则有相反的效果
rabbit.tcp_listen_options.recbuf | 参考前面讨论的TCP 缓冲区。一般取值范围同样在88KB 至128KB 之间。一般是针对发送者或者协议操作
rabbit.tcp_listen_options.backlog | 队列中未接受连接的最大数目。当达到此值时，新连接会被拒绝。对于成千上万的并发连接环境及可能存在大量客户重新连接的场景，可设为4096 或更高
rabbit.tcp_listen_options.linger | 当套接字关闭时，设置为{true, N}，用于设置刷新未发送数据的超时时间，单位为秒
rabbit.tcp_listen_options.keepalive | 当设置为true 时，启用TCP 的存活时间。默认为false。对于长时间空闲的连接（至少10 分钟）是有意义的，虽然更推荐使用heartbeat 的选项

下表是一些可配置的内核选项，注意这一类型的内核参数在/etc/sysctl.conf 文件（Linux 操作系统）中配置：

参 数 项 | 描 述
:---- | :----
fs.file-max | 内核分配的最大文件句柄数。极限值和当前值可以通过/proc/sys/fs/file-nr 来查看。示例如下：<br>[root@node1 ~]# cat /proc/sys/fs/file-nr<br>8480 0 798282
net.ipv4.ip_local_port_range | 本地IP 端口范围，定义为一对值。该范围必须为并发连接提供足够的条目
net.ipv4.tcp_tw_reuse | 当启用时，允许内核重用TIME_WAIT 状态的套接字。当用在NAT 时，此选项是很危险的
net.ipv4.tcp_fin_timeout | 降低此值到5～10 可减少连接关闭的时间，之后会停留在TIME_WAIT 状态，建议用在有大量并发连接的场景
net.core.somaxconn | 监听队列的大小（同一时间建立过程中有多少个连接）。默认为128。增大到4096 或更高，可以支持入站连接的爆发，如clients 集体重连
net.ipv4.tcp_max_syn_backlog | 尚未收到连接客户端确认的连接请求的最大数量。默认为128，最大值为65535。优化吞吐量时，4096 和8192 是推荐的起始值
net.ipv4.tcp_keepalive_* | net.ipv4.tcp_keepalive_time, net.ipv4.tcp_keepalive_intvl 和net.ipv4.tcp_keepalive_probes 用于配置TCP 存活时间
net.ipv4.conf.default.rp_filter | 启用反向地址过滤。如果系统不关心IP 地址欺骗，那么就禁用它

### 参数
RabbitMQ 绝大大多数的配置都可以通过修改rabbitmq.config 配置文件来完成，但是其中有些配置并不太适合在rabbitmq.config 中去实现。比如某项配置不需要同步到集群中的其他节点中，或者某项配置需要在运行时更改，因为rabbitmq.config 需要重启Broker 才能生效。这种类型的配置在RabbitMQ 中的另一种称呼为参数（Parameter），也可以称之为运行时参数（Runtime Parameter）。

Parameter 可以通过rabbitmqctl 工具或者RabbitMQ Management 插件提供的HTTP API 接口来设置。  
RabbitMQ 中一共有两种类型的Parameter：  
> vhost 级别的Parameter  
global 级别的Parameter  

vhost 级别的Parameter 由一个组件名称（component name）、名称（name）和值（value）组成，而global 级别的参数由一个名称和值组成，不管是vhost 级别还是global 级别的参数，其所对应的值都是JSON 类型的。

vhost 级别的参数对应的rabbitmqctl 相关的命令有三种：
> set_parameter  
list_parameters  
clear_parameter  

- rabbitmqctl set_parameter [-p vhost] {component_name} {name} {value}  
用来设置一个参数。

```bash
[root@node1 ~]# rabbitmq-plugins enable rabbitmq_federation
[root@node1 ~]# rabbitmqctl set_parameter federation-upstream f1 '{"uri":"amqp://root:root123@192.168.0.2:5672","ack-mode":"on-confirm"}'
Setting runtime parameter "f1" for component "federation-upstream" to "{\"uri\":\"amqp://root:root123@192.168.0.2:5672\",\"ack-mode\":\"on-confirm\"}"
```

- rabbitmqctl list_parameters [-p vhost]  
用来列出指定虚拟主机上所有的Parameter。

```bash
[root@node1 ~]# rabbitmqctl list_parameters -p /
Listing runtime parameters
federation-upstream f1 {"uri":"amqp://root:root123@192.168.0.2:5672","ackmode":"on-confirm"}
```

- rabbitmqctl clear_parameter [-p vhost] {componenet_name} {key}  
用来清除指定的参数。  

```bash
[root@node1 ~]# rabbitmqctl clear_parameter -p / federation-upstream f1
Clearing runtime parameter "f1" for component "federation-upstream"
```

配置vhost 级别的Parameter 的rabbitmqctl 工具相对应的HTTP API 接口如下：
> 设置一个参数：PUT /api/parameters/{componenet_name}/vhost/name  
清除一个参数：DELETE /api/parameters/{componenet_name}/vhost/name  
列出指定vhost 中的所有参数：GET /api/parameters  

global 级别的Parameter 的操作如下：

方 式 | rabbitmqctl | HTTP API 接口
:---- | :---- | :----
设置参数 | rabbitmqctl set_global_parameter name value | PUT /api/global-parameters/name
罗列参数 | rabbitmqctl list_global_parameters | GET /api/global-parameters/
清除参数 | rabbitmqctl clear_global_parameter name | DELETE /api/global-parameters/name

```bash
[root@node1 ~]# rabbitmqctl set_global_parameter name1 '{}'
Setting global runtime parameter "name1" to "{}"
[root@node1 ~]# rabbitmqctl list_global_parameters
Listing global runtime parameters
cluster_name    "rabbit@node1"
name1           []
[root@node1 ~]# rabbitmqctl clear_global_parameter name1
Clearing global runtime parameter "name1"
```

除了一些固定的参数（比如durable 或者exclusive），客户端在创建交换器或者队列的时候可以配置一些可选的属性参数来获得一些不同的功能，比如x-message-ttl、x-expires、x-max-length 等。  
通过客户端设定的这些属性参数一旦设置成功就不能再改变（不能修改也不能添加），除非删除原来的交换器或队列之后再重新创建新的。

### 策略
Policy可以解决参数不可改变的问题，，它是一种特殊的Parameter 的用法。Policy 是vhost 级别的。  
一个Policy 可以匹配一个或者多个队列（或者交换器，或者两者兼有），这样便于批量管理。  
Policy 也可以支持动态地修改一些属性参数，大大地提高了应用的灵活度。  
一般来说，Policy 用来配置Federation、镜像、备份交换器、死信等功能。

rabbitmq_managemet 插件本身就提供了Policy 的支持。可以在“Admin”->“Policies”->“Add / update a policy”中添加一个Policy。包含以下几个参数：
> Virtual host：表示当前Policy 所在的vhost 是哪个。  
Name：表示当前Policy 的名称。  
Pattern：一个正则表达式，用来匹配相关的队列或者交换器。  
Apply to：用来指定当前Policy 作用于哪一方。一共有三个选项，  
&emsp;&emsp;&emsp;&emsp;“Exchanges and queues”表示作用与Pattern 所匹配的所有队列和交换器；  
&emsp;&emsp;&emsp;&emsp;“Exchanges”表示作用于与Pattern 所匹配的所有交换器；  
&emsp;&emsp;&emsp;&emsp;“Queues”表示作用于与Pattern 所匹配的所有队列。  
Priority：定义优先级。如果有多个Policy 作用于同一个交换器或者队列，那么Priority 最大的那个Policy 才会有用。  
Definition：定义一组或者多组键值对，为匹配的交换器或者队列附加相应的功能。  

Policy 也可以通过rabbitmqctl 工具或者HTTP API 接口来操作。

- rabbitmqctl set_policy [-p vhost] [--priority priority] [--apply-to apply-to] {name} {pattern} {definition}  
用来设置一个Policy。

设置默认的vhost 中所有以“^amq.”开头的交换器为联邦交换器：

```bash
[root@node1 ~]# rabbitmqctl set_policy --apply-to exchanges --priority 1 p1 "^amq." '{"federation-upstream":"f1"}'

# 对应的HTTP API 接口调用
[root@node1 ~]# curl -i -u root:root123 -XPUT -d '{"pattern": "^amq\.","definition":{"federation-upstream":"f1"}, "priority": 1, "apply-to": "exchanges"}' 
http://192.168.0.2:15672/api/policies/%2F/p1
```

- rabbitmqctl list_policies [-p vhost]  
列出默认vhost 中所有的Policy。

```bash
[root@node1 ~]# rabbitmqctl list_policies
Listing policies
/ p1 exchanges ^amq. {"federation-upstream":"f1"} 1

# 对应的HTTP API 接口调用
[root@node1 ~]# curl -i -u root:root123 -XGET http://192.168.0.2:15672/api/policies/%2F
HTTP/1.1 200 OK
server: Cowboy
date: Mon, 21 Aug 2017 12:37:30 GMT
content-length: 125
content-type: application/json
vary: accept, accept-encoding, origin
Cache-Control: no-cache
[{"vhost":"/","name":"p1","pattern":"^amq\\.","apply-to":"exchanges","definition":{"federation-upstream":"f1"},"priority":1}]
```

- rabbitmqctl clear_policy [-p vhost] {name}  
清除指定的Policy。

```bash
[root@node1 ~]# rabbitmqctl clear_policy p1

# 对应的HTTP API 接口调用
[root@node1 ~]# curl -i -u root:root123 -XDELETE http://192.168.0.2:15672/api/policies/%2F/p1
```

<span style="color: red;font-weight: hold;">Tips</span>：如果两个或多个Policy 都作用到同一个交换器或者队列上，且这些Policy 的优先级都是一样的，则参数项最多的Policy 具有决定权。如果参数一样多，则最后添加的Policy 具有决定权。

