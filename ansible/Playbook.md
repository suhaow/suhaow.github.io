# 参考

1. [bilibili马哥视频](https://www.bilibili.com/video/BV1HZ4y1p7Bf?from=search&amp;seid=3229559529999977012)
2. [运维派教程](http://www.yunweipai.com/ansible%e6%95%99%e7%a8%8b )

---

# 介绍

在`Playbook`中，使用`task`来定义一个子任务，并在任务中指定需要使用的模块，以及触发条件等信息。最终一个`Playbook`的`Yml`文件中可由多个`task`组成，按照任务顺序对指定主机进行自动化运维

---

# YAML语言

## 教程

https://www.runoob.com/w3cnote/yaml-intro.html

---

# 核心元素

- Hosts   执行的远程主机列表
- remote_user：指定执行任务时在远程主机上所用的用户
- Tasks   任务集
- Variables 内置变量或自定义变量在playbook中调用
- Templates  模板，可替换模板文件中的变量并实现一些简单逻辑的文件
- Handlers  和 notify 结合使用，由特定条件触发的操作，满足条件方才执行，否则不执行
- tags 标签   指定某条任务执行，用于选择运行playbook中的部分代码。ansible具有幂等性，因此会自动跳过没有变化的部分，即便如此，有些代码为测试其确实没有发生变化的时间依然会非常地长。此时，如果确信其没有变化，就可以通过tags跳过此些代码片断

**注：hosts用于指定要执行指定任务的主机，须事先定义在主机清单中**

---

# playbook命令

## 格式

```bash
ansible-playbook <filename.yml> ... [options]
```

## 常见选项

```bash
-C --check          #只检测可能会发生的改变，但不真正执行操作
--list-hosts        #列出运行任务的主机
--list-tags         #列出tag
--list-tasks        #列出task
--limit 主机列表      #只针对主机列表中的主机执行
-v -vv  -vvv        #显示过程
```

---

# playbook初步

开启远程主机`mysql`的远程访问权限

```yaml
[root@localhost playbook]# cat config-mysql.yml 
---
  - hosts: csmp
    gather_facts: no
    tasks:
    - name: Open mysql remote access permissions
      replace: path=/etc/my.cnf.d/server.cnf regexp='^(bind-address=127.0.0.1)' replace='#\1' backup=true
    - name: restart mysql
      service: name=mysql state=restarted

```

执行

```bash
[root@localhost playbook]# ansible-playbook config-mysql.yml -vv
```

---

# handlers和notify

## 作用

只有当关注的资源发生变化时，才会采取一定的操作。在`notify`中列出的操作成为`handler`

## 示例

改写上面的例子，只有当配置文件修改的时候才重启`mysql`服务

```yaml
---
  - hosts: csmp
    gather_facts: no
    tasks:
    - name: Open mysql remote access permissions
      replace: path=/etc/my.cnf.d/server.cnf regexp='^(bind-address=127.0.0.1)' replace='#\1' backup=true
      notify: restart mysql

    handlers:
    - name: restart mysql
      service: name=mysql state=restarted

```

---

# tags组件

## 作用

在`playbook`中，可以利用`tags`组件，为特定`task`指定标签，当在执行`playbook`时，可以只执行特定`tags`的`tasks`，而非整个`playbook`文件

## 示例

```yaml
---
  - hosts: csmp
    gather_facts: no
    tasks:
    - name: Open mysql remote access permissions
      replace: path=/etc/my.cnf.d/server.cnf regexp='^(bind-address=127.0.0.1)' replace='#\1' backup=true
      tags: replace
    - name: restart mysql
      service: name=mysql state=restarted
      tags: effective
```

此时就可只通过`tags`执行其中某一个`task`

```bash
[root@localhost playbook]# ansible-playbook config-mysql.yml --list-tags

playbook: config-mysql.yml

  play #1 (csmp): csmp	TAGS: []
      TASK TAGS: [effective, replace]
      
#  -t TAGS, --tags TAGS  only run plays and tasks tagged with these values
[root@localhost playbook]# ansible-playbook -t effective config-mysql.yml
```

---

# ansible中使用变量

## 定义

仅能由字母、数字和下划线组成，且只能以字母开头

```
variable=value
```

## 调用

通过{{ variable_name }} 调用变量，且变量名前后建议加空格，有时用“{{ variable_name }}”才生效

## 使用`setup`模块中变量

要求在`playbook`中使用，不要用`ansible`命令调用

```yaml
---
#var.yml
- hosts: all
  remote_user: root
  gather_facts: yes

  tasks:
    - name: create log file
      file: name=/data/{{ ansible_nodename }}.log state=touch owner=wang mode=600

ansible-playbook  var.yml
```

## 在playbook命令行中定义变量

通过`-e`参数使用

```
  -e EXTRA_VARS, --extra-vars EXTRA_VARS
                        set additional variables as key=value or YAML/JSON, if
                        filename prepend with @
```

示例：

```yaml
---
  - hosts: "{{ server }}"
    gather_facts: no
    tasks:
     - name: Open mysql remote access permissions
       replace: path=/etc/my.cnf.d/server.cnf regexp='^(bind-address=127.0.0.1)' replace='#\1' backup=true
       notify: restart mysql

    handlers:
     - name: restart mysql
       service: name=mysql state=restarted

```

在执行时通过`-e`指定`server`变量。

```bash
[root@localhost playbook]# ansible-playbook config-mysql.yml -e server=10.47.119.156
```

## 在playbook文件中定义变量

`vars`中定义变量，在后续直接使用

```yaml
---
  - hosts: "{{ server }}"
    gather_facts: no
    vars:
     - path: /etc/my.cnf.d/server.cnf
    tasks:
     - name: Open mysql remote access permissions
       replace: path={{ path }} regexp='^(bind-address=127.0.0.1)' replace='#\1' backup=true
       notify: restart mysql

    handlers:
     - name: restart mysql
       service: name=mysql state=restarted

```

## 使用变量文件

可以在一个独立的playbook文件中定义变量，在另一个playbook文件中引用变量文件中的变量，比playbook中定义的变量优先级高

```yaml
# 定义变量文件
[root@localhost playbook]# cat config-mysql-var.yml 
---
server_conf_path: /etc/my.cnf.d/server.cnf

# playbook文件中引入使用
[root@localhost playbook]# cat config-mysql.yml 
---
  - hosts: "{{ server }}"
    gather_facts: no
    vars_files:
     - config-mysql-var.yml
    tasks:
     - name: Open mysql remote access permissions
       replace: path={{ server_conf_path }} regexp='^(bind-address=127.0.0.1)' replace='#\1' backup=true
       notify: restart mysql

    handlers:
     - name: restart mysql
       service: name=mysql state=restarted

```

## 主机清单文件中定义变量

- **主机（普通）变量**

在`inventory`主机清单文件中为**指定得主机定义变量**，以便在`playbook`中使用

```bash
[root@localhost playbook]# cat /etc/ansible/hosts 
[server]
10.47.119.156 server_conf_path=/etc/my.cnf.d/server.cnf
10.47.119.157 server_conf_path=/etc/my.cnf.d/server1.cnf

```

-  **组（公共）变量**

在inventory 主机清单文件中赋予给指定组内所有主机上的在playbook中可用的变量，如果和主机变是同名，优先级低于主机变量

```shell
[root@localhost playbook]# cat /etc/ansible/hosts 
[server]
10.47.119.156 
10.47.119.157 

[server:vars]
server_conf_path=/etc/my.cnf.d/server.cnf
```

使用时直接使用变量即可

```yaml
---
  - hosts: "{{ server }}"
    gather_facts: no
    tasks:
     - name: Open mysql remote access permissions
       replace: path={{ server_conf_path }} regexp='^(bind-address=127.0.0.1)' replace='#\1' backup=true
       notify: restart mysql

    handlers:
     - name: restart mysql
       service: name=mysql state=restarted

```

---

# template模板

## jinja2语言

官网：https://jinja.palletsprojects.com/en/2.11.x/

## template

### 功能

可以根据和参考模块文件，动态生成相类似的配置文件

### 要求

` template`文件必须存放于`templates`目录下，且命名为 .j2 结尾
` yaml/yml `文件需和`templates`目录平级，目录结构如下示例：

```
.
├── config.yml
└── templates
	└── nginx.conf.j2
```

### 示例

```shell
# tree
.
├── config-nginx.yml
└── templates
    └── nginx.conf.j2

```

设置`nginx`进程数与`ansible_processor_vcpus+1`相等

```jinja2
# cat templates/nginx.conf.j2 

...
worker_processes {{ ansible_processor_vcpus+1 }};
...
```

`playbook`内容

```yaml
# cat config-nginx.yml 
---
 - hosts: vm
   tasks: 
   - name: template config to remote hosts
     template: src=nginx.conf.j2 dest=/etc/nginx/nginx.conf
     notify: restart nginx

   handlers:
   - name: restart nginx
     service: name=nginx state=restarted

```

使用`setup`模块发现`ansible_processor_vcpus`值为4，所以模板文件中对应`nginx`子进程数应为5

```shell
# ansible vm -m setup -a 'filter=ansible_processor_vcpus'          
10.91.156.205 | SUCCESS => {
    "ansible_facts": {
        "ansible_processor_vcpus": 4, 
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false
}

```

统计`nginx`子进程数目

```shell
# ansible vm -m 'shell' -a "ps aux | grep 'nginx: worker process' "
10.91.156.205 | CHANGED | rc=0 >>
nginx    24585  0.0  0.0 105956  3444 ?        S    15:05   0:00 nginx: worker process
nginx    24586  0.0  0.0 105956  3444 ?        S    15:05   0:00 nginx: worker process
nginx    24587  0.0  0.0 105956  3444 ?        S    15:05   0:00 nginx: worker process
nginx    24588  0.0  0.0 105956  3444 ?        S    15:05   0:00 nginx: worker process
nginx    24589  0.0  0.0 105956  3448 ?        S    15:05   0:00 nginx: worker process
...
```

## 流程控制

template中可使用流程控制 for 循环和 if 条件判断，实现动态生成文件功能

### for循环

示例

`j2`文件：

```jinja2
 ...
 {% for port in listen_ports %}
    server {
        listen {{ port }};
        root /usr/share/nginx/html;

        location / {
        }
    }

    {% endfor %}
...
```

`playbook`文件中`src`指定`j2`文件，拷贝到远程主机后记得要修改名称

```yaml
---
 - hosts: vm
   gather_facts: no
   vars: 
     listen_ports:
     - 8888
     - 8889
   tasks: 
   - name: template config to remote hosts
     template: src=nginx-server.conf.j2 dest=/etc/nginx/nginx.conf backup=true
     notify: restart nginx

   handlers:
   - name: restart nginx
     service: name=nginx state=restarted

```

验证发现服务器`nginx.cnf`中多出了8888，8889两个server

```nginx
...  
    server {
        listen 8888;
        root /usr/share/nginx/html;

        location / {
        }
    }

    server {
        listen 8889;
        root /usr/share/nginx/html;

        location / {
        }
    }
....
```

若`vars`中变量是字典形式，形如

```yaml
vars:
    nginx_vhosts:
      - listen: 8080
        server_name: "web1.magedu.com"
        root: "/var/www/nginx/web1/"
      - listen: 8081
        server_name: "web2.magedu.com"
        root: "/var/www/nginx/web2/"
```

jinja2就需要使用如下方式调用

```jinja2
{% for vhost in nginx_vhosts %}
server {
   listen {{ vhost.listen }}
   server_name {{ vhost.server_name }}
   root {{ vhost.root }}  
}
{% endfor %}
```

### if 判断

在模板文件中可通过条件判断，来决定相关配置得生成信息。例如

```yaml
# cat if-config.yml
---
 - hosts: vm
   gather_facts: no
   vars:
     nginx_vhosts:
      - listen: 80
        server_name: "server1"
        root: "/var/www/nginx/web1"
      - listen: 8080
        root: "var/www/nginx/web2"

   tasks:
    - name: template config if exp
      template: src=if-config.cnf.j2 dest=/root/if-config.cnf
```

模板文件

![image-20210106171553159](https://gitee.com/suhaow/note/raw/master/img/image-20210106171553159.png)

最终生成结果

```nginx
#结果
server {
    listen 80
    server_name server1
    root /var/www/nginx/web1
}
server {
    listen 8080
    root var/www/nginx/web2
}
```

# playbook使用when

when语句，可实现条件测试。如果需要根据变量，facts或此前任务得执行结果来作为某task执行与否的前提时要用到条件测试。通过在task后添加when子句即可使用条件测试。

## 示例

如果`ansible_os_family`值为`RedHat`则创建`/root/redhat`目录，否则创建`/root/others`目录

```yaml
---
 - hosts: vm
   tasks:
   - name: mkdir redhat dir
     shell: mkdir /root/redhat
     when: ansible_os_family == "RedHat"
   - name: mkdir other dir
     shell: mkdir /root/otherOs
     when: ansible_os_family != "RedHat"

```

---

# 迭代with_items

## 作用

当有需要重复性执行的任务时，可以使用迭代机制。对迭代项的引用，固定变量名为"item"。要在`task`中使用`with_items `给定要迭代的元素列表

## 列表元素格式

- 字符串

```yaml
---
 - hosts: vm
   gather_facts: no

   tasks:
    - name: install tools
      yum: name={{ item }} state=installed
      with_items:
       - lrzsz
       - wget

```

等同于

```yaml
---
 - hosts: vm
   gather_facts: no

   tasks:
    - name: install lrzsz
      yum: name=lrzsz state=installed
    - name: install wget
      yum: name=wget state=installed
```

- 字典

```yaml
---
 - hosts: vm
   gather_facts: no

   tasks:
    - name: copy config file
      copy: src={{ item.src }} dest={{ item.dest }}
      with_items:
       - { src: /root/src1.cnf, dest: /root/dest1.txt }
       - { src: /root/src2.cnf, dest: /root/dest2.txt }

```

