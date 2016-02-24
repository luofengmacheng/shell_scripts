## rsync如何工作(翻译)

原文地址：[How Rsync Works](https://rsync.samba.org/how-rsync-works.html)

### 1 Foreword

* rsync算法的非数学描述
* rsync软件中的rsync算法是如何实现的
* rsync软件中使用的协议
* rsync的进程所扮演的角色

目的：

* rsync为什么要这么做？
* rsync的限制？
* 为什么所要求的功能特性与代码库不适应？

### 2 Processes and Roles

<table>
	<tr>
		<td>client</td>
		<td>role</td>
		<td>客户端初始化同步</td>
	</tr>
	<tr>
		<td>server</td>
		<td>role</td>
		<td>远程的rsync进程，可以是通过。当client和server建立连接后，它们之间的不同在于sender和receiver角色</td>
	</tr>
	<tr>
		<td>daemon</td>
		<td>角色和进程</td>
		<td>等待客户端连接的rsync进程</td>
	</tr>
	<tr>
		<td>remote shell</td>
		<td>角色和进程集合</td>
		<td>远程系统上，一个或者多个进程，它们提供rsync客户端和服务器之间的连接</td>
	</tr>
	<tr>
		<td>sender</td>
		<td>角色和进程</td>
		<td>能够访问需要同步的原文件的rsync进程</td>
	</tr>
	<tr>
		<td>receiver</td>
		<td>角色和进程</td>
		<td>接收更新数据，并将数据写入到磁盘</td>
	</tr>
	<tr>
		<td>generator</td>
		<td>process</td>
		<td>计算old文件的</td>
	</tr>
</table>

### 3 Process Startup

当客户端启动时，会与服务器进程建立连接。该连接可能通过管道，也可能通过socket。

当rsync通过远程shell与远程非daemon服务器进行通信时，会在远程shell基础上fork出一个rsync服务器。当rsync客户端和服务器端通过远程shell的管道进行通信。在这里rsync进程之间没有网络。在这种模式中，rsync的选项会传递给命令行被用来启动远程shell。

当rsync与daemon进行通信时，它们之间通过网络socket通信。在这种模式中，rsync选项会通过socket传递。

在客户端和服务器最开始的通信时，它们会互相发送各自支持的最大版本号。两端都采用版本号的最小值作为传输的协议级别。如果是daemon模式连接，rsync选项会从客户端发送到服务器端。然后传输排除列表。在这之后，客户端和服务器之间的关系仅与错误和日志信息的传递有关。

本地rsync(源和目的都在本地)与push类似。客户端就是sender，它会fork出一个server进程作为receiver角色。它们之间的通信都采用管道。

### 4 The File List

文件列表不仅包含路径，还包含所有者、模式、权限、大小、最后修改时间。如果带有--checksum选项，还会包含文件checksum。

当两边建立连接后，sender会创建文件列表。创建完成后，每个条目会以网络优化的方式传送到接收端。

当这个完成后，两端会相对于传输的基础目录对文件列表进行排序。(具体的排序算法依赖于传输的协议版本号)

如果有必要的话，sender会在文件列表后面加上id->name表，receiver会用它对每个文件进行翻译。

receiver收到文件列表后，它会fork出generator和receiver完成流水线。

### 5 The Pipeline

rsync采用流水线的机制。这就意味着，进程之间进行单向通信。当共享文件列表后，流水线以这样的方向运作：

generator -> sender -> receiver

generator的输出是sender的输入，sender的输出是receiver的输入。每个进程独立运行，只有当流水线暂停或者等待磁盘IO或是CPU资源时，流水线才会有延迟。

### 6 The Generator

generator会将文件列表与本地目录树比较。如果有--delete选项，在它开始执行主要工作之前，它会找到在文件列表中没有的文件并删除它们。

首先，generator会遍历文件列表，检查该文件是否可以被跳过。在大多数的操作模式中，如果最后修改时间或者文件大小改变了，就不能跳过该文件。如果指定了--checksum，会创建文件的checksum，并进行比较。不能跳过目录、设备文件、符号链接。当目录找不到时，会创建该目录。

如果没有跳过某个文件，在接收端的对应文件会成为传输的基础文件，并成为数据源，这有助于减少sender传输的数据。创建基础文件的块checksum，然后放在文件的索引号后面立即发送给sender。如果指定了--whole-file选项，对于新文件而言，就是整个空块的checksum。

在后来的版本中，根据文件大小计算每个文件的块大小和块checksum的大小。

### 7 The Sender

sender进程从generator读取文件索引和对应的块checksum集合。

对于每个文件，sender会将块checksum保存到哈希表中。

然后，sender会读取本地文件，对第一个块生成checksum。然后在哈希表中查找该checksum，如果没有找到，没有匹配的字节会被放到非匹配数据中，然后再比较下一个字节开始的块checksum。这就是"rolling checksum"。

如果找到了某个块checksum，它就是匹配块，任何累积的非匹配数据将被发送到receiver，然后，就是匹配块的偏移和长度，就会计算匹配块的下一个字节开始的块的checksum。(If a block checksum match is found it is considered a matching block and any accumulated non-matching data will be sent to the receiver followed by the offset and length in the receiver's file of the matching block and the block checksum generator will be advanced to the next byte after the matching block.这句比较难翻译)

即使块的顺序乱了，或者偏移不同，也可以通过这种方式找到匹配的块。这个过程就是rsync算法的核心。

这样的话，sender通过指令可以告诉receiver如何从原始文件构建出目标文件。这些指令说明了可以从原始文件拷贝哪些数据，还包括原始文件不包含的数据。在处理完一个文件后，会生成整个文件的checksum，再去处理下一个文件。

生成rolling checksum和查找操作需要大量的CPU计算资源，sender是所有的rsync进程中最耗费CPU资源的。

### 8 The Receiver

receiver会读取sender发送的数据。receiver会打开本地文件，并创建一个临时文件。

receiver会读取不匹配的数据，并找到匹配的记录，形成最终文件。当读取到不匹配的数据时，写入到临时文件。当读取到匹配的记录时，receiver会从原始文件中对应的偏移位置读取块，然后拷贝到临时文件。

当生成完临时文件时，就会生成文件的checksum。然后将该checksum与sender发送过来的checksum比较。如果不一样，就删除临时文件，然后重新尝试该过程，如果失败了两次，就会报错。

当创建临时文件完成后，设置它的所有者、权限、最后修改时间，然后重命名并替换原始文件。

将数据从原始文件拷贝到临时文件会使得receiver成为所有rsync进程中最消耗磁盘资源的。小文件保存在磁盘缓存中，但是对于大文件而言，缓存就没有太大的用处了。由于数据从一个文件写入到另一个文件中可能是随机的，因此，如果工作集比磁盘缓存大，就会发生所谓的"seek storm"，这会有损性能。

### 9 The Daemon

rsync的daemon跟其它的daemon进程一样，当收到一个连接时，就会fork出一个进程。当rsync作为daemon进程启动后，会读取rsyncd.conf文件，从而决定存在哪些模块，并设置全局选项。

当收到一个定义好的模块的连接时，daemon会fork一个新的进程处理该连接。该子进程会读取rsyncd.conf文件，然后针对请求的模块设置对应的选项，这可能是chroot，也可能是丢弃setuid，也可能是setgid。之后，它会像其它rsync服务器进程一样成为sender或者receiver。

### 10 The Rsync Protocol

一个设计良好的通信协议有以下特点：

* 要发送的内容放在一个定义良好的包中，该包有头部和选择性的包体或者有效载荷。
* 在包头指明类型或者命令。
* 包的长度有限。

除了这些特点之外，协议还应该有不同程度的状态性、包间独立性、可读性、重连机制。

rsync的协议没有以上这些好的特性。数据作为一个连续的字节流传输。除了不匹配的数据，没有指定长度和数量。相反，每个字节的意义依赖于已经定义好的协议级别的上下文。

例如，当sender发送文件列表时，它只是简单地发送每个文件列表项，最后以空字节结束。在文件列表项中，有一个位指明所预期的结构域，并且，当长度可变时，以空结束。generator发送文件数和块checksum集采用相同的方式。

这种通信方式在可靠连接可以工作的很好，而且它比正常的协议有更少的数据开销。但是，这就使得协议本身很难归档、调试和扩展。每个协议版本有很小的差别，而这只能通过协议版本号进行区分。

### 11 小结(非原文内容)

这篇文件大致地讲解了rsync的一些原理性的内容，对于理解rsync的工作原理有一定的指导作用，但是，或者是由于rsync本身的发展，文章在介绍rsync的核心算法时，少了一个过程：比较rolling checksum后，并不能断定两个块就是一样，还需要计算MD5。