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


## Federation


## Shovel


