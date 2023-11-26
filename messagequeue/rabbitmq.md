### 用RabbitTemplate时保证消息可靠性

一.确保消息发送到 RabbitMQ 的 Exchange
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

二.确保消息路由到 RabbitMQ 的队列
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

三.确保消息在 RabbitMQ 正常存储 
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

四.消费确认
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


