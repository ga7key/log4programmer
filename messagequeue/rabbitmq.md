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

### RabbitMQ连接

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

##### 当mandatory 为 true 时，添加ReturnListener 监听器
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

#### 拉模式
通过channel.basicGet 方法可以单条地获取消息，其返回值是GetRespone  

```
GetResponse basicGet(String queue, boolean autoAck) throws IOException;
```
> queue：队列的名称  
autoAck：设置是否自动确认。建议设成false，即不自动确认  

<span style="color: red;font-weight: bold;">Tips</span>：Basic.Consume 将信道（Channel）置为接收模式，直到取消队列的订阅为止。在接收模式期间，RabbitMQ 会不断地推送消息给消费者，当然推送消息的个数还是会受到Basic.Qos 的限制。如果只想从队列获得单条消息而不是持续订阅，建议还是使用Basic.Get 进行消费。但是不能将Basic.Get 放在一个循环里来代替Basic.Consume，这样做会严重影响RabbitMQ的性能。如果要实现高吞吐量，消费者理应使用Basic.Consume 方法。  

#### 消费端的确认与拒绝








### 用RabbitTemplate时保证消息可靠性

**一.确保消息发送到 RabbitMQ 的 Exchange**
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
- 路由保证
1. 失败通知（mandatory+ReturnListener）  
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
            * 参数2：是否批量确认，属于一个队列中的消息，全部确认，false:只确认当前消息
            */
            channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
        }catch (Exception e) {
            count.incrementAndGet();
            if(count.get() == 3){
                log.error("[ackErr]消费者出现异常，共执行{}次，开启确认失败！：{}", count, e.getMessage());
                /*
                * 异常后确认
                * 参数1：消息标签
                * 参数2：是否批量处理 true：批量(消费当前消息后，后面的不管成没成功都会被应答，不安全，只有在确保通道中的消息百分百消费成功时才可使用)，false:只确认当前消息
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


