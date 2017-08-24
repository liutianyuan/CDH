
转载自JavaChen Blog，作者：JavaChen
原文链接地址：http://blog.javachen.com/2014/11/12/config-ldap-with-kerberos-in-cdh-hadoop.html


CDH集成LDAP配置

本文主要记录 cdh hadoop 集群集成 ldap 的过程，这里 ldap 安装的是 OpenLDAP 。LDAP 用来做账号管理，Kerberos作为认证。CDH集群已经开启Kerberos认证。

# 一、环境说明
## 软件版本
操作系统：CentOs 6.8
CDH版本：Hadoop 2.6.0-cdh5.9.0
JDK版本：jdk1.7.0_67-cloudera
OpenLDAP 版本：2.4.39
Kerberos 版本：1.10.3
运行用户：root

## 集群主机角色划分
1. sunmvm20 作为master节点，安装kerberos Server
2. 其他节点作为slave节点，安装kerberos client

# 二、安装服务端

## 1.安装

同安装 kerberos 一样，这里使用 sunmvm20 作为服务端安装 openldap。

```shell
$ yum install db4 db4-utils db4-devel cyrus-sasl* krb5-server-ldap -y
$ yum install openldap openldap-servers openldap-clients openldap-devel compat-openldap -y
```
查看安装的版本：

```shell
$ rpm -qa openldap
openldap-2.4.39-8.el6.x86_64

$ rpm -qa krb5-server-ldap
krb5-server-ldap-1.10.3-33.el6.x86_64
1.2 OpenSSL
```
## 2.LDAP 服务端配置

更新配置库：
```shell
rm -rf /var/lib/ldap/*
cp /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG
chown -R ldap.ldap /var/lib/ldap
```
在2.4以前的版本中，OpenLDAP 使用 slapd.conf 配置文件来进行服务器的配置，而2.4开始则使用 slapd.d 目录保存细分后的各种配置，这一点需要注意，其数据存储位置即目录 /etc/openldap/slapd.d 。尽管该系统的数据文件是透明格式的，还是建议使用 ldapadd, ldapdelete, ldapmodify 等命令来修改而不是直接编辑。

```shell
# 默认配置文件保存在 /etc/openldap/slapd.d，将其备份：
cp -rf /etc/openldap/slapd.d /etc/openldap/slapd.d.bak

# 添加一些基本配置，并引入 kerberos 和 openldap 的 schema：
$ cp /usr/share/doc/krb5-server-ldap-1.10.3/kerberos.schema /etc/openldap/schema/

$ vi /etc/openldap/slapd.conf
# 插入下面内容
include /etc/openldap/schema/corba.schema
include /etc/openldap/schema/core.schema
include /etc/openldap/schema/cosine.schema
include /etc/openldap/schema/duaconf.schema
include /etc/openldap/schema/dyngroup.schema
include /etc/openldap/schema/inetorgperson.schema
include /etc/openldap/schema/java.schema
include /etc/openldap/schema/misc.schema
include /etc/openldap/schema/nis.schema
include /etc/openldap/schema/openldap.schema
include /etc/openldap/schema/ppolicy.schema
include /etc/openldap/schema/collective.schema
include /etc/openldap/schema/kerberos.schema

pidfile /var/run/openldap/slapd.pid
argsfile /var/run/openldap/slapd.args

# 更新slapd.d
$ slaptest -f /etc/openldap/slapd.conf -F /etc/openldap/slapd.d

$ chown -R ldap:ldap /etc/openldap/slapd.d && chmod -R 700 /etc/openldap/slapd.d
```
## 3.启动服务

启动 LDAP 服务：
```shell
chkconfig --add slapd
chkconfig --level 345 slapd on

/etc/init.d/slapd start
```
查看状态，验证服务端口：
```shell
$ ps aux | grep slapd | grep -v grep
  ldap      9225  0.0  0.2 581188 44576 ?        Ssl  15:13   0:00 /usr/sbin/slapd -h ldap:/// -u ldap

$ netstat -tunlp  | grep :389
  tcp        0      0 0.0.0.0:389                 0.0.0.0:*                   LISTEN      8510/slapd
  tcp        0      0 :::389                      :::*                        LISTEN      8510/slapd

# 如果启动失败，则运行下面命令来启动 slapd 服务并查看日志：
$ slapd -h ldap://127.0.0.1 -d 481
```
待查明原因之后，停止该进程使用正常方式启动 slapd 服务。

## 4.LDAP 和 Kerberos

在Kerberos安全机制里，一个principal就是realm里的一个对象，一个principal总是和一个密钥（secret key）成对出现的。

这个principal的对应物可以是service，可以是host，也可以是user，对于Kerberos来说，都没有区别。

Kdc(Key distribute center)知道所有principal的secret key，但每个principal对应的对象只知道自己的那个secret key。这也是 "共享密钥" 的由来。

为了使 Kerberos 能够绑定到 OpenLDAP 服务器，请创建一个管理员用户和一个 principal，并生成 keytab 文件，设置该文件的权限为 LDAP 服务运行用户可读（ LDAP 服务运行用户一般为 ldap）：

```shell
$ kadmin.local -q "addprinc ldapadmin@JAVACHEN.COM"
$ kadmin.local -q "addprinc -randkey ldap/cdh1@JAVACHEN.COM"
$ kadmin.local -q "ktadd -k /etc/openldap/ldap.keytab ldap/cdh1@JAVACHEN.COM"
# ktadd 后面的-k 指定把 key 存放在一个本地文件中。

$ chown ldap:ldap /etc/openldap/ldap.keytab && chmod 640 /etc/openldap/ldap.keytab
```

使用 ldapadmin 用户测试：

```shell
kinit ldapadmin
```
系统会提示输入密码，如果一切正常，那么会安静的返回。实际上，你已经通过了kerberos的身份验证，且获得了一个Service TGT(Ticket-Granting Ticket). Service TGT的意义是， 在一段时间内，你都可以用此TGT去请求某些service，比如ldap service，而不需要再次通过kerberos的认证。

确保 LDAP 启动时使用上一步中创建的keytab文件，在 /etc/sysconfig/ldap 增加 KRB5_KTNAME 配置：
```shell
export KRB5_KTNAME=/etc/openldap/ldap.keytab
```
然后，重启 slapd 服务。

## 5.创建数据库

进入到 /etc/openldap/slapd.d 目录，查看 etc/openldap/slapd.d/cn\=config/olcDatabase={2}bdb.ldif 可以看到一些默认的配置，例如：

```shell
olcRootDN: cn=Manager,dc=my-domain,dc=com  
olcRootPW: secret  
olcSuffix: dc=my-domain,dc=com
```
接下来更新这三个配置，建立 modify.ldif 文件，内容如下：

```shell
dn: olcDatabase={2}bdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=0hkj,dc=com

dn: olcDatabase={2}bdb,cn=config
changetype: modify
replace: olcRootDN
# Temporary lines to allow initial setup
olcRootDN: uid=ldapadmin,ou=people,dc=0hkj,dc=com

dn: olcDatabase={2}bdb,cn=config
changetype: modify
add: olcRootPW
olcRootPW: secret

dn: cn=config
changetype: modify
add: olcAuthzRegexp
olcAuthzRegexp: uid=([^,]*),cn=GSSAPI,cn=auth uid=$1,ou=people,dc=0hkj,dc=com

dn: olcDatabase={2}bdb,cn=config
changetype: modify
add: olcAccess
# Everyone can read everything
olcAccess: {0}to dn.base="" by * read
# The ldapadm dn has full write access
olcAccess: {1}to * by dn="uid=ldapadmin,ou=people,dc=0hkj,dc=com" write by * read
```
说明：

* 上面的密码使用的是明文密码 secret ，你也可以使用 slappasswd -s secret 生成的字符串作为密码。
* 上面的权限中指明了只有用户 uid=ldapadmin,ou=people,dc=0hkj,dc=com 有写权限。
使用下面命令导入更新配置：

```shell
$ ldapmodify -Y EXTERNAL -H ldapi:/// -f modify.ldif
```
这时候数据库没有数据，需要添加数据，你可以手动编写 ldif 文件来导入一些用户和组，或者使用 migrationtools 工具来生成 ldif 模板。创建 setup.ldif 文件如下：

```shell
dn: dc=0hkj,dc=com
objectClass: top
objectClass: dcObject
objectclass: organization
o: javachen com
dc: javachen

dn: ou=people,dc=0hkj,dc=com
objectclass: organizationalUnit
ou: people
description: Users

dn: ou=group,dc=0hkj,dc=com
objectClass: organizationalUnit
ou: group

dn: uid=ldapadmin,ou=people,dc=0hkj,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: LDAP admin account
uid: ldapadmin
sn: ldapadmin
uidNumber: 1001
gidNumber: 100
homeDirectory: /home/ldap
loginShell: /bin/bash

# 使用下面命令导入数据，密码是前面设置的 secret 。
$ ldapadd -x -D "uid=ldapadmin,ou=people,dc=0hkj,dc=com" -w secret -f setup.ldif
```
参数说明：

* -w 指定密码
* -x 是使用一个匿名的绑定

## 6.LDAP 的使用

接下来你可以从 /etc/passwd, /etc/shadow, /etc/groups 中生成 ldif 更新 ldap 数据库，这需要用到 migrationtools 工具。

* 配置 migrationtools 工具
```shell
# 安装：
$ yum install migrationtools -y

# 利用迁移工具生成模板，先修改默认的配置：
$ vim /usr/share/migrationtools/migrate_common.ph

# line 71 defalut DNS domain
$DEFAULT_MAIL_DOMAIN = "0hkj.com";

# line 74 defalut base
$DEFAULT_BASE = "dc=0hkj,dc=com";

# 生成模板文件：
/usr/share/migrationtools/migrate_base.pl > /opt/base.ldif

# 然后，可以修改该文件，然后执行导入命令：
$ ldapadd -x -D "uid=ldapadmin,ou=people,dc=0hkj,dc=com" -w secret -f /opt/base.ldif
```

* 添加
```shell
# 先添加用户
$ useradd test hive
# 查找系统上的 test、hive 等用户
$ grep -E "test|hive" /etc/passwd  >/opt/passwd.txt
$ /usr/share/migrationtools/migrate_passwd.pl /opt/passwd.txt /opt/passwd.ldif
$ ldapadd -x -D "uid=ldapadmin,ou=people,dc=javachen,dc=com" -w secret -f /opt/passwd.ldif

# 将用户组导入到 ldap 中：

# 生成用户组的 ldif 文件，然后导入到 ldap
$ grep -E "test|hive" /etc/group  >/opt/group.txt
$ /usr/share/migrationtools/migrate_group.pl /opt/group.txt /opt/group.ldif
$ ldapadd -x -D "uid=ldapadmin,ou=people,dc=javachen,dc=com" -w secret -f /opt/group.ldif
```

* 查询
```shell
# 查询新添加的 test 用户：
$ ldapsearch -LLL -x -D 'uid=ldapadmin,ou=people,dc=javachen,dc=com' -w secret -b 'dc=javachen,dc=com' 'uid=test'
  dn: uid=test,ou=people,dc=javachen,dc=com
  objectClass: inetOrgPerson
  objectClass: posixAccount
  objectClass: shadowAccount
  cn: test account
  sn: test
  uid: test
  uidNumber: 1001
  gidNumber: 100
  homeDirectory: /home/test
  loginShell: /bin/bash
```
             
* 修改
```shell
# 用户添加好以后，需要给其设定初始密码，运行命令如下：
$ ldappasswd -x -D 'uid=ldapadmin,ou=people,dc=javachen,dc=com' -w secret "uid=test,ou=people,dc=javachen,dc=com" -S
```

* 删除
```shell
# 删除用户或组条目：
$ ldapdelete -x -w secret -D 'uid=ldapadmin,ou=people,dc=javachen,dc=com' "uid=test,ou=people,dc=javachen,dc=com"
$ ldapdelete -x -w secret -D 'uid=ldapadmin,ou=people,dc=javachen,dc=com' "cn=test,ou=group,dc=javachen,dc=com"
```

# 三、客户端配置

```shell
# 在 cdh2 和 cdh3上，使用下面命令安装openldap客户端
$ yum install openldap-clients -y

# 修改 /etc/openldap/ldap.conf 以下两个配置

BASE    dc=javachen,dc=com
URI     ldap://cdh1

# 然后，运行下面命令测试：

# 先删除 ticket
$ kdestroy

$ ldapsearch -b 'dc=0hkj,dc=com'
  SASL/GSSAPI authentication started
  ldap_sasl_interactive_bind_s: Local error (-2)
    additional info: SASL(-1): generic failure: GSSAPI Error: Unspecified GSS failure.  Minor code may provide more information (No credentials cache found)
重新获取 ticket：

$ kinit root/admin
$ ldapsearch -b 'dc=0hkj,dc=com'
# 没有报错了

$ ldapwhoami
  SASL/GSSAPI authentication started
  SASL username: root/admin@0HKJ.COM
  SASL SSF: 56
  SASL installing layers
  dn:uid=root/admin,ou=people,dc=0hkj,dc=com
  Result: Success (0)

# 直接输入 ldapsearch 不会报错
$ ldapsearch  
```
