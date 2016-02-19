## rsync配置、使用和原理

### 1 rsync简介

rsync是一款远程同步的软件，它能够识别本地和远程文件的差异，只传输必要的部分，从而减少传输量。

### 2 rsync的配置

客户端和服务器安装完[rsync](https://rsync.samba.org/)后，就可以进行rsync的配置。

客户端和服务器的主要差别在于配置和rsync命令运行的机器。

#### 2.1 服务器配置

1 修改/etc/xinetd.d/rsync，将`disable = yes`改为`disable = no`。

2 创建rsync配置：/etc/rsyncd.conf

```
gid = users
port = 873       # 运行端口，默认是873
read only = true # 是否只读
transfer logging = true # 是否传送日志
log format = %a %o %P %f %l
log file = /var/log/rsyncd.log
pid file = /var/run/rsyncd.pid
max connections = 100 # 最大连接数
secrets file = /etc/rsyncd.secrets # 同步时使用的密码文件，用于认证

[test] # 模块名，可以自己指定
path = /home/test # 模块的同步目录
comment = rsync # 说明
auth users = rsync # 同步模块时，使用的用户
hosts allow = 192.168.0.1 # 同步时，允许访问的机器IP
```

需要注意的是：模块中使用的用户既可以是本机的用户，也可以是自定义的用户，
如果是本机的用户，就不需要配置密码文件，如果是自定义的用户，就需要配置密码文件。

3 创建认证文件：/etc/rsyncd.secrets

内容为：`帐号:密码`，例如，

```
rsync:pass
haha:hh
```

4 启动

rsync --daemon

#### 2.2 客户端配置

1 创建配置文件：/etc/rsyncd.conf，文件内容为空

2 创建认证文件：rsyncd.pass，文件内容为与rsync服务器同步的用户的密码，如使用test模块，则用户名为rsync，密码为pass，那么rsyncd.pass的内容就是pass。

### 3 rsync的使用

常见的使用方式有两种：

#### 3.1 不使用模块

```
rsync [option] src [user@]host:dest
rsync [option] [user@]host:src dest
```

在客户端，既可以从本地同步到远程，又可以从远程同步到本地。其中，user即为同步使用的用户，由于没有使用模块的方式，因此，就要使用服务器的用户，并且，在执行时，需要输入host机器的user用户的密码。

#### 3.2 使用模块

```
rsync [option] src [user@]host::module_name[dest]
rsync [option] [user@]host::module_name[src] dest
```

使用模块时，就可以不用写很长的路径名，直接使用模块中配置的path路径。而且，只能使用该模块中配置的auth users进行同步。

### 4 rsync的原理

rsync的工作流程(以拉取文件为例)：

![](https://github.com/luofengmacheng/shell_scripts/blob/master/pics/rsync_liucheng.png)

由于以拉取文件为例，因此，server上的文件为新文件，称为new_file，client上的文件为旧文件，称为old_file。

1 当在客户端发起rsync命令时，客户端的rsync进程会与服务器端的rsync进程建立连接。然后，服务器端会fork一个进程(Sender)，该进程会创建要同步的文件列表，并发送给客户端。

2 客户端rsync进程在收到Sender发送来的文件列表，会fork两个进程(Generator和Receiver)，Generator会对old_file分块计算Rolling checksum和MD5值。

3 Generator将Rolling checksum和MD5值发送给Sender。

4 Sender在收到Generator发送来的校验和与MD5值时，会与new_file比对差异，生成文件的差异以及变更的内容。

5 Sender将差异以及变更的内容发送给Receiver。

6 Receiver在收到Sender发送来的差异和变更的内容时，会合并到要同步的文件。

7 Receiver通知Generator更新文件完成。

#### 4.1 比对差异

基于效率的原因，rsync同步时只传输有变化的部分，因此，rsync的关键算法是：当两个文件不在同一个机器上时，如何快速计算两个文件的差异？

首先，客户端和服务器进程可以协商一个块大小，例如1KB，Generator会对old_file进行分块，对每块计算checksum和MD5值。Sender在收到checksum和MD5数组时，会将checksum和MD5数组保存到一个哈希表中，然后依次遍历new_file，例如，先对1~1KB字节计算checksum，在哈希表中查找是否存在，如果不存在，说明1~1KB字节没有在old_file中有相同的块。然后往后移动一个字节，计算2~1KB+1字节的checksum，依然在哈希表中查找，如果有，说明old_file中可能会有个块与2~1KB+1字节一样，但不能确定，因为checksum是不严格的校验，当某些位改变时，计算得来的checksum也可能相同，此时，需要计算2~1KB+1字节的MD5值，再与刚找到的checksum对应的MD5值比较，如果不一样，则说明也找不到与2~1KB+1字节相同的块，于是，再往后移动1个字节。当checksum和MD5值都相同时，则找到一个old_file相同的块，则记录下该块的起始和终止位置，再从终止位置的下一个字节开始处理。

理论上来说，这里的checksum用网络中通常的算法也可以，但是，当已经计算出1~1KB字节的checksum时，如何能够快速地计算出2~1KB+1字节的checksum？

为了解决这个问题，rsync使用了[adler32算法](https://rsync.samba.org/tech_report/node3.html)。

checksum由A和B组成，2~1KB+1的checksum可以由1~1kB字节的checksum计算而得，因此，这种checksum被称为Rolling checksum，具体算法可以查看上面的文档。

#### 4.2 rsync vs rsync over ssh

上面说到rsync的使用时，提到了两种使用方式，一种是不采用模块的形式，一种是采用模块的形式。不采用模块的形式时，不需要服务器以守护进程启动rsync，只需要安装有rsync即可，在这种情况下，rsync会以ssh方式连接到远程机器，然后启动rsync进程，源和目的采用管道进行通信。而采用模块的形式时，源和目的采用socket进行通信。或许是由于ssh本身的不稳定，在使用第一种方式时，对端的rsync进程有时候会拉不起来，从而导致本机的rsync会一直等待，给人的感觉就是卡死。基于这个原因，在自动化运营中，尽量使用模块的形式。