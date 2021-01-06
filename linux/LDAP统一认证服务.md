# 目的

通过以下步骤最终可使用`ldap server`中的用户登录一台`ldap client`，并允许有`sudo`权限。平常公司中所用的域账号以及服务器账号也许就是使用如下方式，但是应该没有这么简陋，只是借机了解一波`ldap`

---

# 环境信息

```bash
[root@suhw ~]# hostnamectl 
   Static hostname: suhw
         Icon name: computer-vm
           Chassis: vm
        Machine ID: c9006a74a3674913a1e2bf3f582ec2a3
           Boot ID: 1421443538404e6aa9c26cebe889446c
    Virtualization: kvm
  Operating System: CentOS Linux 7 (Core)
       CPE OS Name: cpe:/o:centos:centos:7
            Kernel: Linux 3.10.0-957.el7.x86_64
      Architecture: x86-64

```

---

# 名词解释

`LDAP(Light Directory Access Portocol)`轻量级目录访问协议。现在许多产品都加入了对`LDAP`的支持，可以通过简单的配置与服务器做认证服务。用户只需要使用一个密码登录众多个支持`LDAP`协议的应用，由应用自己去`LDAP Server`去认证用户信息，不仅做到了用户信息的统一管理，对于应用认证也十分方便。接下来了解一些基本概念

## 目录树概念

1.  目录树：在一个目录服务系统中，整个目录信息集可以表示为一个目录信息树，树中的每个节点是一个条目。
2. 条目：每个条目就是一条记录，每个条目有自己的唯一可区别的名称（DN）。
3. 对象类：与某个实体类型对应的一组属性，对象类是可以继承的，这样父类的必须属性也会被继承下来。
4. 属性：描述条目的某个方面的信息，一个属性由一个属性类型和一个或多个属性值组成，属性有必须属性和非必须属性。

## 关键字解释

| **关键字** | **英文全称**       | **含义**                                                     |
| ---------- | ------------------ | ------------------------------------------------------------ |
| **dc**     | Domain Component   | 域名的部分，其格式是将完整的域名分成几部分，如域名为example.com变成dc=example,dc=com（一条记录的所属位置） |
| **ou**     | Organization Unit  | 组织单位，组织单位可以包含其他各种对象（包括其他组织单元），如“oa组”（一条记录的所属组织） |
| **cn**     | Common Name        | 公共名称，如“Thomas Johansson”（一条记录的名称）             |
| **dn**     | Distinguished Name | “uid=songtao.xu,ou=oa组,dc=example,dc=com”，一条记录的位置（唯一） |

---

# 服务端安装

```bash
[root@suhw ~]# yum -y install openldap-servers openldap-clients 
```

---

# 启动

## 生成密钥

记住输出结果，后续会将加密后的结果设置为初始用户的密码

```bash
[root@suhw ~]# slappasswd -s 123456
{SSHA}naG35hsGaMHojVoOoZQvjinkza4XUSBr
```

## 修改所属用户与组

```bash
[root@suhw ~]# chown ldap:ldap /var/lib/ldap/*
```

## 启动 LDAP 服务

```bash
[root@suhw ~]# systemctl start slapd
[root@suhw ~]# systemctl enable slapd
```

---

# 配置

## 添加基础模块

```bash
[root@suhw ~]# ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif 
[root@suhw ~]# ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif
[root@suhw ~]# ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
```

导入生效

```bash
[root@suhw ~]# ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif 
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
adding new entry "cn=cosine,cn=schema,cn=config"

[root@suhw ~]# ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
adding new entry "cn=nis,cn=schema,cn=config"

[root@suhw ~]# ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
adding new entry "cn=inetorgperson,cn=schema,cn=config"

```

## 配置域信息

创建`/etc/openldap/schema/changes.ldif`文件，将要替换的部分更改为自己的数据。

```
# 修改域名
dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcSuffix
# 注意修改
olcSuffix: dc=suhw,dc=com

# 修改管理员用户
dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcRootDN
# 注意修改
olcRootDN: cn=admin,dc=suhw,dc=com

# 修改管理员密码
dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcRootPW
# 替换为 slappasswd 生成后的结果
olcRootPW: {SSHA}naG35hsGaMHojVoOoZQvjinkza4XUSBr

# 修改访问权限
dn: olcDatabase={1}monitor,cn=config
changetype: modify
replace: olcAccess
olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" read by 
#修改dn.base
dn.base="cn=admin,dc=suhw,dc=com" read by * none
```

导入配置

```bash
[root@suhw ~]# ldapmodify -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/changes.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "olcDatabase={2}hdb,cn=config"

modifying entry "olcDatabase={2}hdb,cn=config"

modifying entry "olcDatabase={2}hdb,cn=config"

modifying entry "olcDatabase={1}monitor,cn=config"

```

## 创建域和组织

```bash
[root@suhw ~]# vi /etc/openldap/schema/basedomain.ldif
```

`dn`,`dc`的地方修改为自己对应的内容

```
dn: dc=suhw,dc=com
dc: suhw
objectClass: top
objectClass: domain

dn: ou=People,dc=suhw,dc=com
ou: People
objectClass: top
objectClass: organizationalUnit

dn: ou=Group,dc=suhw,dc=com
ou: Group
objectClass: top
objectClass: organizationalUnit
```

导入配置，回车后输入之前创建用户未加密之前的密码

```bash
[root@suhw ~]# ldapadd -x -D cn=admin,dc=suhw,dc=com -W -f /etc/openldap/base.ldif
Enter LDAP Password: 
adding new entry "dc=suhw,dc=com"

adding new entry "ou=People,dc=suhw,dc=com"

adding new entry "ou=Group,dc=suhw,dc=com"

```

## 查看用户列表

```bash
[root@suhw ~]# ldapsearch -x -b "ou=People,dc=suhw,dc=com" | grep dn
```

---

# 安装图形化界面

了解到的两种图像化界面，一种是`LDAP Admin`，另外一种是`PHPLDAPAdmin`。通过这两个工具，可以方便的进行用户的增删查改等操作。

## PHPLDAPAdmin安装

直接采用`docker`安装方便。首先记得放开`ldapserver`的389端口或关闭防火墙，否则登录会出错

```bash
[root@suhw openldap]# firewall-cmd --zone=public --add-port=389/tcp --permanent
success
[root@suhw openldap]# systemctl restart firewalld
```

执行`docker`安装命令

```bash
docker run -d --privileged -p 10004:80 --name myphpldapadmin \
  --env PHPLDAPADMIN_HTTPS=false --env PHPLDAPADMIN_LDAP_HOSTS=10.91.156.198  \
  --detach osixia/phpldapadmin
```

配置的`Ldap`地址：`--env PHPLDAPADMIN_LDAP_HOSTS=10.91.156.198`

配置不开启`HTTPS`：`--env PHPLDAPADMIN_HTTPS=false`（默认是true）

容器运行以后访问`http://host:10004`即可。

![image-20201112204311658](https://gitee.com/suhaow/note/raw/master/img/20201116194900.png)

---

## LDAP Admin安装

`Ldap Admin`是一个用于`LDAP`目录管理的免费`Windows LDAP`客户端和管理工具。此应用程序允许您在`LDAP`服务器上浏览，搜索，修改，创建和删除对象。它还支持更复杂的操作，例如目录复制和在远程服务器之间移动，并扩展常用编辑功能以支持特定对象类型（例如组和帐户）。

下载地址：http://www.ldapadmin.org/

安装成功后登录界面如下：

![image-20201116195648930](https://gitee.com/suhaow/note/raw/master/img/20201116195650.png)

---

# 客户端配置

## 安装

在任意一台机器上首先安装`nss-pam-ldapd `

```bash
[root@node ~]# yum install openldap-client nss-pam-ldapd -y
```

使用`authconfig`进行`ldap`认证的相关设置

```bash
[root@node ~]# authconfig --enableldap --enableldapauth --enablemkhomedir --enableforcelegacy --disablesssd --disablesssdauth --disableldaptls --enablelocauthorize --ldapserver=10.91.156.205 --ldapbasedn="dc=suhw,dc=com" --enableshadow --update
```

`--ldapserver`填写对应`ldapserver`的地址，`basedn`填写基础的域信息。

假设我在10.91.156.209上进行完成上述操作，那以后就可以通过`test01@10.91.156.209`进行登录，209机器会拿用户信息去找`ldapserver`进行验证。

## 添加用户

1. 新建`sudoers `组织

![image-20201116202922208](https://gitee.com/suhaow/note/raw/master/img/20201116202923.png)

2. 新建 cloud 组

   ![image-20201116203004949](https://gitee.com/suhaow/note/raw/master/img/20201116203007.png)

查看 group 属性

![image-20201116202546170](https://gitee.com/suhaow/note/raw/master/img/20201116202548.png)

在`LDAP server`上添加id为7958的group与之对应

```bash
[root@server ~]# groupadd -g 7958 cloud

[root@server ~]# tail /etc/group -n 1
cloud:x:7958:
```



3. 新建test用户并放置在cloud组下

![image-20201116203610946](https://gitee.com/suhaow/note/raw/master/img/20201116203613.png)

设置user相关属性

![image-20201116203948248](https://gitee.com/suhaow/note/raw/master/img/20201116203950.png)

右键用户设置密码`123456`

![image-20201116204008558](https://gitee.com/suhaow/note/raw/master/img/20201116204010.png)

保存后将`test01`用户挪至`cloud`组下，并修改用户`gid`为之前的7958

![image-20201116204507033](https://gitee.com/suhaow/note/raw/master/img/20201116204508.png)

此时通过`test01@10.91.156.205`访问`ldap client`，即可访问成功，并会在`ldap client`中新建`/home/test01`作为家目录

![image-20201116204647455](https://gitee.com/suhaow/note/raw/master/img/20201116204649.png)

注：添加用户和组时`id`注意别和`ldapserver`中的重复。

## 配置`sudo`

要向给`test01`用户`sudo`权限，可以直接通过`visudo`编辑`/etc/sudoers`配置文件，将`cloud`组加置对应位置即可

```
    106 ## Allows people in group wheel to run all commands
    107 %wheel  ALL=(ALL)       ALL
    108 %cloud  ALL=(ALL)       ALL
```

此时再去重新连接就可拥有`sudo`权限。更详细的权限控制自行搜索

---

# 参考

https://www.cnblogs.com/wilburxu/p/9174353.html

https://blog.mylitboy.com/article/operation/docker-install-openldap.html

https://www.cnblogs.com/xiaomifeng0510/p/9564688.html

https://cloud.tencent.com/developer/article/1563031

https://www.runoob.com/linux/linux-user-manage.html

