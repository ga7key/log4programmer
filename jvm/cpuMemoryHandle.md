#### CPU与内存问题排查  

### CPU高占用定位
1.使用top命令，查看CPU占用情况，找到Java的pid(进程id)  
![top](../images/jvm/2023-08-14_201514.png)  
2.使用ps -mp命令将这个pid下线程占用cpu情况查出来，找到高占用的tid(线程id)，ps -mp 'pid' -o THREAD,tid,time  
![ps-mp](../images/jvm/2023-08-14_202134.png)  
3.使用printf "%x\n"将tid转换成16进制  
![encode](../images/jvm/2023-08-14_202249.png)  
4.使用jstack通过pid和tid查找线程的运行状态，jstack 'pid' |grep 'tid' >> problem.log  
![jstackGrep](../images/jvm/2023-08-14_212059.png)  

### 内存高占用定位
1.使用jmap命令手动生成堆转储快照(dump文件)。jmap -dump:format=b,file='dumpFileName' 'pid'  
    例：jmap -dump:format=b,file=/logFile/dump.dat 3334  
2.使用内存工具分析MAT、jvisualvm等进行分析。以MAT为例：  
File → Open Heap Dump... 打开dump.dat  
查看Overview中的Dominator Tree，确认内存占用高的线程对象，右键 → Java Basics → Thread Overview and Stacks，再通过内存占用情况和具体的对象值定位代码  
![memoryAnalysis](../images/jvm/2023-08-14_224923.png)