> RabbitMQ 版本为3.6.10

### RabbitMQ的优点
- **可靠性**：RabbitMQ 使用一些机制来保证可靠性，如持久化、传输确认及发布确认等。
- **灵活的路由**：在消息进入队列之前，通过交换器来路由消息。对于典型的路由功能，RabbitMQ 已经提供了一些内置的交换器来实现。针对更复杂的路由功能，可以将多个交换器绑定在一起，也可以通过插件机制来实现自己的交换器。
- **扩展性**：多个 RabbitMQ 节点可以组成一个集群，也可以根据实际业务情况动态地扩展集群中节点。
- **高可用性**：队列可以在集群中的机器上设置镜像，使得在部分节点出现问题的情况下队列仍然可用。
- **多种协议**：RabbitMQ除了原生支持AMQP协议，还支持STOMP、MQTT 等多种消息中间件协议。
- **多语言客户端**：RabbitMQ几乎支持所有常用语言，比如Java、Python、Ruby、PHP、C#、JavaScript 等。
- **管理界面**：RabbitMQ 提供了一个易用的用户界面，使得用户可以监控和管理消息、集群中的节点等。
- **插件机制**：RabbitMQ 提供了许多插件，以实现从多方面进行扩展，当然也可以编写自己的插件。

### RabbitMQ的模型架构
![rabbitmq_model_architecture](../images/mq/2024-1-25_RabbitMQ模型架构.png)

### RabbitMQ基本概念
#### Producer：生产者
就是投递消息的一方。生产者创建消息，然后发布到RabbitMQ 中。  
消息一般可以包含2个部分：**消息体**和**标签**（Label）。  
消息体也可以称之为payload，在实际应用中，消息体一般是一个带有业务逻辑结构的数据，比如一个JSON字符串。当然可以进一步对这个消息体进行序列化操作。  
消息的标签用来表述这条消息，比如一个交换器的名称和一个路由键。生产者把消息交由RabbitMQ，RabbitMQ之后会根据标签把消息发送给对应的Broker。

#### Consumer：消费者
就是接收消息的一方。消费者连接到RabbitMQ 服务器，并订阅到队列上。当消费者消费一条消息时，只是消费消息的消息体（payload）。  
在消息路由的过程中，消息的标签会丢弃，存入到队列中的消息只有消息体，消费者也只会消费到消息体，也就不知道消息的生产者是谁，当然消费者也不需要知道。

#### Broker
一个RabbitMQ Broker 可以简单地看作一个RabbitMQ 服务节点，或者RabbitMQ 服务实例。大多数情况下也可以将一个RabbitMQ Broker 看作一台RabbitMQ服务器。

#### Queue：队列
RabbitMQ 的内部对象，用于存储消息。  
RabbitMQ 中消息都只能存储在队列中，这一点和Kafka 这种消息中间件相反。Kafka 将消息存储在topic（主题）这个逻辑层面，而相对应的队列逻辑只是topic 实际存储文件中的位移标识。  
多个消费者可以订阅同一个队列，这时队列中的消息会被平均分摊（Round-Robin，即轮询）给多个消费者进行处理，而不是每个消费者都收到所有的消息并处理。  
<span style="color: red;">tips</span>：RabbitMQ 不支持队列层面的广播消费，如果需要广播消费，需要在其上进行二次开发，处理逻辑会变得异常复杂，同时也不建议这么做。

#### Exchange：交换器
生产者将消息发送到Exchange（交换器，通常也可以用大写的“X”来表示），由交换器将消息路由到一个或者多个队列中。如果路由不到，或许会返回给生产者，或许直接丢弃。

##### 交换器类型
RabbitMQ 常用的交换器类型有fanout、direct、topic、headers 这四种。  
<span style="color: red;">tips</span>：AMQP 协议里还提到另外两种类型：System 和自定义  

- **fanout**  
它会把所有发送到该交换器的消息路由到所有与该交换器绑定的队列中。

- **direct**  
它会把消息路由到那些BindingKey 和RoutingKey 完全匹配的队列中。

- **topic**  
按照一定的匹配规则将消息路由到BindingKey 和RoutingKey 相匹配的队列中。
> RoutingKey 和BindingKey 都是用一个点号“.”分隔的字符串（被点号“.”分隔开的每一段独立的字符串称为一个单词），如“com.rabbitmq.client”  
> BindingKey 中可以存在两种特殊字符串“\*”和“#”，用于做模糊匹配，其中“\*”用于匹配一个单词，“#”用于匹配多规格单词（可以是零个）  

例："*.log" 将匹配任何以 .log 结尾的消息路由键，如 "info.log" 或 "error.log"，但不会匹配 "debug.user.log"  
&emsp;&emsp; "topic.#" 将匹配任何以 topic. 开头的所有路由键，如 "topic.info"、"topic.error.subitem" 等

- **headers**  
headers 类型的交换器不依赖于路由键的匹配规则来路由消息，而是根据发送的消息内容中的headers 属性进行匹配。在绑定队列和交换器时制定一组键值对，当发送消息到交换器时，RabbitMQ 会获取到该消息的headers（也是一个键值对的形式），对比其中的键值对是否完全匹配队列和交换器绑定时指定的键值对，如果完全匹配则消息会路由到该队列，否则不会路由到该队列。headers 类型的交换器性能会很差，而且也不实用，基本上不会看到它的存在。

#### RoutingKey：路由键
生产者将消息发给交换器的时候，一般会指定一个RoutingKey，用来指定这个消息的路由规则，而这个Routing Key 需要与交换器类型和绑定键（BindingKey）联合使用才能最终生效。

##### BindingKey
在绑定的时候使用的路由键（the routing key to use for the binding）。大多数时候，包括官方文档和RabbitMQ Java API中都把BindingKey 和RoutingKey 看作RoutingKey。

#### Binding：绑定
RabbitMQ 中通过绑定将交换器与队列关联起来，在绑定的时候一般会指定一个绑定键（BindingKey），生产者将消息发送给交换器时，需要一个RoutingKey，当BindingKey 和RoutingKey 相匹配时，消息会被路由到对应的队列中。  
在绑定多个队列到同一个交换器的时候，这些绑定允许使用相同的BindingKey。  
BindingKey 并不是在所有的情况下都生效，它依赖于交换器类型，比如fanout 类型的交换器就会无视BindingKey，而是将消息路由到所有绑定到该交换器的队列中。

### RabbitMQ的运转过程
![rabbitmq_running_process](../images/mq/2024-1-26_RabbitMQ运转过程.png)
- **生产者发送消息的过程**  
（1）生产者连接到RabbitMQ Broker，建立一个连接（Connection），开启一个信道（Channel）  
（2）生产者声明一个交换器，并设置相关属性，比如交换机类型、是否持久化等  
（3）生产者声明一个队列并设置相关属性，比如是否排他、是否持久化、是否自动删除等  
（4）生产者通过路由键将交换器和队列绑定起来  
（5）生产者发送消息至RabbitMQ Broker，其中包含路由键、交换器等信息  
（6）相应的交换器根据接收到的路由键查找相匹配的队列  
（7）如果找到，则将从生产者发送过来的消息存入相应的队列中  
（8）如果没有找到，则根据生产者配置的属性选择丢弃还是回退给生产者  
（9）关闭信道  
（10）关闭连接  
- **消费者接受消息的过程**  
（1）消费者连接到RabbitMQ Broker，建立一个连接（Connection），开启一个信道（Channel）  
（2）消费者向RabbitMQ Broker 请求消费相应队列中的消息，可能会设置相应的回调函数，以及做一些准备工作  
（3）等待RabbitMQ Broker 回应并投递相应队列中的消息，消费者接收消息  
（4）消费者确认（ack）接收到的消息  
（5）RabbitMQ 从队列中删除相应已经被确认的消息  
（6）关闭信道  
（7）关闭连接  

### 连接RabbitMQ

#### Connection
Connection 是指客户端与RabbitMQ服务器之间建立的TCP连接。它是一个底层的网络连接，用于承载AMQP的消息交换。  
RabbitMQ 采用类似NIO的做法，选择TCP 连接复用，不仅可以减少性能开销，同时也便于管理。

#### Channel
Channel 是一种轻量级的通信通道，它是建立在 Connection 之上的逻辑单元。每个 Connection 可以拥有多个 Channel，并且所有通信都在 Channel 上进行，而不是直接在 Connection 上。  
Channel 提供了一种在单个TCP连接上执行多路复用的方法，这意味着通过一个TCP连接，可以处理多个并发的AMQP 会话，而不必为每个会话维护独立的连接。  
使用 Channel 而不是直接在 Connection 上操作，可以减少TCP连接的创建和销毁；但是Channel 实例不能在线程间共享，在网络上出现错误的通信帧交错，同时也会影响发送方确认（publisher confirm）机制的运行，所以多线程间共享Channel 实例是非线程安全的。

#### 使用交换器和队列
生产者和消费者都可以声明一个交换器或者队列。  
如果尝试声明一个已经存在的交换器或者队列，只要声明的参数完全匹配现存的交换器或者队列，RabbitMQ就可以什么都不做，并成功返回。如果声明的参数不匹配则会抛出异常。  

#### 何时创建交换器和队列
按照RabbitMQ 官方建议，生产者和消费者都应该尝试创建（这里指声明操作）队列。这是一个很好的建议，但不适用于所有的情况。如果业务本身在架构设计之初已经充分地预估了队列的使用情况，完全可以在业务程序上线之前在服务器上创建好（比如通过页面管理、RabbitMQ命令或者更好的是从配置中心下发），这样业务程序也可以免去声明的过程，直接使用即可。预先创建好资源还有一个好处是，可以确保交换器和队列之间正确地绑定匹配。

- ##### exchangeDeclare
这个方法的返回值是Exchange.DeclareOK，用来标识成功声明了一个交换器。  
```
Exchange.DeclareOk exchangeDeclare(String exchange, String type, boolean durable, boolean autoDelete, 
                                boolean internal, Map<String, Object> arguments) throws IOException;
```
> exchange：交换器的名称。  
  type：交换器的类型，常见的如fanout、direct、topic。  
  durable：设置是否持久化。durable 设置为true 表示持久化，反之是非持久化。持久化可以将交换器存盘，在服务器重启的时候不会丢失相关信息。  
  autoDelete：设置是否自动删除。autoDelete 设置为true 则表示自动删除。自动删除的前提是至少有一个队列或者交换器与这个交换器绑定，之后所有与这个交换器绑定的队列或者交换器都与此解绑。注意不能错误地把这个参数理解为：“当与此交换器连接的客户端都断开时，RabbitMQ 会自动删除本交换器”。  
  internal：设置是否是内置的。如果设置为true，则表示是内置的交换器，客户端程序无法直接发送消息到这个交换器中，只能通过交换器路由到交换器这种方式。  
  argument：其他一些结构化参数，比如alternate-exchange

exchangeDeclare 的其他重载方法如下：
```
Exchange.DeclareOk exchangeDeclare(String exchange, String type) throws IOException;

Exchange.DeclareOk exchangeDeclare(String exchange, String type, boolean durable) throws IOException;

Exchange.DeclareOk exchangeDeclare(String exchange, String type, boolean durable, boolean autoDelete, Map<String, Object> arguments) throws IOException;
```
<span style="color: red;">tips</span>：第二个参数String type 可以换成BuiltInExchangeType type

**其他创建交换器的方法**
```
// 客户端声明了一个交换器之后，不需要服务器任何返回值（如果服务器还并未完成交换器的创建，紧接着使用会发生异常，所以不建议使用该方法）
void exchangeDeclareNoWait(String exchange, String type, boolean durable, boolean autoDelete,
                            boolean internal, Map<String, Object> arguments) throws IOException;

// 检测相应的交换器是否存在。如果存在则正常返回；如果不存在则抛出异常：404 channel exception，同时Channel 也会被关闭
Exchange.DeclareOk exchangeDeclarePassive(String name) throws IOException;
```

**删除交换器的方法**
```
Exchange.DeleteOk exchangeDelete(String exchange) throws IOException;

void exchangeDeleteNoWait(String exchange, boolean ifUnused) throws IOException;

Exchange.DeleteOk exchangeDelete(String exchange, boolean ifUnused) throws IOException;
```
> exchange：表示交换器的名称  
ifUnused：用来设置是否在交换器没有被使用的情况下删除。如果isUnused 设置为true，则只有在此交换器没有被使用的情况下才会被删除；如果设置false，则无论如何这个交换器都要被删除

- ##### queueDeclare
声明一个队列。

```
// 创建一个由RabbitMQ 命名的（类似这种amq.gen-LhQz1gv3GhDOv8PIDabOXA 名称，这种队列也称之为匿名队列）、排他的、自动删除的、非持久化的队列
Queue.DeclareOk queueDeclare() throws IOException;

Queue.DeclareOk queueDeclare(String queue, boolean durable, boolean exclusive, boolean autoDelete, Map<String, Object> arguments) throws IOException;
```
> queue：队列的名称。  
durable：设置是否持久化。为true 则设置队列为持久化。持久化的队列会存盘，在服务器重启的时候可以保证不丢失相关信息。  
exclusive：设置是否排他。为true 则设置队列为排他的。如果一个队列被声明为排他队列，该队列仅对首次声明它的连接可见，并在连接断开时自动删除。这里需要注意三点：排他队列是基于连接（Connection）可见的，同一个连接的不同信道（Channel）是可以同时访问同一连接创建的排他队列；“首次”是指如果一个连接已经声明了一个排他队列，其他连接是不允许建立同名的排他队列的，这个与普通队列不同；即使该队列是持久化的，一旦连接关闭或者客户端退出，该排他队列都会被自动删除，这种队列适用于一个客户端同时发送和读取消息的应用场景。  
autoDelete：设置是否自动删除。为true 则设置队列为自动删除。自动删除的前提是：至少有一个消费者连接到这个队列，之后所有与这个队列连接的消费者都断开时，才会自动删除。不能把这个参数错误地理解为：“当连接到此队列的所有客户端断开时，这个队列自动删除”，因为生产者客户端创建这个队列，或者没有消费者客户端与这个队列连接时，都不会自动删除这个队列。  
arguments：设置队列的其他一些参数，如x-message-ttl、x-expires、x-max-length、x-max-length-bytes、x-dead-letter-exchange、x-deadletter-routing-key、x-max-priority 等。

<span style="color: red;">tips：生产者和消费者都能够使用queueDeclare 来声明一个队列，但是如果消费者在同一个信道上订阅了另一个队列，就无法再声明队列了。必须先取消订阅，然后将信道置为“传输”模式，之后才能声明队列。</span>

**其他创建队列的方法**
```
// 客户端声明了一个队列之后，不需要服务器任何返回值（如果服务器还并未完成队列的创建，紧接着使用会发生异常，所以不建议使用该方法）
void queueDeclareNoWait(String queue, boolean durable, boolean exclusive,
                        boolean autoDelete, Map<String, Object> arguments) throws IOException;

// 检测相应的队列是否存在。如果存在则正常返回，如果不存在则抛出异常：404 channel exception，同时Channel 也会被关闭
Queue.DeclareOk queueDeclarePassive(String queue) throws IOException;
```

**删除队列的方法**
```
Queue.DeleteOk queueDelete(String queue) throws IOException;

Queue.DeleteOk queueDelete(String queue, boolean ifUnused, boolean ifEmpty) throws IOException;

void queueDeleteNoWait(String queue, boolean ifUnused, boolean ifEmpty) throws IOException;
```
> queue：表示队列的名称  
ifUnused：用来设置是否在队列没有被使用的情况下删除。如果isUnused 设置为true，则只有在此队列没有被使用的情况下才会被删除；如果设置false，则无论如何这个队列都要被删除
ifEmpty：设置为true 表示在队列为空（队列里面没有任何消息堆积）的情况下才能够删除

```
// 清空队列中的内容，而不删除队列本身
Queue.PurgeOk queuePurge(String queue) throws IOException;
```

- ##### queueBind
将队列和交换器绑定。  

```
Queue.BindOk queueBind(String queue, String exchange, String routingKey) throws IOException;

Queue.BindOk queueBind(String queue, String exchange, String routingKey, Map<String, Object> arguments) throws IOException;

void queueBindNoWait(String queue, String exchange, String routingKey, Map<String, Object> arguments) throws IOException;
```
> queue：队列名称  
exchange：交换器的名称  
routingKey：用来绑定队列和交换器的路由键  
argument：定义绑定的一些参数  

**队列和交换器解绑**
```
Queue.UnbindOk queueUnbind(String queue, String exchange, String routingKey) throws IOException;

Queue.UnbindOk queueUnbind(String queue, String exchange, String routingKey, Map<String, Object> arguments) throws IOException;
```

- ##### exchangeBind
将交换器与交换器绑定。

```
Exchange.BindOk exchangeBind(String destination, String source, String routingKey) throws IOException;

Exchange.BindOk exchangeBind(String destination, String source, String routingKey, Map<String, Object> arguments) throws IOException;

void exchangeBindNoWait(String destination, String source, String routingKey, Map<String, Object> arguments) throws IOException;
```
> destination：目的交换器  
source：源交换器，消息从source 交换器转发到destination 交换器  
routingKey：用来绑定两个交换器的路由键  
argument：定义绑定的一些参数  

#### 关闭连接
```
// 显式地关闭Channel 是个好习惯，但这不是必须的，在Connection 关闭的时候，Channel 也会自动关闭
channel.close();
connection.close();
```

Connection 和Channel 的生命周期：
> Open：开启状态，代表当前对象可以使用。  
Closing：正在关闭状态。当前对象被显式地通知调用关闭方法（shutdown），这样就产生了一个关闭请求让其内部对象进行相应的操作，并等待这些关闭操作的完成。  
Closed：已经关闭状态。当前对象已经接收到所有的内部对象已完成关闭动作的通知，并且其也关闭了自身。

在Connection 和Channel 中与关闭相关的方法：
```
// 当Connection 或者Channel 的状态转变为Closed 的时候会调用ShutdownListener。
// 如果将一个ShutdownListener 注册到一个已经处于Closed状态的对象（这里特指Connection 和Channel 对象）时，会立刻调用ShutdownListener
addShutdownListener(ShutdownListener listener)  
removeShutdownListener (ShutdownListner listener)

// 获取对象关闭的原因（ShutdownSignalException）
getCloseReason

// 检测对象当前是否处于开启状态
isOpen

// 显式地通知当前对象执行关闭操作
close(int closeCode, String closeMessage)
```

当触发ShutdownListener 的时候，可以获取到关闭的原因（ShutdownSignalException）
```
connection.addShutdownListener(new ShutdownListener() {
    public void shutdownCompleted(ShutdownSignalException cause) {
        // isHardError 方法可以知道是Connection 的还是Channel 的错误
        if (cause.isHardError()) {
            Connection conn = (Connection)cause.getReference();
            if (!cause.isInitiatedByApplication()) {
                // getReason 方法可以获取异常相关的信息
                Method reason = cause.getReason();
                ...
            }
        ...
        } else {
            Channel ch = (Channel)cause.getReference();
            ...
        }
    }
});
```

### 发送消息
发送一个消息，可以使用Channel 类的basicPublish 方法  

```
void basicPublish(String exchange, String routingKey, BasicProperties props, byte[] body) throws IOException;

void basicPublish(String exchange, String routingKey, boolean mandatory, BasicProperties props, byte[] body) throws IOException;

void basicPublish(String exchange, String routingKey, boolean mandatory, boolean immediate, BasicProperties props, byte[] body) throws IOException;
```

> exchange：交换器的名称，指明消息需要发送到哪个交换器中。如果设置为空字符串，则消息会被发送到RabbitMQ 默认的交换器中。  
routingKey：路由键，交换器根据路由键将消息存储到相应的队列之中。  
props：消息的基本属性集，其包含14 个属性成员，分别有contentType、contentEncoding、headers(Map<String,Object>)、deliveryMode、priority、correlationId、replyTo、expiration、messageId、timestamp、type、userId、appId、clusterId。  
byte[] body：消息体（payload），真正需要发送的消息。  
mandatory：当mandatory 参数设为true 时，交换器无法根据自身的类型和路由键找到一个符合条件的队列，那么RabbitMQ 会调用Basic.Return 命令将消息返回给生产者。当mandatory 参数设置为false 时，出现上述情形，则消息直接被丢弃。  
immediate：当immediate 参数设为true 时，如果交换器在将消息路由到队列时发现队列上并不存在任何消费者，那么这条消息将不会存入队列中。当与路由键匹配的所有队列都没有消费者时，该消息会通过Basic.Return 返回至生产者。<span style="color: red;">RabbitMQ 3.0 版本开始去掉了对immediate 参数的支持，对此RabbitMQ 官方解释是：immediate 参数会影响镜像队列的性能，增加了代码复杂性，建议采用TTL 和DLX 的方法替代</span>  

##### 当mandatory 为 true 时，添加ReturnListener 监听器来接收返回的消息
```
    channel.basicPublish(EXCHANGE_NAME, "", true, MessageProperties.PERSISTENT_TEXT_PLAIN, "测试消息".getBytes());
    channel.addReturnListener(new ReturnListener() {
        public void handleReturn(int replyCode, String replyText, String exchange, String routingKey, 
                                AMQP.BasicProperties basicProperties, byte[] body) throws IOException {
            String message = new String(body);
            System.out.println("Basic.Return 返回的结果是："+message);
        }
    });
```

##### BasicProperties props 示例
```
// MessageProperties.PERSISTENT_TEXT_PLAIN 相当于设置投递模式（delivery mode）为2，即消息会被持久化，
// 设置优先级（priority）为1，设置content-type为“text/plain”
channel.basicPublish(exchangeName, routingKey, mandatory, MessageProperties.PERSISTENT_TEXT_PLAIN, messageBodyBytes);

// 自己设定同上的属性
channel.basicPublish(exchangeName, routingKey,
    new AMQP.BasicProperties.Builder().contentType("text/plain").deliveryMode(2).priority(1).userId("hidden").build(), messageBodyBytes);

// 发送一条带有headers、过期时间（expiration）的消息
Map<String, Object> headers = new HashMap<String, Object>();
headers.put("localtion", "here");
headers.put("time","today");
channel.basicPublish(exchangeName, routingKey, new AMQP.BasicProperties.Builder().headers(headers).expiration("60000").build(), messageBodyBytes);
```

### 消费消息
RabbitMQ 的**消费模式**分两种：推（Push）模式和拉（Pull）模式。推模式采用Basic.Consume 进行消费，而拉模式则是调用Basic.Get 进行消费。

#### 推模式
接收消息一般通过实现Consumer 接口或者继承DefaultConsumer 类来实现。  

Channel 类中basicConsume 方法有如下几种形式：
```
String basicConsume(String queue, Consumer callback) throws IOException;

String basicConsume(String queue, boolean autoAck, Consumer callback) throws IOException;

String basicConsume(String queue, boolean autoAck, Map<String, Object> arguments, Consumer callback) throws IOException;

String basicConsume(String queue, boolean autoAck, String consumerTag, Consumer callback) throws IOException;

String basicConsume(String queue, boolean autoAck, String consumerTag, boolean noLocal, boolean exclusive,
                    Map<String, Object> arguments, Consumer callback) throws IOException;
```
> queue：队列的名称  
autoAck：设置是否自动确认。建议设成false，即不自动确认  
consumerTag：消费者标签，用来区分多个消费者  
noLocal：设置为true 则表示不能将同一个Connection 中生产者发送的消息传送给这个Connection 中的消费者  
exclusive：设置是否排他
arguments：设置消费者的其他参数
callback：设置消费者的回调函数。用来处理RabbitMQ 推送过来的消息，比如DefaultConsumer，使用时需要客户端重写（override）其中的方法

callback 中被重写的方法：
```
// 处理推送的消息
void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException;

// 会在其他方法之前调用，返回消费者标签
void handleConsumeOk(String consumerTag);

// 消费端可以在显式地取消订阅的时候调用
void handleCancelOk(String consumerTag);

// 消费端可以在隐式地取消订阅的时候调用
void handleCancel(String consumerTag) throws IOException;

// 当Channel或者Connection关闭的时候会调用
void handleShutdownSignal(String consumerTag, ShutdownSignalException sig);

// 请求重新消费未确认的消息时调用
void handleRecoverOk(String consumerTag);
```

通过channel.basicCancel 方法可以显式地取消一个消费者的订阅
```
// 相当于首先触发handleConsumerOk 方法，之后触发handleDelivery方法，最后触发handleCancelOk 方法
channel.basicCancel(consumerTag);
```
和生产者一样，消费者客户端同样需要考虑线程安全的问题。消费者客户端的这些callback 会被分配到与Channel 不同的线程池上，这意味着消费者客户端可以安全地调用这些阻塞方法，比如channel.queueDeclare、channel.basicCancel 等。  
每个Channel 都拥有自己独立的线程。最常用的做法是一个Channel 对应一个消费者,也就是意味着消费者彼此之间没有任何关联。当然也可以在一个Channel 中维持多个消费者,但是要注意一个问题，如果Channel 中的一个消费者一直在运行，那么其他消费者的callback 会被“耽搁”。

##### 弃用QueueingConsumer
RabbitMQ 4.x 版本开始将QueueingConsumer 标记为@Deprecated，因为它有如下缺陷：
> 1. QueueingConsumer 内部使用LinkedBlockingQueue 来缓存这些消息，消息太多会导致堆内存变大直至内存溢出，可以使用Basic.Qos 限制消费者所保持未确认消息的数量来解决
2. QueueingConsumer 会拖累同一个Connection 下的所有信道，使其性能降低
3. 同步递归调用QueueingConsumer 会产生死锁
4. RabbitMQ 的自动连接恢复机制（automatic connection recovery）不支持Queueing Consumer 的这种形式
5. QueueingConsumer 不是事件驱动的

#### 拉模式
通过channel.basicGet 方法可以单条地获取消息，其返回值是GetRespone  

```
GetResponse basicGet(String queue, boolean autoAck) throws IOException;
```
> queue：队列的名称  
autoAck：设置是否自动确认。建议设成false，即不自动确认  

<span style="color: red;font-weight: bold;">Tips</span>：Basic.Consume 将信道（Channel）置为接收模式，直到取消队列的订阅为止。在接收模式期间，RabbitMQ 会不断地推送消息给消费者，当然推送消息的个数还是会受到Basic.Qos 的限制。如果只想从队列获得单条消息而不是持续订阅，建议还是使用Basic.Get 进行消费（Basic.Qos 的限制对拉模式消费无效）。但是不能将Basic.Get 放在一个循环里来代替Basic.Consume，这样做会严重影响RabbitMQ的性能。如果要实现高吞吐量，消费者理应使用Basic.Consume 方法。  

### 备份交换器
Alternate Exchange，简称AE  
生产者在发送消息时如果不设置mandatory 参数，那么消息在未被路由的情况下将会丢失；如果设置了mandatory 参数，那么需要添加ReturnListener 的编程逻辑，生产者的代码将变得复杂。  
如果既不想复杂化生产者的编程逻辑，又不想消息丢失，那么可以使用备份交换器，这样可以将未被路由的消息存储在RabbitMQ 中，再在需要的时候去处理这些消息。

<span style="color: red;font-weight: bold;">Tips：</span>
> 如果设置的备份交换器不存在，客户端和RabbitMQ 服务端都不会有异常出现，此时消息会丢失。  
如果备份交换器没有绑定任何队列，客户端和RabbitMQ 服务端都不会有异常出现，此时消息会丢失。  
如果备份交换器没有任何匹配的队列，客户端和RabbitMQ 服务端都不会有异常出现，此时消息会丢失。  
如果备份交换器和mandatory 参数一起使用，那么mandatory 参数无效。  

```
// 一、通过在声明交换器（ 调用channel.exchangeDeclare 方法）的时候添加alternate-exchange
Map<String, Object> args = new HashMap<String, Object>();
args.put("alternate-exchange", "myAe");
channel.exchangeDeclare("normalExchange", "direct", true, false, args);
channel.exchangeDeclare("myAe", "fanout", true, false, null);
channel.queueDeclare("normalQueue", true, false, false, null);
channel.queueBind("normalQueue", "normalExchange", "normalKey");
channel.queueDeclare("unroutedQueue", true, false, false, null);
channel.queueBind("unroutedQueue", "myAe", "");
```
> 上面的代码中声明了两个交换器normalExchange 和myAe，分别绑定了normalQueue 和unroutedQueue 这两个队列，同时将myAe 设置为normalExchange 的备份交换器。  
注意myAe 的交换器类型为fanout（建议设置为fanout 类型，其它类型要进行路由键匹配，否则消息丢失）  
如果此时发送一条消息到normalExchange 上，当路由键等于“normalKey”的时候，消息能正确路由到normalQueue 这个队列中。  、
如果路由键设为其他值，比如“errorKey”，即消息不能被正确地路由到与normalExchange 绑定的任何队列上，此时就会发送给myAe，进而发送到unroutedQueue 这个队列。

```
// 二、采用策略（Policy）方式来设置备份交换器
rabbitmqctl set_policy AE "^normalExchange$" ‘{"alternate-exchange": "myAE"}’
```

<span style="color: red;font-weight: bold;">Tips：</span>调用channel.exchangeDeclar 声明备份交换器的优先级更高，会覆盖掉Policy 的设置。


### 过期时间（TTL）
Time to Live 的简称，RabbitMQ 可以对消息和队列设置TTL。  

#### 设置消息的TTL
1. 通过队列属性设置，队列中所有消息都有相同的过期时间
2. 对消息本身进行单独设置，每条消息的TTL 不同

如果两种方法一起使用，则消息的TTL 以两者之间较小的那个数值为准。  

消息在队列中的生存时间一旦超过设置的TTL 值时，就会变成“死信”（Dead Message），消费者将无法再收到该消息（这点不是绝对的）。  

如果不设置TTL，则表示此消息不会过期。  

如果将TTL 设置为0，则表示除非此时可以直接将消息投递到消费者，否则该消息会被立即丢弃。  

“通过队列属性设置”队列TTL 属性的方法，一旦消息过期，就会从队列中抹去，而在“对消息本身进行单独设置”的情况，即使消息过期，也不会马上从队列中抹去，因为每条消息是否过期是在即将投递到消费者之前判定的。

1. 通过队列属性设置消息TTL 的代码：

```
// 1.在channel.queueDeclare 方法中加入x-message-ttl 参数实现的，这个参数的单位是毫秒
Map<String, Object> argss = new HashMap<String, Object>();
argss.put("x-message-ttl",6000);
channel.queueDeclare(queueName, durable, exclusive, autoDelete, argss);

// 2.通过Policy 的方式来设置
rabbitmqctl set_policy TTL ".*" '{"message-ttl":60000}' --apply-to queues

// 3.通过调用HTTP API 接口设置
$ curl -i -u root:root -H "content-type:application/json"-X PUT -d
'{"auto_delete":false,"durable":true,"arguments":{"x-message-ttl":60000}}'
http://localhost:15672/api/queues/{vhost}/{queuename}

```

2. 对消息本身进行单独设置TTL 的代码：

```
// 1.通过BasicProperties.Builder 设置
AMQP.BasicProperties.Builder builder = new AMQP.BasicProperties.Builder();
builder.deliveryMode(2);//持久化消息
builder.expiration("60000");//设置TTL=60000ms
AMQP.BasicProperties properties = builder.build();
channel.basicPublish(exchangeName,routingKey,mandatory,properties,"ttlTestMessage".getBytes());

// 2.通过BasicProperties 设置
AMQP.BasicProperties properties = new AMQP.BasicProperties();
Properties.setDeliveryMode(2);
properties.setExpiration("60000");
channel.basicPublish(exchangeName,routingKey,mandatory,properties,"ttlTestMessage".getBytes());

// 3.通过HTTP API 接口设置
$ curl -i -u root:root -H "content-type:application/json" -X POST -d
'{"properties":{"expiration":"60000"},"routing_key":"routingkey","payload":"ttlTestMessage","payload_encoding":"string"}'
http://localhost:15672/api/exchanges/{vhost}/{exchangename}/publish
```

#### 设置队列的TTL

通过channel.queueDeclare 方法中的x-expires 参数可以控制队列被自动删除前处于未使用状态的时间。  
未使用的意思是队列上没有任何的消费者，队列也没有被重新声明，并且在过期时间段内也未调用过Basic.Get 命令。

设置队列里的TTL 可以应用于类似RPC 方式的回复队列，在RPC 中，许多队列会被创建出来，但是却是未被使用的。  
RabbitMQ 会确保在过期时间到达后将队列删除，但是不保障删除的动作有多及时。

在RabbitMQ 重启后，持久化的队列的过期时间会被重新计算。

```
// 用于表示过期时间的x-expires 参数以毫秒为单位，并且服从和x-message-ttl 一样的约束条件，不过不能设置为0
Map<String, Object> args = new HashMap<String, Object>();
args.put("x-expires", 1800000);
channel.queueDeclare("myqueue", false, false, false, args);
```

### 死信队列
DLX，全称为Dead-Letter-Exchange，可以称之为死信交换器。  
当消息在一个队列中变成死信（dead message）之后，它能被重新被发送到另一个交换器中，这个交换器就是DLX，绑定DLX 的队列就称之为死信队列。

消息变成死信的原因：
> 消息被拒绝（Basic.Reject/Basic.Nack），并且设置requeue 参数为false；  
  消息过期；  
  队列达到最大长度。  

为某个队列设置DLX （实际上就是设置某个队列的属性），当这个队列中存在死信时，RabbitMQ 就会自动地将这个消息重新发布到设置的DLX 上去，进而被路由到DLX 的死信队列。

```
// 在channel.queueDeclare 方法中设置x-dead-letter-exchange 参数来为这个队列添加DLX
channel.exchangeDeclare("dlx_exchange", "direct");//创建DLX: dlx_exchange
Map<String, Object> args = new HashMap<String, Object>();
args.put("x-dead-letter-exchange", " dlx_exchange ");
//为队列myqueue 添加DLX
channel.queueDeclare("myqueue", false, false, false, args);

// 也可以为这个DLX 指定路由键，如果没有特殊指定，则使用原队列的路由键
args.put("x-dead-letter-routing-key", "dlx-routing-key");

// 通过Policy 的方式添加DLX
rabbitmqctl set_policy DLX ".*" '{"dead-letter-exchange":" dlx_exchange "}' --apply-to queues
```

代码示例：
```
channel.exchangeDeclare("exchange.dlx", "direct", true);
channel.exchangeDeclare("exchange.normal", "fanout", true);
Map<String, Object> args = new HashMap<String, Object>();
args.put("x-message-ttl", 10000);
args.put("x-dead-letter-exchange", "exchange.dlx");
args.put("x-dead-letter-routing-key", "routingkey");
channel.queueDeclare("queue.normal", true, false, false, args);
channel.queueBind("queue.normal", "exchange.normal", "");
channel.queueDeclare("queue.dlx", true, false, false, null);
channel.queueBind("queue.dlx", "exchange.dlx", "routingkey");
channel.basicPublish("exchange.normal", "rk", MessageProperties.PERSISTENT_TEXT_PLAIN, "dlx".getBytes());
```
> 这段代码创建了两个交换器exchange.normal 和exchange.dlx，分别绑定两个队列queue.normal 和queue.dlx，再将exchange.dlx 设置到queue.normal  
生产者首先发送一条携带路由键为“rk”的消息，然后经过交换器exchange.normal 顺利地存储到队列queue.normal 中。由于队列queue.normal 设置了过期时间为10s，在这10s 内没有消费者消费这条消息，那么判定这条消息为过期。由于设置了DLX，过期之时，消息被丢给交换器exchange.dlx 中，这时找到与exchange.dlx 匹配的队列queue.dlx，最后消息被存储在queue.dlx 这个死信队列中。


### 延迟队列
延迟队列存储的对象是对应的延迟消息，所谓“延迟消息”是指当消息被发送以后，并不想让消费者立刻拿到消息，而是等待特定时间后，消费者才能拿到这个消息进行消费。

延迟队列的使用场景：
> 超时未支付的订单，需要关闭  
智能设备定时开关

在AMQP 协议中，或者RabbitMQ 本身没有直接支持延迟队列的功能，但是可以通过DLX 和TTL 模拟出延迟队列的功能。

死信队列可以被视为延迟队列，假设一个应用中将每条消息都设置为10 秒的延迟，生产者通过交换器将发送的消息存储到队列中。消费者订阅的并非是这个队列，而是给这个队列配置的死信队列。当消息这个队列中过期之后被存入死信队列中，消费者就恰巧消费到了延迟10 秒的这条消息。

![延迟队列](../images/mq/2024-1-29_RabbitMQ延迟队列.png ':size=50%')  
> 根据应用需求的不同，生产者在发送消息的时候通过设置不同的路由键，以此将消息发送到与交换器绑定的不同的队列中。这里队列分别设置了过期时间为5 秒、10 秒、30 秒、1 分钟，同时也分别配置了DLX 和相应的死信队列。当相应的消息过期时，就会转存到相应的死信队列（即延迟队列）中，这样消费者根据业务自身的情况，分别选择不同延迟等级的延迟队列进行消费。


### 优先级队列
具有高优先级的队列具有高的优先权，优先级高的消息具备优先被消费的特权。

```
// 1.配置一个队列的最大优先级可以通过设置队列的x-max-priority 参数来实现
Map<String, Object> args = new HashMap<String, Object>();
args.put("x-max-priority", 10);
channel.queueDeclare("queue.priority", true, false, false, args);

// 2.发送时在消息中设置消息当前的优先级
AMQP.BasicProperties.Builder builder = new AMQP.BasicProperties.Builder();
builder.priority(5);
AMQP.BasicProperties properties = builder.build();
channel.basicPublish("exchange_priority","rk_priority",properties,("messages").getBytes());
```
上面的代码中设置消息的优先级为5。默认最低为0，最高为队列设置的最大优先级。  

如果在消费者的消费速度大于生产者的速度且Broker 中没有消息堆积的情况下，对发送的消息设置优先级也就没有什么实际意义。

### RPC实现
Remote Procedure Call 的简称，即远程过程调用。  
客户端发送请求消息，服务端回复响应的消息。为了接收响应的消息，我们需要在请求消息中发送一个回调队列（可以使用默认的队列）。

![rpc_process](../images/mq/2024-1-29_RPC流程.png)

上图的处理流程：
> 1. 当客户端启动时，创建一个匿名的回调队列（名称由RabbitMQ 自动创建，图中的回调队列为amq.gen-LhQz1gv3GhDOv8PIDabOXA）
2. 客户端为RPC 请求设置2 个属性：replyTo 用来告知RPC 服务端回复请求时的目的队列，即回调队列；correlationId 用来标记一个请求。
3. 请求被发送到rpc_queue 队列中。
4. RPC 服务端监听rpc_queue 队列中的请求，当请求到来时，服务端会处理并且把带有结果的消息发送给客户端。接收的队列就是replyTo 设定的回调队列。
5. 客户端监听回调队列，当有消息时，检查correlationId 属性，如果与请求匹配，那就是结果了。

发送消息时，参数BasicProperties 类中包含14 个属性，这里用到两个：  
1. replyTo：通常用来设置一个回调队列。
2. correlationId：用来关联请求（request）和其调用RPC 之后的回复（response）。

为每个RPC请求单独创建一个回调队列。效率会非常低。解决方案是可以为每个客户端创建一个单一的回调队列。  
对于回调队列而言，在其接收到一条回复的消息之后，它并不知道这条消息应该和哪一个请求匹配。这里就用到correlationId 这个属性了，我们应该为每一个请求设置一个唯一的correlationId。之后在回调队列接收到回复的消息时，可以根据这个属性匹配到相应的请求。如果回调队列接收到一条未知correlationId 的回复消息，可以简单地将其丢弃。

RabbitMQ 官方用一个例子来做说明，RPC 客户端通过RPC 调用服务端的方法以便得到相应的斐波那契值：
```
// 服务端关键代码
public class RPCServer {
    private static final String RPC_QUEUE_NAME = "rpc_queue";

    public static void main(String args[]) throws Exception {
        //省略了创建Connection 和Channel 的过程
        channel.queueDeclare(RPC_QUEUE_NAME, false, false, false, null);
        channel.basicQos(1);
        System.out.println(" [x] Awaiting RPC requests");
        Consumer consumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, 
            AMQP.BasicProperties properties, byte[] body) throws IOException {
                // 将客户端发送的correlationId 装载到回复的消息中
                AMQP.BasicProperties replyProps = new AMQP.BasicProperties.Builder()
                .correlationId(properties.getCorrelationId()).build();
                String response = "";
                try {
                    String message = new String(body, "UTF-8");
                    int n = Integer.parseInt(message);
                    System.out.println(" [.] fib(" + message + ")");
                    response += fib(n);
                } catch (RuntimeException e) {
                    System.out.println(" [.] " + e.toString());
                } finally {
                    channel.basicPublish("", properties.getReplyTo(),
                    replyProps, response.getBytes("UTF-8"));
                    channel.basicAck(envelope.getDeliveryTag(), false);
                }
            }
        };
        channel.basicConsume(RPC_QUEUE_NAME, false, consumer);
    }

    private static int fib(int n){
        if (n == 0) return 0;
        if (n == 1) return 1;
        return fib(n - 1) + fib(n - 2);
    }
}

// 客户端关键代码
public class RPCClient {
    private Connection connection;
    private Channel channel;
    private String requestQueueName = "rpc_queue";
    private String replyQueueName;
    private QueueingConsumer consumer;

    public RPCClient() throws IOException, TimeoutException {
        //省略了创建Connection 和Channel 的过程
        replyQueueName = channel.queueDeclare().getQueue();
        consumer = new QueueingConsumer(channel);
        channel.basicConsume(replyQueueName, true, consumer);
    }

    public String call(String message) throws IOException,ShutdownSignalException,ConsumerCancelledException,InterruptedException {
        String response = null;
        String corrId = UUID.randomUUID().toString();
        BasicProperties props = new BasicProperties.Builder()
        .correlationId(corrId).replyTo(replyQueueName).build();
        channel.basicPublish("", requestQueueName, props, message.getBytes());
        while(true){
            QueueingConsumer.Delivery delivery = consumer.nextDelivery();
            if(delivery.getProperties().getCorrelationId().equals(corrId)){
                response = new String(delivery.getBody());
                break;
            }
        }
        return response;
    }

    public void close() throws Exception{
        connection.close();
    }

    public static void main(String args[]) throws Exception{
        RPCClient fibRpc = new RPCClient();
        System.out.println(" [x] Requesting fib(30)");
        String response = fibRpc.call("30");
        System.out.println(" [.] Got '"+response+"'");
        fibRpc.close();
    }
}
```

### 消息可靠性
保障消息可靠性的方法：  
1. 消息生产者需要开启事务机制或者publisher confirm 机制，以确保消息可以可靠地传输到RabbitMQ 中
2. 消息生产者需要配合使用mandatory 参数或者备份交换器来确保消息能够从交换器路由到队列中，进而能够保存下来而不会被丢弃
3. 交换器、消息和队列都需要进行持久化处理，以确保RabbitMQ 服务器在遇到异常情况时不会造成消息丢失
4. 消费者在消费消息的同时需要将autoAck 设置为false，然后通过手动确认的方式去确认已经正确消费的消息，以避免在消费端引起消息丢失

#### 生产者确认
如果在消息到达服务器之前已经丢失，不进行特殊配置，默认情况下生产者是不知道消息有没有正确地到达服务器。  
对此，RabbitMQ 提供了两种解决方式：
1. 通过事务机制实现
2. 通过发送方确认（publisher confirm）机制实现

##### 事务机制
RabbitMQ 客户端中与事务机制相关的方法有三个：
> channel.txSelect 用于将当前的信道设置成事务模式  
> channel.txCommit 用于提交事务  
> channel.txRollback 用于事务回滚  

开启事务机制后，相比不开启事务会多四个步骤：
> 客户端发送Tx.Select，将信道置为事务模式；  
Broker 回复Tx.Select-Ok，确认已将信道置为事务模式；  
在发送完消息之后，客户端发送Tx.Commit 提交事务；  
Broker 回复Tx.Commit-Ok，确认事务提交。  

事务确实能够解决消息发送方和RabbitMQ 之间消息确认的问题，只有消息成功被RabbitMQ 接收，事务才能提交成功，否则便可在捕获异常之后进行事务回滚，与此同时可以进行消息重发。  
<span style="color: red;">但是使用事务机制会“吸干”RabbitMQ 的性能。</span>

##### 发送方确认机制
生产者将信道设置成confirm（确认）模式，一旦信道进入confirm 模式，所有在该信道上面发布的消息都会被指派一个唯一的ID（从1 开始），一旦消息被投递到所有匹配的队列之后，RabbitMQ 就会发送一个确认（Basic.Ack）给生产者（包含消息的唯一ID），这就使得生产者知晓消息已经正确到达了目的地了。如果RabbitMQ 因为自身内部错误导致消息丢失，就会发送一条nack（Basic.Nack）命令，生产者应用程序同样可以在回调方法中处理该nack 命令。

如果消息和队列是可持久化的，那么确认消息会在消息写入磁盘之后发出。

RabbitMQ 回传给生产者的确认消息中的deliveryTag 包含了确认消息的序号，此外RabbitMQ 也可以设置channel.basicAck 方法中的multiple 参数，表示到这个序号之前的所有消息都已经得到了处理。

相比事务机制，发送方确认机制最大的好处在于它是异步的。  
如果publisher confirm 模式是每发送一条消息后就调用channel.waitForConfirms 方法，之后等待服务端的确认，这实际上是一种串行同步等待的方式。同步等待的方式只比事务机制快一点。

publisher confirm 的优势在于并不一定需要同步确认。改进方式有如下两种：
1. 批量confirm ：每发送一批消息后，调用channel.waitForConfirms 方法，等待服务器的确认返回。
2. **异步confirm** ：提供一个回调方法，服务端确认了一条或者多条消息后客户端会回调这个方法进行处理。

批量confirm 虽然能提升了confirm 的效率，但是问题在于出现返回Basic.Nack 或者超时情况时，客户端需要将这一批次的消息全部重发，这会带来明显的重复消息数量。如果异常情况经常发生，反而会降低发送消息的效率。

异步confirm 在客户端Channel 接口中提供的addConfirmListener 方法可以添加ConfirmListener 这个回调接口， 这个ConfirmListener 接口包含两个方法：handleAck 和handleNack，分别用来处理RabbitMQ 回传的Basic.Ack 和Basic.Nack 。在这两个方法中都包含有一个参数deliveryTag（在publisher confirm 模式下用来标记消息的唯一有序序号）。我们需要为每一个信道维护一个“unconfirm”的消息序号集合，每发送一条消息，集合中的元素加1。每当调用ConfirmListener 中的handleAck 方法时，“unconfirm”集合中删掉相应的一条（multiple 设置为false）或者多条（multiple 设置为true）记录。这个“unconfirm”集合最好采用有序集合SortedSet 的存储结构。  
<span style="color: red;">建议使用异步confirm 的方式</span>

```
// 异步confirm 的示例代码
channel.confirmSelect();
channel.addConfirmListener(new ConfirmListener() {
    public void handleAck(long deliveryTag, boolean multiple) throws IOException {
        System.out.println("Ack, SeqNo: " + deliveryTag + ", multiple: " + multiple);
        if (multiple) {
            confirmSet.headSet(deliveryTag - 1).clear();
        } else {
            confirmSet.remove(deliveryTag);
        }
    }
    public void handleNack(long deliveryTag, boolean multiple) throws IOException {
        System.out.println("Nack, SeqNo: " + deliveryTag + ", multiple: " + multiple);
        if (multiple) {
            confirmSet.headSet(deliveryTag - 1).clear();
        } else {
            confirmSet.remove(deliveryTag);
        }
        //注意这里需要添加处理消息重发的场景
    }
});

//下面是演示一直发送消息的场景
while (true) {
    long nextSeqNo = channel.getNextPublishSeqNo();
    channel.basicPublish(ConfirmConfig.exchangeName, ConfirmConfig.routingKey,MessageProperties.PERSISTENT_TEXT_PLAIN,ConfirmConfig.msg_10B.getBytes());
    confirmSet.add(nextSeqNo);
}
```
生产者确认的4种方式QPS对比（横坐标表示测试的次数，纵坐标表示QPS）：  
![publish_confirm_qps](../images/mq/2024-1-30_生产者确认的4种方式QPS对比.jpg)

<span style="color: red;font-weight: bold;">Tips</span>：事务机制和publisher confirm 机制两者是互斥的，不能共存。  
&emsp;&emsp;生产者确认确保的是消息能够正确地发送至RabbitMQ，如果此交换器没有匹配的队列，那么消息也会丢失。所以需要配合使用mandatory 参数或者备份交换器来确保消息能够从交换器路由到队列。

#### 持久化
持久化可以提高RabbitMQ 的可靠性，以防在异常情况（重启、关闭、宕机等）下的数据丢失。  
RabbitMQ 的持久化分为三个部分：  
1. 交换器的持久化  
    在声明交换器时将durable 参数置为true，如果交换器不设置持久化，那么出现异常状况后，相关的交换器元数据会丢失，不过消息不会丢失，只是不能将消息发送到这个交换器了。
2. 队列的持久化  
    在声明队列时将durable 参数置为true，。如果队列不设置持久化，那么出现异常状况后，相关队列的元数据会丢失，此时数据也会丢失。
3. 消息的持久化  
    将消息的投递模式（BasicProperties 中的deliveryMode 属性）设置为2 即可实现消息的持久化。常用的MessageProperties.PERSISTENT_TEXT_PLAIN 实际上是封装了这个属性。

<span style="color: red;font-weight: bold;">Tips</span>：可以将所有的消息都设置为持久化，但是这样会严重影响RabbitMQ 的性能（随机）。写入磁盘的速度比写入内存的速度慢得不只一点点。对于可靠性不高的消息可以不采用持久化处理以提高整体的吞吐量。在选择是否要将消息持久化时，需要在可靠性和吐吞量之间做一个权衡。

持久化不能完全保障消息不丢失，RabbitMQ 并不会为每条消息都进行同步存盘（调用Linux内核的fsync方法）的处理，而是先存放到本地的一块缓存，等到缓存写满或内核需要重用该缓存时，再将该缓存排入输出队列，进而同步到磁盘上（这种策略可以减少磁盘IO）。此时可以引入RabbitMQ 的镜像队列机制来提升可靠性，如果主节点（master）在此特殊时间内挂掉，可以自动切换到从节点（slave），这样有效地保证了高可用性。除非整个集群都挂掉才会导致消息丢失。

#### 消费者确认与拒绝
##### 确认消息
为了保证消息从队列可靠地达到消费者，RabbitMQ 提供了消息确认机制（message acknowledgement）。  
消费者在订阅队列时，可以指定autoAck 参数，当autoAck 等于false 时，RabbitMQ 会等待消费者显式地回复确认信号后才从内存（或者磁盘）中移去消息（实质上是先打上删除标记，之后再删除）。当autoAck 等于true 时，RabbitMQ 会自动把发送出去的消息置为确认，然后从内存（或者磁盘）中删除，而不管消费者是否真正地消费到了这些消息。  
采用消息确认机制后，只要设置autoAck 参数为false，消费者就有足够的时间处理消息（任务），不用担心处理消息过程中消费者进程挂掉后消息丢失的问题，因为RabbitMQ 会一直等待持有消息直到消费者显式调用Basic.Ack 命令为止。  
当autoAck 参数置为false，对于RabbitMQ 服务端而言，队列中的消息分成了两个部分：一部分是等待投递给消费者的消息；一部分是已经投递给消费者，但是还没有收到消费者确认信号的消息。如果RabbitMQ 一直没有收到消费者的确认信号，并且消费此消息的消费者已经断开连接，则RabbitMQ 会安排该消息重新进入队列，等待投递给下一个消费者，当然也有可能还是原来的那个消费者。  
RabbitMQ 不会为未确认的消息设置过期时间，它判断此消息是否需要重新投递给消费者的唯一依据是消费该消息的消费者连接是否已经断开，这么设计的原因是RabbitMQ 允许消费者消费一条消息的时间可以很久很久。

RabbtiMQ 的Web 管理平台上可以看到当前队列中的“Ready”状态和“Unacknowledged”状态的消息数，也可以通过命令查看：
```
[root@gackey-pc ~]# rabbitmqctl list_queues name messages_ready messages_unacknowledged
```

消费者通过调用 channel.basicAck 方法，能够确认特定的消息
```
void basicAck(long deliveryTag, boolean multiple);
```
> deliveryTag：这是消息的唯一标识符，每个从队列中投递给消费者的消息都有一个递增的delivery tag，用来追踪和确认每条消息。它是一个64 位的长整型值，最大值是9223372036854775807  
multiple（可选）：如果设置为 true，则表示确认deliveryTag 编号以及之前所有未被当前消费者确认的消息。如果设置为 false，则只确认编号为deliveryTag 的这一条消息。

##### 拒绝消息

```
// 采用channel.basicReject 方法来拒绝这个消息，一次只能拒绝一条消息 
void basicReject(long deliveryTag, boolean requeue) throws IOException;

// 如果想要批量拒绝消息，可以调用channel.basicNack 方法来实现
void basicNack(long deliveryTag, boolean multiple, boolean requeue) throws IOException;
```
> deliveryTag：这是消息的唯一标识符，每个从队列中投递给消费者的消息都有一个递增的delivery tag，用来追踪和确认每条消息。  
requeue：如果设置为true，则RabbitMQ 会重新将这条消息存入队列，以便可以发送给下一个订阅的消费者；如果设置为false，则RabbitMQ立即会把消息从队列中移除，而不会把它发送给新的消费者。  
multiple：如果设置为 true，则表示拒绝deliveryTag 编号以及之前所有未被当前消费者确认的消息。如果设置为 false，则只拒绝编号为deliveryTag 的这一条消息。

<span style="color: red;font-weight: bold;">Tips</span>：将channel.basicReject 或者channel.basicNack 中的requeue 设置为false，可以启用“死信队列”的功能。死信队列可以通过检测被拒绝或者未送达的消息来追踪问题。  

##### 手动恢复消息
通常用channel.basicRecover 在消费者崩溃后恢复未处理的消息
```
// 将所有未确认的消息重新加入到原始队列，再次投递给消费者
channel.basicRecover();

// requeue 参数默认设置为true，则未被确认的消息会被重新加入到队列中，这样对于同一条消息来说，可能会被分配给与之前不同的消费者
// 如果requeue 参数设置为false，那么同一条消息会被路由到死信队列（如果配置过），再分配给与之前相同的消费者
channel.basicRecover(boolean requeue);
```

### 用RabbitTemplate时保障消息可靠性

**一.确保消息发送到 RabbitMQ 的交换器**
- 启用消息确认机制  
yml中的配置
```
spring:
    rabbitmq:
        # 消息发送交换机，开启确认回调模式
        publisher-confirm-type: correlated
        # 消息发送交换机，开启确认机制，并且返回回调
        publisher-returns: true
```
提供一个实现了 ConfirmCallback 接口的回调方法。当消息被成功发送到 RabbitMQ 时，这个方法会被调用。
```
rabbitTemplate.setConfirmCallback(new ConfirmCallback() {  
        @Override  
        public void confirm(CorrelationData correlationData, Exception exception) {  
            if (exception != null) {  
                // 处理发送消息失败的情况  
                System.out.println("发送消息失败：" + exception.getMessage());  
            } else {  
                // 处理消息发送成功的情况  
                System.out.println("消息发送成功，correlationData：" + correlationData);  
            }  
        }  
    });  
```

**二.确保消息路由到 RabbitMQ 的队列**
1. 路由保证的失败通知（mandatory+ReturnListener）  
生产者的 RabbitTemplate 里开启路由失败通知并添加失败通知的回调
```
//如果消息不能被路由到队列，那么不应该继续尝试发送消息。如果这个消息没有被路由，那么它会返回一个异常
rabbitTemplate.setMandatory(true);
//当消息不能被路由或消费者拒绝消息时，这个方法会被调用
rabbitTemplate.setReturnCallback(new ReturnCallback() {  
        @Override  
        public void returned(Exchange exchange, Message message, String cause) {  
            System.out.println("Message returned. Cause: " + cause);  
        }  
    });
```
2. 备用交换器  
在 RabbitMQ 的控制台配置 exchanges 时，Arguments 选项里点击 “Add Alternate exchange” 即可  

**三.确保消息在 RabbitMQ 正常存储**
- 交换器持久化
```  
rabbitTemplate.setPersistent(true);
```
- 队列持久化
```
// 使用RabbitTemplate的queue()方法创建持久化队列
Queue queue = new Queue("my-queue", true);
```
或者
```
Map<String, Object> args = new HashMap<String, Object>();
// 设置durable属性为true以创建持久化队列
args.put("durable", true);
// 使用declareQueue()方法创建持久化队列
channel.queueDeclare("my-queue", false, false, false, args);
```
- 消息持久化
```
rabbitTemplate.convertAndSend("exchange", "routing-key", message, new MessagePostProcessor() {  
    @Override  
    public Message postProcessMessage(Message message) throws AmqpException {  
        message.setPersistent(true); // 设置消息为持久化消息  
        return message;  
    }  
});
```

**四.消费确认**
- 手动确认  
yml中配置

```
spring:
    rabbitmq:
        host: 192.168.0.100
        port: 5672
        username: guest
        password: guest
        virtual-host: /
        connection-timeout: 2000000
        listener:
            direct:
                # 采用手动应答
                acknowledge-mode: manual
            simple:
                # 指定最大的消费者数量
                max-concurrency: 50
                # 指定最小的消费者数量
                concurrency: 1
                # 采用手动应答
                acknowledge-mode: manual
                retry:
                    # 是否开启重试机制
                    enabled: true
                    # 默认是3,是一共三次，而不是重试三次
                    max-attempts: 3
                    # 重试间隔时间，毫秒
                    initial-interval: 2000
                default-requeue-rejected: false
```
Java代码处理
```
@Component
@Slf4j
public class AckCustomer {

    //多线程环境下有问题，需要消息唯一性处理
    private AtomicInteger count = new AtomicInteger(0); 

    @RabbitListener(queuesToDeclare = @Queue(value = "ack_err", durable = "true", autoDelete = "false"))
    public void process(@Payload String data, @Headers Map<String, Object> headers, Channel channel, Message message) throws IOException {
        try {
            //处理业务逻辑
            // ......
            /*
            * 成功后确认
            * 参数1：消息标签
            * 参数2：是否批量确认，之前消费的未确认消息，全部确认，false:只确认当前消息
            */
            channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
        }catch (Exception e) {
            count.incrementAndGet();
            if(count.get() == 3){
                log.error("[ackErr]消费者出现异常，共执行{}次，开启确认失败！：{}", count, e.getMessage());
                /*
                * 异常后确认
                * 参数1：消息标签
                * 参数2：是否批量处理 true：批量，false:只确认当前消息
                * 参数3：被拒绝的消息是否回归队列 true：回归，false：丢弃
                */
                channel.basicNack(message.getMessageProperties().getDeliveryTag(), false, false);
                count.set(0);
            }
            throw e;
        }
    }
}
```

### 消息顺序性
消息的顺序性是指消费者消费到的消息和发送者发布的消息的顺序是一致的。

当出现下列情况时，消息的顺序性难以保证
> 1. 有多个生产者同时发送消息，无法确定消息到达Broker 的前后顺序
2. 生产者确认机制下，发生异常回滚，补偿发送会导致消息错序
3. 给消息设置了不同的过期时间，并配置了死信队列，那么死信队列中的消息顺序与发送时的消息顺序也会不一致
4. 为消息设置了优先级，也会导致顺序不一致
5. 消费者拒绝消息，RabbitMQ 会重发，也会导致消息错序

如果要保证消息的顺序性，需要业务中使用RabbitMQ 之后做进一步的处理，比如在消息体内添加全局有序标识（类似Sequence ID）来实现。

### 消息重复性
一般消息中间件的消息传输保障分为三个层级：
> At most once：最多一次。消息可能会丢失，但绝不会重复传输。  
> At least once：最少一次。消息绝不会丢失，但可能会重复传输。  
> Exactly once：恰好一次。每条消息肯定会被传输一次且仅传输一次。  

“最少一次”方式投递实现需要通过消息的可靠性保障。  
“最多一次”方式，生产者随意发送，消费者随意消费，不过这样很难确保消息不会丢失。  
“恰好一次”方式是RabbitMQ 目前无法保障的。受网络波动的影响，生产者可能重复发送，消费者也可能重复消费。

去重处理一般是在业务客户端实现，比如引入GUID（Globally Unique Identifier）的概念。针对GUID，如果从客户端的角度去重，那么需要引入集中式缓存，必然会增加依赖复杂度，另外缓存的大小也难以界定。建议在实际生产环境中，业务方根据自身的业务特性进行去重，比如业务消息本身具备幂等性，或者借助Redis 等其他产品进行去重处理。

### 消息分发
当RabbitMQ 队列拥有多个消费者时，队列收到的消息将以轮询（round-robin）的分发方式发送给消费者。每条消息只会发送给订阅列表里的一个消费者。  
如果现在负载加重，那么只需要创建更多的消费者来消费处理消息即可。

默认情况下，如果有n 个消费者，那么RabbitMQ 会将第m 条消息分发给第m%n（取余的方式）个消费者，RabbitMQ 不管消费者是否消费并已经确认（Basic.Ack）了消息。  
如果消费者的业务复杂度不同，会影响自身的消费能力，就会造成整体应用吞吐量的下降。

使用channel.basicQos(int prefetchCount) 方法限制信道上的消费者所能保持的最大未确认消息的数量。  
channel.basicQos 必须在channel.basicConsume 前设置才会生效。  
RabbitMQ 会保存一个消费者的列表，每发送一条消息都会为对应的消费者计数加1，如果达到了所设定的上限，那么RabbitMQ 就不会向这个消费者再发送任何消息。直到消费者确认了某条消息之后，RabbitMQ 将相应的计数减1，之后消费者可以继续接收消息，直到再次到达计数上限。这种机制可以类比于TCP/IP 中的“滑动窗口”。

<span style="color: red;font-weight: bold;">Tips</span>：Basic.Qos 的使用对于拉模式的消费方式无效。

channel.basicQos 有三种类型的重载方法：

```
void basicQos(int prefetchCount) throws IOException;
void basicQos(int prefetchCount, boolean global) throws IOException;
void basicQos(int prefetchSize, int prefetchCount, boolean global) throws IOException;
```

> prefetchCount：表示消费者所能保持的最大未确认消息的数量，设置为0 则表示没有上限  
prefetchCount：表示消费者所能接收未确认消息的总体大小的上限，单位为B，设置为0 则表示没有上限  
global：当一个信道同时消费多个队列，并设置了prefetchCount 大于0 时，这个信道需要和各个队列协调以确保发送的消息都没有超过所限定的prefetchCount 的值，这样会使RabbitMQ 的性能降低，尤其是这些队列分散在集群中的多个Broker 节点之中。

global参数 | AMQP 0-9-1 | RabbitMQ
---- | :---- | :----
false | 信道上所有的消费者都需要遵从prefetchCount 的限定值 | 信道上新的消费者需要遵从prefetchCount 的限定值
true | 当前通信链路（Connection）上所有的消费者都需要遵从prefetchCount 的限定值 | 信道上所有的消费者都需要遵从prefetchCount 的限定值

```
// 同一个信道上有多个消费者，如果设置了prefetchCount 的值，那么都会生效
Channel channel = ...;
Consumer consumer1 = ...;
Consumer consumer2 = ...;
channel.basicQos(10); // Per consumer limit
channel.basicConsume("my-queue1", false, consumer1);
channel.basicConsume("my-queue2", false, consumer2);

// 在订阅消息之前，既设置了global 为true 的限制，又设置了global 为false 的限制，RabbitMQ 会确保两者都会生效
// 每个消费者最多只能收到3 个未确认的消息，两个消费者能收到的未确认的消息个数之和的上限为5
Channel channel = ...;
Consumer consumer1 = ...;
Consumer consumer2 = ...;
channel.basicQos(3, false); // Per consumer limit
channel.basicQos(5, true); // Per channel limit
channel.basicConsume("queue1", false, consumer1);
channel.basicConsume("queue2", false, consumer2);
```

同时使用两种global模式，会增加RabbitMQ的负载，因为RabbitMQ 需要更多的资源来协调完成这些限制。  
如无特殊需要，最好只使用global 为false 的设置，这也是默认的设置。



### RabbitMQ 安装

##### 安装Erlang
1. 将erlang安装到/opt/erlang 目录下：

```
[root@hidden ~]# tar zxvf otp_src_19.3.tar.gz
[root@hidden ~]# cd otp_src_19.3
[root@hidden otp_src_19.3]# ./configure --prefix=/opt/erlang
```

2. 如果在安装的过程中出现类似“No ***** found”的提示，可根据提示信息安装相应的包，例如出现“No curses library functions found”报错：

```
[root@hidden otp_src_19.3]# yum install ncurses-devel
```

3. 安装Erlang：

```
[root@hidden otp_src_19.3]# make
[root@hidden otp_src_19.3]# make install
```

4. 修改/etc/profile 配置文件，添加下面的环境变量：

```
ERLANG_HOME=/opt/erlang
export PATH=$PATH:$ERLANG_HOME/bin
export ERLANG_HOME
```

最后执行如下命令让配置文件生效：

```
[root@hidden otp_src_19.3]# source /etc/profile
```

5. 输入erl 命令来验证Erlang 是否安装成功：

```
[root@hidden ~]# erl
Erlang/OTP 19 [erts-8.1] [source] [64-bit] [smp:4:4] [async-threads:10] [hipe] [kernel-poll:false]
Eshell V8.1 (abort with ^G)
```

##### RabbitMQ 的安装与运行

```
[root@hidden ~]# tar zvxf rabbitmq-server-generic-unix-3.6.10.tar.gz -C /opt
[root@hidden ~]# cd /opt
[root@hidden ~]# mv rabbitmq_server-3.6.10 rabbitmq
```

修改/etc/profile 文件，添加下面的环境变量：

```
export PATH=$PATH:/opt/rabbitmq/sbin
export RABBITMQ_HOME=/opt/rabbitmq
```

之后执行 source /etc/profile 命令让配置文件生效。

运行RabbitMQ，“-detached”参数是为了能够让RabbitMQ服务以守护进程的方式在后台运行：

```
rabbitmq-server –detached
```

默认情况下，访问RabbitMQ 服务的用户名和密码都是“guest”，这个账户有限制，默认只能通过本地网络（如localhost）访问，远程网络访问受限。

添加新用户，用户名为“root”，密码为“root123”：

```
[root@hidden ~]# rabbitmqctl add_user root root123
```

为root 用户设置所有权限：

```
[root@hidden ~]# rabbitmqctl set_permissions -p / root ".*" ".*" ".*"
```

设置root 用户为管理员角色：

```
[root@hidden ~]# rabbitmqctl set_user_tags root administrator
```



### RabbitMQ 管理
**rabbitmqctl** 工具是用来管理RabbitMQ 中间件的命令行工具，它通过连接各个RabbitMQ 节点来执行所有操作。  
rabbitmqctl 工具的标准语法如下（[]表示可选参数，{}表示必选参数）：

```
rabbitmqctl [-n node] [-t timeout] [-q] {command} [command options...]
```

- [-n node]：默认节点是“rabbit@hostname”，此处的hostname 是主机名称。在一个名为“node.hidden.com”的主机上，RabbitMQ 节点的名称通常是rabbit@node（除非RABBITMQ_NODENAME 参数在启
动时被设置成了非默认值）。hostname -s 命令的输出通常是“@”标志后的东西。
- [-q]：使用-q 标志来启用quiet 模式，这样可以屏蔽一些消息的输出。 默认不开启quiet 模式。
- [-t timeout]：操作超时时间（秒为单位），只适用于“list_xxx”类型的命令，默认是无穷大。

操作命令示例：

```
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

#### 多租户与权限
每一个RabbitMQ 服务器都能创建虚拟的消息服务器，我们称之为虚拟主机（virtual host），简称为vhost。  
每一个vhost 本质上是一个独立的小型RabbitMQ 服务器，拥有自己独立的队列、交换器及绑定关系等，并且它拥有自己独立的权限。vhost 就像是虚拟机与物理服务器一样，它们在各个实例间提供逻辑上的分离，为不同程序安全保密地运行数据，它既能将同一个RabbitMQ 中的众多客户区分开，又可以避免队列和交换器等命名冲突。vhost 之间是绝对隔离的，无法将vhost1 中的交换器与vhost2 中的队列进行绑定，这样既保证了安全性，又可以确保可移植性。  
vhost 是AMQP 概念的基础，客户端在连接的时候需要制定一个vhost。RabbitMQ 默认创建的vhost 为“/”  

**创建一个新的vhost**，大括号里的参数表示vhost 的名称：  
rabbitmqctl add_vhost {vhost}
```
[root@node1 ~]# rabbitmqctl add_vhost vhost1
Creating vhost "vhost1"
```

**罗列当前vhost 的相关信息**，目前vhostinfoitem 的取值有2 个：  
rabbitmqctl list_vhosts [vhostinfoitem...]

> name：表示vhost 的名称。  
tracing：表示是否使用了RabbitMQ 的trace 功能。

```
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

```
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

```
// 授予root 用户可访问虚拟主机vhost1，并在所有资源上都具备可配置、可写及可读的权限
[root@node1 ~]# rabbitmqctl set_permissions -p vhost1 root ".*" ".*" ".*"
Setting permissions for user "root" in vhost "vhost1"
// 授予root 用户可访问虚拟主机vhost2，在以“queue”开头的资源上具备可配置权限，并在所有资源上拥有可写、可读的权限
[root@node1 ~]# rabbitmqctl set_permissions -p vhost2 root "^queue.*" ".*" ".*"
Setting permissions for user "root" in vhost "vhost2"
```

**清除权限的命令**为：  
rabbitmqctl clear_permissions [-p vhost] {username}
> vhost ：设置禁止用户访问的虚拟主机的名称，默认为“/”。  
username ：表示禁止访问特定虚拟主机的用户名称。

```
[root@node1 ~]# rabbitmqctl clear_permissions -p vhost1 root
Clearing permissions for user "root" in vhost "vhost1"
```

**显示虚拟主机上的权限**：  
rabbitmqctl list_permissions [-p vhost]

**显示用户的权限**：  
rabbitmqctl list_user_permissions {username}

```
[root@node1 ~]# rabbitmqctl list_permissions -p vhost1
Listing permissions in vhost "vhost1"
root .* .* .*
[root@node1 ~]# rabbitmqctl list_user_permissions root
Listing permissions for user "root"
/ .* .* .*
vhost1 .* .* .*
```

#### 用户管理
用户是访问控制（Access Control）的基本单元，且单个用户可以跨越多个vhost 进行授权。针对一至多个vhost，用户可以被赋予不同级别的访问权限，并使用标准的用户名和密码来认证用户。

**创建用户**：  
rabbitmqctl add_user {username} {password}  
> username ：表示要创建的用户名称  
password ：表示创建用户登录的密码  

```
// 创建一个用户名为root、密码为root123 的用户
[root@node1 ~]# rabbitmqctl add_user root root123
Creating user "root"
```

**更改指定用户的密码**：  
rabbitmqctl change_password {username} {newpassword}
> username ：表示要变更密码的用户名称  
newpassword ：表示要变更的新的密码

```
// 将root 用户的密码变更为root321
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

```
// 采用密码root321 来验证root 用户
[root@node1 ~]# rabbitmqctl authenticate_user root root321
Authenticating user "root"
Success
// 采用密码root322 来验证root 用户
[root@node1 ~]# rabbitmqctl authenticate_user root root322
Authenticating user "root"
Error: failed to authenticate user "root"
```

**删除用户**：  
rabbitmqctl delete_user {username}
> username ：表示要删除的用户名称

```
[root@node1 ~]# rabbitmqctl delete_user root
Deleting user "root"
```

**罗列当前的所有用户**：  
rabbitmqctl list_users

```
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

```
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

#### Web 端管理
为了运行rabbitmqctl 工具，当前的用户需要拥有访问Erlang cookie 的权限，由于服务器可能是以guest 或者root 用户身份来运行的，因此需要获得这些文件的访问权限，这样就引申出来一些权限管理的问题。  
RabbitMQ management 插件可以提供Web 管理界面用来管理如前面所述的虚拟主机、用户等，也可以用来管理队列、交换器、绑定关系、策略、参数等，还可以用来监控RabbitMQ服务的状态及一些数据统计类信息，基本上能够涵盖所有RabbitMQ 管理的功能。

先启用RabbitMQ management 插件。RabbitMQ 提供了很多的插件，默认存放在$RABBITMQ_HOME/plugins 目录下：

```
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

```
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

#### 应用与集群管理
- rabbitmqctl stop [pid_file]  
用于停止运行RabbitMQ 的Erlang 虚拟机和RabbitMQ 服务应用。  
如果指定了pid_file，还需要等待指定进程的结束。其中pid_file 是通过调用rabbitmq-server 命令启动RabbitMQ 服务时创建的，默认情况下存放于Mnesia 目录中，可以通过RABBITMQ_PID_FILE 这个环境变量来改变存放路径。注意，如果使用rabbitmq-server –detach 这个带有-detach 后缀的命令来启动RabbitMQ 服务则不会生成pid_file 文件。

```
[root@node1 ~]# rabbitmqctl stop /opt/rabbitmq/var/lib/rabbitmq/mnesia/rabbit\@node1.pid
Stopping and halting node rabbit@node1
[root@node1 ~]# rabbitmqctl stop
Stopping and halting node rabbit@node1
```

- rabbitmqctl shutdown  
用于停止运行RabbitMQ 的Erlang 虚拟机和RabbitMQ 服务应用。  
执行这个命令会阻塞直到Erlang 虚拟机进程退出。如果RabbitMQ 没有成功关闭，则会返回一个非零值。  
这个命令和rabbitmqctl stop 不同的是，它不需要指定pid_file 而可以阻塞等待指定进程的关闭。

```
[root@node1 ~]# rabbitmqctl shutdown
Shutting down RabbitMQ node rabbit@node1 running at PID 1706
Waiting for PID 1706 to terminate
RabbitMQ node rabbit@node1 running at PID 1706 successfully shut down
```

- rabbitmqctl stop_app  
停止RabbitMQ 服务应用，但是Erlang 虚拟机还是处于运行状态。  
此命令的执行优先于其他管理操作（这些管理操作需要先停止RabbitMQ 应用），比如rabbitmqctl reset。

```
[root@node1 ~]# rabbitmqctl stop_app
Stopping rabbit application on node rabbit@node1
```

- rabbitmqctl start_app  
启动RabbitMQ 应用。  
此命令典型的用途是在执行了其他管理操作之后，重新启动之前停止的RabbitMQ 应用，比如rabbitmqctl reset。

```
[root@node1 ~]# rabbitmqctl start_app
Starting node rabbit@node1
```

- rabbitmqctl wait [pid_file]  
等待RabbitMQ 应用的启动。  
它会等到pid_file 的创建，然后等待pid_file 中所代表的进程启动。当指定的进程没有启动RabbitMQ 应用而关闭时将会返回失败。

```
[root@node1 ~]# rabbitmqctl wait /opt/rabbitmq/var/lib/rabbitmq/mnesia/rabbit\@node1.pid
Waiting for rabbit@node1
pid is 3468
```

- rabbitmqctl reset  
将RabbitMQ 节点重置还原到最初状态。包括从原来所在的集群中删除此节点，从管理数据库中删除所有的配置数据，如已配置的用户、vhost 等，以及删除所有的持久化消息。  
执行rabbitmqctl reset 命令前必须停止RabbitMQ 应用（比如先执行rabbitmqctl stop_app）。

```
[root@node1 ~]# rabbitmqctl stop_app
Stopping rabbit application on node rabbit@node1
[root@node1 ~]# rabbitmqctl reset
Resetting node rabbit@node
```

- rabbitmqctl force_reset  
强制将RabbitMQ 节点重置还原到最初状态。  
rabbitmqctl force_reset 命令不论当前管理数据库的状态和集群配置是什么，都会无条件地重置节点。它只能在数据库或集群配置已损坏的情况下使用。  
执行rabbitmqctl force_reset 命令前必须先停止RabbitMQ 应用。

```
[root@node1 ~]# rabbitmqctl stop_app
Stopping rabbit application on node rabbit@node1
[root@node1 ~]# rabbitmqctl force_reset
Forcefully resetting node rabbit@node1
```

- rabbitmqctl rotate_logs {suffix}  
指示RabbitMQ 节点轮换日志文件。RabbitMQ节点会将原来的日志文件中的内容追加到“原始名称+后缀”的日志文件中，然后再将新的日志内容记录到新创建的日志中（与原日志文件同名）。  
当目标文件不存在时，会重新创建。如果不指定后缀suffix，则日志文件只是重新打开而不会进行轮换。

```
// 原日志文件为rabbit@node1.log 和rabbit@node1-sasl.log
// 轮换日志之后，原日志文件中的内容就被追加到rabbit@node1.log.bak 和 rabbit@node1-sasl.log.bak 日志中
// 之后重新建立rabbit@node1.log 和rabbit@node1-sasl.log 文件用来接收新的日志
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

```
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

```
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

```
[root@node2 ~]# rabbitmqctl force_boot
Forcing boot for Mnesia dir /opt/rabbitmq/var/lib/rabbitmq/mnesia/rabbit@node2
[root@node2 ~]# rabbitmq-server –detached
```

- rabbitmqctl sync_queue [-p vhost] {queue}  
指示未同步队列queue 的slave 镜像可以同步master 镜像行的内容。  
同步期间此队列会被阻塞（所有此队列的生产消费者都会被阻塞），直到同步完成。此条命令执行成功的前提是队列queue 配置了镜像。  
注意，未同步队列中的消息被耗尽后，最终也会变成同步，此命令主要用于未耗尽的队列。

```
[root@node1 ~]# rabbitmqctl sync_queue queue
Synchronising queue 'queue' in vhost '/'
```

- rabbitmqctl cancel_sync_queue [-p vhost] {queue}  
取消队列queue 同步镜像的操作

```
[root@node1 ~]# rabbitmqctl cancel_sync_queue queue
Stopping synchronising queue 'queue' in vhost '/'
```

- rabbitmqctl set_cluster_name {name}  
设置集群名称。集群名称在客户端连接时会通报给客户端。  
Federation 和Shovel 插件也会有用到集群名称的地方。集群名称默认是集群中第一个节点的名称。  
在Web 管理界面的右上角有个“（change）”的地方，点击也可以修改集群名称。

```
[root@node1 ~]# rabbitmqctl set_cluster_name cluster_hidden
Setting cluster name to cluster_hidden
[root@node1 ~]# rabbitmqctl cluster_status
Cluster status of node rabbit@node1
[{nodes,[{disc,[rabbit@node1,rabbit@node2]}]},{running_nodes,[rabbit@node2,rabbit@node1]},
{cluster_name,<<"cluster_hidden">>},{partitions,[]},{alarms,[{rabbit@node2,[]},{rabbit@node1,[]}]}]
```

#### 服务端状态
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

```
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

```
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

```
// 交换器exchange1 和队列queue1 通过rk1 进行绑定，还有一个独立的队列queue2。显示的第一行是默认的交换器与queue1 进行绑定，这个是RabbitMQ 内置的功能。
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

```
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

```
[root@node1 ~]# rabbitmqctl list_channels -q
<rabbit@node1.1.631.0> root 0 0
<rabbit@node1.1.643.0> root 1 0
```

- rabbitmqctl list_consumers [-p vhost]  
列举消费者信息。  
每行将显示由制表符分隔的已订阅队列的名称、相关信道的进程标识、consumerTag、是否需要消费端确认、prefetch_count 及参数列表这些信息。

```
[root@node1 ~]# rabbitmqctl list_consumers -p default -q
queue4 <rabbit@node1.1.1628.11> consumer_zzh true 0 []
```

- rabbitmqctl status  
显示Broker 的状态，比如当前Erlang 节点上运行的应用程序、RabbitMQ/Erlang 的版本信息、OS 的名称、内存及文件描述符等统计信息。

- rabbitmqctl node_health_check  
对RabbitMQ 节点进行健康检查，确认应用是否正常运行、list_queues 和list_channels是否能够正常返回等。

```
[root@node1 ~]# rabbitmqctl node_health_check
Timeout: 70.0 seconds
Checking health of node rabbit@node1
Health check passed
```

- rabbitmqctl environment  
显示每个运行程序环境中每个变量的名称和值。

- rabbitmqctl report  
为所有服务器状态生成一个服务器状态报告，并将输出重定向到一个文件。

```
[root@node1 ~]# rabbitmqctl report > report.txt
```

- rabbitmqctl eval {expr}  
执行任意Erlang 表达式。  
用户、Parameter、vhost、权限等都可以通过rabbitmqctl 工具来完成创建（或删除）的操作，而交换器、队列及绑定关系的创建（或删除）操作可以通过rabbitmqctl eval {expr} 实现。

```
// 返回rabbitmqctl连接的节点名称
[root@node1 ~]# rabbitmqctl eval 'node().'
rabbit@node1
```

<span style="color: red;font-weight: bold;">Tips</span>：若要删除所有的交换器、队列及绑定关系，删除对应的vhost 就可以“一键搞定”，而不需要一个个遍历删除。

#### HTTP API 接口管理
RabbitMQ Management 插件不仅提供了Web 管理界面，还提供了HTTP API 接口来方便调用。  
HTTP API 是完全基于RESTful 风格的，不同的HTTP API 接口所对应的HTTP 方法各不相同，这里一共涉及4 种HTTP 方法：GET、PUT、DELETE 和POST。

    GET 方法一般用来获取如集群、节点、队列、交换器等信息。  
    PUT 方法用来创建资源，如交换器、队列之类的。  
    DELETE 方法用来删除资源。  
    POST 方法也是用来创建资源的，与PUT 不同的是，POST 创建的是无法用具体名称的资源。比如绑定关系（bindings）和发布消息（publish）无法指定一个具体的名称。

所有的HTTP API 接口都需要HTTP 基础认证（使用标准的RabbitMQ 用户数据库），默认的是guest/guest（非localhost 的不能使用这组认证，除非特殊设置）

```
// 通过curl 命令创建队列queue ，“%2F”是指默认的vhost，即“/”，这类特殊字符在HTTP URL 中是需要转义的
[root@node1 ~]# curl -i -u root:root123 -H "content-type:application/json"
-XPUT -d '{"auto_delete":false,"durable":true,"node":"rabbit@node2"}'
http://192.168.0.2:15672/api/queues/%2F/queue
HTTP/1.1 201 Created
server: Cowboy
date: Fri, 25 Aug 2017 06:03:17 GMT
content-length: 0
content-type: application/json
vary: accept, accept-encoding, origin

// 通过GET 方法来获取队列queue 的信息
[root@node1 ~]# curl -i -u root:root123 –XGET http://192.168.0.2:15672/api/queues/%2F/queue

// 通过DELETE 方法来删除队列queue
[root@node1 ~]# curl -i -u root:root123 -XDELETE http://192.168.0.2:15672/api/queues/%2F/queue
```

<span style="color: red;font-weight: bold;">Tips</span>：Web 管理界面左下角的“HTTP API”可以跳转到相应的“RabbitMQ Management HTTP API”帮助页面，里面有详细的接口信息。

#### rabbitmqadmin
rabbitmqadmin是RabbitMQ Management 插件提供的功能，它会包装HTTP API 接口，使其调用显得更加简洁方便。

rabbitmqadmin 是需要安装的，可以点击Web 管理页面左下角的“Command Line”跳转到“rabbitmqadmin”页面进行下载，或者通过下面的示例进行下载并添加可执行权限。在使用rabbitmqadmin前还要确保已经成功安装Python。

```
[root@node1 ~]# wget http://192.168.0.2:15672/cli/rabbitmqadmin
[root@node1 ~]# chmod +x rabbitmqadmin
```

<span style="color: red;font-weight: bold;">Tips</span>：通过rabbitmqadmin --help 命令可以获得相应的使用方式。

```
// 创建队列
[root@node1 ~]# ./rabbitmqadmin -u root -p root123 declare queue name=queue1
queue declared
// 显示队列
[root@node1 ~]# ./rabbitmqadmin list queues
+--------+----------+
| name   | messages |
+--------+----------+
| queue1 | 0        |
+--------+----------+
// 删除队列
[root@node1 ~]# ./rabbitmqadmin -u root -p root123 delete queue name=queue1
queue deleted
```


### RabbitMQ 配置