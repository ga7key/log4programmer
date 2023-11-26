### 减少上下文切换

Lmbench3 可以测量上下文切换带来的消耗。  
vmstat 可以测量上下文切换的次数。  

> $ vmstat 3 7 // 每 3 秒进行一次采样，共进行 7 次

**减少上下文切换的方法**  
- 无锁并发编程  
避免使用锁，如将数据的 id 按照 HASH 算法取模分段，不同线程处理不同分段的数据。
- CAS 算法  
Compare and Swap，比较与交换。Java 的 Atomic 包使用 CAS 算法来更新数据，不用创建锁。
- 使用最少线程  
避免创建多余的线程，创建多余的线程会造成大量线程处于等待状态。
- 协程  
在单线程里实现多任务的调度，并在单线程里维持多个任务间的切换。

**查看 WAITING 的线程数可以确认上下文切换次数**  
1.查找要处理的线程信息  
> $ ps -ef | grep xxx

2.用 jstack 命令 dump 线程信息，查看第 1 步的线程内容，例如第一步的线程是 9527  
> $ sudo -u admin /opt/java/bin/jstack 9527 > /home/mydump/dump9527

3.统计所有线程状态，查看处于 WAITING (on object monitor) 的状态的线程数量  
> $ grep java.lang.Thread.State dump9527 | awk '{print $2$3$4$5}' | sort | uniq -c

4.打开 dump9527 文件查看 WAITING (on object monitor) 状态的线程，发现其处于 await 状态  
5.减少对应的线程数量，重启服务。再次 dump 线程信息，查看上下文切换次数是否减少  