## ssh登录认证过程

### 1 ssh登录认证

#### 1.1 普通登录

![](https://github.com/luofengmacheng/shell_scripts/blob/master/pics/ssh_learn1.png)

需要注意的是，当客户端已经登录过某个服务器时，会将服务器的公钥保存到known_hosts文件中，当下次登录时，会先验证该公钥是否与服务器发送过来的一致，如果不一致，则会提示失败。

解决办法是：

* 删除known_hosts文件
* 修改客户端的配置，使得不用验证公钥是否一致

```
UserKnownHostsFile /dev/null
```

#### 1.2 免密码登录

![](https://github.com/luofengmacheng/shell_scripts/blob/master/pics/ssh_learn2.png)