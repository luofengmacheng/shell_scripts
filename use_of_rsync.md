## rsync的使用

rsync的常用形式为：

```
rsync options source destination
```

常用的选项有：

* -v 详细模式
* -r 递归拷贝数据(但是不保留时间戳和权限)
* -a 归档模式，递归拷贝数据，并且，还会保留符号链接、权限、属主和时间戳
* -z 压缩模式
* -h 可读模式
* -e 指定协议
* --progress 显示进度
* --include & --exclude 包含和剔除文件
* --delete 删除选项，当目标机器有文件在原始机器上没有，就删除目标机器的该文件，也就是说，有了该选项，源和目标一定一致
* --max-size 传输文件的最大大小
* --remove-source-files 传输结束时删除原始文件
* --bwlimit 最大传输带宽

参考文档：

1 [Rsync (Remote Sync): 10 Practical Examples of Rsync Command in Linux](http://www.tecmint.com/rsync-local-remote-file-synchronization-commands/)