> RabbitMQ 版本为3.6.10

RabbitMQ 集群中的所有节点都会备份所有的元数据信息：

    队列元数据：队列的名称及属性；
    交换器：交换器的名称及属性；
    绑定关系元数据：交换器与队列或者交换器与交换器之间的绑定关系；
    vhost元数据：为vhost 内的队列、交换器和绑定提供命名空间及安全属性。

但是不会备份消息（通过特殊的配置比如镜像队列可以解决这个问题）。集群只会在单个节点（而不是在所有节点上）创建队列的进程并包含完整的队列信息（元数据、状态、内容）。这样只有队列的宿主节点，即所有者节点知道队列的所有信息，其他非所有者节点只知道队列的元数据和指向该队列存在的那个节点的指针。因此当集群节点崩溃时，该节点的队列进程和关联的绑定都会消失。附加在那些队列上的消费者也会丢失其所订阅的信息，并且任何匹配该队列绑定信息的新消息也都会消失。

## 搭建集群
RabbitMQ 集群对延迟非常敏感，应当只在本地局域网内使用。  
在广域网中不应该使用集群，而应该使用Federation 或者Shovel 来代替。

### 多机多节点配置
#### 集群配置流程
假设现在一共有三台物理主机，均已正确地安装了RabbitMQ，且主机名分别为node1、node2和node3。

1. 配置各个节点的hosts 文件，让各个节点都能互相识别对方的存在。  
比如在Linux 系统中可以编辑/etc/hosts 文件，在其上添加IP 地址与节点名称的映射信息：

```bash
192.168.0.2 node1
192.168.0.3 node2
192.168.0.4 node3
```

2. 编辑RabbitMQ 的cookie 文件，以确保各个节点的cookie 文件使用的是同一个值。cookie 相当于密钥令牌，集群中的RabbitMQ 节点需要通过交换密钥令牌以获得相互认证。  
可以读取node1 节点的cookie 值，然后将其复制到node2 和node3 节点中。cookie 文件默认路径为/var/lib/rabbitmq/.erlang.cookie 或者$HOME/.erlang.cookie。  
如果节点的密钥令牌不一致，那么在配置节点时就会有如下的报错：

```bash
[root@node2 ~]# rabbitmqctl join_cluster rabbit@node1
Clustering node rabbit@node2 with rabbit@node1
Error: unable to connect to nodes [rabbit@node1]: nodedown
DIAGNOSTICS
===========
attempted to contact: [rabbit@node1]
rabbit@node1:
* connected to epmd (port 4369) on node1
* epmd reports node 'rabbit' running on port 25672
* TCP connection succeeded but Erlang distribution failed
* Authentication failed (rejected by the remote node), please check the Erlang cookie    #报错原因
current node details:
- node name: 'rabbitmq-cli-53@node2'
- home dir: /root
- cookie hash: kLtTY75JJGZnZpQF7CqnYg==
```

3. 配置集群。配置集群有三种方式：
- 通过rabbitmqctl 工具配置；
- 通过rabbitmq.config 配置文件配置；
- 通过rabbitmq-autocluster1 插件配置。

其中rabbitmqctl 工具配置是最常用的方式。

首先启动node1、node2 和node3 这三个节点的RabbitMQ 服务。
```bash
[root@node1 ~]# rabbitmq-server –detached
[root@node2 ~]# rabbitmq-server -detached
[root@node3 ~]# rabbitmq-server -detached
```

这三个节点目前都是以独立节点存在的单个集群。通过rabbitmqctl cluster_status 命令来查看各个节点的状态。
```bash
[root@node1 ~]# rabbitmqctl cluster_status
Cluster status of node rabbit@node1
[{nodes,[{disc,[rabbit@node1]}]},
 {running_nodes,[rabbit@node1]},
 {cluster_name,<<"rabbit@node1">>},
 {partitions,[]},
 {alarms,[{rabbit@node1,[]}]}]
```

如果以node1 节点为基准，将node2 和node3 节点加入node1 节点的集群中。（这三个节点是平等的，如果想调换彼此的加入顺序也未尝不可。）  
以node2 节点加入node1 节点的集群为例，步骤如下：
```bash
[root@node2 ~]# rabbitmqctl stop_app
Stopping rabbit application on node rabbit@node2
[root@node2 ~]# rabbitmqctl reset
Resetting node rabbit@node2
[root@node2 ~]# rabbitmqctl join_cluster rabbit@node1
Clustering node rabbit@node2 with rabbit@node1
[root@node2 ~]# rabbitmqctl start_app
Starting node rabbit@node2
```

之后将node3 节点也加入node1 节点所在的集群中，这三个节点组成了一个完整的集群。  
在任意一个节点中执行rabbitmqctl cluster_status都可以看到如下的集群状态。
```bash
[{nodes,[{disc,[rabbit@node1,rabbit@node2,rabbit@node3]}]},
 {running_nodes,[rabbit@node1,rabbit@node2,rabbit@node3]},
 {cluster_name,<<"rabbit@node1">>},
 {partitions,[]},
 {alarms,[{rabbit@node1,[]},{rabbit@node2,[]} ,{rabbit@node3,[]}]}]
```

#### 集群节点关闭引发异常
在node2 节点上执行rabbitmqctl stop_app 命令来主动关闭RabbitMQ 应用。此时在node1 上看到的集群状态可以参考下方信息，可以看到在running_nodes 这一选项中已经没有了rabbit@node2 这一节点。
```bash
[{nodes,[{disc,[rabbit@node1,rabbit@node2,rabbit@node3]}]},
 {running_nodes,[rabbit@node1 ,rabbit@node3]},
 {cluster_name,<<"rabbit@node1">>},
 {partitions,[]},
 {alarms,[{rabbit@node1,[]} ,{rabbit@node3,[]}]}]
```

如果关闭了集群中的所有节点，则需要确保在启动的时候**最后关闭的那个节点是第一个启动的**。如果第一个启动的不是最后关闭的节点，那么这个节点会等待最后关闭的节点启动。这个等待时间是30 秒，如果没有等到，那么这个先启动的节点也会失败。  
在最新的版本中会有重试机制，默认重试10 次30 秒以等待最后关闭的节点启动。  
在重试失败之后，当前节点也会因失败而关闭自身的应用。

如果最后一个关闭的节点最终由于某些异常而无法启动，则可以通过rabbitmqctl forget_cluster_node 命令来将此节点剔出当前集群。  
如果集群中的所有节点由于某些非正常因素，比如断电而关闭，那么集群中的节点都会认为还有其他节点在它后面关闭，此时需要调用rabbitmqctl force_boot 命令来启动一个节点，之后集群才能正常启动。
```bash
[root@node2 ~]# rabbitmqctl force_boot
Forcing boot for Mnesia dir /opt/rabbitmq/var/lib/rabbitmq/mnesia/rabbit@node2
[root@node2 ~]# rabbitmq-server –detached
```

### 集群节点类型
在使用rabbitmqctl cluster_status 命令来查看集群状态时会有{nodes,[{disc,[rabbit@node1,rabbit@node2,rabbit@node3]}]这一项信息，其中的disc 标注了RabbitMQ 节点的类型。  
RabbitMQ 中的每一个节点，不管是单一节点系统或者是集群中的节点，都只会是以下两种类型之一：
- 内存节点
- 磁盘节点

内存节点将所有的队列、交换器、绑定关系、用户、权限和vhost的元数据定义都存储在内存中，而磁盘节点则将这些信息存储到磁盘中。  
单节点的集群中必然只有磁盘类型的节点，否则当重启RabbitMQ 之后，所有关于系统的配置信息都会丢失。  
不过在集群中，可以选择配置部分节点为内存节点，这样可以获得更高的性能。