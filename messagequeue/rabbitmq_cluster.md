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

在使用rabbitmqctl cluster_status 命令来查看集群状态时会有{nodes,[{disc,[rabbit@node1,rabbit@node2,rabbit@node3]}]}这一项信息，其中的disc 标注了RabbitMQ 节点的类型。  
RabbitMQ 中的每一个节点，不管是单一节点系统或者是集群中的节点，都只会是以下两种类型之一：

- 内存节点
- 磁盘节点

内存节点将所有的队列、交换器、绑定关系、用户、权限和vhost的元数据定义都存储在内存中，而磁盘节点则将这些信息存储到磁盘中。  
单节点的集群中必然只有磁盘类型的节点，否则当重启RabbitMQ 之后，所有关于系统的配置信息都会丢失。  
不过在集群中，可以选择配置部分节点为内存节点，这样可以获得更高的性能。

在将节点加入到集群中时，可以通过“--ram”参数指定节点的类型为内存节点：

```bash
[root@node2 ~]# rabbitmqctl join_cluster rabbit@node1 --ram
```

默认不添加“--ram”参数则表示此节点为磁盘节点。

如果集群已经搭建好了，那么也可以使用rabbitmqctl change_cluster_node_type {disc,ram}命令来切换节点的类型，其中disc 表示磁盘节点，而ram 表示内存节点。

```bash
[root@node2 ~]# rabbitmqctl stop_app
Stopping rabbit application on node rabbit@node2
# 将node2 节点由磁盘节点转变为内存节点
[root@node2 ~]# rabbitmqctl change_cluster_node_type ram
Turning rabbit@node2 into a disc node
[root@node2 ~]# rabbitmqctl start_app
Starting node rabbit@node2
[root@node2 ~]# rabbitmqctl cluster_status
Cluster status of node rabbit@node2
[{nodes,[{disc,[rabbit@node1]},{ram,[rabbit@node2]}]},
{running_nodes,[rabbit@node1,rabbit@node2]},
{cluster_name,<<"rabbit@node1">>},
{partitions,[]},
{alarms,[{rabbit@node1,[]},{rabbit@node2,[]}]}]
```

在集群中创建队列、交换器或者绑定关系这些操作，直到所有集群节点都成功提交元数据变更后才会返回。

###### 集群中磁盘节点的关键性

RabbitMQ 只要求在集群中至少有一个磁盘节点，所有其他节点可以是内存节点。  
当节点加入或者离开集群时，它们必须将变更通知到至少一个磁盘节点。  
如果只有一个磁盘节点，而且不凑巧的是它刚好崩溃了，那么集群可以继续发送或者接收消息，但是不能执行创建队列、交换器、绑定关系、用户，以及更改权限、添加或删除集群节点的操作了。  
如果集群中唯一的磁盘节点崩溃，集群仍然可以保持运行，但是直到将该节点恢复到集群前，你无法更改任何东西。所以在建立集群的时候应该保证有两个或者多个磁盘节点的存在。

<span style="color: red;font-weight: bold;">Tips</span>：在内存节点重启后，会连接到预先配置的磁盘节点，下载当前集群元数据的副本。当在集群中添加内存节点时，确保告知其所有的磁盘节点（内存节点唯一存储在本地磁盘的元数据信息就是集群中磁盘节点的地址）。只要内存节点可以找到至少一个磁盘节点，那么它就能在重启后重新加入集群中。

<span style="color: red;font-weight: bold;">Tips</span>：为了确保集群信息的可靠性，或者在不确定使用磁盘节点或者内存节点的时候，建议全部使用磁盘节点。

### 剔除单个节点

以node1、node2 和node3 组成的集群为例，有两种方式将node2 剥离出当前集群。  

1. 首先在node2 节点上执行rabbitmqctl stop_app 或者rabbitmqctl stop 命令来关闭RabbitMQ 服务。之后再在node1 节点或者node3 节点上执行rabbitmqctl forget_cluster_node rabbit@node2 命令将node2 节点剔除出去。这种方式适合node2 节点不再运行RabbitMQ 的情况。  

```bash
[root@node1 ~]# rabbitmqctl forget_cluster_node rabbit@node2
Removing node rabbit@node2 from cluster
```

在关闭集群中的每个节点之后，如果最后一个关闭的节点最终由于某些异常而无法启动，则可以通过rabbitmqctl forget_cluster_node 命令来将此节点剔除出当前集群。例如：

```bash
# 集群中节点按照node3、node2、node1 的顺序关闭
[root@node3 ~]# rabbitmqctl stop
Stopping and halting node rabbit@node3
[root@node2 ~]# rabbitmqctl stop
Stopping and halting node rabbit@node2
[root@node1 ~]# rabbitmqctl stop
Stopping and halting node rabbit@node1
# 若要启动集群，就要从node1 节点开始启动，假如node1 出现问题无法启动
# 可以在node2 节点中执行命令将node1 节点剔除出当前集群
[root@node2 ~]# rabbitmqctl forget_cluster_node rabbit@node1 --offline
```

上面在使用rabbitmqctl forget_cluster_node 命令的时候用到了“--offline”参数。如果不添加这个参数，就需要保证node2 节点中的RabbitMQ 服务处于运行状态。这种情况node2 无法先行启动，则“--offline”参数的添加让其可以在非运行状态下将node1 剥离出当前集群。

2. 在node2 上执行rabbitmqctl reset 命令。如果不是像上面由于启动顺序的缘故而不得不删除一个集群节点，建议采用这种方式。

```bash
[root@node2 ~]# rabbitmqctl stop_app
Stopping rabbit application on node rabbit@node2
[root@node2 ~]# rabbitmqctl reset
Resetting node rabbit@node2
[root@node2 ~]# rabbitmqctl start_app
Starting node rabbit@node2
```

如果从node2 节点上检查集群的状态，会发现它现在是独立的节点。同样在集群中剩余的节点node1 和node3 上看到node2 已不再是集群中的一部分了。

rabbitmqctl reset 命令将清空节点的状态，并将其恢复到空白状态。  
当重置的节点是集群中的一部分时，该命令也会和集群中的磁盘节点进行通信，告诉它们该节点正在离开集群。不然集群会认为该节点出了故障，并期望其最终能够恢复过来。

### 集群节点的升级

如果RabbitMQ 集群由**单独的一个节点**组成，只需关闭原来的服务，然后解压新的版本再运行即可。只要保留原节点的Mnesia 数据，然后解压新版本到相应的目录，再将新版本的Mnesia 路径指向保留的Mnesia 数据的路径（也可以直接复制保留的Mnesia 数据到新版本中相应的目录），最后启动新版本的服务即可。

如果RabbitMQ 集群由**多个节点**组成，升级步骤如下：

1. 关闭所有节点的服务，注意采用rabbitmqctl stop 命令关闭。
2. 保存各个节点的Mnesia 数据。
3. 解压新版本的RabbitMQ 到指定的目录。
4. 指定新版本的Mnesia 路径为步骤2 中保存的Mnesia 数据路径。
5. 启动新版本的服务，注意先重启原版本中最后关闭的那个节点。

<span style="color: red;font-weight: bold;">Tips</span>：在对不同版本升级的过程中，最好先测试两个版本互通的可能性，然后再在线上环境中实地操作。

###### 节点升级前的准备工作

首先要保存元数据，之后再关闭所有生产者并等待消费者消费完队列中的所有数据，紧接着关闭所有消费者，然后重新安装RabbitMQ 并重建元数据等。

### 单机多节点配置

在一台机器上部署多个RabbitMQ 服务节点，需要确保每个节点都有独立的名称、数据存储位置、端口号（包括插件的端口号）等。  
下面演示在主机名称为node1 的机器上创建一个由rabbit1@node1、rabbit2@node1 和rabbit3@node1 这三个节点组成RabbitMQ 集群。

首先需要确保机器上已经安装了Erlang 和RabbitMQ 的程序。  
其次，为每个RabbitMQ 服务节点设置不同的端口号和节点名称来启动相应的服务。
```bash
[root@node1 ~]# RABBITMQ_NODE_PORT=5672 RABBITMQ_NODENAME=rabbit1
rabbitmq-server -detached
[root@node1 ~]# RABBITMQ_NODE_PORT=5673 RABBITMQ_NODENAME=rabbit2
rabbitmq-server -detached
[root@node1 ~]# RABBITMQ_NODE_PORT=5674 RABBITMQ_NODENAME=rabbit3
rabbitmq-server –detached
```

如果开启了RabbitMQ Management 插件，就需要为每个服务节点配置一个对应插件端口号：
```bash
[root@node1 ~]# RABBITMQ_NODE_PORT=5672 RABBITMQ_NODENAME=rabbit1 RABBITMQ_SERVER_START_ARGS="-rabbitmq_management listener [{port,15672}]"
rabbitmq-server -detached
[root@node1 ~]# RABBITMQ_NODE_PORT=5673 RABBITMQ_NODENAME=rabbit2 RABBITMQ_SERVER_START_ARGS="-rabbitmq_management listener [{port,15673}]"
rabbitmq-server -detached
[root@node1 ~]# RABBITMQ_NODE_PORT=5674 RABBITMQ_NODENAME=rabbit3 RABBITMQ_SERVER_START_ARGS="-rabbitmq_management listener [{port,15674}]"
rabbitmq-server -detached
```

启动各节点服务之后，将rabbit2@node1 节点加入rabbit1@node1 的集群之中：
```bash
[root@node1 ~]# rabbitmqctl -n rabbit2@node1 stop_app
Stopping rabbit application on node rabbit2@node1
[root@node1 ~]# rabbitmqctl -n rabbit2@node1 reset
Resetting node rabbit2@node1
[root@node1 ~]# rabbitmqctl -n rabbit2@node1 join_cluster rabbit1@node1
Clustering node rabbit2@node1 with rabbit1@node1
[root@node1 ~]# rabbitmqctl -n rabbit2@node1 start_app
Starting node rabbit2@node1
```

按照rabbit2@node1 的操作将rabbit3@node1 也加入到集群中。然后通过 rabbitmqctl cluster_status 命令查看各个服务节点的集群状态：
```bash
[root@node1 ~]# rabbitmqctl -n rabbit1@node1 cluster_status
Cluster status of node rabbit1@node1
[{nodes,[{disc,[rabbit1@node1,rabbit2@node1,rabbit3@node1]}]},
{running_nodes,[rabbit3@node1,rabbit2@node1,rabbit1@node1]},
{cluster_name,<<"rabbit1@node1">>},
{partitions,[]},
{alarms,[{rabbit3@node1,[]},{rabbit2@node1,[]},{rabbit1@node1,[]}]}]
```


## 查看服务日志
RabbitMQ 日志中包含各种类型的事件，比如连接尝试、服务启动、插件安装及解析请求时的错误等。  
RabbitMQ 的日志默认存放在$RABBITMQ_HOME/var/log/rabbitmq 文件夹内。在这个文件夹内RabbitMQ 会创建两个日志文件： RABBITMQ_NODENAME-sasl.log 和RABBITMQ_NODENAME.log。

- SASL（System Application Support Libraries，系统应用程序支持库）是库的集合，作为Erlang-OTP 发行版的一部分。它们帮助开发者在开发Erlang 应用程序时提供一系列标准，其中之一是日志记录格式。所以当RabbitMQ 记录Erlang 相关信息时，它会将日志写入文件RABBITMQ_NODENAME-sasl.log 中。举例来说，可以在这个文件中找到Erlang 的崩溃报告，有助于调试无法启动的RabbitMQ 节点。

- 如果想查看RabbitMQ 应用服务的日志，则需要查阅RABBITMQ_NODENAME.log 这个文件，RabbitMQ 服务日志指的就是这个文件。

1. 启动阶段，日志包含了RabbitMQ 的版本号、Erlang 的版本号、RabbitMQ 服务节点名称、cookie 的hash 值、RabbitMQ 配置文件地址、内存限制、磁盘限制、默认账户guest 的创建及权限配置、插件信息、统计值信息等。  
在RabbitMQ 中，日志级别有none、error、warning、info、debug 这5 种，none 表示不输出日志。日志级别可以通过rabbitmq.config 配置文件中的log_levels 参数来进行设置，默认为[{connection, info}]。

2. 使用rabbitmqctl stop 命令，会将Erlang 虚拟机一同关闭，而rabbitmqctl stop_app 只关闭RabbitMQ 应用服务，在关闭的时候打印的日志会有区别。

<span style="color: red;font-weight: bold;">Tips</span>：在执行任何RabbitMQ 操作之前，打开一个新的窗口运行tail -f $RABBITMQ_HOME/var/log/rabbitmq/rabbit@$HOSTNAME.log -n 200 命令来实时查看相应操作所对应的服务日志。

**手工切换当前的日志**的命令：
```bash
rabbitmqctl rotate_logs .bak
```

之后可以看到在日志目录下会建立新的日志文件，并且将老的日志文件以添加“.bak”后缀的方式进行区分保存：
```bash
[root@node1 rabbitmq]# ls -al
-rw-r--r-- 1 root root 0 Jul 23 00:50 rabbit@node1.log
-rw-r--r-- 1 root root 22646 Jul 23 00:50 rabbit@node1.log.bak
-rw-r--r-- 1 root root 0 Jul 23 00:50 rabbit@node1-sasl.log
-rw-r--r-- 1 root root 0 Jul 23 00:50 rabbit@node1-sasl.log.bak
```

也可以使用Linux crontab 执行一个定时任务，以当前日期为后缀，每天执行一次切换日志的任务，这样在后面需要查阅日志的时候可以根据日期快速定位到相应的日志文件。

###### 通过程序化的方式查看日志
RabbitMQ 默认会创建一些交换器，其中amq.rabbitmq.log 就是用来收集RabbitMQ 日志的，集群中所有的服务日志都会发往这个交换器中。这个交换器的类型为topic，可以收集如前面所说的debug、info、warning 和error 这4 个级别的日志。  
创建4 个日志队列queue.debug、queue.info、queue.warning 和queue.error，分别采用debug、info、warning 和error 这4 个路由键来绑定amq.rabbitmq.log。如果要使用一个队列来收集所有级别的日志，可以使用“#”这个路由键。

集群情况下，会在任一节点创建交换器amq.rabbitmq.log ，它负责收集集群中所有节点的日志，但这些日志是互相交错的。  
对于日志的监控处理可以用客户端消费队列中的日志信息，也可以采用Logstash等第三方工具实现。  
接收服务日志示例程序：
```java
public class ReceiveLog {
    public static void main(String[] args) {
        try {
        //省略创建connection…详细内容
        Channel channelDebug = conncection.createChannel();
        Channel channelInfo = conncection.createChannel();
        Channel channelWarn = conncection.createChannel();
        Channel channelError = conncection.createChannel();
        //省略channel.basicQos(int prefetch_count);
        channelDebug.basicConsume("queue.debug", false, "DEBUG",
        new ConsumerThread(channelDebug));
        channelInfo.basicConsume("queue.info", false, "INFO",
        new ConsumerThread(channelInfo));
        channelWarn.basicConsume("queue.warning", false, "WARNING",
        new ConsumerThread(channelWarn));
        channelError.basicConsume("queue.error", false, "ERROR",
        new ConsumerThread(channelError));
        } catch (IOException e) {
            e.printStackTrace();
        } catch (TimeoutException e) {
            e.printStackTrace();
        }
    }

    public static class ConsumerThread extends DefaultConsumer {
        public ConsumerThread(Channel channel) {
            super(channel);
        }

        @Override
        public void handleDelivery(String consumerTag, Envelope envelope, 
        AMQP.BasicProperties properties, byte[] body) throws IOException {
            String log = new String(body);
            System.out.println("="+consumerTag+" REPORT====\n"+log);
            //对日志进行相应的处理
            getChannel().basicAck(envelope.getDeliveryTag(),false);
        }
    }
}
```


## 单节点故障恢复
单点故障是指集群中单个节点发生了故障，有可能会引起集群服务不可用、数据丢失等异常。配置数据节点冗余（镜像队列）可以有效地防止由于单点故障而降低整个集群的可用性、可靠性。

单节点故障包括：机器硬件故障、机器掉电、网络异常、服务进程异常。

- 单节点机器硬件故障包括机器硬盘、内存、主板等故障造成的死机，无法从软件角度来恢复。此时需要在集群中的其他节点中执行rabbitmqctl forget_cluster_node {nodename}命令来将故障节点剔除，其中nodename 表示故障机器节点名称。如果之前有客户端连接到此故障节点上，在故障发生时会有异常报出，此时需要将故障节点的IP 地址从连接列表里删除，并让客户端重新与集群中的节点建立连接，以恢复整个应用。如果此故障机器修复或者原本有备用机器，那么也可以选择性的添加到集群中。
- 当遇到机器掉电故障，需要等待电源接通之后重启机器。此时这个机器节点上的RabbitMQ 处于stop 状态，但是此时不要盲目重启服务，否则可能会引起网络分区。此时同样需要在其他节点上执行rabbitmqctl forget_cluster_node {nodename}命令将此节点从集群中剔除，然后删除当前故障机器的RabbitMQ 中的Mnesia 数据（相当于重置），然后再重启RabbitMQ 服务，最后再将此节点作为一个新的节点加入到当前集群中。
- 网线松动或者网卡损坏都会引起网络故障的发生。对于网线松动，无论是彻底断开，还是“藕断丝连”，只要它不降速，RabbitMQ 集群就没有任何影响。但是为了保险起见，建议先关闭故障机器的RabbitMQ 进程，然后对网线进行更换或者修复操作，之后再考虑是否重新开启RabbitMQ 进程。而网卡故障极易引起网络分区的发生，如果监控到网卡故障而网络分区尚未发生时，理应第一时间关闭此机器节点上的RabbitMQ 进程，在网卡修复之前不建议再次开启。
- 对于服务进程异常，如RabbitMQ 进程非预期终止，需要预先思考相关风险是否在可控范围之内。如果风险不可控，可以选择抛弃这个节点。一般情况下，重新启动RabbitMQ 服务进程即可。


## 集群迁移
在为集群扩容时，向集群中加入新的集群节点即可，不过新的机器节点中是没有队列创建的，只有后面新创建的队列才有可能进入这个新的节点中。  
迁移可以解决扩容遇到的问题，将旧的集群中的数据（包括元数据信息和消息）迁移到新的且容量更大的集群中即可。  
RabbitMQ 中的集群迁移更多的是用来解决集群故障不可短时间内修复而将所有的数据、客户端连接等迁移到新的集群中，以确保服务的可用性。

RabbitMQ 集群迁移包括元数据重建、数据迁移，以及与客户端连接的切换。

### 元数据重建
通过手工或者使用客户端重建元数据极其烦琐、低效，且时效性太差，不到万不得已不建议使用。  
高效的手段莫过于通过Web管理界面的方式重建，在Web 管理界面的首页最下面有如下图所示功能：  
![export_import_definitions](../images/rabbitmq/2024-03-29_export_import_definitions.png)  
可以在原集群上点击“Download broker definitions”按钮下载集群的元数据信息文件，此文件是一个JSON 文件，比如命名为metadata.json，其内部详细内容可以参考附录A。之后再在新集群上的Web 管理界面中点击“Upload broker definitions”按钮上传metadata.json 文件，如果导入成功则会跳转到成功页面，这样就迅速在新集群中创建了元数据信息。  
<span style="color: red;font-weight: bold;">Tips</span>：如果新集群有数据与metadata.json 中的数据相冲突，对于交换器、队列及绑定关系这类非可变对象而言会报错，而对于其他可变对象如Parameter、用户等则会被覆盖，没有发生冲突的则不受影响。如果过程中发生错误，则导入过程终止，导致metadata.json 中只有部分数据加载成功。

以上重建元数据的方式需要考虑三个问题。
1. 如果原集群突发故障，又或者开启RabbitMQ Management 插件的那个节点机器故障不可修复，就无法获取原集群的元数据metadata.json，这样元数据重建就无从谈起。  
解决方法可以采取一个通用的备份任务，在元数据有变更或者达到某个存储周期时将最新的metadata.json 备份至另一处安全的地方。这样在遇到需要集群迁移时，可以获取到最新的元数据。
2. 如果新旧集群的RabbitMQ 版本不一致时会出现异常情况，比如新建立了一个3.6.10 版本的集群，旧集群版本为3.5.7，这两个版本的元数据就不相同。  
一般情况下，RabbitMQ 是能够做到向下兼容的，在高版本的RabbitMQ 中可以上传低版本的元数据文件。然而如果在低版本中上传高版本的元数据文件就麻烦了。  
例如密码加密方式变了，可以简单地在Shell 控制台输入变更密码的方式来解决这个问题：
```bash
rabbitmqctl change_password {username} {new_password}
```

如果还是不能成功上传元数据，就说明还有其它变更的元数据定义。我们首先需要明确一个概念，就是对于用户、策略、权限这种元数据来说内容相对固定，且内容较少，手工重建的代价较小。而且在一个新集群中要能让Web 管理界面运作起来，本身就需要创建用户、设置角色及添加权限等。相反，集群中元数据最多且最复杂的要数队列、交换器和绑定这三项的内容，这三项内容还涉及其内容的参数设置，如果采用人工重建的方式代价太大，重建元数据的意义其实就在于重建队列、交换器及绑定这三项的相关信息。  
解决方法就是将低版本的metadata.json 文件中queues 这一项前面的所有内容（包括rabbit_version、users、vhosts、permissions、parameters、global_parameters、policies）覆盖掉高版本的metadata.json 文件中queues 项前面的内容，再尝试导入。

3. 如果采用上面的方法将元数据在新集群上重建，则所有的队列都只会落到同一个集群节点上，而其他节点处于空置状态，这样所有的压力将会集中到这单台节点之上。  
解决方法有两种方式，都是通过程序（或者脚本）的方式在新集群上建立元数据，而非简单地在页面上上传元数据文件而已。  
    - 第一种方式是通过HTTP API 接口创建相应的数据。这里需要节点名称的列表。
    - 第二种方式是随机连接集群中不同的节点的IP 地址，然后再创建队列。这里需要的是节点IP 地址列表。

第一种方式：
```java
/* 创建Queue, Exchange, Binding对象 */
public class Queue {
    //与附录A 中相关内容一一对应
    private String name;
    private String vhost;
    private Boolean durable;
    private Boolean auto_delete;
    private Map<String, Object> arguments;
    //省略Getter 和Setter 方法…
}
public class Exchange {
    private String name;
    private String vhost;
    private String type;
    private Boolean durable;
    private Boolean auto_delete;
    private Boolean internal;
    private Map<String, Object> arguments;
    //省略Getter 和Setter 方法…
}
public class Binding {
    private String source;
    private String vhost;
    private String destination;
    private String destination_type;
    private String routing_key;
    private Map<String, Object> arguments;
    //省略Getter 和Setter 方法…
}

/* 用Gson解析metadata.json */
private static List<Queue> queueList = new ArrayList<Queue>();
private static List<Exchange> exchangeList = new ArrayList<Exchange>();
private static List<Binding> bindingList = new ArrayList<Binding>();
private static void parseJson(String filename) {
    JsonParser parser = new JsonParser();
    try {
        JsonObject json = (JsonObject) parser.parse(new FileReader(filename));
        JsonArray jsonQueueArray = json.get("queues").getAsJsonArray();
        for (int i = 0; i < jsonQueueArray.size(); i++) {
            JsonObject subObject = jsonQueueArray.get(i).getAsJsonObject();
            Queue queue = parseQueue(subObject);
            queueList.add(queue);
        }
        JsonArray jsonExchangeArray = json.get("exchanges").getAsJsonArray();
        for(int i=0;i<jsonExchangeArray.size();i++) {
            JsonObject subObject = jsonExchangeArray.get(i).getAsJsonObject();
            Exchange exchange = parseExchange(subObject);
            exchangeList.add(exchange);
        }
        JsonArray jsonBindingArray = json.get("bindings").getAsJsonArray();
        for(int i=0;i<jsonBindingArray.size();i++) {
            JsonObject subObject = jsonBindingArray.get(i).getAsJsonObject();
            Binding binding = parseBinding(subObject);
            bindingList.add(binding);
        }
    } catch (FileNotFoundException e) {
        e.printStackTrace();
    }
}
//解析队列信息
private static Queue parseQueue(JsonObject subObject) {
    Queue queue = new Queue();
    queue.setName(subObject.get("name").getAsString());
    queue.setVhost(subObject.get("vhost").getAsString());
    queue.setDurable(subObject.get("durable").getAsBoolean());
    queue.setAuto_delete(subObject.get("auto_delete").getAsBoolean());
    JsonObject argsObject = subObject.get("arguments").getAsJsonObject();
    Map<String, Object> map = parseArguments(argsObject);
    queue.setArguments(map);
    return queue;
}
//解析交换器信息
private static Exchange parseExchange(JsonObject subObject) {
    //省略，具体参考parseQueue 方法进行推演
}
//解析绑定信息
private static Binding parseBinding(JsonObject subObject) {
    //省略，具体参考parseQueue 方法进行推演
}
//解析参数arguments 这一项内容
private static Map<String,Object> parseArguments(JsonObject argsObject){
    Map<String, Object> map = new HashMap<String, Object>();
    Set<Map.Entry<String, JsonElement>> entrySet = argsObject.entrySet();
    for (Map.Entry<String, JsonElement> mapEntry : entrySet) {
        map.put(mapEntry.getKey(), mapEntry.getValue());
    }
    return map;
}
//在解析完队列、交换器及绑定关系之后，只需要遍历queueList、exchangeList 和bindingList，然后调用HTTP API 创建相应的数据即可。

/* 创建元数据 */
private static final String ip = "192.168.0.2";
private static final String username = "root";
private static final String password = "root123";
private static final List<String> nodeList = new ArrayList<String>(){{
    add("rabbit@node1");
    add("rabbit@node2");
    add("rabbit@node3");
}};
//创建队列
private static Boolean createQueues() {
    try {
        for(int i=0;i<queueList.size();i++) {
            Queue queue = queueList.get(i);
            //注意将特殊字符转义，比如默认的vhost="/"，将其转成%2F
            String url = String.format("http://%s:15672/api/queues/%s/%s", ip,
            encode(queue.getVhost(),"UTF-8"), encode(queue.getName(),"UTF-8"));
            Map<String, Object> map = new HashMap<String, Object>();
            map.put("auto_delete", queue.getAuto_delete());
            map.put("durable", queue.getDurable());
            map.put("arguments", queue.getArguments());
            //随机挑选一个节点，并在此节点上创建相应的队列，可以解决集群内部队列分布不均匀的问题
            Collections.shuffle(nodeList);
            map.put("node", nodeList.get(0));
            String data = new Gson().toJson(map);
            System.out.println(url);
            System.out.println(data);
            httpPut(url,data,username,password);
        }
    } catch (UnsupportedEncodingException e) {
        e.printStackTrace();
        return false;
    } catch (IOException e) {
        e.printStackTrace();
        return false;
    }
    return true;
}
//创建交换器
private static Boolean createExchanges(){
    //省略，具体参考createQueues 方法进行推演，关键信息如url
    String url = String.format("http://%s:15672/api/exchanges/%s/%s",ip,
    encode(exchange.getVhost(),"UTF-8"),encode(exchange.getName(),"UTF-8"));
}
//创建绑定关系
private static Boolean createBindings(){
    //省略，具体参考createQueues 方法进行推演，关键信息如url
    String url = null;
    //绑定有两种：交换器与队列，交换器与交换器
    if (binding.getDestination_type().equals("queue")) {
        url = String.format("http://%s:15672/api/bindings/%s/e/%s/q/%s", ip, encode(binding.getVhost(),"UTF-8"),
        encode(binding.getSource(),"UTF-8"), encode(binding.getDestination(),"UTF-8"));
    } else {
        url = String.format("http://%s:15672/api/bindings/%s/e/%s/e/%s", ip, encode(binding.getVhost(),"UTF-8"),
        encode(binding.getSource(),"UTF-8"), encode(binding.getDestination(),"UTF-8"));
    }
}
// http Put
public static int httpPut(String url, String data, String username, String password) throws IOException {
    HttpClient client = new HttpClient();
    client.getState().setCredentials(AuthScope.ANY, new UsernamePasswordCredentials(username, password));
    PutMethod putMethod = new PutMethod(url);
    putMethod.setRequestHeader("Content-Type","application/json;charset=UTF-8");
    putMethod.setRequestEntity(new StringRequestEntity(data,"application/json","UTF-8"));
    int statusCode = client.executeMethod(putMethod);
    //System.out.println(statusCode);
    return statusCode;
}
public static int httpPost(String url, String data, String username, String password) throws IOException {
    //省略，具体参考httpPut 方法进行推演
}
```

第二种方式：
```java
/* 节点IP 地址列表 */
private static List<String> ipList = new ArrayList<String>(){{
add("192.168.0.2");
add("192.168.0.3");
add("192.168.0.4");
}};

//客户端通过连接不同的IP 地址来创建不同的connection 和channel，然后将channel存入一个缓冲池channelList。
//之后随机从channelList 中获取一个channel，再根据queueList 中的信息创建相应的队列
//每一个channel 对应一个connection，而每一个connection 又对应一个IP，这样串起来就能保证channelList 中不会遗留任何节点
/* 创建队列 */
private static void createQueuesNew(){
    List<Channel> channelList = new ArrayList<Channel>();
    List<Connection> connectionList = new ArrayList<Connection>();
    try {
        for (int i = 0; i < ipList.size(); i++) {
            String ip = ipList.get(i);
            ConnectionFactory connectionFactory = new ConnectionFactory();
            connectionFactory.setUsername(username);
            connectionFactory.setPassword(password);
            connectionFactory.setHost(ip);
            connectionFactory.setPort(5672);
            Connection connection = connectionFactory.newConnection();
            Channel channel = connection.createChannel();
            channelList.add(channel);
            connectionList.add(connection);
        }
        createQueueByChannel(channelList);
    } catch (IOException e) {
        e.printStackTrace();
    } catch (TimeoutException e) {
        e.printStackTrace();
    } finally {
        for (Connection connection : connectionList) {
            try {
                connection.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}
private static void createQueueByChannel(List<Channel> channelList) {
    for(int i=0;i<queueList.size();i++) {
        Queue queue = queueList.get(i);
        //随机获取相应的channel
        Collections.shuffle(channelList);
        Channel channel = channelList.get(0);
        try {
            Map<String, Object> mapArgs = queue.getArguments();
            //do something with mapArgs.
            channel.queueDeclare(queue.getName(), queue.getDurable(), false, queue.getAuto_delete(), mapArgs);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

### 数据迁移和客户端连接的切换
将生产者的客户端与原RabbitMQ 集群的连接断开，然后再与新的集群建立新的连接，这样就可以将新的消息流转入到新的集群中。

至于消费者客户端，一种是等待原集群中的消息全部消费完之后再将连接断开，然后与新集群建立连接进行消费作业。  
可以通过Web 页面查看消息是否消费完成。也可以通过rabbitmqctl list_queues name messages messages_ready messages_unacknowledged 命令来查看是否有未被消费的消息。  
当原集群服务不可用或者出现故障造成服务质量下降而需要迅速将消息流切换到新的集群中时，此时就不能等待消费完原集群中的消息，这里需要及时将消费者客户端的连接切换到新的集群中，那么在原集群中就会残留部分未被消费的消息，此时需要做进一步的处理。如果原集群损坏，可以等待修复之后将数据迁移到新集群中，否则会丢失数据。  
数据迁移的主要原理是先从原集群中将数据消费出来，然后存入一个缓存区中，另一个线程读取缓存区中的消息再发布到新的集群中，如此便完成了数据迁移的动作。作者将此命名为“RabbitMQ ForwardMaker”，可以自行编写一个小工具来实现这个功能。RabbitMQ 本身提供的Federation 和Shovel 插件都可以实现RabbitMQ ForwardMaker 的功能，确切地说Shovel 插件更贴近RabbitMQ ForwardMaker，不过自定义的RabbitMQ ForwardMaker 工具可以让迁移系统更加高效、灵活。

### 自动化迁移
要实现集群自动化迁移，需要在使用相关资源时就做好一些准备工作，方便在自动化迁移过程中进行无缝切换。  
与生产者和消费者客户端相关的是交换器、队列及集群的信息，如果这3 种类型的资源发生改变时需要让客户端迅速感知，以便进行相应的处理，则可以通过将相应的资源加载到ZooKeeper 的相应节点中，然后在客户端为对应的资源节点加入watcher 来感知变化，当然这个功能使用etcd或者集成到公司层面的资源配置中心中会更加标准、高效。  
etcd 是一个分布式一致性k-v 存储系统，可用于服务注册发现与共享配置。官网地址为https://coreos.com/etcd/

如图所示，将整个RabbitMQ 集群资源的使用分为三个部分：客户端、集群、ZooKeeper配置管理。  
![auto-transfer](../images/rabbitmq/2024-03-29_auto-transfer.png)  
在集群中创建元数据资源时都需要在ZooKeeper 中生成相应的配置，比如在cluster1 集群中创建交换器exchange1 之后，需要在/rmqNode/exchanges 路径下创建实节点exchange1，并赋予节点的数据内容为：

```bash
cluster=cluster1 #表示此交换器所在的集群名称
exchangeType=direct #表示此交换器的类型
vhost=vhost1 #表示此交换器所在的vhost
username=root #表示用户名
password=root123 #表示密码
```

同样，在cluster1 集群中创建队列queue1 之后，需要在/rmqNode/queues 路径下创建实节点queue1，并赋予节点的数据内容为：

```bash
cluster=cluster1
bindings=exchange1 #表示此队列所绑定的交换器
#如果有需要，也可以添加一些其他信息，比如路由键等
vhost=vhost1
username=root
password=root123
```

对应集群的数据在/rmqNode/clusters 路径下，比如cluster1 集群，其对应节点的数据内容包含IP 地址列表信息：

```bash
ipList=192.168.0.2,192.168.0.3,192.168.0.4 #集群中各个节点的IP 地址信息
```

客户端程序如果与其上的交换器或者队列进行交互，那么需要在相应的ZooKeeper 节点中添加watcher，以便在数据发生变更时进行相应的变更，从而达到自动化迁移的目的。

生产者客户端在发送消息之前需要先连接ZooKeeper，然后根据指定的交换器名称如exchange1 到相应的路径/rmqNode/exchanges 中寻找exchange1 的节点，之后再读取节点中的数据，并同时对此节点添加watcher。在节点的数据第一条“cluster=cluster1”中找到交换器所在的集群名称，然后再从路径/rmqNode/clusters 中寻找cluster1 节点，然后读取其对应的IP 地址列表信息。这样整个发送端所需要的连接串数据（IP 地址列表、vhost、username、password 等）都已获取，接下就可以与RabbitMQ 集群cluster1 建立连接然后发送数据了。

对于消费者客户端而言，同样需要连接ZooKeeper，之后根据指定的队列名称（queue1）到相应的路径/rmqNode/queues 中寻找queue1 节点，继而找到相应的连接串，然后与RabbitMQ 集群cluster1 建立连接进行消费。当然对/rmqNode/queues/queue1 节点的watcher 必不可少。

当cluster1 集群需要迁移到cluster2 集群时，首先需要将cluster1 集群中的元数据在cluster2 集群中重建。之后通过修改zk 的channel 和queue 的元数据信息，比如原cluster1 集群中有交换器exchange1、exchange2 和队列queue1、queue2，现在通过脚本或者程序将其中的“cluster=cluster1”数据修改为“cluster=cluster2”。客户端会立刻感知节点的变化，然后迅速关闭当前连接之后再与新集群cluster2 建立新的连接后生产和消费消息，在此切换客户端连接的过程中是可以保证数据零丢失的。迁移之后，生产者和消费者都会与cluster2 集群进行互通，此时原cluster1 集群中可能还有未被消费完的数据，此时需要使用RabbitMQ ForwardMaker 工具将cluster1 集群中未被消费完的数据同步到cluster2 集群中。

如果没有准备RabbitMQ ForwardMaker 工具，也不想使用Federation 或者Shovel 插件，那么在变更完交换器相关的ZooKeeper 中的节点数据之后，需要等待原集群中的所有队列都消费完全之后，再将队列相关的ZooKeeper 中的节点数据变更，进而使得消费者的连接能够顺利迁移到新的集群之上。可以通过下面的命令来查看是否有队列中的消息未被消费完：
```bash
rabbitmqctl list_queues -p / -q | awk '{if($2>0) print $0}'
```

<span style="color: red;font-weight: bold;">Tips</span>：上面的自动化迁移立足于将现有集群迁移到空闲的备份集群，如果有很多个正在运行的RabbitMQ 集群，为每个集群都配备一个空闲的备份集群无疑是一种资源的浪费。  
对此，可以采用“两两互备”或“以1备2”的方案解决这个难题。但是要注意多集群间互备的解决方案需要配套一个完备的实施系统，比如具备资源管理、执行下发、数据校对等功能，具体细节也需要仔细斟酌。


## 集群监控
集群监控可以提供运行时的数据为应用提供参考依据、迅速定位问题、提供预防及告警等功能，增强了整体服务的鲁棒性。  
RabbitMQ Management 插件就能提供一定的监控功能，Web 管理界面提供了很多的统计值信息：如发送速度、确认速度、消费速度、消息总数、磁盘读写速度、句柄数、Socket 连接数、Connection 数、Channel 数、内存信息等。虽然RabbitMQ Management 插件提供的监控页面是相对完善的，但是难以和公司内部系统平台关联。

### 通过HTTP API 接口提供监控数据
假设集群中一共有4 个节点node1、node2、node3 和node4，有一个交换器exchange 通过同一个路由键“rk”绑定了3 个队列queue1、queue2 和queue3。  
集群节点的信息可以通过/api/nodes 接口来获取。有关从/api/nodes 接口中获取到数据的结构可以参考附录B。

收集数据的代码示例：  
创建ClusterNode 类
```java
public class ClusterNode {
    private long diskFree;//磁盘空闲
    private long diskFreeLimit;
    private long fdUsed;//句柄使用数
    private long fdTotal;
    private long socketsUsed;//Socket 使用数
    private long socketsTotal;
    private long memoryUsed;//内存使用值
    private long memoryLimit;
    private long procUsed;//Erlang 进程使用数
    private long procTotal;
    @Override
    public String toString() {
        return "{disk_free="+diskFree+", disk_free_limit="+diskFreeLimit+", fd_used="+fdUsed+", "+
        "fd_total="+fdTotal+", sockets_used="+socketsUsed+", sockets_total="+socketsTotal+", "+
        "mem_used="+memoryUsed+", mem_limit="+memoryLimit+", proc_used="+procUsed+", proc_total="+procTotal+"}";
    }
    //省略Getter 和Setter 方法
}
```

封装HTTP GET
```java
public class HttpUtils {
    public static String httpGet(String url, String username, String password) throws IOException {
        HttpClient client = new HttpClient();
        client.getState().setCredentials(AuthScope.ANY, new UsernamePasswordCredentials(username, password));
        GetMethod getMethod = new GetMethod(url);
        int ret = client.executeMethod(getMethod);
        String data = getMethod.getResponseBodyAsString();
        System.out.println(data);
        return data;
    }
}
```

采集集群节点数据
```java
public static List<ClusterNode> getClusterData(String ip, int port, String username, String password) {
    List<ClusterNode> list = new ArrayList<ClusterNode>();
    String url = "http://" + ip + ":" + port + "/api/nodes";
    System.out.println(url);
    try {
        String urlData = HttpUtils.httpGet(url, username, password);
        parseClusters(urlData,list);
    } catch (IOException e) {
        e.printStackTrace();
    }
    System.out.println(list);
    return list;
}
private static void parseClusters(String urlData, List<ClusterNode> list) {
    JsonParser parser = new JsonParser();
    JsonArray jsonArray = (JsonArray) parser.parse(urlData);
    for(int i=0;i<jsonArray.size();i++) {
        JsonObject jsonObjectTemp = jsonArray.get(i).getAsJsonObject();
        ClusterNode cluster = new ClusterNode();
        cluster.setDiskFree(jsonObjectTemp.get("disk_free").getAsLong());
        cluster.setDiskFreeLimit(jsonObjectTemp. get("disk_free_ limit").
        getAsLong());
        cluster.setFdUsed(jsonObjectTemp.get("fd_used").getAsLong());
        cluster.setFdTotal(jsonObjectTemp.get("fd_total").getAsLong());
        cluster.setSocketsUsed(jsonObjectTemp.get("sockets_used"). getAsLong());
        cluster.setSocketsTotal(jsonObjectTemp.get("sockets_total").getAsLong());
        cluster.setMemoryUsed(jsonObjectTemp.get("mem_used").getAsLong());
        cluster.setMemoryLimit(jsonObjectTemp.get("mem_limit").getAsLong());
        cluster.setProcUsed(jsonObjectTemp.get("proc_used").getAsLong());
        cluster.setProcTotal(jsonObjectTemp.get("proc_total").getAsLong());
        list.add(cluster);
    }
}
```

对于交换器而言的数据采集可以调用/api/exchanges/vhost/name 接口，比如需要调用虚拟主机为默认的“/”、交换器名称为exchange 的数据，只需要使用HTTP GET 方法获取http://xxx.xxx.xxx.xxx:15672/api/exchanges/%2F/exchange 的数据即可。注意，这里需要将“/”进行HTML 转义成“%2F”，否则会出错。对应的数据内容可以参考下方：
```json
{
    "message_stats": {
        "publish_in_details": {
            "rate": 0.4//数据流入的速率
        },
        "publish_in": 9,//数据流入的总量
        "publish_out_details": {
            "rate": 1.2//数据流出的速率
        },
        "publish_out": 27//数据流出的总量
    },
    "outgoing": [],
    "incoming": [],
    "arguments": {},
    "internal": false,
    "auto_delete": false,
    "durable": true,
    "type": "direct",
    "vhost": "/",
    "name": "exchange"
}
```

采集交换器数据
```java
public class Exchange {
    private double publishInRate;
    private long publishIn;
    private double publishOutRate;
    private long publishOut;
    @Override
    public String toString() {
    return "{publish_in_rate=" + publishInRate + ", publish_in=" + publishIn +
    ", publish_out_rate=" + publishOutRate + ", publish_out=" + publishOut+"}";
    }
    //省略Getter 和Setter 方法
}
public class ExchangeMonitor {
    public static void main(String[] args) {
        try {
            getExchangeData("192.168.0.2", 15672, "root", "root123", "/", "exchange");
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    public static Exchange getExchangeData(String ip, int port, String username,
    String password, String vhost, String exchange) throws IOException {
        String url = "http://" + ip + ":" + port + "/api/exchanges/"
            + encode(vhost, "UTF-8") + "/" + encode(exchange, "UTF-8");
        System.out.println(url);
        String urlData = HttpUtils.httpGet(url, username, password);
        System.out.println(urlData);
        Exchange exchangeAns = parseExchange(urlData);
        System.out.println(exchangeAns);
        return exchangeAns;
    }
    //解析程序
    private static Exchange parseExchange(String urlData) {
        Exchange exchange = new Exchange();
        JsonParser parser = new JsonParser();
        JsonObject jsonObject = (JsonObject) parser.parse(urlData);
        JsonObject msgStats = jsonObject.get("message_stats").getAsJsonObject();
        double publish_in_details_rate = msgStats.get("publish_in_details").getAsJsonObject().get("rate").getAsDouble();
        double publish_out_details_rate = msgStats.get("publish_out_details").getAsJsonObject().get("rate").getAsDouble();
        long publish_in = msgStats.get("publish_in").getAsLong();
        long publish_out = msgStats.get("publish_out").getAsLong();
        exchange.setPublishInRate(publish_in_details_rate);
        exchange.setPublishOutRate(publish_out_details_rate);
        exchange.setPublishIn(publish_in);
        exchange.setPublishOut(publish_out);
        return exchange;
    }
}
```

对于队列而言的数据采集相关的接口为/api/queues/vhost/name，对应的数据结构可以参考下方内容。代码逻辑省略。
```json
{
    "consumer_details": [],
    "incoming": [],
    "deliveries": [],
    "messages_details": { "rate": 0 },
    "messages": 12,
    "messages_unacknowledged_details": { "rate": 0 },
    "messages_unacknowledged": 0,
    "messages_ready_details": { "rate": 0 },
    "messages_ready": 12,
    "reductions_details": { "rate": 0 },
    "reductions": 577759,
    "message_stats": { "publish_details": { "rate": 0 }, "publish": 12 },
    "node": "rabbit@node2",
    "arguments": {},
    "exclusive": false,
    "auto_delete": false,
    "durable": true,
    "vhost": "/",
    "name": "queue1",
    "message_bytes_paged_out": 0,
    "messages_paged_out": 0,
    "backing_queue_status": {
        "mode": "default", "q1": 0, "q2": 0,
        "delta": ["delta", "undefined", 0, 0, "undefined" ],
        "q3": 0, "q4": 12, "len": 12, "target_ram_count": "infinity",
        "next_seq_id": 12,
        "avg_ingress_rate": 0.0501007133625864,
        "avg_egress_rate": 0, "mirror_seen": 0,
        "avg_ack_ingress_rate": 0, "avg_ack_egress_rate": 0,
        "mirror_senders": 0
    },
    "head_message_timestamp": null,
    "message_bytes_persistent": 28,
    "message_bytes_ram": 28,
    "message_bytes_unacknowledged": 0,
    "message_bytes_ready": 28,
    "message_bytes": 28,
    "messages_persistent": 12,
    "messages_unacknowledged_ram": 0,
    "messages_ready_ram": 12,
    "messages_ram": 12,
    "garbage_collection": {
        "minor_gcs": 492, "fullsweep_after": 65535, "min_heap_size": 233,
        "min_bin_vheap_size": 46422, "max_heap_size": 0
    },
    "state": "running",
    "recoverable_slaves": [ "rabbit@node1" ],
    "synchronised_slave_nodes": [ "rabbit@node1" ],
    "slave_nodes": [ "rabbit@node1" ],
    "memory": 143272,
    "consumer_utilisation": null,
    "consumers": 0,
    "exclusive_consumer_tag": null,
    "policy": "policy1"
}
```

数据采集完之后并没有结束，采集程序通过定时调用HTTP API 接口获取JSON 数据，然后进行JSON 解析之后再进行持久化处理。对于这种基于时间序列的数据非常适合使用OpenTSDB来进行存储。监控管理系统可以根据用户的检索条件来从OpenTSDB 中获取相应的数据并展示到页面之中。监控管理系统本身还可以具备报表、权限管理等功能，同时也可以实时读取所采集的数据，对其进行分析处理，对于异常的数据需要及时报告给相应的人员。  
OpenTSDB：基于Hbase 的分布式的，可伸缩的时间序列数据库。主要用途就是做监控系统，比如收集大规模集群（包括网络设备、操作系统、应用程序）的监控数据并进行存储、查询。

### 通过客户端提供监控数据
除了HTTP API 接口可以提供监控数据，Java 版客户端（3.6.x 版本开始）中Channel 接口中也提供了两个方法来获取数据。方法定义如下：
```java
//查询队列中的消息个数，可以为监控消息堆积的情况提供数据
long messageCount(String queue) throws IOException;
//查询队列中的消费者个数，可以为监控消费者的情况提供数据
long consumerCount(String queue) throws IOException;
```

也可以通过连接的状态进行监控。Java 客户端中Connection 接口提供了addBlockedListener（BlockedListener listener）方法（用来监听连接阻塞信息）和addShutdownListener（ShutdownListener listener）方法（用来监听连接关闭信息）。  
监听Connection 的状态：
```java
try {
    Connection connection = connectionFactory.newConnection();
    connection.addShutdownListener(new ShutdownListener() {
        public void shutdownCompleted(ShutdownSignalException cause) {
            //处理并记录连接关闭事项
        }
    });
    connection.addBlockedListener(new BlockedListener() {
        public void handleBlocked(String reason) throws IOException {
            //处理并记录连接阻塞事项
        }
        public void handleUnblocked() throws IOException {
            //处理并记录连接阻塞取消事项
        }
    });
    Channel channel = connection.createChannel();
    long msgCount = channel.messageCount("queue1");
    long consumerCount = channel.consumerCount("queue1");
    //记录msgCount 和consumerCount
} catch (IOException e) {
    e.printStackTrace();
} catch (TimeoutException e) {
    e.printStackTrace();
}
```

自定义埋点数据：
```java
public static volatile int successCount = 0;//记录发送成功的次数
public static volatile int failureCount = 0;//记录发送失败的次数
//下面代码内容包含在某方法体内
try {
    channel.confirmSelect();
    channel.addReturnListener(new ReturnListener() {
        public void handleReturn(int replyCode, String replyText, String exchange, String routingKey, 
            AMQP.BasicProperties properties, byte[] body) throws IOException {
            failureCount++;
        }
    });
    channel.basicPublish("","",true,MessageProperties.PERSISTENT_TEXT_PLAIN, "msg".getBytes());
    if (channel.waitForConfirms() == true) {
        successCount++;
    } else {
        failureCount++;
    }
} catch (IOException e) {
    e.printStackTrace();
    failureCount++;
} catch (InterruptedException e) {
    e.printStackTrace();
    failureCount++;
}
```

上面的代码中只是简单地对successCount 和failureCount 进行累加操作，这里推荐引入metrics 工具（比如com.codahale.metrics.*）来进行埋点，同样的方式也可以统计消费者消费成功的条数和消费失败的条数。

### 检测RabbitMQ 服务是否健康
通过某些工具或方法可以检测RabbitMQ 进程是否在运行（如ps aux | grep rabbitmq），或者5672 端口是否开启（如telnet xxx.xxx.xxx.xxx 5672），但是这样依旧不能真正地评判RabbitMQ 是否还具备服务外部请求的能力。  
这里就需要使用AMQP 协议来构建一个类似于TCP 协议中的Ping 的检测程序。当这个测试程序与RabbitMQ 服务无法建立TCP 协议层面的连接，或者无法构建AMQP 协议层面的连接，再或者构建连接超时，则可判定RabbitMQ 服务处于异常状态而无法正常为外部应用提供相应的服务。

AMQP-ping 测试程序：
```java
/**
* AMQP-ping 测试程序返回的状态
*/
enum PING_STATUS{
    OK,//正常
    EXCEPTION//异常
}
public class AMQPPing {
    private static String host = "localhost";
    private static int port = 5672;
    private static String vhost = "/";
    private static String username = "guest";
    private static String password = "guest";
    /**
    * 读取rmq_cfg.properties 中的内容，如果没有配置相应的项则采用默认值
    */
    static {
        Properties properties = new Properties();
        try {
            properties.load(AMQPPing.class.getClassLoader().getResourceAsStream("rmq_cfg.properties"));
            host = properties.getProperty("host");
            port = Integer.valueOf(properties.getProperty("port"));
            vhost = properties.getProperty("vhost");
            username = properties.getProperty("username");
            password = properties.getProperty("password");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    /**
    * AMQP-ping 测试程序，如有IOException 或者TimeoutException，则说明RabbitMQ
    * 服务出现异常情况
    */
    public static PING_STATUS checkAMQPPing(){
        PING_STATUS ping_status = PING_STATUS.OK;
        ConnectionFactory connectionFactory = new ConnectionFactory();
        connectionFactory.setHost(host);
        connectionFactory.setPort(port);
        connectionFactory.setVirtualHost(vhost);
        connectionFactory.setUsername(username);
        connectionFactory.setPassword(password);
        Connection connection = null;
        Channel channel = null;
        try {
            connection = connectionFactory.newConnection();
            channel = connection.createChannel();
        } catch (IOException | TimeoutException e ) {
            e.printStackTrace();
            ping_status = PING_STATUS.EXCEPTION;
        } finally {
            if (connection != null) {
                try {
                    connection.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
        return ping_status;
    }
}
```

示例中涉及rmq_cfg.properties 配置文件，这个文件用来灵活地配置与RabbitMQ 服务的连接所需的连接信息，包括IP 地址、端口号、vhost、用户名和密码等。如果没有配置相应的项则可以采用默认的值。  
监控应用时，可以定时调用AMQPPing.checkAMQPPing()方法来获取检测信息，AMQPPing 类能够检测RabbitMQ 是否能够接收新的请求和构造AMQP 信道。

RabbitMQ Management 插件提供了/api/aliveness-test/vhost 的HTTP API 形式的接口，这个接口通过三个步骤来验证RabbitMQ 服务的健康性：

    （1）创建一个以“aliveness-test”为名称的队列来接收测试消息。
    （2）用队列名称，即“aliveness-test”作为消息的路由键，将消息发往默认交换器。
    （3）到达队列时就消费该消息，否则就报错。

这个HTTP API 接口背后的检测程序也称之为aliveness-test，其运行在Erlang 虚拟机内部，因此它不会受到网络问题的影响。如果在虚拟机外部，则网络问题可能会阻止外部客户端连接到RabbitMQ 的5672 端口。aliveness-test 程序不会删除创建的队列，对于频繁调用这个接口的情况，它可以避免数以千计的队列元数据事务对Mnesia 数据库造成巨大的压力。如果RabbitMQ 服务完好，调用/api/aliveness-test/vhost 接口会返回{"status":"ok"}，HTTP 状态码为200。

AlivenessTest 程序：
```java
/**
* AlivenessTest 程序返回的状态
* OK 表示健康，EXCEPTION 表示异常
*/
enum ALIVE_STATUS{
    OK,
    EXCEPTION
}
public class AlivenessTest {
    public static ALIVE_STATUS checkAliveness(String url, String username, String password){
        ALIVE_STATUS alive_status = ALIVE_STATUS.OK;
        HttpClient client = new HttpClient();
        client.getState().setCredentials(AuthScope.ANY, new UsernamePasswordCredentials(username, password));
        GetMethod getMethod = new GetMethod(url);
        String data = null;
        int ret = -1;
        try {
            ret = client.executeMethod(getMethod);
            data = getMethod.getResponseBodyAsString();
            if (ret != 200 || !data.equals("{\"status\":\"ok\"}")) {
                alive_status = ALIVE_STATUS.EXCEPTION;
            }
        } catch (IOException e) {
            e.printStackTrace();
            alive_status = ALIVE_STATUS.EXCEPTION;
        }
        return alive_status;
    }
}
//调用示例
//AlivenessTest.checkAliveness("http://192.168.0.2:15672/api/aliveness-test/%2F","root", "root123");
```

aliveness-test 程序配合前面的AMQPPing 程序一起使用可以从内部和外部这两个方面来全面地监控RabbitMQ 服务。  
HTTP API 接口中还有另外两个接口/api/healthchecks/node 和/api/healthchecks/node/node，这两个HTTP API 接口分别表示对当前节点或指定节点进行基本的健康检查，包括RabbitMQ 应用、信道、队列是否运行正常，是否有告警产生等。

### 元数据管理与监控
删除队列、删除交换器，或者修改了绑定信息，再或者是胡乱建立了一个队列绑定到现有的一个交换器中，同时又没有消费者订阅消费此队列，从而留下消
息堆积的隐患等都会对使用RabbitMQ 服务的业务应用造成影响。所以对于RabbitMQ 元数据的管理与监控也尤为重要。  
许多应用场景是在业务逻辑代码中创建相应的元数据资源（交换器、队列及绑定关系）并使用。对于排他的、自动删除的这类非高可靠性要求的元数据资源可以在一定程度上忽略元数据变更的影响。但是对于两个非常重要的且通过消息中间件交互的业务应用，在使用相应的元数据资源时最好进行相应的管控，如果一方或者其他方肆意变更所使用的元数据，必然对另一方造成不小的损失。管控的介入自然会降低消息中间件的灵活度，但是可以增强系统的可靠性。比如提供给业务方使用的用户只有可读和可写的权限。

RabbitMQ 中在创建元数据资源的时候是以一种声明的形式完成的：无则创建、有则不变，不过在对应的元数据存在的情况下，对其再次声明时使用不同的属性会报出相应的错误信息。  
可以利用这一特性来监控元数据的变更，通过定时程序来将记录中的元数据信息重新声明一次，查看是否有异常报出。不过这种方法非常具有局限性，只能增加元数据的信息而不能减少。

**下面提供一个简单的元数据管控和监控的思路：**  
所有的业务应用都需要通过元数据审核系统来申请变更（创建、查询、修改及删除）相应的元数据信息。在申请动作完成之后，由专门的人员进行审批，之后在数据库中存储和在RabbitMQ 集群中创建相应的元数据，这两个步骤可以同时进行，而且也无须为这两个动作添加强一致性的事务逻辑。在数据库和RabbitMQ 集群之间用一个元数据一致性校验程序来检测元数据不一致的地方，然后将不一致的数据上送到监控管理系统。监控管理系统中可以显示元数据不一致的记录信息，也可以以告警的形式推送出来，然后相应的管理人员可以选择手动或者自动地进行元数据修正。这里的不一致有可能是由于数据库的记录未被正确及时地更新，也有可能是RabbitMQ 集群中元数据被异常篡改。元数据修正需慎之又慎，在整个系统修正逻辑完备之前，建议优先采用人工的方式，毕竟不一致的元数据仅占少数，人工修正的工作量并不太大。

RabbitMQ 中主要的元数据是queues、exchanges 和bindings，可以分别建立三张表，参考附录A。
- Table 1：队列信息表，名称为rmq_queues。列名有name、vhost、durable、auto_delete、arguments、cluster_name、description。
- Table 2：交换器信息表，名称为rmq_exchanges。列名有name、vhost、type、durable、auto_delete、internal、arguments、cluster_name、description。
- Table 3：绑定信息表，名称为rmq_bindings。列名有source、vhost、destination、destination_type、routing_key、arguments、cluster_name、description。

元数据一致性检测程序可以通过/api/definitions 的HTTP API 接口获取集群的元数据信息，通过解析之后与数据库中的记录一一比对，查看是否有不一致的地方。


---

**RabbitMQ 可以通过3 种方式实现分布式部署：集群、Federation 和Shovel。这3 种方式不是互斥的，可以根据需要选择其中的一种或者以几种方式的组合来达到分布式部署的目的。Federation 和Shovel 可以为RabbitMQ 的分布式部署提供更高的灵活性，但同时也提高了部署的复杂性。**

## Federation
Federation 插件的设计目标是使RabbitMQ 在不同的Broker 节点之间进行消息传递而无须建立集群，该功能在很多场景下都非常有用：
- Federation 插件能够在不同管理域（可能设置了不同的用户和vhost，也可能运行在不同版本的RabbitMQ 和Erlang 上）中的Broker 或者集群之间传递消息。
- Federation 插件基于AMQP 0-9-1 协议在不同的Broker 之间进行通信，并设计成能够容忍不稳定的网络连接情况。
- 一个Broker 节点中可以同时存在联邦交换器（或队列）或者本地交换器（或队列），只需要对特定的交换器（或队列）创建Federation 连接（Federation link）。
- Federation 不需要在N 个Broker 节点之间创建O(N2)个连接（尽管这是最简单的使用方式），这也就意味着Federation 在使用时更容易扩展。

Federation 插件可以让多个交换器或者多个队列进行联邦。一个联邦交换器（federated exchange）或者一个联邦队列（federated queue）接收上游（upstream）的消息，这里的上游是指位于其他Broker 上的交换器或者队列。联邦交换器能够将原本发送给上游交换器（upstream exchange）的消息路由到本地的某个队列中；联邦队列则允许一个本地消费者接收到来自上游队列（upstream queue）的消息。

### 联邦交换器
假设有broker1 部署在北京，broker2 部署在上海，而broker3 部署在广州，彼此之间相距甚远，网络延迟是一个不得不面对的问题。  
有一个在广州的业务ClientA 需要连接brokerA，并向其中的交换器exchangeA 发送消息，此时的网络延迟很小，ClientA 可以迅速将消息发送至exchangeA 中，就算在开启了publisher confirm 机制或者事务机制的情况下，也可以迅速收到确认信息。此时又有一个在北京的业务ClientB 需要向exchangeA 发送消息，那么ClientB 与broker3 之间有很大的网络延迟，ClientB将发送消息至exchangeA 会经历一定的延迟，尤其是在开启了publisher confirm 机制或者事务机制的情况下，ClientB 会等待很长的延迟时间来接收broker3 的确认信息，进而必然造成这条发送线程的性能降低，甚至造成一定程度上的阻塞。

**使用Federation 插件就以解决以上问题**  
如下图所示，在broker3 中为交换器exchangeA（broker3 中的队列queueA 通过“rkA”与exchangeA 进行了绑定）与北京的broker1 之间建立一条单向的Federation link。此时Federation 插件会在broker1 上会建立一个同名的交换器exchangeA（这个名称可以配置，默认同名），同时建立一个内部的交换器“exchangeA→broker3 B”，并通过路由键“rkA”将这两个交换器绑定起来。这个交换器“exchangeA→broker3 B”名字中的“broker3”是集群名，可以通过rabbitmqctl set_cluster_name {new_name}命令进行修改。与此同时Federation 插件还会在broker1 上建立一个队列“federation: exchangeA→broker3 B”，并与交换器“exchangeA→broker3 B”进行绑定。Federation 插件会在队列“federation: exchangeA→broker3 B”与broker3 中的交换器exchangeA 之间建立一条AMQP 连接来实时地消费队列“federation: exchangeA→broker3 B”中的数据。这些操作都是内部的，对外部业务客户端来说这条Federation link 建立在broker1 的exchangeA 和broker3 的exchangeA 之间。  
![federation_link](../images/rabbitmq/2024-03-30_federation_link.png) _建立Federation link_  
部署在北京的业务ClientB 可以连接broker1 并向exchangeA 发送消息，这样ClientB 可以迅速发送完消息并收到确认信息，而之后消息会通过Federation link 转发到broker3 的交换器exchangeA 中。最终消息会存入broker3 的与exchangeA 绑定的队列queueA 中，消费者最终可以消费队列queueA 中的消息。  
经过Federation link 转发的消息会带有特殊的headers 属性标记。例如向broker1 中的交换器exchangeA 发送一条内容为“federation test payload.”的持久化消息，之后可以在broker3 中的队列queueA 中消费到这条消息，如下图：  
![federation_headers](../images/rabbitmq/2024-03-30_federation_headers.png)

上图 _建立Federation link_ 中broker1 的队列“federation: exchangeA -> broker3 B”是一个相对普通的队列，可以直接通过客户端进行消费。假设此时还有一个客户端ClientD 通过Basic.Consume 来消费队列“federation: exchangeA→broker3 B”的消息，那么发往broker1 中exchangeA 的消息会有一部分（一半）被ClientD 消费掉，而另一半会发往broker3 的exchangeA。所以如果业务应用有要求所有发往broker1 中exchangeA 的消息都要转发至broker3 的exchangeA 中，此时就要注意队列“federation: exchangeA→broker3 B”不能有其他的消费者；而对于“异地均摊消费”这种特殊需求，队列“federation: exchangeA→broker3 B”这种天生特性提供了支持。对于broker1 的交换器exchangeA 而言，它是一个普通的交换器，可以创建一个新的队列绑定它，对它的用法没有什么特殊之处。

一个federated exchange 同样可以成为另一个交换器的upstream exchange。例如将broker2 作为broker1 的联邦交换器，再将broker3 作为broker2 的联邦交换器，此时broker2 即时联邦交换器又是上游交换器。  
两方的交换器可以互为federated exchange 和upstream exchange。其中参数“max_hops=1”表示一条消息最多被转发的次数为1。如下图：  
![mutual_federated_exchange](../images/rabbitmq/2024-03-30_mutual_federated_exchange.png) _两方的交换器互为federated exchange 和upstream exchange_

对于联邦交换器而言，还有更复杂的拓扑逻辑部署方式。如下图：  
![federation_complex_topology](../images/rabbitmq/2024-03-30_federation_complex_topology.png)

<span style="color: red;font-weight: bold;">Tips</span>：对于默认的交换器（每个vhost 下都会默认创建一个名为“”的交换器）和内部交换器而言，不能对其使用Federation 的功能。

### 联邦队列
联邦队列（federated queue）可以在多个Broker 节点（或者集群）之间为单个队列提供均衡负载的功能。一个联邦队列可以连接一个或者多个上游队列（upstream queue），并从这些上游队列中获取消息以满足本地消费者消费消息的需求。

如下图所示，位于两个Broker 中的几个联邦队列（灰色）和非联邦队列（白色）。队列queue1 和queue2 原本在broker2 中，由于某种需求将其配置为federated queue 并将broker1 作为upstream。Federation 插件会在broker1 上创建同名的队列queue1 和queue2，与broker2 中的队列queue1 和queue2 分别建立两条单向独立的Federation link。当有消费者ClinetA 连接broker2 并通过Basic.Consume 消费队列queue1（或queue2）中的消息时，如果队列queue1（或queue2）中本身有若干消息堆积，那么ClientA 直接消费这些消息，此时broker2 中的queue1（或queue2）并不会拉取broker1 中的queue1（或queue2）的消息；如果队列queue1（或queue2）中没有消息堆积或者消息被消费完了，那么它会通过Federation link 拉取在broker1 中的上游队列queue1（或queue2）中的消息（如果有消息），然后存储到本地，之后再被消费者ClientA 进行消费。  
![_federated_queue](../images/rabbitmq/2024-03-30_federated_queue.png)

消费者既可以消费broker2 中的队列，又可以消费broker1 中的队列，Federation 的这种分布式队列的部署可以提升单个队列的容量。如果在broker1 一端部署的消费者来不及消费队列queue1 中的消息，那么broker2 一端部署的消费者可以为其分担消费，也可以达到某种意义上的负载均衡。

与federated exchange 不同，如果两个队列互为联邦队列，一条消息可以在联邦队列间转发无限次。  
队列中的消息除了被消费，还会转向有多余消费能力的一方，如果这种“多余的消费能力”在broker1 和broker2 中来回切换，那么消费也会在broker1 和broker2 中的队列queue 中来回转发。  
可以在其中一个队列上发送一条消息“msg”，然后再分别创建两个消费者ClientB 和ClientC 分别连接broker1 和broker2，并消费队列queue 中的消息，但是并不需要确认消息（消费完消息不需要调用Basic.Ack）。来回开启/关闭ClientB 和ClientC 可以发现消息“msg”会在broker1 和broker2 之间串来串去。

如下图所示，broker2 的队列queue 没有消息堆积或者消息被消费完之后并不能通过Basic.Get 来获取broker1 中队列queue 的消息。因为Basic.Get 是一个异步的方法，如果要从broker1 中队列queue 拉取消息，必须要阻塞等待通过Federation link 拉取消息存入broker2 中的队列queue 之后再消费消息，所以对于federated queue 而言只能使用Basic.Consume 进行消费。

![federated_queue_transitivity](../images/rabbitmq/2024-03-30_federated_queue_transitivity.png)

federated queue 并不具备传递性。如上图所示，队列queue2 作为federated queue 与队列queue1 进行联邦，而队列queue2 又作为队列queue3 的upstream queue，但是这样队列queue1 与queue3 之间并没有产生任何联邦的关系。如果队列queue1 中有消息堆积，消费者连接broker3 消费queue3 中的消息，无论queue3 处于何种状态，这些消费者都消费不到queue1 中的消息，除非queue2 有消费者。

<span style="color: red;font-weight: bold;">Tips</span>：理论上可以将一个federated queue 与一个federated exchange 绑定起来，不过这样会导致一些不可预测的结果，如果对结果评估不足，建议慎用这种搭配方式。

### Federation 的使用
为了能够使用Federation 功能，需要配置以下2 个内容：
1. 需要配置一个或多个upstream，每个upstream 均定义了到其他节点的Federation link。这个配置可以通过设置运行时的参数（Runtime Parameter）来完成，也可以通过federation management 插件来完成。
2. 需要定义匹配交换器或者队列的一种/多种策略（Policy）。

Federation 插件默认在RabbitMQ 发布包中， 执行rabbitmq-plugins enable rabbitmq_federation 命令可以开启Federation 功能，因为Federation 内部基于AMQP 协议拉取数据，所以在开启rabbitmq_federation 插件的时候，默认会开启amqp_client 插件。示例如下：
```bash
[root@node1 ~]# rabbitmq-plugins enable rabbitmq_federation
The following plugins have been enabled:
  amqp_client
  rabbitmq_federation
Applying plugin configuration to rabbit@node1... started 2 plugins.
```

同时，如果要开启Federation 的管理插件，需要执行rabbitmq-plugins enable rabbitmq_federation_management 命令，示例如下：
```bash
[root@node1 ~]# rabbitmq-plugins enable rabbitmq_federation_management
The following plugins have been enabled:
  cowlib
  cowboy
  rabbitmq_web_dispatch
  rabbitmq_management_agent
  rabbitmq_management
  rabbitmq_federation_management
Applying plugin configuration to rabbit@node1... started 6 plugins.
```

开启rabbitmq_federation_management 插件之后，在RabbitMQ 的管理界面中“Admin”的右侧会多出“Federation Status”和“Federation Upstreams”两个Tab 页。  
rabbitmq_federation_management 插件依附于rabbitmq_management 插件，所以开启rabbitmq_federation_management 插件的同时默认也会开启rabbitmq_management 插件。

<span style="color: red;font-weight: bold;">Tips</span>：当需要在集群中使用Federation 功能的时候，集群中所有的节点都应该开启Federation 插件。

有关Federation upstream 的信息全部都保存在RabbitMQ 的Mnesia 数据库中，包括用户信息、权限信息、队列信息等。在Federation 中存在三种级别的配置：
1. Upstreams：每个upstream 用于定义与其他Broker 建立连接的信息。
2. Upstream sets：每个upstream set 用于对一系列使用Federation 功能的upstream 进行分组。
3. Policies：每一个Policy 会选定出一组交换器，或者队列，亦或者两者皆有而进行限定，进而作用于一个单独的upstream 或者upstream set 之上。

在简单使用场景下，基本上可以忽略upstream set 的存在，因为存在一种名为“all”并且隐式定义的upstream set，所有的upstream 都会添加到这个set 之中。Upstreams 和Upstream sets 都属于运行时参数，就像交换器和队列一样，每个vhost 都持有不同的参数和策略的集合。

Federation 相关的运行时参数和策略都可以通过下面三种方式进行设置：
1. 通过rabbitmqctl 工具。
2. 通过RabbitMQ Management 插件提供的HTTP API 接口。
3. 通过rabbitmq_federation_management 插件提供的Web 管理界面的方式（最方便且通用）。不过基于Web 管理界面的方式不能提供全部功能，比如无法针对upstream set 进行管理。

---

下面就详细讲解如何正确地使用Federation 插件，以章节Federation - 联邦交换器 - _建立Federation link_ 这张图为例，讲述如何在broker1（IP 地址：192.168.0.2）和broker3（IP 地址：192.168.0.4）之间**建立federated exchange** 关系。  
##### 第一步
需要在broker1 和broker3 中开启rabbitmq_federation 插件，最好同时开启rabbitmq_federation_management 插件。

##### 第二步
在broker3 中定义一个upstream。

**第一种**，通过rabbitmqctl 工具的方式：
```bash
rabbitmqctl set_parameter federation-upstream f1 '{"uri":"amqp://root:root123@192.168.0.2:5672","ack-mode":"on-confirm"}'
```

**第二种**，通过调用HTTP API 接口的方式：
```bash
curl -i -u root:root123 -XPUT -d '{"value":{"uri":"amqp://root:root123@192.168.0.2:5672","ack-mode":"on-confirm"}}'
 http://192.168.0.4:15672/api/parameters/federation-upstream/%2f/f1
```

**第三种**，通过在Web 管理界面中添加的方式，在“Admin”→“Federation Upstreams”→“Add a new upstream”中创建。  
各个参数的含义如下，括号中对应的是采用设置Runtime Parameter 或者调用HTTP API 接口的方式所对应的相关参数名称。  
通用的参数如下：
- Name：定义这个upstream 的名称。必填项。  
- URI (uri) ： 定义upstream 的AMQP 连接。必填项。本示例中可以填写为amqp://root:root123@192.168.0.2:5672。  
- Prefetch count (prefetch_count)：定义Federation 内部缓存的消息条数，即在收到上游消息之后且在发送到下游之前缓存的消息条数。  
- Reconnect delay (reconnect-delay)：Federation link 由于某种原因断开之后，需要等待多少秒开始重新建立连接。  
- Acknowledgement Mode (ack-mode)：定义Federation link 的消息确认方式。共有三种：  
    ▶ on-confirm，默认方式，表示在接收到下游目标节点的确认消息（等待下游的Basic.Ack）之后再向上游发送消息确认，这个选项可以确保网络失败或者Broker 宕机时不会丢失消息，但也是处理速度最慢的选项。  
    ▶ on-publish，表示消息发送到下游目标节点的交换器后立即向上游的源节点发送确认，这个选项可以确保在网络失败的情况下不会丢失消息，但不能确保下游的Broker 宕机时不会丢失消息。  
    ▶ no-ack 表示无须进行消息确认，这个选项处理速度最快，但也最容易丢失消息。  
- Trust User-ID (trust-user-id)：设定Federation 是否使用“Validated User-ID”这个功能。如果设置为false 或者没有设置，那么Federation 会忽略消息的user_id 这个属性；如果设置为true，则Federation 只会转发user_id 为上游任意有效的用户的消息。  
“Validated User-ID”功能是指发送消息时验证消息的user_id 的属性，channel.basicPublish() 方法中有个参数是BasicProperties，这个BasicProperties 类中有个属性为userId。如果在连接Broker 时所用的用户名为“root”，当发送消息时设置的user_id 的属性为“guest”，那么这条消息会发送失败，具体报错为 406 PRECONDITION_FAILED - user_id property set to 'guest' but authenticated user was 'root'，只有当user_id 设置为“root”时这条消息才会发送成功。

只适合federated exchange 的参数如下：
- Exchange (exchange)：指定upstream exchange 的名称，默认情况下和federated exchange 同名，即图中的exchangeA。
- Max hops (max-hops)：指定消息被丢弃前在Federation link 中最大的跳转次数。默认为1。注意即使设置max-hops 参数为大于1 的值，同一条消息也不会在同一个Broker 中出现2 次，但是有可能会在多个节点中被复制。
- Expires (expires)：指定Federation link 断开之后，federated queue 所对应的upstream queue（即中的队列“federation: exchangeA→broker3 B”）的超时时间，默认为“none”，表示为不删除，单位为ms。这个参数相当于设置普通队列的x-expires 参数。设置这个值可以避免Federation link 断开之后，生产者一直在向broker1 中的exchangeA 发送消息，这些消息又不能被转发到broker3 中而被消费掉，进而造成broker1 中有大量的消息堆积。
- Message TTL (message-ttl)：为federated queue 所对应的upstream queue（即图中的队列“federation: exchangeA→broker3 B”）设置，相当于普通队列的x-message-ttl 参数。默认为“none”，表示消息没有超时时间。
- HA policy (ha-policy)：为federated queue 所对应的upstream queue（即图中的队列“federation: exchangeA→broker3 B”）设置，相当于普通队列的x-ha-policy参数，默认为“none”，表示队列没有任何HA。

只适合federated queue 的参数如下：
- Queue (queue)：执行upstream queue 的名称，默认情况下和federated queue 同名。

##### 第三步
定义一个Policy 用于匹配交换器exchangeA，并使用第二步中所创建的upstream。

**第一种**，通过rabbitmqctl 工具的方式，如下（定义所有以“exchange”开头的交换器作为federated exchange）：
```bash
rabbitmqctl set_policy --apply-to exchanges p1 "^exchange" '{"federationupstream":"f1"}'
```

**第二种**，通过HTTP API 接口的方式，如下：
```bash
curl -i -u root:root123 -XPUT -d '{"pattern":"^exchange","definition":{"federation-upstream":"f1"},"apply-to":"exchanges"} 
'http://192.168.0.4:15672/api/policies/%2F/p1
```

**第三种**，通过在Web 管理界面中添加的方式，在 “Admin”→“Policies”→“Add/ update a policy”中创建。

创建Federation link后，可以在Web 管理界面中“Admin”→“Federation Status”→“Running Links”查看到相应的链接。  
还可以通过rabbitmqctl eval 'rabbit_federation_status:status().'命令来查看相应的Federation link。示例如下：
```bash
[root@node2 ~]# rabbitmqctl eval 'rabbit_federation_status:status().'
[[{exchange,<<"exchangeA">>},
  {upstream_exchange,<<"exchangeA">>},
  {type,exchange},
  {vhost,<<"/">>},
  {upstream,<<"f1">>},
  {id,<<"fad51c1713586d453b7dc9cd1a28641192a94f41">>},
  {status,running},
  {local_connection,<<"<rabbit@node2.1.15217.10>">>},
  {uri,<<"amqp://192.168.0.2:5672">>},
  {timestamp,{{2017,10,15},{0,7,48}}}]]
```

<span style="color: red;font-weight: bold;">Tips</span>：通常情况下，针对每个upstream 都会有一条Federation link，该Federation link 对应到一个交换器上。例如，3 个交换器与2 个upstream 分别建立Federation link 的情况下，会有6 条连接。

**建立federated queue**，首先同样也是定义一个upstream。之后定义Policy 的时候略微有变化，比如使用rabbitmqctl 工具的情况（定义所有以“queue”开头的队列作为federated queue）：
```bash
rabbitmqctl set_policy --apply-to queues p2 "^queue" '{"federation-upstream":"f1"}'
```


## Shovel
Shovel 能够可靠、持续地从一个Broker 中的队列（作为源端，即source）拉取数据并转发至另一个Broker 中的交换器（作为目的端，即destination）。作为源端的队列和作为目的端的交换器可以同时位于同一个Broker 上，也可以位于不同的Broker 上。  
Shovel 可以翻译为“铲子”，是一种比较形象的比喻，这个“铲子”可以将消息从一方“挖到”另一方。Shovel 的行为就像优秀的客户端应用程序能够负责连接源和目的地、负责消息的读写及负责连接失败问题的处理。

Shovel 的主要优势在于：
- 松耦合。Shovel 可以移动位于不同管理域中的Broker（或者集群）上的消息，这些Broker（或者集群）可以包含不同的用户和vhost，也可以使用不同的RabbitMQ 和Erlang 版本。
- 支持广域网。Shovel 插件同样基于AMQP 协议在Broker 之间进行通信，被设计成可以容忍时断时续的连通情形，并且能够保证消息的可靠性。
- 高度定制。当Shovel 成功连接后，可以对其进行配置以执行相关的AMQP 命令。

### Shovel 的原理
![shovel_structure](../images/rabbitmq/2024-03-31_shovel_structure.png) _Shovel的结构_  
上图有两个Broker：broker1（IP 地址：192.168.0.2）和broker2（IP 地址：192.168.0.3）。broker1 中有交换器exchange1 和队列queue1，且这两者通过路由键“rk1”进行绑定；broker2 中有交换器exchange2 和队列queue2，且这两者通过路由键“rk2”进行绑定。在队列queue1 和交换器exchange2 之间配置一个Shovel link，当一条内容为“shovel test payload”的消息从客户端发送至交换器exchange1 的时候，这条消息会经过数据流转最后存储在队列queue2 中。如果在配置Shovel link 时设置了add_forward_headers 参数为true，则在消费到队列queue2 中这条消息的时候会有特殊的headers 属性标记，如下图。  
![shovel_message_details](../images/rabbitmq/2024-03-31_shovel_message_details.png)

通常情况下，使用Shovel 时配置源节点的队列作为源端，目标节点的交换器作为目的端。  
同样可以将目标节点的队列配置为目的端，如下图。虽然看起来队列queue1 是通过Shovel link 直接将消息转发至queue2 的，其实中间也是经由broker2 的交换器转发，只不过这个交换器是默认的交换器而已。  
![shovel_queue_destination](../images/rabbitmq/2024-03-31_shovel_queue_destination.png) _将队列配置为目的端_

如下图，配置源节点的交换器为源端也是可行的。虽然看起来交换器exchange1 是通过Shovel link 直接将消息转发至exchange2 上的，实际上在broker1 中会新建一个队列（名称由RabbitMQ 自定义，比如中的“amq.gen-ZwolUsoUchY6a7xaPyrZZH”）并绑定exchange1，消息从交换器exchange1 过来先存储在这个队列中，然后Shovel 再从这个队列中拉取消息进而转发至交换器exchange2。  
![shovel_exchange_source](../images/rabbitmq/2024-03-31_shovel_exchange_source.png) _配置交换器为源端_

以上Shovel 结构中的broker1、broker2 中的exchange1、queue1、exchange2 及queue2 都可以在Shovel 成功连接源端或者目的端Broker 之后再第一次创建（执行一系列相应的AMQP 配置声明时），它们并不一定需要在Shovel link 建立之前创建。Shovel 可以为源端或者目的端配置多个Broker 的地址，这样可以使得源端或者目的端的Broker 失效后能够尝试重连到其他Broker 之上（随机挑选）。可以设置reconnect_delay 参数以避免由于重连行为导致的网络泛洪，或者可以在重连失败后直接停止连接。针对源端和目的端的所有配置声明会在重连成功之后被重新发送。

### Shovel 的使用
Shovel 插件默认也在RabbitMQ 的发布包中，执行rabbitmq-plugins enable rabbitmq_shovel 命令可以开启Shovel 功能，Shovel 内部是基于AMQP 协议转发数据的，所以在开启rabbitmq_shovel 插件的时候，默认会开启amqp_client 插件。示例如下：
```bash
[root@node2 opt]# rabbitmq-plugins enable rabbitmq_shovel
The following plugins have been enabled:
  amqp_client
  rabbitmq_shovel
Applying plugin configuration to rabbit@node2... started 2 plugins.
```

如果要开启Shovel 的管理插件，需要执行rabbitmq-plugins enable rabbitmq_shovel_management 命令，示例如下：
```bash
[root@node2 opt]# rabbitmq-plugins enable rabbitmq_shovel_management
The following plugins have been enabled:
  cowlib
  cowboy
  rabbitmq_web_dispatch
  rabbitmq_management_agent
  rabbitmq_management
  rabbitmq_shovel_management
Applying plugin configuration to rabbit@node2... started 6 plugins.
```

之后，在RabbitMQ 的管理界面中“Admin”的右侧会多出“Shovel Status”和“Shovel Management”两个Tab 页。

Shovel 既可以部署在源端，也可以部署在目的端。有两种方式可以部署Shovel：
- 静态方式（static）是指在rabbitmq.config 配置文件中设置。
- 动态方式（dynamic）是指通过Runtime Parameter 设置。

##### 静态方式
在rabbitmq.config 配置文件中针对Shovel 插件的配置信息是一种Erlang 项式，由单条Shovel 条目构成（shovels 部分的下一层）：
```tex
{rabbitmq_shovel, [ {shovels, [ {shovel_name, [ {sources, [ ... ]}, 
                                                {destinations, [ ... ]}, 
                                                {queue, queue_name}, 
                                                {prefetch_count, count}, 
                                                {ack_mode, a_mode}, 
                                                {publish_properties, [ ... ]}, 
                                                {publish_fields, [ ... ]}, 
                                                {reconnect_delay, reconn_delay}
]}, ... ]} ]}
```

每一条Shovel 条目定义了源端与目的端的转发关系，其名称（shovel_name）必须是独一无二的。其中sources、destination 和queue 这三项是必需的，其余的都可以默认。

Shovel 的配置：
```tex
[{rabbitmq_shovel,
    [{shovels,
        [{hidden_shovel,
            [{sources,
                [{broker, "amqp://root:root123@192.168.0.2:5672"},
                {declarations,
                [
                    {'queue.declare',[{queue, <<"queue1">>}, durable]},
                    {'exchange.declare',[
                            {exchange, <<"exchange1">>},
                            {type, <<"direct">>},
                            durable
                        ]
                    },
                    {'queue.bind',[
                            {exchange, <<"exchange1">>},
                            {queue, <<"queue1">>},
                            {routing_key, <<"rk1">>}
                        ]
                    }]}]},
            {destinations,
                [{broker, "amqp://root:root123@192.168.0.3:5672"},
                {declarations,
                [
                    {'queue.declare',[{queue, <<"queue2">>}, durable]},
                    {'exchange.declare',[
                            {exchange, <<"exchange2">>},
                            {type, <<"direct">>},
                            durable
                        ]
                    },
                    {'queue.bind',[
                            {exchange, <<"exchange2">>},
                            {queue, <<"queue2">>},
                            {routing_key, <<"rk2">>}
                        ]
                    }]}]},
            {queue, <<"queue1">>},
            {ack_mode, no_ack},
            {prefetch_count, 64},
            {publish_properties, [{delivery_mode, 2}]},
            {add_forward_headers, true},
            {publish_fields, [{exchange, <<"exchange2">>}, {routing_key,<<"rk2">>}]},
            {reconnect_delay, 5}]
        }]
    }]
}].
```

- sources 和destinations 两者都包含了同样类型的配置：
```tex
{sources, [ {broker[或brokers], broker_list},
            {declarations, declaration_list}
        ]}
```
▶ broker 项配置的是URI，定义了用于连接Shovel 两端的服务器地址、用户名、密码、vhost 和端口号等。如果sources 或者destinations 是RabbitMQ 集群，那么就使用brokers，并在其后用多个URI 字符串以“[]”的形式包裹起来，比如{brokers,["amqp://root:root123@192.168.0.2:5672","amqp://root:root123@192.168.0.4:5672"]}，这样的定义能够使得Shovel 在主节点故障时转移到另一个集群节点上。  
▶ declarations 这一项是可选的。  
▶ declaration_list 指定了可以使用的AMQP 命令的列表，声明了队列、交换器和绑定关系。比如Shovel 的配置中sources 的declarations 这一项声明了队列queue1（'queue.declare'）、交换器exchange1（'exchange.declare'）及其之间的绑定关系（'queue.bind'）。注意其中所有的字符串并不是简单地用引号标注，而是同时用双尖括号包裹，比如<<"queue1">>。这里的双尖括号是要让Erlang 程序不要将其视为简单的字符串，而是binary 类型的字符串。如果没有双尖括号包裹，那么Shovel 在启动的时候就会出错。与queue1 一起的还有一个durable 参数，它不需要像其他参数一样需要包裹在大括号内，这是因为像durable 这种类型的参数不需要赋值，它要么存在，要么不存在，只有在参数需要赋值的时候才需要加上大括号。  
▶ queue 表示源端服务器上的队列名称。可以将queue 设置为“<<>>”，表示匿名队列（队列名称由RabbitMQ 自动生成）。  
▶ prefetch_count 参数表示Shovel 内部缓存的消息条数，Shovel 的内部缓存是源端服务器和目的端服务器之间的中间缓存部分。  
▶ ack_mode 表示在完成转发消息时的确认模式，和Federation 的ack_mode 一样也有三种取值：no_ack 表示无须任何消息确认行为；on_publish 表示Shovel 会把每一条消息发送到目的端之后再向源端发送消息确认；on_confirm 表示Shovel 会使用publisher confirm 机制，在收到目的端的消息确认之后再向源端发送消息确认。Shovel 的ack_mode 默认也是on_confirm，并且官方强烈建议使用该值。如果选择使用其他值，整体性能虽然会有略微提升，但是发生各种失效问题的情况时，消息的可靠性得不到保障。  
▶ publish_properties 是指消息发往目的端时需要特别设置的属性列表。默认情况下，被转发的消息的各个属性是被保留的，但是如果在publish_properties 中对属性进行了设置则可以覆盖原先的属性值。publish_properties 的属性列表包括content_type、content_encoding 、headers 、delivery_mode 、priority 、correlation_id 、reply_to、expiration、message_id、timestamp、type、user_id、app_id 和cluster_id。  
▶ add_forward_headers 如果设置为true，则会在转发的消息内添加x-shovelled 的header 属性。  
▶ publish_fields 定义了消息需要发往目的端服务器上的交换器以及标记在消息上的路由键。如果交换器和路由键没有定义，则Shovel 会从原始消息上复制这些被忽略的设置。  
▶ reconnect_delay 指定在Shovel link 失效的情况下，重新建立连接前需要等待的时间，单位为秒。如果设置为0，则不会进行重连动作，即Shovel 会在首次连接失效时停止工作。reconnect_delay 默认为5 秒。

##### 动态方式
Shovel 动态部署方式的配置信息会被保存到RabbitMQ 的Mnesia 数据库中，包括权限信息、用户信息和队列信息等内容。每一个Shovel link 都由一个相应的Parameter 定义，这个Parameter 同样可以通过rabbitmqctl 工具、RabbitMQ Management 插件的HTTP API 接口或者rabbitmq_shovel_management 提供的Web 管理界面的方式设置。

下面展示三种设置Parameter 的示例用法。

**第一种**是通过rabbitmqctl 工具的方式，详细如下：
```bash
rabbitmqctl set_parameter shovel hidden_shovel \
'{"src-uri":"amqp://root:root123@192.168.0.2:5672","src-queue":"queue1",
"dest-uri":"amqp://root:root123@192.168.0.3:5672","src-exchange-key":"rk2",
"prefetch-count":64, "reconnect-delay":5, "publish-properties":[],
"add-forward-headers":true, "ack-mode":"on-confirm"}'
```

**第二种**是通过调用HTTP API 接口的方式，详细如下：
```bash
curl -i -u root:root123 -XPUT -d 
'{"value":{"src-uri":"amqp://root:root123@192.168.0.2:5672","src-queue":"queue1",
"dest-uri":"amqp://root:root123@192.168.0.3:5672","src-exchange-key":"rk2",
"prefetch-count":64, "reconnect-delay":5, "publish-properties":[],
"add-forward-headers":true, "ack-mode":"on-confirm"}}' 
http://192.168.0.2:15672/api/parameters/shovel/%2f/hidden_shovel
```

**第三种**是通过Web 管理界面中添加的方式，在“Admin”→“Shovel Management”→“Add a new shovel”中创建。  
在创建了一个Shovel link 之后，可以在Web 管理界面中“Admin”→“Shovel Status”中查看到相应的信息，也可以通过rabbitmqctl eval 'rabbit_shovel_status:status().'命令直接查询Shovel的状态信息，该命令会调用rabbitmq_shovel插件模块中的status方法，该方法将返回一个Erlang 列表，其中每一个元素对应一个已配置好的Shovel。示例如下：
```bash
[root@node1 ~]# rabbitmqctl eval 'rabbit_shovel_status:status().'
[{{<<"/">>,<<"hidden_shovel">>},
  dynamic,
  {running,[{src_uri,<<"amqp://192.168.0.2:5672">>},
          {dest_uri,<<"amqp://192.168.0.3:5672">>}]},
  {{2017,10,16},{11,41,58}}}]
```

列表中的每一个元素都以一个四元组的形式构成：{Name, Type, Status, Timestamp}。具体含义如下：
- Name 表示Shovel 的名称。
- Type 表示类型，有2 种取值——static 和dynamic。
- Status 表示目前Shovel 的状态。当Shovel 处于启动、连接和创建资源时状态为starting；当Shovel 正常运行时是running；当Shovel 终止时是terminated。
- Timestamp 表示该Shovel 进入当前状态的时间戳，具体格式是{{YYYY, MM, DD}，{HH, MM, SS}}。

### 案例：消息堆积的治理
消息堆积是一把双刃剑，适量的堆积可以有削峰、缓存之用，但是如果堆积过于严重，那么就可能影响到其他队列的使用，导致整体服务质量的下降。  
对于一台普通的服务器来说，在一个队列中堆积1 万至10 万条消息，丝毫不会影响什么。但是如果这个队列中堆积超过1 千万乃至一亿条消息时，可能会引起一些严重的问题，比如引起内存或者磁盘告警而造成所有Connection 阻塞。

消息堆积的治理方案：
- 消息堆积严重时，可以选择清空队列，或者采用空消费程序丢弃掉部分消息。不过对于重要的数据而言，丢弃消息的方案并无用武之地。  
- 另一种方案是增加下游消费者的消费能力，这个思路可以通过后期优化代码逻辑或者增加消费者的实例数来实现。但是后期的代码优化在面临紧急情况时总归是“远水解不了近渴”，并且有些业务场景也并非可以简单地通过增加消费实例而得以增强消费能力。
- 当某个队列中的消息堆积严重时，比如超过某个设定的阈值，就可以通过Shovel 将队列中的消息移交给另一个集群。

运用Shovel 转移消息的流程：
1. 情形1：当检测到当前运行集群cluster1 中的队列queue1 中有严重消息堆积，比如通过/api/queues/vhost/name 接口获取到队列的消息个数（messages）超过2 千万或者消息占用大小（messages_bytes）超过10GB 时，就启用shovel1 将队列queue1 中的消息转发至备份集群cluster2 中的队列queue2。
2. 情形2：紧随情形1，当检测到队列queue1 中的消息个数低于1 百万或者消息占用大小低于1GB 时就停止shovel1，然后让原本队列queue1 中的消费者慢慢处理剩余的堆积。
3. 情形3：当检测到队列queue1 中的消息个数低于10 万或者消息占用大小低于100MB时，就开启shovel2 将队列queue2 中暂存的消息返还给队列queue1。
4. 情形4：紧随情形3，当检测到队列queue1 中的消息个数超过1 百万或者消息占用大小高于1GB 时就将shovel2 停掉。

如此，队列queue1 就拥有了队列queue2 这个“保镖”为它保驾护航。这里是“一备一”的情形，如果需要要“一备多”，可以采用镜像队列或者引入Federation。


## Federation/Shovel 与集群
Federation/Shovel | 集群
 :---- | :----
各个Broker 节点之间逻辑分离 | 逻辑上是一个Broker 节点
各个Broker 节点之间可以运行不同版本的Erlang 和RabbitMQ | 各个Broker 节点之间必须运行相同版本的Erlang 和RabbitMQ
各个Broker 节点之间可以在广域网中相连，当然必须要授予适当的用户和权限 | 各个Broker 节点之间必须在可信赖的局域网中相连，通过Erlang 内部节点传递消息，但节点间需要有相同的Erlang cookie
各个Broker 节点之间能以任何拓扑逻辑部署，连接可以是单向的或者是双向的 | 所有Broker 节点都双向连接所有其他节点
从CAP 理论中选择可用性和分区耐受性，即AP | 从CAP 理论中选择一致性和可用性，CA
一个Broker 中的交换器可以是Federation 生成的或者是本地的 | 集群中所有Broker 节点中的交换器都是一样的，要么全有要么全无
客户端所能看到它所连接的Broker 节点上的队列 | 客户端连接到集群中的任何Broker 节点都可以看到所有的队列

### Federation与Shovel的异同
通过Shovel 来连接各个RabbitMQ Broker，概念上与Federation 的情形类似，不过Shovel 工作在更低一层。鉴于Federation 从一个交换器中转发消息到另一个交换器（如果必要可以确认消息是否被转发），Shovel 只是简单地从某个Broker 上的队列中消费消息，然后转发消息到另一个Broker 上的交换器而已。Shovel 也可以在单独的一台服务器上去转发消息，比如将一个队列中的数据移动到另一个队列中。如果想获得比Federation 更多的控制，可以在广域网中使用Shovel 连接各个RabbitMQ Broker 来生产或消费消息。


## 网络分区
### 网络分区的意义
RabbitMQ 集群的网络分区的容错性并不是很高，一般都是使用Federation 或者Shovel 来解决广域网中的使用问题。  
不过即使是在局域网环境下，网络分区也不可能完全避免，网络设备（比如中继设备、网卡）出现故障也会导致网络分区。当出现网络分区时，不同分区里的节点会认为不属于自身所在分区的节点都已经挂（down）了，对于队列、交换器、绑定的操作仅对当前分区有效。  
在RabbitMQ 的默认配置下，即使网络恢复了也不会自动处理网络分区带来的问题。RabbitMQ 从3.1 版本开始会自动探测网络分区，并且提供了相应的配置来解决这个问题。  
当一个集群发生网络分区时，这个集群会分成两个部分或者更多，它们各自为政，互相都认为对方分区内的节点已经挂了，包括队列、交换器及绑定等元数据的创建和销毁都处于自身分区内，与其他分区无关。如果原集群中配置了镜像队列，而这个镜像队列又牵涉两个或者更多个网络分区中的节点时，每一个网络分区中都会出现一个master 节点，对于各个网络分区，此队列都是相互独立的。当然也会有一些其他未知的、怪异的事情发生。当网络恢复时，网络分区的状态还是会保持，除非你采取了一些措施去解决它。  
网络分区带来的影响大多是负面的，极端情况下不仅会造成数据丢失，还会影响服务的可用性。

引入网络分区的设计理念与它本身的数据一致性复制原理有关，RabbitMQ 采用的镜像队列是一种环形的逻辑结构，这种复制原理和ZooKeeper 的Quorum 原理不同，它可以保证更强的一致性。如果出现网络波动或者网络故障等异常情况，那么整个数据链的性能就会大大降低。如果环形链中的节点出现网络异常，那么整个数据链就会被阻塞，继而相关服务也会被阻塞，所以这里就需要引入网络分区来将异常的节点剥离出整个分区，以确保RabbitMQ 服务的可用性及可靠性。等待网络恢复之后，可以进行相应的处理来将此前的异常节点加入集群中。

### 网络分区的判定
RabbitMQ 集群节点内部通信端口默认为25672，两两节点之间都会有信息交互。如果某节点出现网络故障，或者是端口不通，则会致使与此节点的交互出现中断，这里就会有个超时判定机制，继而判定网络分区。  
对于网络分区的判定是与net_ticktime 这个参数息息相关的，此参数默认值为60 秒。注意与heartbeat_time 的区别，heartbeat_time 是指客户端与RabbitMQ 服务之间通信的心跳时间，针对5672 端口而言。  
如果发生超时则会有net_tick_timeout 的信息报出。在RabbitMQ 集群内部的每个节点之间会每隔四分之一的net_ticktime 计一次应答（tick）。如果有任何数据被写入节点中，则此节点被认为已经被应答（ticked）了。如果连续4 次，某节点都没有被ticked，则可以判定此节点已处于“down”状态，其余节点可以将此节点剥离出当前分区。  
将连续4 次的tick 时间记为T，那么T 的取值范围为：0.75×net_ticktime < T < 1.25×net_ticktime。  
下图可以形象地描绘出这个取值范围的缘由。图中每个节点代表一次tick 判定的时间戳，在2 个临界值0.75×net_ticktime 和1.25×net_ticktime 之间可以连续执行4 次的tick 判定。默认情况下，在45s < T < 75s之间会判定出net_tick_timeout。  
![ticked_value_range](../images/rabbitmq/2024-04-02_ticked_value_range.png)

RabbitMQ 不仅会将队列、交换器及绑定等信息存储在Mnesia 数据库中，而且许多围绕网络分区的一些细节也都和这个Mnesia 的行为相关。

##### 查看是否出现网络分区
1. 如果一个节点不能在T时间连上另一个节点，那么Mnesia 通常认为这个节点已经挂了，就算之后两个节点又重新恢复了内部通信，但是这两个节点都会认为对方已经挂了，Mnesia 此时认定了发生网络分区的情况。这些会被记录到RabbitMQ 的服务日志之中，如下：
```tex
=ERROR REPORT==== 16-Oct-2017::18:20:55 ===
Mnesia('rabbit@node1'): ** ERROR ** mnesia_event got
    {inconsistent_database, running_partitioned_network, 'rabbit@node2'}
```

2. 采用rabbitmqctl 工具来查看，即采用rabbitmqctl cluster_status 命令。
```bash
# 未发生网络分区时，在partitions 这一项中没有相关记录，则说明没有产生网络分区
[root@node1 ~]# rabbitmqctl cluster_status
[{nodes,[{disc,[rabbit@node1,rabbit@node2,rabbit@node3]}]},
 {running_nodes,[rabbit@node2,rabbit@node3,rabbit@node1]},
 {cluster_name,<<"rabbit@node1">>},
 {partitions,[]}]
# partitions 项中有相关内容，则说明产生了网络分区
[root@node1 ~]# rabbitmqctl cluster_status
[{nodes,[{disc,[rabbit@node1,rabbit@node2,rabbit@node3]}]},
 {running_nodes,[rabbit@node3,rabbit@node1]},
 {cluster_name,<<"rabbit@node1">>},
 {partitions,[{rabbit@node3,[rabbit@node2]},{rabbit@node1,[rabbit@node2]}]}]
```

上面partitions 项中的内容表示：  
▶ rabbit@node3 与rabbit@node2 发生了分区，即{rabbit@node3,[rabbit@node2]}  
▶ rabbit@node1 与rabbit@node2 发生了分区，即{rabbit@node3,[rabbit@node2]}

3. 通过Web 管理界面的方式查看。如果出现了下图这种告警，即发生了网络分区。  
![web_network_partition_exception](../images/rabbitmq/2024-04-02_web_network_partition_exception.png)

4. 通过HTTP API 的方式调取节点信息来检测是否发生网络分区。比如通过curl 命令来调取节点信息：
```bash
curl -i -u root:root123 -H "content-type:application/json" -X GET http://localhost:15672/api/nodes
```
/api/nodes 这个接口返回一个JSON 字符串，详细内容可以参考附录A，其中会有partitions 的相关项，如果在其中发现partitions 项中有内容则为发生了网络分区。
```bash
"sockets_used": 1,
"sockets_used_details": {
"rate": 0
},
# node2 发生了网络分区
"partitions": [
"rabbit@node2"
],
"os_pid": "2155",
"fd_total": 1024,
"sockets_total": 829,
```

### 网络分区的模拟
模拟网络分区的方式有多种，主要分为以下三大类：
- iptables 封禁/解封IP 地址或者端口号。
- 关闭/开启网卡。
- 挂起/恢复操作系统。

###### iptables 的方式
由于RabbitMQ 集群内部节点通信端口默认为25672，可以封禁这个端口来模拟出net_tick_timeout，然后再开启此端口让集群判定网络分区的发生。  
举例说明，整个RabbitMQ 集群由3 个节点组成，分别为node1、node2 和 node3。此时我们要模拟node2 节点被剥离出当前分区的情形，即模拟[node1, node3]和[node2]两个分区。可以在node2 上执行如下命令以封禁25672 端口。
```bash
iptables -A INPUT -p tcp --dport 25672 -j DROP
iptables -A OUTPUT -p tcp --dport 25672 -j DROP
```

同时需要监测各个节点的服务日志， 当有如下相似信息出现时即为已经判定出net_tick_timeout：
```bash
=INFO REPORT==== 10-Oct-2017::11:53:03 ===
rabbit on node rabbit@node2 down

=INFO REPORT==== 10-Oct-2017::11:53:03 ===
node rabbit@node2 down: net_tick_timeout
```

也可以等待75 秒（45s < T < 75s）之后以确保出现net_tick_timeout。  
确认判定出net_tick_timeout，在恢复node2 的网络连接（即解封25672 端口）之后才会判定出现网络分区。解封命令如下：
```bash
iptables -D INPUT 1
iptables -D OUTPUT 1
```

恢复node2 节点与其他节点的内部通信之后，如果此时查看集群的状态可以发现[node1, node3]和[node2]已形成两个独立的分区。

---

还可以使用iptables 封禁IP 地址的方法模拟网络分区。假设整个RabbitMQ 集群的节点名称与其IP 地址对应如下：
```bash
node1 192.168.0.2
node2 192.168.0.3
node3 192.168.0.4
```

如果要模拟出[node1, node3]和[node2]两个分区的情形，可以在node2 节点上执行：
```bash
iptables -I INPUT -s 192.168.0.2 -j DROP
iptables -I INPUT -s 192.168.0.4 -j DROP
# 对应的解封命令为
iptables -D INPUT 1
iptables -D INPUT 1
```

或者也可以分别在node1 和node3 节点上执行：
```bash
iptables -I INPUT -s 192.168.0.3 -j DROP
# 对应的解封命令为
iptables -D INPUT 1
```

---

如果集群的节点部署跨网段，可以采取禁用整个网络段的方式模拟网络分区。假设RabbitMQ 集群中3 个节点和其对应的IP 关系如下：
```bash
node1 192.168.0.2
node2 192.168.1.3  //注意这里的网段
node3 192.168.0.4
```

模拟出[node1, node3]和[node2]两个分区的情形，可以在node2 节点上执行：
```bash
iptables -I INPUT -s 192.168.0.0/24 -j DROP
# 对应的解封命令为
iptables -D INPUT 1
```

###### 封禁/解封网卡的方式
首先需要使用ifconfig 命令来查询出当前的网卡编号，一般情况下单台机器只有一个网卡（这里暂时不考虑多网卡的情形，因为对于RabbitMQ 来说，多网卡的情况造成的网络分区异常复杂。）  
假设node1、node2 和node3 这三个节点组成RabbitMQ 集群，node2 的网卡编号为eth0，此时要模拟网络分区[node1, node3]和[node2]的情形，需要在node2 上执行以下命令关闭网卡：
```bash
ifdown eth0
```

待判定出net_tick_timeout 之后，再开启网卡：
```bash
ifup eth0
```

也可以使用service network stop 和service network start 这两个命令来模拟网络分区，原理同ifdown/ifup eth0 的方式。

###### 挂起/恢复操作系统的方式
操作系统的挂起和恢复操作也会导致集群内节点的网络分区。因为发生挂起的节点不会认为自身已经失败或者停止工作，但是集群内的其他节点会这么认为。如果集群中的一个节点运行在一台笔记本电脑上，然后你合上了笔记本电脑，那么这个节点就挂起了。或者一个更常见的现象，集群中的一个节点运行在某台虚拟机上，然后虚拟机的管理程序挂起了这个虚拟机节点，这样节点就被挂起了。在等待了(0.75×net_ticktime, 1.25×net_ticktime)这个区间大小的时间之后，判定出net_tick_timeout，再恢复挂起的节点即可以复现网络分区。

### 网络分区的影响
RabbitMQ 集群在发生网络分区之后对于数据可靠性、服务可用性、客户端的影响，这里主要针对未配置镜像和配置镜像两种情况展开探讨。

#### 未配置镜像
node1、node2 和node3 这3 个节点组成一个RabbitMQ 集群，且在这三个节点中分别创建queue1、queue2 和queue3 这三个队列，并且相应的交换器与绑定关系如下：

节点名称 | 交换器 | 绑定 | 队列
 :----: | :----: | :----: | :----:
node1 | exchange | rk1 | queue1
node2 | exchange | rk2 | queue2
node3 | exchange | rk3 | queue3

**情形一**  
客户端分别连接node1 和node2 并分别向queue1 和queue2 发送消息，对应关系如下所示：

客户端 | 节点名称 | 交换器 | 绑定 | 队列
 :----: | :----: | :----: | :----: | :----:
client1 | node1 | exchange | rk1 | queue1
client2 | node2 | exchange | rk2 | queue2

client1 那条信息表示：客户端client1 连接node1 的IP 地址，并通过路由键rk1 向交换器exchange 发送消息。如果发送成功，消息可以存入队列queue1 中。其对应的发送代码如下：
```java
channel.basicPublish("exchange", "rk1", true, MessageProperties.PERSISTENT_TEXT_PLAIN, message.getBytes());
```

采用iptables 封禁/解封25672 端口的方式模拟网络分区，使node1 和node2 存在于两个不同的分区之中，对于客户端client1 和client2 而言，没有任何异常，消息正常发送也没有消息丢失。如果这里采用关闭/开启网卡的方式来模拟网络分区，在关闭网卡的时候客户端的连接也会关闭，这样就检测不出在网络分区发生时对客户端的影响。

**情形二**  
如果client1 连接node1 的IP，并向queue2 发送消息会发生何种情形？对应关系如下所示：

客户端 | 节点名称 | 交换器 | 绑定 | 队列
 :----: | :----: | :----: | :----: | :----:
client1 | node1 | exchange | rk2 | queue2
client2 | node2 | exchange | rk1 | queue1

采用iptables的方式模拟网络分区，使得node1 和node2 处于两个不同的分区。  
如果客户端在发送消息的时候将mandatory 参数设置为true，那么在网络分区之后可以通过抓包工具（如wireshark 等）看到有Basic.Return 将发送的消息返回过来。这里表示在发生网络分区之后，client1 不能将消息正确地送达到queue2 中，同样client2 不能将消息送达到queue1 中，因此消息也就不能存储。如果客户端中设置了ReturnListener 来监听Basic.Return 的信息，并附带有消息重传机制，那么在整个网络分区前后的过程中可以保证发送端的消息不丢失。  
如果客户端没有设置mandatory 参数并且没有通过ReturnListener 进行消息重试（或者其他措施）来保障消息可靠性，那么在发送端就会有消息丢失。

在网络分区之前，queue1 进程存在于node1 节点中，queue2 的进程存在于node2 节点中。在网络分区之后，在node1 所在的分区并不会创建新的queue2 进程，同样在node2 所在的分区也不会创建新的queue1 的进程。这样在网络分区发生之后，虽然可以通过rabbitmqctl list_queues name 命令在node1 节点上查看到queue2，但是在node1 上已经没有真实的queue2 进程的存在。

**情形三**  
在网络分区之前，分别有客户端连接node1 和node2 并订阅消费其上队列中的消息，对应关系如下所示：

客户端 | 节点名称 | 队列
 :----: | :----: | :----:
client3 | node1 | queue1
client4 | node2 | queue2

client3 连接node1 的ip 并订阅消费queue1。模拟网络分区置node1 和node2 于不同的分区之中。在发生网络分区的前后，消费端client3 和client4 都能正常消费，无任何异常发生。

**情形四**  
client3 连接node1 的IP 消费queue2，对应关系如下所示：

客户端 | 节点名称 | 队列
 :----: | :----: | :----:
client3 | node1 | queue2
client4 | node2 | queue1

在发生网络分区前，消费一切正常。在网络分区发生之后，虽然客户端没有异常报错，且可以消费到相关数据，但是此时会有一些怪异的现象发生，比如对于已消费消息的ack 会失效。在从网络分区中恢复之后，数据不会丢失。

如果分区之后，重启client3 或者有个新的客户端client5 连接node1 的IP 来消费queue2，则会有如下报错：
```bash
com.rabbitmq.client.ShutdownSignalException: channel error; protocol method:#method<channel.close>
(reply-code=404, reply-text=NOT_FOUND - home node
'rabbit@node2' of durable queue 'queue2' in vhost '/' is down or inaccessible,class-id=60, method-id=20)
```

同样在node1 的服务日志中也有相关记录：
```bash
=ERROR REPORT==== 12-Oct-2017::14:14:48 ===
Channel error on connection <0.9528.9> (192.168.0.9:61294 -> 192.168.0.2:5672,vhost: '/', user: 'root'), channel 1:
{amqp_error,not_found,"home node 'rabbit@node2' of durable queue 'queue2' 
in vhost '/' is down or inaccessible",'basic.consume'}
```

综上所述，对于未配置镜像的集群，网络分区发生之后，队列也会伴随着宿主节点而分散在各自的分区之中。  
对于消息发送方而言，可以成功发送消息，但是会有路由失败的现象，需要配合mandatory 等机制保障消息的可靠性。  
对于消息消费方来说，有可能会有诡异、不可预知的现象发生，比如对于已消费消息的ack 会失效。如果网络分区发生之后，客户端与某分区重新建立通信链路，其分区中如果没有相应的队列进程，则会有异常报出。如果从网络分区中恢复之后，数据不会丢失，但是客户端会重复消费。

#### 已配置镜像
如果集群中配置了镜像队列，那么在发生网络分区时，情形比未配置镜像队列的情况复杂得多，尤其是发生多个网络分区的时候。  
集群中有node1、node2 和node3 这3 个节点，分别在这些节点上创建队列queue1、queue2 和queue3，并配置镜像队列。采用iptables 的方式将集群模拟分裂成[node1, node3]和[node2]这两个网络分区。  
镜像队列的相关配置可以参考如下：
```bash
ha-mode:exactly
ha-param:2
ha-sync-mode:automatic
```

**情形一**  
分区之前，3 个队列的master 镜像和slave 镜像分别做相应分布：

队列 | master | slave
 :----: | :----: | :----:
queue1 | node1 | node3
queue2 | node2 | node3
queue3 | node3 | node2

在发生网络分区之后，[node1, node3]分区中的队列有了新的部署。除了queue1 未发生改变，queue2 由于原宿主节点node2 已被剥离当前分区，那么node3 提升为master，同时选择node1 作为slave。在queue3 重新选择node1 作为其新的slave。对于[node2]分区而言，queue2 和queue3 的分布比较容易理解，此分区中只有一个节点，所有slave 这一列为空。但是对于queue1 而言，其部署还是和分区前如出一辙。不管是在网络分区前，还是在网络分区之后，再或者是又从网络分区中恢复，对于queue1 而言生产和消费消息都不会受到任何的影响，就如未发生过网络分区一样。对于队列queue2 和queue3 的情形可以参考上面未配置镜像的相关细节，从网络分区中恢复（即恢复成之前的[node1, node2, node3]组成的完整分区）之后可能会有数据丢失。  
分区之后，相应分布关系如下：

&nbsp; | [node1, node3]分区 | [node2]分区
 :----: | :---- | :----
队列 | master&emsp;&emsp;slave | master&emsp;&emsp;slave
queue1 | node1&emsp;&emsp;node3 | node1&emsp;&emsp;node3
queue2 | node3&emsp;&emsp;node1 | node2&emsp;&emsp;[]
queue3 | node3&emsp;&emsp;node1 | node2&emsp;&emsp;[]

**情形二**  
分区之前，3 个队列的master 镜像和slave 镜像分别做相应分布：

队列 | master | slave
 :----: | :----: | :----:
queue1 | node1 | node2
queue2 | node2 | node3
queue3 | node3 | node1

分区之后：

&nbsp; | [node1, node3]分区 | [node2]分区
 :----: | :---- | :----
队列 | master&emsp;&emsp;slave | master&emsp;&emsp;slave
queue1 | node1&emsp;&emsp;node3 | node2&emsp;&emsp;[]
queue2 | node3&emsp;&emsp;node1 | node2&emsp;&emsp;[]
queue3 | node3&emsp;&emsp;node1 | node3&emsp;&emsp;node1

如果要实现在ha-mode=exactly 和ha-params=2 的镜像配置下准确地指定对应的slave 镜像所在节点，可以实现编写一个脚本，命名为rmq_mirror_create.sh，具体内容如下：

```bash
rabbitmqctl clear_policy p1
rabbitmqctl set_policy --priority 0 --apply-to queues p1 ".*" '{"ha-mode":"exactly","ha-params":2}'
rabbitmqctl list_queues name pid slave_pids
```

之后再为rmq_mirror_create.sh 添加可执行权限：

```bash
chmod a+x rmq_mirror_create.sh
```

最后反复运行脚本直到有如下相似输出即可（主要是观察list_queues 中slave_pids 的信息）：

```bash
[root@node1 ~]# ./ rmq_mirror_create.sh
Clearing policy "p1" ...
Setting policy "p1" for pattern ".*" to "{\"ha-mode\":\"exactly\",\"ha-params\":2}" with priority "0" ...
Listing queues ...
queue1 <rabbit@node1.1.279.0> [<rabbit@node2.3.3804.1>]
queue2 <rabbit@node2.3.1391.1> [<rabbit@node3.1.10567.0>]
queue3 <rabbit@node3.1.9625.0> [<rabbit@node1.1.13080.1>]
```

如果镜像配置是ha-sync-mode=automatic 的情况，当有新的slave 出现时，此slave 会自动同步master 中的数据。注意在同步的过程中，集群的整个服务都不可用，客户端连接会被阻塞。如果master 中有大量的消息堆积，必然会造成slave 的同步时间增长，进一步影响了集群服务的可用性。如果配置ha-sync-mode=manual，有新的slave 创建的同时不会去同步master 上旧的数据，如果此时master 节点又发生了异常，那么此部分数据将会丢失。同样ha-promote-on-shutdown 这个参数的影响也需要考虑进来。

网络分区的发生可能会引起消息的丢失，当然这点也有办法解决。首先消息发送端要有能够处理Basic.Return 的能力。其次，在监测到网络分区发生之后，需要迅速地挂起所有的生产者进程。之后连接分区中的每个节点消费分区中所有的队列数据。在消费完之后再处理网络分区。最后在从网络分区中恢复之后再恢复生产者的进程。整个过程可以最大程度上保证网络分区之后的消息的可靠性。同样也要注意的是，在整个过程中会伴有大量的消息重复，消费者客户端需要做好相应的幂等性处理。当然也可以采用集群迁移，将所有旧集群的资源都迁移到新集群来解决这个问题。

### 手动处理网络分区
为了从网络分区中恢复，首先需要挑选一个信任分区，这个分区才有决定Mnesia 内容的权限，发生在其他分区的改变将不会被记录到Mnesia 中而被直接丢弃。在挑选完信任分区之后，重启非信任分区中的节点，如果此时还有网络分区的告警，紧接着重启信任分区中的节点。

这里有三个要点需要详细阐述：
- 如何挑选信任分区？
- 如何重启节点？
- 重启的顺序有何考究？

挑选信任分区一般可以按照这几个指标进行：  
分区中要有disc 节点；分区中的节点数最多；分区中的队列数最多；分区中的客户端连接数最多。优先级从前到后，例如信任分区中要有disc 节点；如果有两个或者多个分区满足，则挑选节点数最多的分区作为信任分区；如果又有两个或者多个分区满足，那么挑选队列数最多的分区作为信任分区。依次类推，如果有两个或者多个分区对于这些指标都均等，那么随机挑选一个分区也不失为一良策。

RabbitMQ 中有两种重启方式：
- 第一种方式是使用rabbitmqctl stop 命令关闭，然后再用rabbitmq-server –detached 命令启动；
- 第二种方式是使用rabbitmqctl stop_app 关闭，然后使用rabbitmqctl start_app 命令启动。

第一种方式需要同时重启Erlang 虚拟机和RabbitMQ 应用，而第二种方式只是重启RabbitMQ 应用。两种方式都可以从网络分区中恢复，但是更加推荐使用第二种方式。

RabbitMQ 的重启顺序也比较讲究，必须在以下两种重启顺序中择其一进行重启操作：
1. 停止其他非信任分区中的所有节点，然后再启动这些节点。如果此时还有网络分区的告警，则再重启信任分区中的节点以去除告警。
2. 关闭整个集群中的节点，然后再启动每一个节点，这里需要确保启动的第一个节点在信任的分区之中。

在选择哪种重启顺序之前，首先考虑一下队列“漂移”的现象。所谓的队列“漂移”是在配置镜像队列的情况下才会发生的。  
假设一共集群中有node1、node2 和node3 这三个节点，且配置全镜像（ha-mode=all），队列分布情况如下：

队列 | master | slaves
 :----: | :---- | :----:
queue1 | node1 | node2, node3
queue2 | node2 | node3, node1
queue3 | node3 | node2, node1

**情形一**  
这里首先关闭node3 节点，那么queue3 中的某个slave 提升为master，队列分布情况如下：

队列 | master | slaves
 :----: | :---- | :----:
queue1 | node1 | node2
queue2 | node2 | node1
queue3 | node2 | node1

然后在再关闭node2 节点，继续演变为以下情况：

队列 | master | slaves
 :----: | :---- | :----:
queue1 | node1 | []
queue2 | node1 | []
queue3 | node1 | []

此时，如果关闭node1 节点，然后再启动这3 个节点。或者不关闭node1 节点，而启动node2 和node3 节点都只会增加slave 的个数，而不会改变master 的分布，最终如情形如下所示。注意这里哪怕关闭了node1，然后并非先启动node1，而是先启动node2 或者node3，对于master 节点的分布都不会受影响。

队列 | master | slaves(按节点启动顺序排列)
 :----: | :---- | :----:
queue1 | node1 | node2, node3
queue2 | node1 | node2, node3
queue3 | node1 | node2, node3

这里就可以看出，随着节点的重启，所有的队列的master 都“漂移”到了node1 节点上，因为在RabbitMQ 中，除了发布消息，所有的操作都是在master 上完成的，如此大部分压力都集中到了node1 节点上，从而不能很好地实现负载均衡。

**情形二**  
如果在关闭节点node3 之后，又重新启动节点node3，队列分布情况如下：

队列 | master | slaves
 :----: | :---- | :----:
queue1 | node1 | node2, node3
queue2 | node2 | node1, node3
queue3 | node2 | node1, node3

之后再重启（先关闭，后启动）node2 节点，队列分布情况如下：

队列 | master | slaves
 :----: | :---- | :----:
queue1 | node1 | node3, node2
queue2 | node1 | node3, node2
queue3 | node1 | node3, node2

继续重启node1 节点，队列分布情况如下：

队列 | master | slaves
 :----: | :---- | :----:
queue1 | node3 | node2, node1
queue2 | node3 | node2, node1
queue3 | node3 | node2, node1

如此顺序演变，在配置镜像的集群中重启会有队列“漂移”的情况发生，造成负载不均衡。这里采用的是全镜像以作说明，不管如何，都难以避免队列“漂移”的发生。

<span style="color: red;font-weight: bold;">Tips</span>：一定要按照前面提及的两种方式择其一进行重启。如果选择挨个节点重启的方式，同样可以处理网络分区，但是这里会有一个严重的问题，即Mnesia 内容权限的归属问题。比如有两个分区[node1, node2]和[node3, node4]，其中[node1, node2]为信任分区。此时若按照挨个重启的方式进行重启，比如先重启node3，在node3 节点启动之时无法判断其节点的Mnesia 内容是向[node1, node2]分区靠齐还是向node4 节点靠齐。至此，如果挨个一轮重启之后，最终集群中的Mnesia 数据是[node3, node4]这个非信任分区，就会造成无法估量的损失。挨个节点重启也有可能会引起二次网络分区的发生。

如果原本配置了镜像队列，从发生网络分区到恢复的过程中队列可能会出现“漂移”的现象。可以重启之前先删除镜像队列的配置，这样能够在一定程度上阻止队列的“过分漂移”，即阻止可能所有队列都“漂移”到一个节点上的情况。  
删除镜像队列的配置：
- 可以采用rabbitmqctl 工具删除：
```bash
rabbitmqctl clear_policy [-p vhost] {mirror_queue_name}
```

- 可以通过Web 管理界面进行删除。
- 可以通过HTTP API 的方式进行删除：
```bash
curl -s -u {username:password} -X DELETE http://localhost:15672/api/policies/default/{mirror_queue_name}
```

需要在每个分区上都执行删除镜像队列配置的操作，以确保每个分区中的镜像都被删除。

--- 

**具体的网络分区处理步骤如下所述**
1. 步骤1：挂起生产者和消费者进程。这样可以减少消息不必要的丢失，如果进程数过多，情形又比较紧急，也可跳过此步骤。
2. 步骤2：删除镜像队列的配置。
3. 步骤3：挑选信任分区。
4. 步骤4：关闭非信任分区中的节点。采用rabbitmqctl stop_app 命令关闭。
5. 步骤5：启动非信任分区中的节点。采用与步骤4 对应的rabbitmqctl start_app 命令启动。
6. 步骤6：检查网络分区是否恢复，如果已经恢复则转步骤8；如果还有网络分区的报警则转步骤7。
7. 步骤7：重启信任分区中的节点。
8. 步骤8：添加镜像队列的配置。
9. 步骤9：恢复生产者和消费者的进程。

### 自动处理网络分区
RabbitMQ 提供了三种方法自动地处理网络分区：pause-minority 模式、pause-if-all-down 模式和autoheal 模式。  
默认是ignore 模式，即不自动处理网络分区，所以在这种模式下，当网络分区的时候需要人工介入。在rabbitmq.config 配置文件中配置cluster_partition_handling 参数即可实现相应的功能。默认的ignore 模式的配置如下：
```bash
[{
    rabbit, [
        {cluster_partition_handling, ignore}
    ]
}].
```

#### pause-minority 模式
当发生网络分区时，集群中的节点在观察到某些节点“down”的时候，会自动检测其自身是否处于“少数派”（分区中的节点小于或者等于集群中一半的节点数），RabbitMQ 会自动关闭这些节点的运作。根据CAP 原理，这里保障了P，即分区耐受性。这样确保了在发生网络分区的情况下，大多数节点（当然这些节点得在同一个分区中）可以继续运行。“少数派”中的节点在分区开始时会关闭，当分区结束时又会启动。  
这里关闭是指RabbitMQ 应用的关闭，而Erlang 虚拟机并不关闭，类似于执行了rabbitmqctl stop_app 命令。处于关闭的节点会每秒检测一次是否可连通到剩余集群中，如果可以则启动自身的应用。相当于执行rabbitmqctl start_app 命令。  
pause-minority 模式相应的配置如下：
```bash
[{
    rabbit, [
        {cluster_partition_handling, pause_minority}
    ]
}].
```

<span style="color: red;font-weight: bold;">Tips</span>：RabbitMQ 也会关闭不是严格意义上的大多数，比如在一个集群中只有两个节点的时候并不适合采用pause-minority 的模式，因为其中任何一个节点失败而发生网络分区时，两个节点都会关闭。当网络恢复时，有可能两个节点会自动启动恢复网络分区，也有可能仍保持关闭状态。然而如果集群中的节点数远大于2 个时，pause_minority 模式比ignore 模式更加可靠，特别是网络分区通常是由单节点网络故障而脱离原有分区引起的。  
不过也需要考虑2v2、3v3 这种被分裂成对等节点数的分区的情况。所谓的2v2 这种对等分区表示原有集群的组成为[node1, node2, node3, node4]，由于某种原因分裂成类似[node1, node2]和[node3, node4]这两个网络分区的情形。这种情况在跨机架部署时就有可能发生，当node1 和node2 部署在机架A 上，而node3 和node4 部署在机架B 上，那么有可能机架A 与机架B 之间网络的通断会造成对等分区的出现。  
接下来说明如何模拟对等的网络分区。可以在node1 和node2 上分别执行iptables 命令去封禁node3 和node4 的IP。如果node1、node2和node3、node4 处于不同的网段，那么也可以采用封禁网段的做法。更有甚者，可以将node1、node2 部署到物理机A 上的两台虚拟机中，然后将node3、node4 部署到物理机B 上的两台虚拟机中，之后切断物理机A 与B 之间的通信即可。  
当对等分区出现时，会关闭这些分区内的所有节点，对于前面的[node1, node2]和[node3, node4]的例子而言，这四个节点上的RabbitMQ 应用都会被关闭。只有等待网络恢复之后，才会自动启动所有的节点以求从网络分区中恢复。

#### pause-if-all-down 模式
RabbitMQ 集群中的节点在和所配置的列表中的任何节点不能交互时才会关闭，语法为{pause_if_all_down, [nodes], ignore|autoheal}，其中[nodes]为前面所说的列表，也可称之为受信节点列表。参考配置如下：
```bash
[{
    rabbit, [
    {cluster_partition_handling, 
        {pause_if_all_down, ['rabbit@node1'], ignore}}
    ]
}].
```

如果一个节点与rabbit@node1 节点无法通信时，则会关闭自身的RabbitMQ 应用。如果是rabbit@node1 本身发生了故障造成网络不可用，而其他节点都是正常的情况下，这种规则会让所有的节点中RabbitMQ 应用都关闭，待rabbit@node1 中的网络恢复之后，各个节点再启动自身应用以从网络分区中恢复。  
pause-if-all-down 模式下有ignore 和autoheal 两种不同的配置。考虑前面
pause-minority 模式中提及的一种情形，node1 和node2 部署在机架A 上，而node3 和node4 部署在机架B 上。此时配置{cluster_partition_handling, {pause_if_all_down,['rabbit@node1', 'rabbit@node3'], ignore}}，那么当机架A 和机架B 的通信出现异常时，由于node1 和node2 保持着通信，node3 和node4 保持着通信，这4 个节点都不会自行关闭，但是会形成两个分区，所以这样不能实现自动处理的功能。所以如果将配置中的ignore 替换成autoheal 就可以处理此种情形。

#### autoheal 模式
当认为发生网络分区时，RabbitMQ 会自动决定一个获胜（winning）的分区，然后重启不在这个分区中的节点来从网络分区中恢复。一个获胜的分区是指客户端连接最多的分区，如果产生一个平局，即有两个或者多个分区的客户端连接数一样多，那么节点数最多的一个分区就是获胜分区。如果此时节点数也一样多，将以节点名称的字典序来挑选获胜分区，相关源码如下：
```erlang
make_decision(AllPartitions)->
    Sorted = lists:sort([{partition_value(P),P} || P <- AllPartitions]),
    [[Winner | _] | Rest] = lists:reverse([P || {_, P} <- Sorted]),
    {Winner, lists:append(Rest)}.
partition_value(Partition) ->
    Connections = [Res || Node <- Partition,
        Res <- [rpc:call(Node, rabbit_networking,
            Connections_local,[])],
        is_list(Res)],
    {length(lists:append(Connections)), length(Partition)}.
```

autoheal 模式参考配置如下：
```bash
[{
    rabbit, [
        {cluster_partition_handling, autoheal}
    ]
}].
```

autoheal 模式在判定出net_tick_timeout 之时不做动作，要等到网络恢复之时，在判定出网络分区之后才会有相应的动作，即重启非获胜分区中的节点。  
在autoheal 模式下，如果集群中有节点处于非运行状态，那么当发生网络分区的时候，将不会有任何自动处理的动作。

##### 挑选哪种模式
允许RabbitMQ 能够自动处理网络分区并不一定会有正面的成效，也有可能会带来更多的问题。网络分区会导致RabbitMQ 集群产生众多的问题，需要对遇到的问题做出一定的选择。如果置RabbitMQ 于一个不可靠的网络环境下，需要使用Federation 或者Shovel。就算从网络分区中恢复了之后，也要谨防发生二次网络分区。  
每种模式都有自身的优缺点，没有哪种模式是万无一失的，希望根据实际情形做出相应的选择，下面简要概论以下4 个模式：
- ignore 模式：发生网络分区时，不做任何动作，需要人工介入。
- pause-minority 模式：对于对等分区的处理不够优雅，可能会关闭所有的节点。一般情况下，可应用于非跨机架、奇数节点数的集群中。
- pause-if-all-down 模式：对于受信节点的选择尤为考究，尤其是在集群中所有节点硬件配置相同的情况下。此种模式可以处理对等分区的情形。
- autoheal 模式：可以处于各个情形下的网络分区。但是如果集群中有节点处于非运行状态，则此种模式会失效。

### 案例：多分区情形
之前的讨论大多基于分成两个分区的情形。在实际应用中，如果集群中节点所在物理机是多网卡，当某节点网卡发生故障就有可能会发生多个分区的情形。

案例：集群中有6 个节点，分别为node1、node2、node3、node4、node5 和node6，每个节点所在物理机都是4 网卡（网卡名称分别为eth0、eth1、eth2 和eth3）配置，并采用bind0 的绑定模式。当node6 的eth0 故障之后，整个集群演变成为了6 个分区，即每个节点为一个独立的分区。  
网络分区之前，集群中的各个节点相互通信，为了简要说明，先只展示node1、node3 和node6 节点。如下图：  
![multi_partition](../images/rabbitmq/2024-04-03_multi_partition.png)

若node6 的网被关闭之后，对于bond0 的网卡绑定模式，交换机无法感知eth0 网卡的故障，但是node6 节点能够感知本地eth0 的故障。对于node3 节点而言，其与node6 的eth0 网卡建立的长连接没有被关闭，node3 会向node6 重试发送数据（TCP retransmission），但是node6 节点无法回应。除非主动关闭或者等待长连接超时（默认为7200s，即2 小时），此条链路才会被关闭。  
当node6 网卡关闭之后，node1、node3 和node6 有如下变化：
1. 与node6 上eth0 相关的链路不通，node3 此时需要等待net_ticktime 的超时。节点node3 中的相关日志如下：
```bash
rabbit on node 'rabbit@node6' down
node 'rabbit@node6' down: net_tick_timeout
```

2. 待超时之后，主动关闭连接。节点node3 的相关日志如下：
```bash
node 'rabbit@node6' down: connection_closed
```
与此同时Erlang 虚拟机尝试让node3 与node6 重新建立连接，由于node6 上的其他网卡正常，最后node3 和node6 可以建立。
```bash
node 'rabbit@node6' up
```

3. 判定node3 和node6 之间产生了网络分区。
```bash
Mnesia('rabbit@node3'): ** ERROR ** mnesia_event got {inconsistent_database,
running_partitioned_network, 'rabbit@node6'}
```

4. 到这里还没有结束，网络分区会继续演变。此时node3 和node1 还处于连通状态，同样node6 和node1 也处于连通状态。进一步查看node3 的日志如下，这段日志是说：node3 和node6 之间发生了网络分区，但是node3 又发现node1 和node6 内部通信还没有断，此时认为node1 和node6 处于同一个分区，那么node3 就准备主动关闭与node1 之间的内部通信，最后node3 和node1 之间也发生了分区。与此同时，对于节点node6 而言，node1 和node3 还处于同一个分区，那么node6 也要将node1 置于node6 本身的分区之外。最后node1、node3 与node6 都处于不同的网络分区。

```bash
Partial partition detected:
* We saw DOWN from rabbit@node6
* We can still see rabbit@node1 which can see rabbit@node6
We will therefore intentionally disconnect from rabbit@node1
```

5. 继续查看node1 中的日志，可以看到node1 此时察觉到node4 与node3 之间还有内部通信交换，那么就会主动将node4剥离出自身的分区。如此演变，最终node1、node2、node3、node4、node5 和node6 处于6 个不同的分区。

```bash
=ERROR REPORT==== 16-Oct-2017::14:20:54 ===
Partial partition detected:
* We saw DOWN from rabbit@node3
* We can still see rabbit@node4 which can see rabbit@node3
We will therefore intentionally disconnect from rabbit@node4
=INFO REPORT==== 16- Oct -2017::14:20:55 ===
rabbit on node 'rabbit@node4' down
=INFO REPORT==== 16- Oct -2017::14:20:55 ===
node 'rabbit@node4' down: disconnect
=INFO REPORT==== 16- Oct -2017::14:20:55 ===
node 'rabbit@node4' up
=ERROR REPORT==== 16- Oct -2017::14:20:55 ===
Mnesia('rabbit@node1'): ** ERROR ** mnesia_event got
{inconsistent_database, running_partitioned_network, 'rabbit@node4
```

**在此种故障下， 如果选择自动处理网络分区会有什么不同的效果呢？**  
对于pause_if_all_down 模式而言，如果挑选1 个节点作为受信节点，那么会重启剩余的5 个节点以作恢复。对于autoheal 同样如此，且查看日志可以发现，autoheal 会等网络分区判定之后罗列出所有分区信息，然后再重启非获胜分区中的节点，同样需要重启5 个节点。然而对于pause_minority 的配置而言，对此种情形的处理要优雅很多，当有节点检测到net_tick_timeout 之后会自行重启当前节点，这样就阻止了网络分区进一步演变，且处理效率最高。


## 负载均衡
面对大量业务访问、高并发请求，可以使用高性能的服务器来提升RabbitMQ 服务的负载能力。当单机容量达到极限时，可以采取集群的策略来对负载能力做进一步的提升，但这里还存在一个负载不均衡的问题。试想如果一个集群中有3 个节点，那么所有的客户端都与其中的单个节点node1 建立TCP 连接，那么node1 的网络负载必然会大大增加而显得难以承受，其他节点又由于没有那么多的负载而造成硬件资源的浪费，所以负载均衡显得尤为重要。  
对于RabbitMQ 而言，客户端与集群建立的TCP 连接不是与集群中所有的节点建立连接，而是挑选其中一个节点建立连接。如图所示，在引入了负载均衡之后，各个客户端的连接可以分摊到集群的各个节点之中，进而避免了前面所讨论的缺陷。  
![load_balance](../images/rabbitmq/2024-04-04_load_balance.png)

负载均衡（Load balance）是一种计算机网络技术，用于在多个计算机（计算机集群）、网络连接、CPU、磁盘驱动器或其他资源中分配负载，以达到最佳资源使用、最大化吞吐率、最小响应时间及避免过载的目的。使用带有负载均衡的多个服务器组件，取代单一的组件，可以通过冗余提高可靠性。  
负载均衡通常分为软件负载均衡和硬件负载均衡两种：
- **软件负载均衡**是指在一个或者多个交互的网络系统中的多台服务器上安装一个或多个相应的负载均衡软件来实现的一种均衡负载技术。软件可以很方便地安装在服务器上，并且实现一定的均衡负载功能。软件负载均衡技术配置简单、操作也方便，最重要的是成本很低。
- **硬件负载均衡**是指在多台服务器间安装相应的负载均衡设备，也就是负载均衡器（如F5）来完成均衡负载技术，与软件负载均衡技术相比，能达到更好的负载均衡效果。由于硬件负载均衡技术需要额外增加负载均衡器，成本比较高，所以适用于流量高的大型网站系统。

目前主流的对RabbitMQ 集群使用软件负载均衡的方式有在客户端内部实现负载均衡，或者使用HAProxy、LVS 等负载均衡软件来实现。

#### 客户端内部实现负载均衡
对于RabbitMQ 而言可以在客户端连接时简单地使用负载均衡算法来实现负载均衡。负载均衡算法有很多种，主流的有以下几种。

1. 轮询法  
将请求按顺序轮流地分配到后端服务器上，它均衡地对待后端的每一台服务器，而不关心服务器实际的连接数和当前的系统负载。
```java
public class RoundRobin {
    private static List<String> list = new ArrayList<String>(){{
        add("192.168.0.2");
        add("192.168.0.3");
        add("192.168.0.4");
    }};
    private static int pos = 0;
    private static final Object lock = new Object();
    //调用该方法来获取相应的连接地址
    public static String getConnectionAddress(){
        String ip = null;
        synchronized (lock) {
            ip = list.get(pos);
            if (++pos >= list.size()) {
                pos = 0;
            }
        }
        return ip;
    }
}
```

2. 加权轮询法  
不同的后端服务器的配置可能和当前系统的负载并不相同，因此它们的抗压能力也不相同。给配置高、负载低的机器配置更高的权重，让其处理更多的请求；而配置低、负载高的集群，给其分配较低的权重，降低其系统负载，加权轮询能很好地处理这一问题，并将请求顺序和权重分配到后端。

3. 随机法  
通过随机算法，根据后端服务器的列表大小值来随机选取其中的一台服务器进行访问。由概率统计理论可以得知，随着客户端调用服务端的次数增多，其实际效果越来越接近于平均分配调用量到后端的每一台服务器，也就是轮询的结果。
```java
public class RandomAccess {
    private static List<String> list = new ArrayList<String>(){{
        add("192.168.0.2");
        add("192.168.0.3");
        add("192.168.0.4");
    }};
    public static String getConnectionAddress(){
        Random random = new Random();
        int pos = random.nextInt(list.size());
        return list.get(pos);
    }
}
```

4. 加权随机法  
与加权轮询法一样，加权随机法也根据后端机器的配置、系统的负载分配不同权重。不同的是，它按照权重随机请求后端服务器，而非顺序。

5. 源地址哈希法  
源地址哈希的思想是根据获取的客户端IP 地址，通过哈希函数计算得到的一个数值，用该数值对服务器列表的大小进行取模运算，得到的结果便是客户端要访问服务器的序号。采用源地址哈希法进行负载均衡，同一IP 地址的客户端，当后端服务器列表不变时，它每次都会映射到同一台后端服务器进行访问。
```java
public class IpHash {
    private static List<String> list = new ArrayList<String>(){{
        add("192.168.0.2");
        add("192.168.0.3");
        add("192.168.0.4");
    }};
    public static String getConnectionAddress() throws UnknownHostException {
        int ipHashCode = InetAddress.getLocalHost().getHostAddress().hashCode();
        int pos = ipHashCode % list.size();
        return list.get(pos);
    }
}
```

6. 最小连接数法  
最小连接数算法比较灵活和智能，由于后端服务器的配置不尽相同，对于请求的处理有块有慢，它根据后端服务器当前的连接情况，动态地选取其中当前积压连接数最少的一台服务器来处理当前的请求，尽可能地提高后端服务的利用效率，将负载合理地分流到每一台服务器。

#### 使用HAProxy 实现负载均衡
HAProxy 提供高可用性、负载均衡及基于TCP 和HTTP 应用的代理，支持虚拟主机，它是免费、快速并且可靠的一种解决方案，包括Twitter、Reddit、StackOverflow、GitHub 在内的多家知名互联网公司在使用。HAProxy 实现了一种事件驱动、单一进程模型，此模型支持非常大的并发连接数。

- 安装HAProxy  
首先需要去HAProxy 的官网下载HAProxy 的安装文件， 目前最新的版本为haproxy-1.7.8.tar.gz。下载地址为http://www.haproxy.org/#down ，相关文档地址为http://www.haproxy.org/#doc1.7

将haproxy-1.7.8.tar.gz 复制至/opt 目录下，与RabbitMQ 存放在同一个目录中，之后进行解压缩：

```bash
[root@node1 opt]# tar zxvf haproxy-1.7.8.tar.gz
```

将源码解压之后，需要运行make 命令来将HAProxy 编译为可执行程序。在执行make 之前需要先选择目标平台，通常对于UNIX 系的操作系统可以选择TARGET=generic。下面是详细操作：

```bash
[root@node1 opt]# cd haproxy-1.7.8
[root@node1 haproxy-1.7.8]# make TARGET=generic
gcc -Iinclude -Iebtree -Wall -O2 -g -fno-strict-aliasing
    -Wdeclaration-after-statement -fwrapv
-DTPROXY -DENABLE_POLL
-DCONFIG_HAPROXY_VERSION=\"1.7.8\"
-DCONFIG_HAPROXY_DATE=\"2017/07/07\" \
    -DBUILD_TARGET='"generic"' \
    -DBUILD_ARCH='""' \
    -DBUILD_CPU='"generic"' \
    -DBUILD_CC='"gcc"' \
    -DBUILD_CFLAGS='"-O2 -g -fno-strict-aliasing -Wdeclaration-after-statement
        -fwrapv"' \
    -DBUILD_OPTIONS='""' \
    -c -o src/haproxy.o src/haproxy.c
gcc -Iinclude -Iebtree -Wall -O2 -g -fno-strict-aliasing
    -Wdeclaration-after-statement -fwrapv...
...
gcc -g -o haproxy src/haproxy.o src/base64.o src/protocol.o src/uri_auth.o ...
```

编译完目录下有名为“haproxy”的可执行文件。之后在/etc/profile 中加入haproxy 的路径，内容如下：

```bash
export PATH=$PATH:/opt/haproxy-1.7.8/haproxy
```

最后执行source/etc/profile 让此环境变量生效。

- 配置HAProxy  
HAProxy 使用单一配置文件来定义所有属性，包括从前端IP 到后端服务器。

下面的代码展示了用于3 个RabbitMQ 节点组成集群的负载均衡配置。  
配置相关环境说明如下所述：  
▶ HAProxy 主机：192.168.0.9 5671  
▶ RabbitMQ 1：192.168.02 5672  
▶ RabbitMQ 2：192.168.03 5672  
▶ RabbitMQ 3：192.168.04 5672  

```bash
# HAProxy 的配置
#全局配置
global
    #日志输出配置，所有日志都记录在本机，通过local0 输出
    log 127.0.0.1 local0 info
    #最大连接数
    maxconn 4096
    #改变当前的工作目录
    chroot /opt/haproxy-1.7.8
    #以指定的UID 运行haproxy 进程
    uid 99
    #以指定的GID 运行haproxy 进程
    gid 99
    #以守护进程方式运行haproxy #debug #quiet
    daemon
    #debug
    #当前进程pid 文件
    pidfile /opt/haproxy-1.7.8/haproxy.pid
#默认配置
defaults
    #应用全局的日志配置
    log global
    #默认的模式mode{tcp|http|health}
    #TCP 是4 层，HTTP 是7 层，health 只返回OK
    mode tcp
    #日志类别tcplog
    option tcplog
    #不记录健康检查日志信息
    option dontlognull
    #3 次失败则认为服务不可用
    retries 3
    #每个进程可用的最大连接数
    maxconn 2000
    #连接超时
    timeout connect 5s
    #客户端超时
    timeout client 120s
    #服务端超时
    timeout server 120s
#绑定配置
listen rabbitmq_cluster :5671
    #配置TCP 模式
    mode tcp
    #简单的轮询
    balance roundrobin
    #RabbitMQ 集群节点配置
    server rmq_node1 192.168.0.2:5672 check inter 5000 rise 2 fall 3 weight 1
    server rmq_node2 192.168.0.3:5672 check inter 5000 rise 2 fall 3 weight 1
    server rmq_node3 192.168.0.4:5672 check inter 5000 rise 2 fall 3 weight 1
#haproxy 监控页面地址
listen monitor :8100
    mode http
    option httplog
    stats enable
    stats uri /stats
    stats refresh 5s
```

在前面的配置中“listen rabbitmq_cluster bind 192.168.0.9.5671”定义了客户端连接IP 地址和端口号。这里配置的负载均衡算法是roundrobin，注意roundrobin 是加权轮询。  
和RabbitMQ 最相关的是“server rmq_node1 192.168.0.2:5672 check inter 5000 rise 2 fall 3 weight 1”此类型的3 条配置，它定义了RabbitMQ 服务的负载均衡细节，其中包含6 个部分：  
1. server \<name>：定义RabbitMQ 服务的内部标识，注意这里的“rmq_node1”是指包含有含义的字符串名称，不是指RabbitMQ 的节点名称。
2. \<ip>:\<port>：定义RabbitMQ 服务连接的IP 地址和端口号。
3. check inter \<value>：定义每隔多少毫秒检查RabbitMQ 服务是否可用。
4. rise \<value>：定义RabbitMQ 服务在发生故障之后，需要多少次健康检查才能被再次确认可用。
5. fall \<value>：定义需要经历多少次失败的健康检查之后，HAProxy 才会停止使用此RabbitMQ 服务。
6. weight \<value>：定义当前RabbitMQ 服务的权重。

HAProxy 的配置代码中最后一段配置定义的是HAProxy 的数据统计页面。数据统计页面包含各个服务节点的状态、连接、负载等信息。在调用haproxy -f haproxy.cfg 命令运行HAProxy 服务之后，可以在浏览器上输入http://192.168.0.9:8100/stats 来加载相关的页面。

#### 使用Keepalived 实现高可靠负载均衡
如果前面配置的HAProxy 主机192.168.0.9 突然宕机或者网卡失效，那么虽然RabbitMQ 集群没有任何故障，但是对于外界的客户端来说所有的连接都会被断开，结果将是灾难性的。确保负载均衡服务的可靠性同样显得十分重要。这里就需要引入Keepalived 工具，它能够通过自身健康检查、资源接管功能做高可用（双机热备），实现故障转移。

Keepalived 采用VRRP（Virtual Router Redundancy Protocol，虚拟路由冗余协议），以软件的形式实现服务的热备功能。通常情况下是将两台Linux 服务器组成一个热备组（Master 和Backup），同一时间内热备组只有一台主服务器Master 提供服务，同时Master 会虚拟出一个公用的虚拟IP 地址，简称VIP。这个VIP 只存在于Master 上并对外提供服务。如果Keepalived 检测到Master 宕机或者服务故障，备份服务器Backup 会自动接管VIP 并成为Master，Keepalived 将原Master 从热备组中移除。当原Master 恢复后，会自动加入到热备组，默认再抢占成为Master，起到故障转移的功能。  
Keepalived 工作在OSI 模型中的第3 层、第4 层和第7 层。  
工作在第3 层是指Keepalived 会定期向热备组中的服务器发送一个ICMP 数据包来判断某台服务器是否故障，如果故障则将这台服务器从热备组移除。  
工作在第4 层是指Keepalived 以TCP 端口的状态判断服务器是否故障，比如检测RabbitMQ的5672 端口，如果故障则将这台服务器从热备组中移除。  
工作在第7 层是指Keepalived 根据用户设定的策略（通常是一个自定义的检测脚本）判断服务器上的程序是否正常运行，如果故障将这台服务器从热备组移除。

1. Keepalived 的安装  
首先需要去Keepalived 的官网下载安装文件，目前最新的版本为keepalived-1.3.5.tar.gz，下载地址为http://www.keepalived.org/download.html

将keepalived-1.3.5.tar.gz 解压并安装，详细步骤如下：

```bash
[root@node1 ~]# tar zxvf keepalived-1.3.5.tar.gz
[root@node1 ~]# cd keepalived-1.3.5
[root@node1 keepalived-1.3.5]# ./configure --prefix=/opt/keepalived --withinit=SYSV
#注：(upstart|systemd|SYSV|SUSE|openrc) #根据你的系统选择对应的启动方式
[root@node1 keepalived-1.3.5]# make
[root@node1 keepalived-1.3.5]# make install
```

之后将安装过后的Keepalived 加入系统服务中，详细步骤如下（注意千万不要输错命令）：

```bash
#复制启动脚本到/etc/init.d/下
[root@node1 ~]# cp /opt/keepalived/etc/rc.d/init.d/keepalived /etc/init.d/
[root@node1 ~]# cp /opt/keepalived/etc/sysconfig/keepalived /etc/sysconfig
[root@node1 ~]# cp /opt/keepalived/sbin/keepalived /usr/sbin/
[root@node1 ~]# chmod +x /etc/init.d/keepalived
[root@node1 ~]# chkconfig --add keepalived
[root@node1 ~]# chkconfig keepalived on
#Keepalived 默认会读取/etc/keepalived/keepalived.conf 配置文件
[root@node1 ~]# mkdir /etc/keepalived
[root@node1 ~]# cp /opt/keepalived/etc/keepalived/keepalived.conf /etc/keepalived/
```

执行完之后就可以使用如下命令来重启、启动、关闭和查看keepalived 状态：

```bash
service keepalived restart
service keepalived start
service keepalived stop
service keepalived status
```

2. 配置  
在安装的时候我们已经创建了/etc/keepalived 目录，并将keepalived.conf 配置文件复制到此目录下，如此Keepalived 便可以读取这个默认的配置文件了。如果要将Keepalived 与前面的HAProxy 服务结合起来需要更改/etc/keepalived/keepalived.conf 这个配置文件。

如下图所示，两台Keepalived 服务器之间通过VRRP 进行交互，对外部虚拟出一个VIP 为192.168.0.10。Keepalived 与HAProxy 部署在同一台机器上，两个Keepalived 服务实例匹配两个HAProxy 服务实例，这样通过Keeaplived 实现HAProxy 的双机热备。所以在上一节的192.168.0.9 的基础之上，还要再部署一台HAProxy 服务，IP 地址为192.168.0.8。  
![VRRP_interaction](../images/rabbitmq/2024-04-04_VRRP_interaction.png) _通过VRRP交互_

整条调用链路为：客户端通过VIP 建立通信链路；通信链路通过Keeaplived 的Master 节点路由到对应的HAProxy 之上；HAProxy 通过负载均衡算法将负载分发到集群中的各个节点之上。正常情况下客户端的连接通过上图中左侧部分进行负载分发。当Keepalived 的Master 节点挂掉或者HAProxy 挂掉无法恢复时，Backup 提升为Master，客户端的连接通过上图中右侧部分进行负载分发。

接下来修改/etc/keepalived/keepalived.conf 文件，在Keepalived 的Master 上配置详情如下：
```bash
#Keepalived 配置文件
global_defs {
    router_id NodeA #路由ID、主/备的ID 不能相同
}
#自定义监控脚本
vrrp_script chk_haproxy {
    script "/etc/keepalived/check_haproxy.sh"
    interval 5
    weight 2
}
vrrp_instance VI_1 {
    state MASTER #Keepalived 的角色。Master 表示主服务器，从服务器设置为BACKUP
    interface eth0 #指定监测网卡
    virtual_router_id 1
    priority 100 #优先级，BACKUP 机器上的优先级要小于这个值
    advert_int 1 #设置主备之间的检查时间，单位为s
    authentication { #定义验证类型和密码
        auth_type PASS
        auth_pass root123
    }
    track_script {
        chk_haproxy
    }
    virtual_ipaddress { #VIP 地址，可以设置多个
        192.168.0.10
    }
}
```

Backup 中的配置大致和Master 中的相同，不过需要修改global_defs{}的router_id，比如设置为“NodeB”；其次要修改vrrp_instance VI_1{}中的state 为“BACKUP”；最后要将priority 设置为小于100 的值。注意Master 和Backup 中的virtual_router_id 要保持一致。下面简要地展示一下Backup 的配置：
```bash
global_defs {
    router_id NodeB
}
vrrp_script chk_haproxy {
    ...
}
vrrp_instance VI_1 {
    state BACKUP
    ...
    priority 50
    ...
}
```

为了防止HAProxy 服务挂掉之后Keepalived 还在正常工作而没有切换到Backup 上，所以这里需要编写一个脚本来检测HAProxy 服务的状态。当HAProxy 服务挂掉之后该脚本会自动重启HAProxy 的服务，如果不成功则关闭Keepalived 服务，如此便可以切换到Backup 继续工作。这个脚本就对应了上面配置vrrp_script chk_haproxy{}中的script 对应的值，/etc/keepalived/check_haproxy.sh 的内容如下所示（记得添加可执行权限）。  
```bash
#!/bin/bash
if [ $(ps -C haproxy --no-header | wc -l) -eq 0 ];then
    haproxy -f /opt/haproxy-1.7.8/haproxy.cfg
fi
sleep 2
if [ $(ps -C haproxy --no-header | wc -l) -eq 0 ];then
    service keepalived stop
fi
```

如此配置好之后，使用service keepalived start 命令启动192.168.0.8 和192.168.0.9 中的Keepalived 服务即可。之后客户端的应用可以通过192.168.0.10 这个IP 地址来接通RabbitMQ 服务。

3. 查看Keepalived 运行情况  
可以通过tail -f /var/log/messages -n 200 命令查看相应的Keepalived 日志输出。  
Master 启动之后可以通过ip add show 命令查看添加的VIP（Backup 节点是没有VIP 的）：

```bash
[root@node1 ~]# ip add show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
    inet6 ::1/128 scope host
        valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc pfifo_fast state UP
    qlen 1000
    link/ether fa:16:3e:5e:7a:f7 brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.8/18 brd 10.198.255.255 scope global eth0
    # VIP
    inet 192.168.0.10/32 scope global eth0
    inet6 fe80::f816:3eff:fe5e:7af7/64 scope link
        valid_lft forever preferred_lft forever
```

在Master 节点执行service keepalived stop 模拟异常关闭的情况，观察Master 的日志，对应的Master 上的VIP 也会消失。  
Master 关闭后，Backup 会提升为新的Master，可以观察Backup 的日志。通过ip add show 命令可以看到新的Master 节点上虚拟出了VIP，如下所示：
```bash
[root@node2 ~]# ip add show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
    inet6 ::1/128 scope host
        valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc pfifo_fast state UP
    qlen 1000
    link/ether fa:16:3e:23:ac:ec brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.9/18 brd 10.198.255.255 scope global eth0
    # VIP
    inet 192.168.0.10/32 scope global eth0
    inet6 fe80::f816:3eff:fe23:acec/64 scope link
        valid_lft forever preferred_lft forever
```

Keeaplived 的出现让HAProxy 的负载均衡服务更加可靠。如果想要追求要更高的可靠性，可以加入多个Backup 角色的Keepalived 节点来实现一主多从的多机热备。当然这样会提升硬件资源的成本，该如何抉择需要更细致的考量，一般情况下双机热备的配备已足够满足应用需求。

#### 使用Keepalived+LVS 实现负载均衡
LVS 是Linux Virtual Server 的简称，也就是Linux 虚拟服务器，是一个由章文嵩博士发起的自由软件项目，它的官方站点是linuxvirtualserver.org。现在LVS 已经是 Linux 标准内核的一部分，在Linux2.6.32 内核以前，使用LVS 时必须要重新编译内核以支持LVS 功能模块，但是从Linux2.6.32 内核以后，已经完全内置了LVS 的各个功能模块，无须给内核打任何补丁，可以直接使用LVS 提供的各种功能。  
LVS 是4 层负载均衡，也就是说建立在OSI 模型的传输层之上。LVS 支持TCP/UDP 的负载均衡，相对于其他高层负载均衡的解决方案，比如DNS 域名轮流解析、应用层负载的调度、客户端的调度等，它是非常高效的。LVS 自从1998 年开始，发展到现在已经是一个比较成熟的技术项目了。可以利用LVS 技术实现高可伸缩的、高可用的网络服务。例如，WWW 服务、Cache 服务、DNS 服务、FTP 服务、MAIL 服务、视频/音频点播服务等。有许多比较著名网站和组织都在使用LVS 架设的集群系统。例如，Linux 的门户网站（linux.com）、向RealPlayer 提供音频视频服务而闻名的Real 公司（real.com）、全球最大的开源网站（sourceforge.net）等。

LVS 主要由3 部分组成：
- 负载调度器（Load Balancer/Director）：它是整个集群对外面的前端机，负责将客户的请求发送到一组服务器上执行，而客户认为服务是来自一个IP 地址（VIP）上的。
- 服务器池（Server Pool/RealServer）：一组真正执行客户端请求的服务器，如RabbitMQ 服务器。
- 共享存储（Shared Storage）：它为服务器池提供一个共享的存储区，这样很容易使服务器池拥有相同的内容，提供相同的服务。

目前LVS 的负载均衡方式也分为三种：
- VS/NAT：Virtual Server via Network Address Translation 的简称。VS/NAT 是一种最简单的方式，所有的RealServer 只需要将自己的网关指向Director 即可。客户端可以是任意的操作系统，但此方式下一个Director 能够带动的RealServer 比较有限。
- VS/TUN：Virtual Server via IP Tunneling 的简称。IP 隧道（IP Tunneling）是将一个IP 报文封装再另一个IP 报文的技术，这可以使目标为一个IP 地址的数据报文能够被封装和转发到另一个IP 地址。IP 隧道技术也可以称之为IP 封装技术（IP encapsulation）。
- VS/DR：即Virtual Server via Direct Routing 的简称。VS/DR 方式是通过改写报文中的MAC 地址部分来实现的。Director 和RealServer 必须在物理上有一个网卡通过不间断的局域网相连。RealServer 上绑定的VIP 配置在各自Non-ARP 的网络设备上（如lo 或tunl），Director 的VIP 地址对外可见，而RealServer 的VIP 对外是不可见的。RealServer 的地址既可以是内部地址，也可以是真实地址。

对于LVS 而言配合Keepalived 一起使用同样可以实现高可靠的负载均衡，对于上面“通过VRRP交互”图示中的结构，LVS 可以完全替代HAProxy 而其他内容可以保持不变。LVS 不需要额外的配置文件，直接集成在Keepalived 的配置文件之中。修改/etc/keepalived/keepalived.conf 文件内容如下：
```bash
#Keepalived 配置文件（Master）
global_defs {
    router_id NodeA #路由ID、主/备的ID 不能相同
}
vrrp_instance VI_1 {
    state MASTER #Keepalived 的角色。Master 表示主服务器，从服务器设置为BACKUP
    interface eth0 #指定监测网卡
    virtual_router_id 1
    priority 100 #优先级，BACKUP 机器上的优先级要小于这个值
    advert_int 1 #设置主备之间的检查时间，单位为s
    authentication { #定义验证类型和密码
        auth_type PASS
        auth_pass root123
    }
    track_script {
        chk_haproxy
    }
    virtual_ipaddress { #VIP 地址，可以设置多个：
        192.168.0.10
    }
}
virtual_server 192.168.0.10 5672 { #设置虚拟服务器
    delay_loop 6 #设置运行情况检查时间，单位是秒
    #设置负载调度算法，共有rr、wrr、lc、wlc、lblc、lblcr、dh、sh 这8 种
    lb_algo wrr #这里是加权轮询
    lb_kind DR #设置LVS 实现的负载均衡机制方式VS/DR
    #指定在一定的时间内来自同一IP 的连接将会被转发到同一RealServer 中
    persistence_timeout 50
    protocal TCP #指定转发协议类型，有TCP 和UDP 两种
    #这个real_server 即LVS 的三大部分之一的RealServer，这里特指RabbitMQ 的服务
    real_server 192.168.0.2 5672 { #配置服务节点
        weight 1 #配置权重
        TCP_CHECK {
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
            connect_port 5672
        }
    }
    real_server 192.168.0.3 5672 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
            connect_port 5672
        }
    }
    real_server 192.168.0.4 5672 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
            connect_port 5672
        }
    }
}
#为RabbitMQ 的RabbitMQ Management 插件设置负载均衡
virtual_server 192.168.0.10 15672 {
    delay_loop 6
    lb_algo wrr
    lb_kind DR
    persistence_timeout 50
    protocal TCP
    real_server 192.168.0.2 15672 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
            connect_port 15672
        }
    }
    real_server 192.168.0.3 15672 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
            connect_port 15672
        }
    }
    real_server 192.168.0.4 15672 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
            connect_port 15672
        }
    }
}
```

对于Backup 的配置可以参考前一节中的相应配置。在LVS 和Keepalived 环境里面，LVS 主要的工作是提供调度算法，把客户端请求按照需求调度在RealServer 中，Keepalived 主要的工作是提供LVS 控制器的一个冗余，并且对RealServer 进行健康检查，发现不健康的RealServer 就把它从LVS 集群中剔除，RealServer 只负责提供服务。  
通常在LVS 的VS/DR 模式下需要在RealServer 上配置VIP。原因在于当LVS 把客户端的包转发给RealServer 时，因为包的目的IP 地址是VIP，如果RealServer 收到这个包后发现包的目的地址不是自己系统的IP，会认为这个包不是发给自己的，就会丢弃这个包，所以需要将这个IP 地址绑定到网卡下。当发送应答包给客户端时，RealServer 就会把包的源和目的地址调换，直接回复给客户端。下面为所有的RealServer 的lo:0 网卡创建启动脚本（vim /opt/realserver.sh）绑定VIP 地址，详细内容如下：
```bash
#!/bin/bash
VIP=192.168.0.10
/etc/rc.d/init.d/functions

case "$1" in
start)
    /sbin/ifconfig lo:0 $VIP netmask 255.255.255.255 broadcast $VIP
    /sbin/route add -host $VIP dev lo:0
    echo "1" >/proc/sys/net/ipv4/conf/lo/arp_ignore
    echo "2" >/proc/sys/net/ipv4/conf/lo/arp_announce
    echo "1" >/proc/sys/net/ipv4/conf/all/arp_ignore
    echo "2" >/proc/sys/net/ipv4/conf/all/arp_announce
    sysctl -p >/dev/null 2>&1
    echo "RealServer Start Ok"
;;
stop)
    /sbin/ifconfig lo:0 down
    /sbin/route del -host $VIP dev lo:0
    echo "0" >/proc/sys/net/ipv4/conf/lo/arp_ignore
    echo "0" >/proc/sys/net/ipv4/conf/lo/arp_announce
    echo "0" >/proc/sys/net/ipv4/conf/all/arp_ignore
    echo "0" >/proc/sys/net/ipv4/conf/all/arp_announce
;;
status)
    islothere=`/sbin/ifconfig lo:0 | grep $VIP | wc -l`
    isrothere=`netstat -rn | grep "lo:0"|grep $VIP | wc -l`
    if [ $islothere -eq 0 ]
    then
        if [ $isrothere -eq 0 ]
        then
            echo "LVS of RealServer Stopped."
        else
            echo "LVS of RealServer Running."
        fi
    else
        echo "LVS of RealServer Running."
    fi
;;
*)
    echo "Usage:$0{start|stop}"
    exit 1
;;
esac
```

注意上面绑定VIP 的掩码是255.255.255.255，说明广播地址是其自身，那么它就不会将ARP 发送到实际的自己该属于的广播域了，这样防止与LVS 上的VIP 冲突进而导致IP 地址冲突。为/opt/realserver.sh 文件添加可执行权限后，运行/opt/realserver.sh start 命令之后可以通过ip add show 命令查看lo:0 网卡的状态，注意与Keepalived 节点的网卡状态进行区分。
```bash
[root@node1 keepalived]# ip add show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
    inet 192.168.0.10/32 brd 10.198.197.74 scope global lo:0
    inet6 ::1/128 scope host
        valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc pfifo_fast state UP
    qlen 1000
    link/ether fa:16:3e:5e:7a:f7 brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.2/18 brd 10.198.255.255 scope global eth0
    inet6 fe80::f816:3eff:fe5e:7af7/64 scope link
        valid_lft forever preferred_lft forever
```
