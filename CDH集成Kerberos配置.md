## 转载自 JavaChen Blog，作者：JavaChen
## 原文链接地址：http://blog.javachen.com/2014/11/04/config-kerberos-in-cdh-hdfs.html

## 转载自 小黑的博客
## 原文链接地址：http://www.xiaohei.info/2016/09/01/cdh-install-kerberos-ldap-sentry/

## 参考上面两篇基本配置，添加了部分配置

本文主要记录 CDH 集群上集成 Kerberos 的过程，包括 Kerberos 的安装和 CDH 相关配置修改说明。

# 一、安装Kerberos
## 1. 整体说明
* **软件版本**  
操作系统：CentOs 6.8  
CDH版本：Hadoop 2.6.0-cdh5.9.0  
JDK版本：jdk1.7.0_67-cloudera  
运行用户：root

* **集群主机角色划分**  
sunmvm20 作为master节点，安装kerberos Server  
其他节点作为slave节点，安装kerberos client

## 2. 配置host

添加主机名到 /etc/hosts 文件中。
```shell
$ cat /etc/hosts
127.0.0.1       localhost

192.168.1.20        sunmvm20
192.168.1.26        sunmvm26
192.168.1.27        sunmvm27
192.168.1.28        sunmvm28
```
注意：hostname 请使用小写，要不然在集成 kerberos 时会出现一些错误。

## 3. 安装 Kerberos

在 sunmvm20 上安装 krb5、krb5-server 和 krb5-client。
```shell
yum install krb5-server -y
# klist等命令找不大时执行下面安装
yum install -y krb5-server krb5-workstation pam_krb5
```
在其他节点安装 krb5-devel、krb5-workstation
```shell
$ ssh sunmvm26 "yum install krb5-devel krb5-workstation -y"
$ ssh sunmvm27 "yum install krb5-devel krb5-workstation -y"
$ ssh sunmvm28 "yum install krb5-devel krb5-workstation -y"
```
## 4. 修改配置文件

kdc 服务涉及到三个配置文件：

* /etc/krb5.conf
* /var/kerberos/krb5kdc/kdc.conf
* /var/kerberos/krb5kdc/kadm5.acl

1）编辑配置文件 /etc/krb5.conf。默认安装的文件中包含多个示例项。

```shell
[logging]
 default = FILE:/var/log/krb5libs.log
 kdc = FILE:/var/log/krb5kdc.log
 admin_server = FILE:/var/log/kadmind.log

[libdefaults]
 default_realm = 0HKJ.COM
 dns_lookup_realm = false
 dns_lookup_kdc = false
 ticket_lifetime = 24h
 renew_lifetime = 7d
 forwardable = true
 default_tgs_enctypes = aes256-cts-hmac-sha1-96
 default_tkt_enctypes = aes256-cts-hmac-sha1-96
 permitted_enctypes = aes256-cts-hmac-sha1-96
 clockskew = 120
 udp_preference_limit = 1

[realms]
0HKJ.COM = {
  kdc = sunmvm20
  admin_server = sunmvm20
 }

[domain_realm]
 .0hkj.com = 0HKJ.COM
ohkj.com = 0HKJ.COM
```
说明：

* [logging]：表示 server 端的日志的打印位置
* [libdefaults]：每种连接的默认配置，需要注意以下几个关键的小配置
    * default_realm = 0HKJ.COM：设置 Kerberos 应用程序的默认领域。如果您有多个领域，只需向 [realms] 节添加其他的语句。
    * ticket_lifetime： 表明凭证生效的时限，一般为24小时。
    * renew_lifetime： 表明凭证最长可以被延期的时限，一般为一个礼拜。当凭证过期之后，对安全认证的服务的后续访问则会失败。
    * clockskew：时钟偏差是不完全符合主机系统时钟的票据时戳的容差，超过此容差将不接受此票据。通常，将时钟扭斜设置为 300 秒（5 分钟）。这意味着从服务器的角度看，票证的时间戳与它的偏差可以是在前后 5 分钟内。
    * udp_preference_limit= 1：禁止使用 udp 可以防止一个 Hadoop 中的错误
* [realms]：列举使用的 realm。
    * kdc：代表要 kdc 的位置。格式是 机器:端口
    * admin_server：代表 admin 的位置。格式是 机器:端口
    * default_domain：代表默认的域名
* [appdefaults]：可以设定一些针对特定应用的配置，覆盖默认配置。

2）修改 /var/kerberos/krb5kdc/kdc.conf ，该文件包含 Kerberos 的配置信息。例如，KDC 的位置，Kerbero 的 admin 的realms 等。需要所有使用的 Kerberos 的机器上的配置文件都同步。这里仅列举需要的基本配置。详细介绍参考：krb5conf

```shell
[kdcdefaults]
 kdc_ports = 88
 kdc_tcp_ports = 88

[realms]
 0HKJ.COM = {
  #master_key_type = aes256-cts  acl_file = /var/kerberos/krb5kdc/kadm5.acl
  dict_file = /usr/share/dict/words
  max_renewable_life = 7d
  max_life = 1d
  admin_keytab = /var/kerberos/krb5kdc/kadm5.keytab
  supported_enctypes = aes256-cts:normal aes128-cts:normal des3-hmac-sha1:normal arcfour-hmac:normal des-hmac-sha1:normal des-cbc-md5:normal des-cbc-crc:normal
  default_principal_flags = +renewable, +forwardable
 }
``` 
说明：

* 0HKJ.COM： 是设定的 realms。名字随意。Kerberos 可以支持多个 realms，会增加复杂度。大小写敏感，一般为了识别使用全部大写。这个 realms 跟机器的 host 没有大关系。
* master_key_type：和 supported_enctypes 默认使用 aes256-cts。JAVA 使用 aes256-cts 验证方式需要安装 JCE 包，见下面的说明。为了简便，你可以不使用 aes256-cts 算法，这样就不需要安装 JCE 。
* acl_file：标注了 admin 的用户权限，需要用户自己创建。文件格式是：Kerberos_principal permissions [target_principal] [restrictions]
* supported_enctypes：支持的校验方式。
* admin_keytab：KDC 进行校验的 keytab。
关于AES-256加密：

对于使用 centos5. 6 及以上的系统，默认使用 AES-256 来加密的。这就需要集群中的所有节点上安装 JCE，如果你使用的是 JDK1.6 ，则到Java Cryptography Extension (JCE) Unlimited Strength Jurisdiction Policy Files for JDK/JRE 6 页面下载，如果是 JDK1.7，则到 Java Cryptography Extension (JCE) Unlimited Strength Jurisdiction Policy Files for JDK/JRE 7 下载。下载的文件是一个 zip 包，解开后，将里面的两个文件放到下面的目录中：$JAVA_HOME/jre/lib/security.

上面这一步一定要做，否则会报zk，namenode等不支持默认tkt的加密方式错误。

3）为了能够不直接访问 KDC 控制台而从 Kerberos 数据库添加和删除主体，请对 Kerberos 管理服务器指示允许哪些主体执行哪些操作。通过创建 /var/lib/kerberos/krb5kdc/kadm5.acl 完成此操作。

```shell
$ cat /var/kerberos/krb5kdc/kadm5.acl
```
内容如下：
```shell
*/admin@0HKJ.COM *
```
表示principal的名字的第二部分如果是admin,那么该principal就拥有管理员权限

## 5. 同步配置文件

将 kdc 中的 /etc/krb5.conf 拷贝到集群中其他服务器即可。
```shell
$ scp /etc/krb5.conf sunmvm26:/etc/krb5.conf
$ scp /etc/krb5.conf sunmvm27:/etc/krb5.conf
$ scp /etc/krb5.conf sunmvm28:/etc/krb5.conf
```
## 6. 创建数据库

在 sunmvm20 上运行初始化数据库命令。其中 -r 指定对应 realm。
```shell
$ kdb5_util create -r 0HKJ.COM -s
```
出现 Loading random data 的时候另开个终端执行点消耗CPU的命令如 cat /dev/sda > /dev/urandom 可以加快随机数采集。该命令会在 /var/kerberos/krb5kdc/ 目录下创建 principal 数据库。

如果遇到数据库已经存在的提示，可以把 /var/kerberos/krb5kdc/ 目录下的 principal 的相关文件都删除掉。默认的数据库名字都是 principal。可以使用 -d指定数据库名字。

## 7. 启动Kerberos服务

在 sunmvm20 节点上运行：
```shell
$ chkconfig --level 35 krb5kdc on
$ chkconfig --level 35 kadmin on
$ service krb5kdc start
$ service kadmin start
```
## 8. 创建 kerberos 管理员

关于 kerberos 的管理，可以使用 kadmin.local 或 kadmin，至于使用哪个，取决于账户和访问权限：

* 如果有访问 kdc 服务器的 root 权限，但是没有 kerberos admin 账户，使用 kadmin.local
* 如果没有访问 kdc 服务器的 root 权限，但是用 kerberos admin 账户，使用 kadmin
在 sunmvm20 上创建远程管理的管理员：

* 手动输入两次密码
```shell
$ kadmin.local -q "addprinc root/admin"
```
* 也可以不用手动输入密码
```shell
$ echo -e "root\nroot" | kadmin.local -q "addprinc root/admin"
```
抽取密钥并将其储存在本地 keytab 文件 /etc/krb5.keytab 中。这个文件由超级用户拥有，所以必须是 root 用户才能在 kadmin shell 中执行以下命令：
```shell
kadmin.local -q "ktadd kadmin/admin"

# 查看生成的keytab
klist -k /etc/krb5.keytab
```

## 9. 测试 kerberos

```shell
# 列出Kerberos中的所有认证用户，即principals
kadmin.local -q "list_principals"

# 添加认证用户，需要输入密码
kadmin.local -q "addprinc user1"

# 使用该用户登录，获取身份认证，需要输入密码
kinit user1

# 查看当前用户的认证信息ticket
klist

# 更新ticket
kinit -R

# 销毁当前的ticket
kdestroy

# 删除认证用户
kadmin.local -q "delprinc user1"
```

# 二、CDH启用Kerberos 

## CM中的操作
在CM的界面上点击启用Kerberos,启用的时候需要确认几个事情：

    1.KDC已经安装好并且正在运行  
    2.将KDC配置为允许renewable tickets with non-zerolifetime(在之前修改kdc.conf文件的时候已经添加了kdc_tcp_ports、max_life和max_renewable_life这个三个选项)  
    3.在Cloudera Manager Server上安装openldap-clients  
    4.为Cloudera Manager创建一个principal，使其能够有权限在KDC中创建其他的principals，就是上面创建的Kerberos管理员账号

上述确认完了之后点击continue，进入下一页进行配置，要注意的是：这里的『Kerberos Encryption Types』必须跟KDC实际支持的加密类型匹配即 /etc/krb5.conf 中的default_tgs_enctypes、default_tkt_enctypes和permitted_enctypes三个选项的值对应起来，不然会出现集群服务无法认证通过的情况。
填 aes256-cts

点击continue，进入下一页，这一页中可以不勾选『Manage krb5.conf through Cloudera Manager』注意，如果勾选了这个选项就可以通过CM的管理界面来部署krb5.conf，但是实际操作过程中发现有些配置仍然需要手动修改该文件并同步。  

点击continue，进入下一页，输入Cloudera Manager Principal的管理员账号和密码，注意输入账号的时候要使用@前要使用全称，root/admin。  

点击continue，进入下一页，导入KDC Account Manager Credentials。 

点击continue，进入下一页，restart cluster并且enable Kerberos。

之后CM会自动重启集群服务，启动之后会会提示Kerberos已启用。  

## 在CM上启用Kerberos的过程中，CM会自动做以下的事情：

1.集群中有多少个节点，每个账户都会生成对应个数的principal，格式为username/hostname@OHKJ.COM，例如hdfs/hadoop-10-0-8-124@OHKJ.COM。使用如下命令来查看：
```shell
kadmin.local -q "list_principals"
```
2.为每个对应的principal创建keytab

3.部署keytab文件到指定的节点中
    keytab是包含principals和加密principal key的文件，keytab文件对于每个host是唯一的，因为key中包含hostname，keytab文件用于不需要人工交互和保存纯文本密码，实现到kerberos上验证一个主机上的principal。启用之后访问集群的所有资源都需要使用相应的账号来访问，否则会无法通过Kerberos的authenticatin

4.在每个服务的配置文件中加入有关Kerberos的配置，其中包括Zookeeper服务所需要的jaas.conf和keytab文件都会自动设定并读取，如果用户仍然手动修改了Zookeeper的服务，要确保这两个文件的路径和内容正确性。

## 创建HDFS超级用户
此时直接用CM生成的principal访问HDFS会失败，因为那些自动生成的principal的密码是随机的，用户并不知道，而通过命令行的方式访问HDFS需要先使用kinit来登录并获得ticket，所以使用kinit hdfs/hadoop-10-0-8-124@XIAOHEI.INFO需要输入密码的时候无法继续。用户可以通过创建一个hdfs@0HKJ.COM的principal并记住密码从命令行中访问HDFS。登录之后就可以通过认证并访问HDFS,默认hdfs用户是超级用户。
```shell
kadmin.local -q "addprinc hdfs"
kinit hdfs@0HKJ.COM
```
## 为每个用户创建principal 

当集群运行Kerberos后，每一个Hadoop user都必须有一个principal或者keytab来获取Kerberos credentials（即使用密码的方式或者使用keytab验证的方式）这样才能访问集群并使用Hadoop的服务。也就是说，如果Hadoop集群存在一个名为hdfs@0HKJ.COM的principal那么在集群的每一个节点上应该存在一个名为hdfs的Linux用户。同时，在HDFS中的目录/user要存在相应的用户目录（即/user/hdfs），且该目录的owner和group都要是hdfs

一般来说，Hadoop user会对应集群中的每个服务，即一个服务对应一个user。例如impala服务对应用户impala。

至此，集群上的服务都启用了Kerberos的安全认证

# 三、验证Kerberos在集群上是否正常工作

## 1.确认HDFS可以正常使用

登录到某一个节点后，切换到hdfs用户，然后用kinit来获取credentials 

现在用`hadoop hdfs -ls /`应该能正常输出结果
用kdestroy销毁credentials后，再使用`hadoop hdfs -ls /`会发现报错

## 2.确认可以正常提交MapReduce job

获取了hdfs的证书后，提交一个PI程序，如果能正常提交并成功运行，则说明Kerberized Hadoop cluster在正常工作

```shell
hadoop jar /opt/cloudera/parcels/CDH-5.9.0-1.cdh5.9.0.p0.23/lib/hadoop-mapreduce/hadoop-mapreduce-examples-2.6.0-cdh5.9.0.jar pi 2 2
```

# 四、集群集成Kerberos过程中遇到的坑

## hdfs用户提交mr作业无法运行

INFO mapreduce.Job: Job job_1442654915965_0002 failed with state FAILED due to: Application application_1442654915965_0002 failed 2 times due to AM Container for appattempt_1442654915965_0002_000002 exited with exitCode: -1000 due to: Application application_1442654915965_0002 initialization failed (exitCode=255) with output: Requested user hdfs is not whitelisted and has id 496,which is below the minimum allowed 1000  

原因：

    Linux user 的 user id 要大于等于1000，否则会无法提交Job。例如，如果以hdfs（id为490）的身份提交一个job，就会看到以上的错误信息
    
解决方法：

    1.使用命令 usermod -u 修改一个用户的user id   
    2.修改Clouder关于这个该项的设置，Yarn->配置->min.user.id修改为合适的值，当前为0  

## 提交mr作业时可以运行但是有错误信息

INFO mapreduce.Job: Job job_1442722429197_0001 failed with state FAILED due to: Application application_1442722429197_0001 failed 2 times due to AM Container for appattempt_1442722429197_0001_000002 exited with exitCode: -1000 due to: Application application_1442722429197_0001 initialization failed (exitCode=255) with output: Requested user hdfs is banned  

原因：

    hdfs用户被禁止运行 YARN container，yarn的设置中将hdfs用户禁用了
    
解决方法：

    修改Clouder关于这个该项的设置，Yarn->配置->banned.users 将hdfs用户移除
    
## YARN job运行时无法创建缓存目录

异常信息：
main : user is hdfs
main : requested yarn user is hdfs
Can’t create directory /data/data/yarn/nm/usercache/hdfs/appcache/application_1442724165689_0005 - Permission denied

原因：
该缓存目录在集群进入Kerberos状态前就已经存在了。例如当我们还没为集群Kerberos支持的时候，就用该用户跑过YARN应用

解决方法：
在每一个NodeManager节点上删除该用户的缓存目录，对于用户hdfs，是/data/data/yarn/nm/usercache/hdfs
