本文主要介绍CDH5.9中的各个服务集成LDAP，Sentry等的配置。

# 1.Hive集成LDAP
## 配置：在CDH->hive->配置->高级 中找到下面这项：
`hive-site.xml 的 HiveServer2 高级配置代码段（安全阀）`

添加如下配置：

```xml
<property>
  <name>hive.server2.authentication</name>
  <value>LDAP</value>
</property>
<property>
  <name>hive.server2.authentication.ldap.url</name>
  <value>ldap://sunmvm20</value>
</property>
<property>
  <name>hive.server2.authentication.ldap.baseDN</name>
  <value>ou=people,dc=0hkj,dc=com</value>
</property>
```
重启Hive。
## 验证
使用 beeline 验证：
```shell
beeline -u "jdbc:hive2://sunmvm26:10000/default" -n test -p test
```
# 2.Impala集成LDAP
## 配置：CDH->impala->配置  做如下配置
* 启用LDAP身份验证 = true
* LDAP URL = ldap://sunmvm20
* 启用LDAP TLS = true
* LDAP BaseDN = ou=people,dc=0hkj,dc=com
重启impala。

## 验证
```shell
impala-shell -l -u hive --auth_creds_ok_in_clear

beeline -u "jdbc:hive2://sunmvm28:21050/default;" -n test -p test
```
# 3.HUE集成LDAP
在Cloudera Mnager 中修改 HUE 配置，使用搜索绑定进行认证:

```shell
backend = desktop.auth.backend.LdapBackend
ldap_url = ldap://sunmvm20

# 使用搜索绑定认证
Use Search Bind Authentication = true
LDAP 搜索基础base_dn:  dc=ohkj,dc=com
LDAP 绑定用户可分辨名称bind_dn:  uid=ldapadmin,ou=people,dc=ohkj,dc=com
LDAP 绑定密码bind_password = ${BIND_PASSWORD}
LDAP User Filter = (objectClass=posixAccount)
LDAP user_name_attr = uid
LDAP group_filter = (objectClass=posixGroup)
LDAP group_name_attr = cn
LDAP group_member_attr = memberUID
```

# Hive集成Sentry

# Impala集成Sentry

# HUE集成Sentry

# 用户、用户组映射配置（HUE，HDFS，Sentry）
!(image/hdfs-user-group-mapping.png)

# HUE本地调试环境搭建
