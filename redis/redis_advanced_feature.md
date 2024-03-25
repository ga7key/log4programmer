> 基于Redis 2.9，适用于Redis 2.6至Redis 3.0

## 发布与订阅
Redis的发布与订阅功能由PUBLISH、SUBSCRIBE、PSUBSCRIBE等命令组成。  
通过执行SUBSCRIBE命令，客户端可以订阅一个或多个频道，从而成为这些频道的订阅者（subscriber）：每当有其他客户端向被订阅的频道发送消息（message）时，频道的所有订阅者都会收到这条消息。

除了订阅频道之外，客户端还可以通过执行PSUBSCRIBE命令订阅一个或多个模式，从而成为这些模式的订阅者：每当有其他客户端向某个频道发送消息时，消息不仅会被发送给这个频道的所有订阅者，它还会被发送给所有与这个频道相匹配的模式的订阅者。

### 订阅与退订

###### 订阅频道
当一个客户端执行SUBSCRIBE命令订阅某个或某些频道的时候，这个客户端与被订阅频道之间就建立起了一种订阅关系。  
Redis将所有频道的订阅关系都保存在服务器状态的pubsub_channels字典里面，这个字典的键是某个被订阅的频道，而键的值则是一个链表，链表里面记录了所有订阅这个频道的客户端：
```
struct redisServer {
  // ...
  // 保存所有频道的订阅关系
  dict *pubsub_channels;
  // ...
};
```

每当客户端执行SUBSCRIBE命令订阅某个或某些频道的时候，服务器都会将客户端与被订阅的频道在pubsub_channels字典中进行关联。

- 如果频道已经有其他订阅者，那么它在pubsub_channels字典中必然有相应的订阅者链表，程序唯一要做的就是将客户端添加到订阅者链表的末尾。
- 如果频道还未有任何订阅者，那么它必然不存在于pubsub_channels字典，程序首先要在pubsub_channels字典中为频道创建一个键，并将这个键的值设置为空链表，然后再将客户端添加到链表，成为链表的第一个元素。

例如，现有客户端client-10086执行命令：
```
SUBSCRIBE "news.sport" "news.movie"
```

![pubsub_channels](../images/redis/2024-03-25_pubsub_channels.png)

###### 退订频道
UNSUBSCRIBE命令的行为和SUBSCRIBE命令的行为正好相反，当一个客户端退订某个或某些频道的时候，服务器将从pubsub_channels中解除客户端与被退订频道之间的关联：
- 程序会根据被退订频道的名字，在pubsub_channels字典中找到频道对应的订阅者链表，然后从订阅者链表中删除退订客户端的信息。
- 如果删除退订客户端之后，频道的订阅者链表变成了空链表，那么说明这个频道已经没有任何订阅者了，程序将从pubsub_channels字典中删除频道对应的键。

执行命令示例：
```
UNSUBSCRIBE "news.sport" "news.movie"
```

###### 订阅模式
服务器将所有模式的订阅关系都保存在服务器状态的pubsub_patterns属性里面：
```
struct redisServer {
  // ...
  // 保存所有模式订阅关系
  list *pubsub_patterns;
  // ...
};
```
pubsub_patterns属性是一个链表，链表中的每个节点都包含着一个pubsub Pattern结构，这个结构的pattern属性记录了被订阅的模式，而client属性则记录了订阅模式的客户端：
```
typedef struct pubsubPattern {
  // 订阅模式的客户端
  redisClient *client;
  // 被订阅的模式
  robj *pattern;
} pubsubPattern;
```

每当客户端执行PSUBSCRIBE命令订阅某个或某些模式的时候，服务器会对每个被订阅的模式执行以下两个操作：  
1. 新建一个pubsubPattern结构，将结构的pattern属性设置为被订阅的模式，client属性设置为订阅模式的客户端。
2. 将pubsubPattern结构添加到pubsub_patterns链表的表尾。

执行命令示例：
```
PSUBSCRIBE "news.*"
```

![pubsub_patterns](../images/redis/2024-03-25_pubsub_patterns.png)

###### 退订模式
当一个客户端退订某个或某些模式的时候，服务器将在pubsub_patterns链表中查找并删除那些pattern属性为被退订模式，并且client属性为执行退订命令的客户端的pubsubPattern结构。

执行命令示例：
```
PUNSUBSCRIBE "news.*"
```

### 发送消息
当一个Redis客户端执行PUBLISH \<channel> \<message>命令将消息message发送给频道channel的时候，服务器需要执行以下两个动作：
1. 将消息message发送给channel频道的所有订阅者。
2. 如果有一个或多个模式pattern与频道channel相匹配，那么将消息message发送给pattern模式的订阅者。

###### 将消息发送给频道订阅者
因为服务器状态中的pubsub_channels字典记录了所有频道的订阅关系，所以为了将消息发送给channel频道的所有订阅者，PUBLISH命令要做的就是在pubsub_channels字典里找到频道channel的订阅者名单（一个链表），然后将消息发送给名单上的所有客户端。

执行命令示例：
```
PUBLISH "news.it" "hello"
```

###### 将消息发送给模式订阅者
因为服务器状态中的pubsub_patterns链表记录了所有模式的订阅关系，所以为了将消息发送给所有与channel频道相匹配的模式的订阅者，PUBLISH命令要做的就是遍历整个pubsub_patterns链表，查找那些与channel频道相匹配的模式，并将消息发送给订阅了这些模式的客户端。

### 查看订阅信息
PUBSUB命令是Redis 2.8新增加的命令之一，客户端可以通过这个命令来查看频道或者模式的相关信息，比如某个频道目前有多少订阅者，又或者某个模式目前有多少订阅者。

###### PUBSUB CHANNELS
PUBSUB CHANNELS [pattern]子命令用于返回服务器当前被订阅的频道，其中pattern参数是可选的：  
▶ 如果不给定pattern参数，那么命令返回服务器当前被订阅的所有频道。  
▶ 如果给定pattern参数，那么命令返回服务器当前被订阅的频道中那些与pattern模式相匹配的频道。

执行PUBSUB CHANNELS命令将返回服务器目前被订阅的频道：
```
redis> PUBSUB CHANNELS
1) "news.it"
2) "news.sport"
3) "news.business"
4) "news.movie"
```

执行PUBSUB CHANNELS "news.[is]\*"命令将返回"news.it"和"news.sport"两个频道，因为只有这两个频道和"news.[is]\*"模式相匹配：
```
redis> PUBSUB CHANNELS "news.[is]*"
1) "news.it"
2) "news.sport"
```

###### PUBSUB NUMSUB
PUBSUB NUMSUB [channel-1 channel-2...channel-n]子命令接受任意多个频道作为输入参数，并返回这些频道的订阅者数量。  
这个子命令是通过在pubsub_channels字典中找到频道对应的订阅者链表，然后返回订阅者链表的长度来实现的（订阅者链表的长度就是频道订阅者的数量）。

执行命令示例：
```
redis> PUBSUB NUMSUB news.it news.sport news.business news.movie
1) "news.it"
2) "3"
3) "news.sport"
4) "2"
5) "news.business"
6) "2"
7) "news.movie"
8) "1"
```

###### PUBSUB NUMPAT
PUBSUB NUMPAT子命令用于返回服务器当前被订阅模式的数量。  
这个子命令是通过返回pubsub_patterns链表的长度来实现的，因为这个链表的长度就是服务器被订阅模式的数量。

执行命令示例：
```
redis> PUBSUB NUMPAT
(integer) 3
```


## 事务


## Lua脚本


## 排序


## 二进制位数组


## 慢查询日志


## 监视器

