# 简介

`Supervisor` 是可以在类 `UNIX` 系统中进行管理和监控各种进程的小型系统。它自带了客户端和服务端工具。可以通过用户所定义的配置文件来管理和监控单个或多个进程，并且它可以根据配置来对异常崩溃的进程进行重启操作。



---

# 环境信息

```bash
[root@suhw ~]# cat /etc/redhat-release
CentOS Linux release 7.7.1908 (Core)
```



---

# 安装

```bash
[root@suhw ~]# yum install epel-release
[root@suhw ~]# yum install -y supervisor
[root@suhw ~]# systemctl enable supervisord # 开机自启动
[root@suhw ~]# systemctl start supervisord # 启动supervisord服务
[root@suhw ~]# systemctl status supervisord # 查看supervisord服务状态
```



---

# 配置

配置选项可具体参考`/etc/supervisord.conf`中的详细介绍，或者可参考http://supervisord.org/

若想新增要管理的进程，直接在`supervisor`的配置文件默认目录`/etc/supervisord.d/`下添加`ini`文件即可，在`/etc/supervisord.conf`中已定义会将`/etc/supervisord.d/`目录下所有的`ini`文件都会包含进来。

```ini
...
[include]
files = supervisord.d/*.ini
```



---

# 管理`redis`

每个 `program` 代表了一个要进行管理的进程，我们可以在这里进行进程的相关配置。

## 编辑配置文件

假设此时需要使用`supervisor`管理`redis-server`，编辑`/etc/supervisord.d/redis.ini`

```ini
# 项目名
[program:redis]
# 脚本执行命令
command=/usr/bin/redis-server /etc/redis.conf
# 当进程数为 1 时 为 %(program_name)s 
# 当进程数 >1 时 应配置为 %(program_name)s_%(process_num)02d
process_name=%(program_name)s

# 如果numprocs>1，supervisor启动当前program时会实例化好几个子进程，默认情况取1
numprocs=1
# supervisord作为守护进程的时候，会转换路径到该目录
directory=/suhw/data/redis
# supervisord进程的文件权限掩码，默认 022
umask=022
# 进程启动优先级，默认999，值小的优先启动
priority=999

# supervisor启动时是否跟随同时启动(默认为true)
autostart=true
# 启动1秒后没有异常退出，就表示进程正常启动了，默认为1秒
startsecs=2
# 程序退出后自动重启,可选值：[unexpected,true,false]，默认为unexpected，表示进程意外杀死后才重启
autorestart=true
# 启动失败自动重试次数，默认是3
startretries=100

# 脚本运行的用户身份 
# user=csmp
# 把stderr重定向到stdout，默认 false
redirect_stderr=true
# 日志输出
stdout_logfile=/suhw/data/redis/logs/%(program_name)s-stdout.log
# stdout日志文件大小，默认 50MB
stdout_logfile_maxbytes=10MB
# stdout 日志文件备份数
stdout_logfile_backups=10
# 当进程处于stdout capture mode模式的时候，写入capture FIFO的最大字节数限制，默认为0，此时认为stdout capture mode模式关闭；
stdout_capture_maxbytes=10MB

```

## 重新载入配置

```bash
# 重新加载配置并根据需要添加/删除，并将重新启动受影响的程序
[root@suhw ~]# supervisorctl update
# 重新载入 supervisord
[root@suhw ~]# supervisorctl reload
```

## 查看状态

```bash
[root@suhw ~]# supervisorctl status
redis                            RUNNING   pid 19573, uptime 0:00:18
```

此时`redis`服务已经被`supervisor`所接管了，我们以后就可通过`supervisorctl`进行服务的操作

---

# `supervisorctl`操作

`supervisorctl` 是 `supervisord` 的命令行客户端工具，直接输入`supervisorctl`可进入交互界面，也可直接输入`shell`命令操作。

```bash
[root@suhw ~]# supervisorctl
redis                            RUNNING   pid 19573, uptime 0:20:06
supervisor> help

default commands (type help <topic>):
=====================================
add    exit      open  reload  restart   start   tail
avail  fg        pid   remove  shutdown  status  update
clear  maintail  quit  reread  signal    stop    version

```

`supervisorctl`可选参数如上，可以在交互模式中输入 `help xxx`查看对应命令的作用，举例如下

```bash
supervisor> help reload
reload          Restart the remote supervisord.
```

