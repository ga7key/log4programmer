### 重置TCP/IP和DNS配置

1.以管理员的方式运行命令行工具 cmd.exe；  
2.分别执行以下各行命令，以回车结束：  
```
ipconfig /flushdns
nbtstat –r
netsh int ip reset
netsh winsock reset
```
3.重启生效。  