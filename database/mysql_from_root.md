> MySQL版本为 5.7
> 拆抄自《MySQL是怎样运行的：从根儿上理解MySQL》 小孩子4919 著

### MySQL启停
#### UNIX里启动服务器程序
在类UNIX系统中用来启动MySQL服务器程序的可执行文件有很多，大多在MySQL安装目录的bin目录下。

###### mysqld
mysqld这个可执行文件就代表着MySQL服务器程序，运行这个可执行文件就可以直接启动一个服务器进程。但这个命令不常用。

###### mysqld_safe
mysqld_safe是一个启动脚本，它会间接的调用mysqld，而且还顺便启动了另外一个监控进程，在服务器进程挂了的时候，可以帮助重启服务器进程。另外，使用mysqld_safe启动服务器程序时，它会将服务器程序的出错信息和其他诊断信息重定向到某个文件中，产生出错日志。

###### mysql.server
mysql.server也是一个启动脚本，它会间接的调用mysqld_safe，这个 mysql.server 文件其实是一个链接文件，它的实际文件是 ../support-files/mysql.server。  
在调用mysql.server时在后边指定start参数可以启动服务器程序：

```bash
mysql.server start
```

指定stop参数可以停止服务器程序：

```bash
mysql.server stop
```

#### Windows里启动服务器程序

###### mysqld
在MySQL安装目录下的bin目录下有一个mysqld可执行文件，在命令行里输入mysqld，或者直接双击运行它就算启动了MySQL服务器程序了。

###### 以服务的方式运行服务器程序
将MySQL注册为Windows服务：  
"完整的可执行文件路径" --install [-manual] [服务名]  
-manual可以省略，表示手动启动程序；服务名也可以省略，默认的服务名是MySQL。  
例如：

```powershell
"C:\Program Files\MySQL\MySQL Server 5.7\bin\mysqld" --install
```

之后可以用命令行启停MySQL：

```powershell
net start MySQL
net stop MySQL
```

#### 启动MySQL客户端程序

```bash
mysql -h主机名 -u用户名 -p密码
```

- -h 表示服务器进程所在计算机的域名或者IP地址，如果服务器进程就运行在本机的话，可以省略这个参数，或者填localhost或者127.0.0.1。也可以写作 --host=主机名的形式。
- -u 表示用户名。也可以写作 --user=用户名的形式。
- -p 表示密码。也可以写作 --password=密码的形式。

断开客户端与服务器的连接并且关闭客户端的命令：

```mysql
quit
exit
\q
```

<span style="color: red;font-weight: bold;">Tips</span>：  
