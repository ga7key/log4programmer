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

### Producer：生产者
就是投递消息的一方。生产者创建消息，然后发布到RabbitMQ 中。  
消息一般可以包含2个部分：**消息体**和**标签**（Label）。  
消息体也可以称之为payload，在实际应用中，消息体一般是一个带有业务逻辑结构的数据，比如一个JSON字符串。当然可以进一步对这个消息体进行序列化操作。  
消息的标签用来表述这条消息，比如一个交换器的名称和一个路由键。生产者把消息交由RabbitMQ，RabbitMQ之后会根据标签把消息发送给对应的Broker。

### Consumer：消费者
就是接收消息的一方。消费者连接到RabbitMQ 服务器，并订阅到队列上。当消费者消费一条消息时，只是消费消息的消息体（payload）。  
在消息路由的过程中，消息的标签会丢弃，存入到队列中的消息只有消息体，消费者也只会消费到消息体，也就不知道消息的生产者是谁，当然消费者也不需要知道。

### Broker
一个RabbitMQ Broker 可以简单地看作一个RabbitMQ 服务节点，或者RabbitMQ 服务实例。大多数情况下也可以将一个RabbitMQ Broker 看作一台RabbitMQ服务器。

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

### Queue：队列
RabbitMQ 的内部对象，用于存储消息。  
RabbitMQ 中消息都只能存储在队列中，这一点和Kafka 这种消息中间件相反。Kafka 将消息存储在topic（主题）这个逻辑层面，而相对应的队列逻辑只是topic 实际存储文件中的位移标识。  
多个消费者可以订阅同一个队列，这时队列中的消息会被平均分摊（Round-Robin，即轮询）给多个消费者进行处理，而不是每个消费者都收到所有的消息并处理。  
<span style="color: red;">tips</span>：RabbitMQ 不支持队列层面的广播消费，如果需要广播消费，需要在其上进行二次开发，处理逻辑会变得异常复杂，同时也不建议这么做。

### Exchange：交换器
生产者将消息发送到Exchange（交换器，通常也可以用大写的“X”来表示），由交换器将消息路由到一个或者多个队列中。如果路由不到，或许会返回给生产者，或许直接丢弃。

#### 交换器类型
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

### RoutingKey：路由键
生产者将消息发给交换器的时候，一般会指定一个RoutingKey，用来指定这个消息的路由规则，而这个Routing Key 需要与交换器类型和绑定键（BindingKey）联合使用才能最终生效。

#### BindingKey
在绑定的时候使用的路由键（the routing key to use for the binding）。大多数时候，包括官方文档和RabbitMQ Java API中都把BindingKey 和RoutingKey 看作RoutingKey。

### Binding：绑定
RabbitMQ 中通过绑定将交换器与队列关联起来，在绑定的时候一般会指定一个绑定键（BindingKey），生产者将消息发送给交换器时，需要一个RoutingKey，当BindingKey 和RoutingKey 相匹配时，消息会被路由到对应的队列中。  
在绑定多个队列到同一个交换器的时候，这些绑定允许使用相同的BindingKey。  
BindingKey 并不是在所有的情况下都生效，它依赖于交换器类型，比如fanout 类型的交换器就会无视BindingKey，而是将消息路由到所有绑定到该交换器的队列中。

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


