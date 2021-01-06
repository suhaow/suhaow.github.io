# NFS介绍

## 介绍

`NFS`是`Network FileSystem`的缩写，主要功能是通过网络，让不同的机器，不同的操作系统，可以彼此分享指定的资源文件，可简单的认为是一个文件服务器，可以将远程`NFS`服务器共享的目录挂载到本地机器中，在本地机器看起来，被挂载的目录就像是自己的一个磁盘分区一样，使用非常方便。

![image-20200616151317977](https://gitee.com/suhaow/note/raw/master/img/20200616151320.png)

上图中的`NFS`服务器设定好了要共享的目录后，其他客户端就可直接将该目录挂载到自己系统上的某个挂载点，就可通过挂载点直接访问服务器共享目录中的数据了（前提权限要够）

## NFS数据获取流程

**背景**

由于`NFS`支持的功能较多，不通的功能都会使用不同的程序来启动，而每一个程序都会启动一些端口来传输数据，所以`NFS`功能对应的端口没有固定，而是随机获取一些小于1024的端口来用。所以当客户端需要使用这些端口的时候，就需要远程过程调用(RPC)的服务了。当`NFS`服务器在启动时，会将随机获取来的端口都注册到`RPC`中，因为`RPC`知道每个端口对应的`NFS`功能，这样当客户端请求服务端时，只需要让监听在111端口的`RPC`服务来返回客户端正确的端口即可。

**注：**所以启动`NFS`之前，就需要启动`RPC`服务，否则`NFS`无法向`RPC`注册，同时当`RPC`重启时，旧的注册数据就会丢失，他管理的所有服务都需重新向`RPC`注册。



所以，当服务端将端口向`RPC`注册完成后，若有客户端有`NFS`数据存取需求时，执行流程如下：

1. 客户端会向服务器端的 `RPC (port 111) `发出 NFS 档案存取功能的询问要求；
2. 服务器端找到对应的已注册的端口后，会回报给客户端；
3. 客户端了解正确的端口后，就可以直接与运行在该端口的程序来联机。

## 主要配置文件

默认配置文件地址：`/etc/exports`，该配置文件的内容需要自己手动写入

**语法和参数**

```bash
[root@suhw ~]# vim /etc/exports
/tmp         192.168.100.0/24(ro)   localhost(rw)   *.ev.ncku.edu.tw(ro,sync)
[分享目录]    [第一部主机(权限)]   [可用主机]  [可用通配符]
```

详细的用户可参考示例。

权限参考参数，更全的介绍参考 `man exports`

| 参数值         | 内容说明                                                     |
| -------------- | ------------------------------------------------------------ |
| ro             | 该主机对共享目录权限为只读权限                               |
| rw             | 该主机对共享目录权限为读写权限                               |
| root_squash    | 客户端用root用户访问该共享文件夹时，将root用户映射为匿名用户 |
| no_root_squash | 客户端用root用户访问该共享文件夹时，不映射root用户，如果你想要开放客户端使用 root 身份来操作服务器的文件系统，那么这里就得要开 no_root_squash 才行！ |
| all_squash     | 客户机上的任何用户访问该共享目录时都映射成匿名用户           |
| anonuid        | 将客户机上的用户映射成指定的本地用户ID的用户                 |
| anongid        | 将客户机上的用户映射成属于指定的本地用户组ID                 |
| sync           | 资料同步写入到内存与硬盘中                                   |
| async          | 资料会先暂存于内存中，而非直接写入硬盘                       |
| insecure       | 允许从这台机器过来的非授权访问                               |

**举例**

1. 将`/home/public`公开，但是只有`192.168.100.0/24`这个网段的用户可以读写，其他来源只能读取

```
/home/public 192.168.100.0/24(rw) *(ro)
```

2. 将`/home/iso`目录公开，允许所有人进行读操作

```
/home/iso        *(ro,insecure,no_root_squash)
```

注意` no_root_squash `的功能。在这个例子中，如果你是客户端，而且你是以 root 的身份登入你的 Linux 主机，那么当你 mount 上我这部主机的`/home/iso `之后，你在该 mount 的目录当中，将具有`root `的权限！

## `exportfs`指令

利用`exportfs`可以重新分享` /etc/exports` 变更的目录资源、将` NFS Server `分享的目录卸除或重新分享等等。

**语法和参数**

| 参数       | 作用                                                   |
| ---------- | ------------------------------------------------------ |
| -a         | 打开或取消所有目录共享                                 |
| -o options | 指定一列共享选项                                       |
| -i         | 忽略 /etc/exports 文件，只使用默认的和命令行指定的选项 |
| -r         | 重新共享所有目录，使/etc/exportfs生效                  |
| -u         | 取消一个或多个目录的共享                               |
| -v         | 输出详细信息                                           |
| -s         | 显示/etc/exports中的列表                               |

## 查看分享资源记录

在`NFS`服务器的登陆文件都放置到`/var/lib/nfs`目录中，比较重要的有`etab`

`etab`记录了`NFS`所分享出来的目录的完整权限设定值

```
/home/iso     (ro,sync,wdelay,hide,nocrossmnt,insecure,no_root_squash,no_all_squash,no_subtree_check,secure_locks,acl,no_pnfs,anonuid=65534,anongid=65534,sec=sys,insecure,no_root_squash,no_all    _squash)

```

## `showmount`指令

`exportfs`是用于在`NFS server`端维护分享的资源，而`shownmount`则是用于`NFS client`端，该命令可以查看出`NFS server`分享出的目录资源

**语法和参数**

`showmount [ -adehv ] [ --all ] [ --directories ] [ --exports ] [ --help ] [ --version ] [ host ]`

| 参数选项            | 作用                          |
| ------------------- | ----------------------------- |
| -d or --directories | 显示已被nfs客户端加载的目录   |
| -e or --exports     | 显示nfs服务端上所有的共享目录 |

---

# 挂载远程目录示例

## NFS Server 配置

### 环境准备

要想对`NFS server`配置首先需要`rpcbind`和`nfs-utils`

可以使用`rpm -qa | grep nfs-utils`查询是否已经安装该软件，若没安装过执行下列命令

```bash
[root@suhw ~]# yum install -y nfs-utils
```

`rpcbind`：`NFS`可被视为`RPC`服务，而启动`RPC`服务之前，我们首先需要启动`rpcbind`

```bash
[root@suhw ~]# systemctl start rpcbind
[root@suhw ~]# systemctl status rpcbind
● rpcbind.service - RPC bind service
   Loaded: loaded (/usr/lib/systemd/system/rpcbind.service; enabled; vendor preset: enabled)
   Active: active (running) since Tue 2020-06-16 10:26:08 CST; 2s ago
  Process: 29537 ExecStart=/sbin/rpcbind -w $RPCBIND_ARGS (code=exited, status=0/SUCCESS)
 Main PID: 29538 (rpcbind)
    Tasks: 1
   Memory: 660.0K
   CGroup: /system.slice/rpcbind.service
           └─29538 /sbin/rpcbind -w

Jun 16 10:26:07 suhw systemd[1]: Starting RPC bind service...
Jun 16 10:26:08 suhw systemd[1]: Started RPC bind service.

```

由于`rpc`固定使用`111`端口，所以可直接查看111端口是否处于监听状态

```bash
[root@suhw ~]# netstat -tulnp | grep 111
tcp6       0      0 :::111                  :::*                    LISTEN      29538/rpcbind
udp        0      0 0.0.0.0:111             0.0.0.0:*                           29538/rpcbind
udp6       0      0 :::111                  :::*                                29538/rpcbind

```

### 启动服务

确保`rpcbind`启动

```bash
#查看 rpcbind 状态
[root@suhw ~]# systemctl status rpcbind
#若rpcbind未启动，则start
[root@suhw ~]# systemctl start rpcbind
```

确保`nfs`启动

```bash
[root@suhw ~]# systemctl status nfs
#若nfs未启动，则start
[root@suhw ~]# systemctl start nfs
```

### 配置文件编写

允许`10.91`开头的地址共享`/home/test`目录，并具有可读可写操作

```bash
[root@suhw ~]# cat /etc/exports
/home/test 10.91.*(rw,sync,no_root_squash)
```

### 配置文件生效

使用上面介绍的用来管理当前NFS共享的文件系统列表的`exportfs`命令

```bash
# 重新共享，使 exports 生效
[root@suhw ~]# exportfs -r
[root@suhw ~]# exportfs -s
/home/test  10.91.*(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,no_root_squash,no_all_squash)

```

### 验证

`NFS`服务端配置完成后，先在服务端通过`showmount`命令测试下是否已经共享成功

```bash
[root@suhw ~]#  showmount -e localhost
Export list for localhost:
/home/test 10.91.*
```

此时查看之前说的``etab`就会发现多了一条记录i

```bash
[root@suhw ~]# cat /var/lib/nfs/etab
/home/test      10.91.*(rw,sync,wdelay,hide,nocrossmnt,secure,no_root_squash,no_all_squash,no_subtree_check,secure_locks,acl,no_pnfs,anonuid=65534,anongid=65534,sec=sys,rw,secure,no_root_squash,no_all_squash)

```

在挂载目录下创建个文件，方便一会观察现象

```bash
[root@suhw ~]# touch /home/test/test.txt
```

---

## NFS Cient配置

### 环境验证

1、确认本地端已经启动了 rpcbind 服务

```bash
[root@nfs-client ~]# systemctl status rpcbind
● rpcbind.service - RPC bind service
   Loaded: loaded (/usr/lib/systemd/system/rpcbind.service; enabled; vendor preset: enabled)
   Active: active (running) since Mon 2020-06-15 18:11:40 CST; 20h ago
 Main PID: 1264 (rpcbind)
   CGroup: /system.slice/rpcbind.service
           └─1264 /sbin/rpcbind -w

Jun 15 18:11:40 csmp-SP1Fusion systemd[1]: Starting RPC bind service...
Jun 15 18:11:40 csmp-SP1Fusion systemd[1]: Started RPC bind service.
```

2、确保安装`showmount`

### 查看远程服务器共享资源

```bash
[root@nfs-client ~]# showmount -e 10.91.156.174
Export list for 10.91.156.174:
/home/test 10.91.*/
```

### 进行挂载

1、在本地端建立预计要挂载的挂载点目录

```bash
[root@nfs-client ~]# mkdir -p /home/test
```

2、利用 mount 将远程主机直接挂载到相关目录

```bash
[root@nfs-client ~]# mount 10.91.156.174:/home/test /home/test
```

3、查看挂载信息

```bash
[root@nfs-client ~]# df -h
Filesystem                Size  Used Avail Use% Mounted on
'''
10.91.156.174:/home/test   17G  8.0G  9.1G  47% /home/test
```

### 验证

查看本地挂载目录下的内容即可

```bash
[root@nfs-client ~]# ll /home/test/
total 0
-rw-r--r-- 1 root root 0 Jun 16 14:47 test.txt
```

---

## 取消挂载

若不需要挂载时，使用`umount`命令指定本地挂载目录即可

```bash
[root@nfs-client ~]# umount /home/test
```

---

# 参考

http://cn.linux.vbird.org/linux_server/0330nfs.php#What_NFS_0