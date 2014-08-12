### autopass
------------
该脚本由bash shell实现，功能类似`expect`，ssh到远程主机可以实现自动输入密码（密码验证模式下），用法比expect简单很多。

示例如下：
```bash
$ ./autopass 
Usage: 
autopass -h host [-u user] [-p port] [-k password | -K password_file]
        [-o ssh_args] [-t ssh_timeout] [-m module] -a "module_args"

Options:
  -m <module>             要执行的模块，值为ssh表示在远程主机执行命令，scp表示传输文件，默认为ssh
  -a <module_args>        模块参数，对ssh来说就是要执行的命令，对scp来说就是路径
  -h <host>               远程主机
  -u <user>               ssh用户，默认为当前用户
  -p <port>               ssh端口，默认22端口
  -k <password>           ssh密码
  -K <password_file>      ssh密码文件路径，密码文件必须有可执行权限
  -o <ssh_args>           ssh连接选项
  -t <ssh_timeout>        ssh连接超时时间，默认30s

$ ./autopass -h 172.16.9.140 -k 123456 -a "/sbin/ifconfig eth0"             #执行命令
eth0      Link encap:Ethernet  HWaddr 08:00:27:AD:A1:41  
          inet addr:172.16.9.140  Bcast:172.16.9.255  Mask:255.255.255.0
          inet6 addr: fe80::a00:27ff:fead:a141/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:216873 errors:0 dropped:0 overruns:0 frame:0
          TX packets:133789 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:16902661 (16.1 MiB)  TX bytes:44942465 (42.8 MiB)

$ ./autopass -h 172.16.9.140 -k 123456 -m scp -a "-s README.md -d /tmp/"     #传输文件，-s和-d是传递给scp模块的参数，-s是源路径，-d是目的路径
File transfer success
```
你还可以执行ssh模块但不提供`-a`选项，或`-a`选项的值为空字符串，这样可以连接到远程主机并打开一个远程shell来交互。
  

### 密码文件
------------
可以通过`-k`选项在命令行输入密码，也可以构造密码文件然后通过`-K`选项指定，密码文件是一个*可执行脚本*，且该脚本的输出就是密码。

示例如下：
```bash
$ cat /tmp/test_pass_file.sh 
#!/bin/sh
echo "123456"

$ ./autopass -h 172.16.9.140 -K /tmp/test_pass_file.sh -a "uname -a"
Linux centos6 2.6.32-431.el6.i686 #1 SMP Fri Nov 22 00:26:36 UTC 2013 i686 i686 i386 GNU/Linux
```
还可以直接修改脚本的`PASSWORD_FILE`变量，将密码文件直接赋值给该变量，然后就可以省略掉`-k`或`-K`参数了：
```bash
$ grep '^PASSWORD_FILE' autopass 
PASSWORD_FILE="/tmp/test_pass_file.sh"

$ ./autopass -h 172.16.9.140 -a "uname -a"
Linux centos6 2.6.32-431.el6.i686 #1 SMP Fri Nov 22 00:26:36 UTC 2013 i686 i686 i386 GNU/Linux
```
  

### TODO
------------
* 支持sudo
* 支持并发连接多台远程主机
