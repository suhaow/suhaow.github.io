# FTP

## 介绍

FTP`(File transfer protocol)`在TCP/IP协议族中属于应用层协议运行于TCP协议之上是一种可靠的传输协议，主要功能用于实现用户间文件分发共享，以及网络管理者在进行设备版本升级、日志下载和配置保存等业务操作时，均会使用到FTP功能。

FTP不同于其他服务的是它使用了两个端口，一个数据端口（通常为20端口），一个命令端口，也称控制端口（通常为21端口）

## 传输模式

FTP协议有两种工作方式：`PORT`方式和`PASV`方式，中文意思为主动模式和被动模式。其中是否主动是站在`FTP`服务端来讨论的，而选择使用哪种传输方式的选择权则是在`FTP`客户端。

### 主动模式

主动模式下，FTP客户端从任意的非特殊端口（N>1023）连入到FTP服务器的21命令端口，然后客户端在（N+1）端口监听，并且通过N+1端口发送PORT命令将N+1端口告知服务器，然后服务器会**主动**从自己的20数据端口发起到客户端告知的N+1端口的连接。连接成功后，开始传输数据。



![image-20200617203101285](https://gitee.com/suhaow/note/raw/master/img/20200617203105.png)



### 被动模式

被动模式对应命令为 PASV（全称Passive）。在被动模式中，命令连接和数据连接都由客户端发起。当开启FTP连接时，客户端打开任意两个非特权端口（N>1023和N+1）。第一个端口连接服务器的21端口，并发送PASV命令告知服务器启动被动模式，服务端接收到PASV命令后，开启一个任意非特权端口（P>1024），并发送PORT P命令给客户端，然后由客户端主动从本地N+1到服务器的P端口的连接

![image-20200617203211883](https://gitee.com/suhaow/note/raw/master/img/20200617203212.png)



### 区别

简单说 主动模式就是服务器 **主动连接** 到客户端的端口，而被动模式就是客户端主动连接服务器的端口，服务器负责开放端口后进行监听，等待客户端连接。

## 使用场景

当`client`端和`server`端同处一个局域网使用两种模式都不会存在问题，

但现实环境中无论是`Client`端还是`Server`端都是在防火墙后面，在主动模式下`VSFTP`会链接`Client`端的随机+1号端口，`Client`端显然不会将防火墙上所有随机端口开放；而在被动下问题同样的问题仍然会摆在Server端的防火墙面前，这就需要`Server`端的防火墙开启连接追踪功能，即放行与21号端口有关联的端口访问请求，这也就是为什么大部分情况下`VSFTP`是以被动模式工作。

---

# Vsftpd 

## 介绍

vsftpd`(Very Secure FTP Daemon)`，是一个以安全为主的`FTP`服务器

## 安装

```
[root@suhw ~]# yum install -y vsftpd
```

## 启动

```bash
# 启动 vsftpd
[root@suhw ~]# systemctl start vsftpd
# 查看状态
[root@suhw ~]# systemctl status vsftpd
● vsftpd.service - Vsftpd ftp daemon
   Loaded: loaded (/usr/lib/systemd/system/vsftpd.service; disabled; vendor preset: disabled)
   Active: active (running) since Wed 2020-06-17 20:12:50 CST; 6s ago
  Process: 17277 ExecStart=/usr/sbin/vsftpd /etc/vsftpd/vsftpd.conf (code=exited, status=0/SUCCESS)
 Main PID: 17278 (vsftpd)
    Tasks: 1
   Memory: 648.0K
   CGroup: /system.slice/vsftpd.service
           └─17278 /usr/sbin/vsftpd /etc/vsftpd/vsftpd.conf

Jun 17 20:12:47 suhw systemd[1]: Starting Vsftpd ftp daemon...
Jun 17 20:12:50 suhw systemd[1]: Started Vsftpd ftp daemon.

```

## 配置文件

| 文件地址                | 作用                                                         |
| ----------------------- | ------------------------------------------------------------ |
| /etc/vsftpd/vsftpd.conf | vsftpd 配置文件                                              |
| /etc/pam.d/vsftpd       | vsftpd 使用 pam 时的相关破欸之                               |
| /etc/vsftpd/ftpusers    | 记录着 拒绝登录ftp 的账号                                    |
| /etc/vsftpd/user_list   | 记录需要管理的 vsftpd 用户账号 ，与 vsftpd.conf 中的 userlist_anle，userlist_deny 有关，控制列表中的用户是否可登录 vsftpd 账号 |
| /etc/vsftpd/chroot_list | 将指定用户 chroot 在对应的家目录下，与vsftpd.conf 中的 chroot_list_enable 和 chroot_list_file 有关 |
| /var/ftp                | vsftpd 预设匿名者登录的根目录                                |

## 参数介绍

下面只列举`/etc/vsftpd/vsftpd.conf`中部分参数介绍，详细介绍参考`man 5 vsftpd.conf`

| 参数                             | 作用                                                         |
| -------------------------------- | ------------------------------------------------------------ |
| anonymous_enable=NO              | 是否允许匿名帐户登录 FTP 服务器                              |
| local_enable=YES                 | 是否允许本地账户登录 FTP 服务器                              |
| write_enable =YES                | 是否开启目录上次权限                                         |
| local_umask=022                  | 本地用户上传文件的权限掩码                                   |
|                                  |                                                              |
| 目录消息                         |                                                              |
| dirmessage_enable=YES            | 用户第一次进入目录时，vsftpd会查看.message文件，并将其内容显示给用户 |
| message_file                     | 指定文件路径，而非默认的.message                             |
|                                  |                                                              |
| 数据传输日志                     |                                                              |
| xferlog_enable=YES               | 是否开启日志功能                                             |
| xferlog_file=/var/log/vsftpd.log | 日志的存放路径                                               |
| xferlog_std_format=No            | 日志的格式                                                   |
|                                  |                                                              |
| 数据传输模式                     |                                                              |
| connect_from_port_20=YES         | 是否启用PORT模式                                             |
|                                  |                                                              |
| chroot_local_user=YES            |                                                              |
| 修改匿名用户上传的文件的属主     |                                                              |
| chown_uploads                    | 是否修改                                                     |
| chown_username                   | 启用chown_uploads指令时，将文件属主修改为此指令指定的用户，默认为root |
| chown_upload_mode                | 设置匿名用户上传文件的权限，默认600                          |
|                                  |                                                              |
| 设定会话超时时长                 |                                                              |
| idle_session_timeout             | 空闲会话超时时长                                             |
| connect_timeout                  | prot模式下，服务器连接客户端的超时时长                       |
| data_connection_timeout          | 数据传输的超时时长                                           |
|                                  |                                                              |
| 与主机相关的设定值               |                                                              |
| listen_port                      | 默认为21                                                     |
| listen                           | 若设定为YES 表示 vsftpd 以standalone 方式启动，默认为No      |
|                                  |                                                              |
| 设定连接及传输速率               |                                                              |
| local_max_rate                   | 本地用户的最大传输速率                                       |
| anon_max_rate                    | 匿名用户的最大传输速率                                       |
| max_clients                      | 最大并发连接数                                               |
| max_per_ip                       | 每个ip所允许发起的最大连接数                                 |
|                                  |                                                              |
| 禁锢本地用户                     |                                                              |
| chroot_local_user                | 是否禁锢所有本地用户                                         |
| chroot_list_enable               |                                                              |
|                                  |                                                              |
| 用户访问控制                     |                                                              |
| userlist_enable                  | 若置为开启，vsftpd将加载由userlist_file指令指定的用户列表文件，此文件中的用户是否能访问vsftpd服务取决于userlist_deny指令； |
| userlist_deny                    | 表示此列表是否为黑名单(YES表示为黑名单，反之为白名单)        |
|                                  |                                                              |
| pasv_enable                      | 值为 YES 表示启动被动模式                                    |
| pasv_min_port=0 pasv_max_port=0  | 设定被动模式所用的端口号，0代表随机取用                      |
| pasv_address                     | 设置ftp服务器返回的pasv地址                                  |



---

# 被动模式配置

 上面介绍了两种传输模式的区别以及各自应用场景，实际开发过程中被动模式十分常见，相关的配置可参考如下：

```ini
# 启用被动模式
pasv_enable=YES

# 随机端口所用的端口号范围
pasv_min_port=6000
pasv_max_port=7000

#指定被动模式回传的服务器IP地址
pasv_address=10.44.192.126

# 指定vsftpd侦听的地址为非IPv6格式
listen_ipv6=NO

# 以 standalone 方式启动
listen=YES
```

---

# 参考

- http://linux.vbird.org/linux_server/0410vsftpd.php
- https://yq.aliyun.com/articles/545714
- http://blog.sina.com.cn/s/blog_946cb2b70100x4zc.html