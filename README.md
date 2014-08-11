### autopass
------------
该脚本由bash shell实现，功能类似`expect`，ssh到远程主机可以实现自动输入密码（密码验证模式下），用法比expect简单很多。

示例如下：
```bash
$ ./autopass -h 172.16.9.140 -k 123456 "uname -a"
Linux centos6 2.6.32-431.el6.i686 #1 SMP Fri Nov 22 00:26:36 UTC 2013 i686 i686 i386 GNU/Linux

$ ./autopass -h 172.16.9.140 -k 123456 "/sbin/ifconfig eth0|grep -oP '(?<=inet addr:)[^ ]*'"    #执行复杂shell命令
172.16.9.140
```

### TODO
------------
* 支持scp
* 支持sudo
* 支持并发连接多台远程主机
