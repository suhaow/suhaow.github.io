# 参考

1. [bilibili马哥视频](https://www.bilibili.com/video/BV1HZ4y1p7Bf?from=search&amp;seid=3229559529999977012)
2. [运维派教程](http://www.yunweipai.com/ansible%e6%95%99%e7%a8%8b )

---

# 介绍

roles就是通过分别将变量、文件、任务、模板及处理器放置于单独的目录中，并可以便捷地`include`它们的一种机制

---

# 目录组织

```
├──playbook.yml
├── roles
│   ├── project
│       ├── default
│       ├── files
│       ├── handlers
│       ├── meta
│       ├── tasks
│       ├── templates
│       └── vars

```

使用roles的`playbook`需要与`roles`文件目录平级，`roles`中目录作用如下

- files/ ：存放由`copy`或`script`模块等调用的文件

- templates/：`template`模块查找所需要模板文件的目录

- tasks/：定义`task,role`的基本元素，至少应该包含一个名为`main.yml`的文件；其它的文件需要在此文件中通过`include`进行包含

- handlers/：至少应该包含一个名为`main.yml`的文件；其它的文件需要在此文件中通过include进行包含

- vars/：定义变量，至少应该包含一个名为`main.yml`的文件；其它的文件需要在此文件中通过include进行包含

- meta/：定义当前角色的特殊设定及其依赖关系,至少应该包含一个名为`main.yml`的文件，其它文件需在此文件中通过include进行包含

- default/：设定默认变量时使用此目录中的`main.yml`文件，比vars的优先级低

---

# 示例1：实现httpd角色

## 目的

当引入`httpd`角色时，可以达到以下几个目的

1. 为远程主机安装`httpd`并设置为开机自启动
2. 将首页文件以及`httpd`配置文件拷贝到对应位置
3. 重启服务生效

## 目录结构

```
.
├── role-install-httpd.yml
├── roles
│   └── httpd
│       ├── files
│       │   ├── httpd.conf
│       │   └── index.html
│       ├── handlers
│       │   └── main.yml
│       └── tasks
│           ├── config.yml
│           ├── enable.yml
│           ├── index.yml
│           ├── install.yml
│           └── main.yml
```

## `playbbok`内容

```yaml
# cat role-install-httpd.yml
- hosts: vm
  gather_facts: no
  
  roles:
   - httpd
```

## `tasks`内容

```yaml
# cat roles/httpd/tasks/install.yml 
- name: install httpd package
  yum: name=httpd 
  
# cat roles/httpd/tasks/config.yml 
- name: config file
  copy: src=httpd.conf dest=/etc/httpd/conf/ backup=yes
  notify: restart


# cat roles/httpd/tasks/index.yml 
- name: copy index
  copy: src=index.html dest=/var/www/html/


# cat roles/httpd/tasks/enable.yml 
- name: start service
  service: name=httpd state=started enabled=yes
```

## `handlers`内容

```yaml
# cat roles/httpd/handlers/main.yml 
- name: restart
  service: name=httpd state=restarted
```

<br/>

# 示例2：实现nginx角色

## 目的

在上一个例子的基础上增加`templates`以及`vars`。

1. 配置文件由模板文件自动生成
2. `nginx`由指定的远程主机用户`suhw`启动

## 目录结构

```
├── role-install-nginx.yml
├── roles
│   └── nginx
│       ├── handlers
│       │   └── main.yml
│       ├── tasks
│       │   ├── config.yml
│       │   ├── enable.yml
│       │   ├── index.yml
│       │   ├── install.yml
│       │   └── main.yml
│       ├── templates
│       │   └── nginx.conf.j2
│       └── vars
│           └── main.yml

```

## `playbook`内容

```yaml
# cat role-install-nginx.yml 
- hosts: vm
  roles:
  - nginx 
```

## tasks内容

```yaml
# cat tasks/main.yml
- include: install.yml
- include: config.yml
- include: index.yml
- include: enable.yml

# cat tasks/install.yml
- name: install
  yum: name=nginx

# cat tasks/config.yml
- name: config nginx
  template: src=nginx.conf.j2 dest=/etc/nginx/nginx.conf
  notify: restart
  
  # cat tasks/index.yml
- name: copy index
  copy: src=roles/httpd/files/index.html dest=/usr/share/nginx/html
  
# cat tasks/enable.yml
- name: enable and start service
  service: name=nginx state=started enabled=yes
```

## templates内容

```nginx
...
user {{ user }};
worker_processes {{ ansible_processor_vcpus+1 }};
...
```

## handlers内容

```yaml
# cat handlers/main.yml 
- name: restart
  service: name=nginx state=restarted

```

## vars内容

```yaml
cat vars/main.yml
user: suhw
```

