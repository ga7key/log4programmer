> 基于Redis 2.9，适用于Redis 2.6至Redis 3.0

### 数据结构与对象

#### SDS
simple dynamic string，简单动态字符串，Redis的默认字符串表示。  
字符串、哈希表、列表、集合等数据类型的键值对底层都是由SDS 实现的，因为Redis 需要表示一个可以被修改的字符串值。  
SDS还被用作缓冲区（buffer）：AOF模块中的AOF缓冲区，以及客户端状态中的输入缓冲区。

在Redis里面，C语言的字符串只会作为字符串字面量（string literal）用在一些无须对字符串值进行修改的地方，比如打印日志。