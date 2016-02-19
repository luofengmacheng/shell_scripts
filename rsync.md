## rsync

### 1 rsync简介

rsync是一款远程同步的软件，它能够识别本地和远程文件的差异，只传输必要的部分，从而减少传输量。

### 2 rsync的配置

客户端和服务器安装完[rsync](https://rsync.samba.org/)后，就可以进行rsync的配置。

客户端和服务器的主要差别在于配置和rsync命令运行的机器。

#### 2.1 服务器配置

1 修改/etc/xinetd.d/rsync，将`disable = yes`改为`disable = no`。

2 创建rsync配置：/etc/rsyncd.conf

```
uid = haha
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

3 启动

rsync --daemon

### 3 rsync的使用

常见的使用方式有两种：

#### 3.1 使用目录。

```
rsync [option] src [user@]host:dest
rsync [option] [user@]host:src dest
```

如上所示，在客户端，既可以从本地同步到远程，又可以从远程同步到本地。其中，user即为同步使用的用户，由于没有使用模块的方式，因此，只能使用rsyncd.conf中的全局配置，即haha用户。当然，为了使用这种认证方式，必须添加`--password-file=passwd_file`。

#### 3.2 使用模块

```
rsync [option] src [user@]host::module_name[dest]
rsync [option] [user@]host::module_name[src] dest
```

使用模块时，就可以不用写很长的路径名，直接使用模块中配置的path路径。而且，只能使用该模块中配置的auth users进行同步。

### 4 rsync的原理

基于效率的原因，rsync同步时只传输有变化的部分，因此，rsync的关键算法是：当两个文件不在同一个机器上时，如何快速计算两个文件的差异？

为了解决这个问题，rsync使用了两个算法：[adler32算法](https://rsync.samba.org/tech_report/node3.html)和MD5算法。

在了解adler32算法之前，先来了解下rsync的工作流程(以拉取文件为例)：

![](https://github.com/luofengmacheng/shell_scripts/blob/master/pics/rsync_liucheng.png)

1 当在客户端发起rsync命令时，客户端的rsync进程会与服务器端的rsync进程建立连接。然后，服务器端会fork一个进程(Sender)，该进程会创建要同步的文件列表，并发送给客户端。

2 客户端rsync进程在收到Sender发送来的文件列表，会fork两个进程(Generator和Receiver)，Generator会对要同步的文件分块计算adler32 checksum和MD5值。

3 Generator将adler32 checksum和MD5值发送给Sender。

4 Sender在收到Generator发送来的校验和与MD5值时，会与要同步的文件比对差异，生成文件的差异以及变更的内容。

5 Sender将差异以及变更的内容发送给Receiver。

6 Receiver在收到Sender发送来的差异和变更的内容时，会合并到要同步的文件。

7 Receiver通知Generator更新文件完成。