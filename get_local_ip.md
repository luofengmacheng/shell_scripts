## 获取本机IP

> 场景：在服务器上执行脚本时，经常需要获取本机IP。在我们的场景中，eth0是外网IP，eth1是内网IP，我们要获取的是内网IP。

### 1 ifconfig

`ifconfig eth1 | grep "inet " | awk '{print $2}' | awk -F':' '{print $2}'`

### 2 ip addr

`ip addr | grep eth1 | grep global|grep brd | awk '{print $2}' | awk -F'/' '{print $1}'`

### 3 netstat

但是，当面对有多个内网IP时，由于不知道可以通过哪个IP登录上去，上面的办法就无能为力了。

因此，这里的思想是，想获取可以登录的内网IP，那么，可以获取监听36000端口(ssh)的IP。

`netstat -ltn | grep 36000 | awk '{print $4}' | awk -F':' '{print $1}'`

netstat常用的选项：

* -n 以数字形式显示主机、端口和用户
* -p 显示程序的PID和名字
* -l 只显示监听的连接
* -t 只显示TCP协议的连接
* -u 只显示UDP协议的连接

### 4 ss

`ss -ntl|grep 36000|awk '{print $3}'|awk -F ':' '{print $1}'`

### 5 小结

ifconfig和netstat属于net-tools套件，而ip和ss则属于iproute2套件，现在iproute2正在主键代替net-tools，而且，当前的发行版基本都默认安装了iproute2，因此，以后还是多使用ip addr和ss。