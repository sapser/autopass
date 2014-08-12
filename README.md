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
可以通过`-k`选项在命令行输入密码，也可以构造密码文件然后通过`-K`选项指定，密码文件是一个**可执行脚本**，且该脚本的输出就是密码本身。

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
  

### 并发执行
------------
如果你有很多台主机需要维护，且这些主机都是使用密码验证，那么还可以配合下面这段代码实现并发连接多台远程主机执行任务：
```bash
$ cat parall_run.sh 
#!/bin/sh
#
#shell并发执行脚本

#引用未定义的变量时报错并退出脚本
set -o nounset

#主机列表
HOSTLIST=("172.16.9.140" "172.16.9.141")

#并发量
FORKS=5

#命名管道文件描述符
NAMED_PIPE_FD=5


#并发控制
parall() {
    local i= host=

    #建立命名管道文件
    local pipe="$(mktemp -u)" 
    mkfifo "$pipe" 
    #将文件$pipe绑定到指定文件描述符 
    exec 5<>"$pipe"
    rm -f "$pipe" 

    #往命名管道存入值，存入多少个值就表示并发量多大
    for ((i=1;i<=${FORKS};i++)); do 
        echo >&5
    done 
    
    for host in ${HOSTLIST[@]}; do
        #从命名管道读取一个值
        #如果此时命名管道中没有值，read会阻塞直到读取到一个值  
        read -u 5
        #开始执行命令，执行完后放入一个值到命名管道
        (./autopass -h "$host" -k 123456 -a "sleep 2 && /sbin/ifconfig eth0"; \
            echo >&5) &
    done

    #等待所有后台进程执行完毕
    wait 

    #关闭文件描述符
    exec 5>&-
}

parall

$ time ./parall_run.sh 
eth0      Link encap:Ethernet  HWaddr 08:00:27:97:96:30  
          inet addr:172.16.9.141  Bcast:172.16.9.255  Mask:255.255.255.0
          inet6 addr: fe80::a00:27ff:fe97:9630/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:352578 errors:0 dropped:0 overruns:0 frame:0
          TX packets:239097 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:76962448 (73.3 MiB)  TX bytes:45608661 (43.4 MiB)

eth0      Link encap:Ethernet  HWaddr 08:00:27:AD:A1:41  
          inet addr:172.16.9.140  Bcast:172.16.9.255  Mask:255.255.255.0
          inet6 addr: fe80::a00:27ff:fead:a141/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:305956 errors:0 dropped:0 overruns:0 frame:0
          TX packets:189809 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:23875481 (22.7 MiB)  TX bytes:63792630 (60.8 MiB)


real    0m2.179s
user    0m0.030s
sys     0m0.017s
```
连接到两台远程主机并执行命令`sleep 2 && /sbin/ifconfig eth0`，如果不使用并发执行，则最少需要4秒才能执行完，而通过`time`命令计算出脚本执行时间为2秒，可见这里是并发执行的。
  

### 其他
----------------
没有将并发代码写到脚本内，是因为加了并发代码后，脚本不太容易看明白，而且也不是人人都需要兵法功能，有需求的可以自己实现。该脚本不出意外就是这样子了，不会有其他功能添加了。
