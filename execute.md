## shell中exec与source的区别

前言：shell中有两种执行程序的方法：exec和source。它们都是shell的内部命令，在执行程序时，有一些不同之处：

* 执行上下文
* 环境变量

### 1 exec

exec可以用于执行一个shell文件：exec filename。当在shell文件使用该命令时，它的作用是将filename中的代码拷贝到当前执行上下文中，也就是，相当于重新执行filename程序，这就会导致，当前文件后面的代码将不会执行。

### 2 source

source用于执行一个shell文件：source filename。当在shell文件中使用该命令时，它的作用相当于将filename中的代码放在该语句的地方。因此，程序执行的上下文没有改变，并且，filename中的环境变量会导入到当前环境。

### 3 ./

最常用的执行shell文件的方式是：./filename。当在shell文件中使用该命令时，它的作用是，新建一个进程，在该进程中执行file程序。因此，两部分代码同时执行。./ = fork + exec