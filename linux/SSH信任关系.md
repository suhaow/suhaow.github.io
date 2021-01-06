# 介绍

SSH（Secure Shell）是一项创建在应用层和传输层基础上得安全协议，为shell提供安全得传输和使用环境。具体协议以及服务相关知识不再赘述，下面主要介绍如何配置服务器之间的信任关系。以配置A免密登录到B为例

---

# 生成密钥

在A机器上通过`ssh key-gen`生成密钥

```bash
[root@suhw ~]# eval `ssh-agent -s`
[root@suhw ~]# ssh-keygen -t rsa
```

若不指定存放密钥的路径以及名字的话，会默认生成在当前用户家目录下的`.ssh`目录中生成一个私钥和一个公钥

```bash
[root@suhw ~]# ll /root/.ssh/
-rw------- 1 root root 2590 Jun 28 15:41 id_rsa
-rw-r--r-- 1 root root  563 Jun 28 15:41 id_rsa.pub
```

**注：不可能每次生成密钥都覆盖掉之前的 `~/.ssh/id_rsa`文件，建议在生成密钥第一步时指定文件名，便于区分管理**

---

# 配置公钥

将生成的公钥追加到B机器的`authorized_keys`文件中

1、如果B机器`.ssh`目录下不存在`authorized_keys`文件，则通过touch命令新建，并将权限修改为600

```bash
[root@suhw ~]# cd ~/.ssh && touch authorized_keys && chmod 600 authorized_keys
```

2、将A机器生成的`id_rsa》pub`公钥追加到`authorized_keys`中

---

# 相关配置

`ssh`的配置文件包含以下两个：

- ~/.ssh/config：用户配置文件
- /etc/ssh/ssh_config：系统配置文件

其中config配置中最常用的就是管理多组密钥对，对于多个服务器指定对应的密钥对，可以参考如下配置

```
HOST host1 # 关键词
    User root    # 登录用户名
    HostName 10.47.119.96 # 主机名
    IdentityFile /root/.ssh/host1_id_rsa # 私钥存储地址
HOST host2
    User root 
    HostName 10.47.119.97
    IdentityFile /root/.ssh/host2_id_rsa 
```

有了上面的配置，当我们在A机器上执行`ssh host1`时就会匹配到对应`Host`的配置，相当于执行了

```bash
ssh -i /root/.ssh/host1_id_rsa root@10.47.119.96
```

这样不论是`ssh`还是`scp`都会通过`host`匹配到对应的密钥文件进行连接，十分方便

## 配置项说明

- `Host`：执行`ssh`命令时如果匹配到该配置，例如将`HOST`修改为`kylin`，在A机器上执行`ssh kylin`就相当于执行了`ssh -i /data/suhw/.ssh/suhw_id_rsa root@10.44.103.165`
- `HostName`：主机地址
- `IdentityFile`：指定读取的认证文件路径

其他具体参数可参考文章最后的链接

---

# FAQ

1、若遇到报错`Could not open a connection to your authentication agent.`

先执行  `eval ssh-agent -s`

---

# 参考

- https://deepzz.com/post/how-to-setup-ssh-config.html