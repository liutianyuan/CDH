# sqoop 报错：Could not get connectors
拷贝sqoop.properties 文件至 /etc/sqoop2/conf 目录


# sqoop 报错：Could not start job.
* 报错：
Could not start job.
exception    ERROR    {"message":null,"error-code-class":"org.apache.sqoop.error.code.GenericJdbcConnectorError","stack-trace":[{"file":"GenericJdbcFromInitializer.java","line":147,"class":"org.apache.sqoop.connector.jdbc.GenericJdbcFromInitializer"

* 解决办法：
```shell
cd /opt/cloudera/parcels/CDH/lib/hadoop/client
sudo ln -s ../../hadoop-hdfs/lib/jackson-mapper-asl-1.8.8.jar .
sudo ln -s ../../hadoop-hdfs/lib/jackson-core-asl-1.8.8.jar .
```


# log4j
Thu Apr 20 10:29:05 CST 2017
JAVA_HOME=/usr/java/jdk1.7.0_67-cloudera
using 5 as CDH_VERSION
CONF_DIR=/run/cloudera-scm-agent/process/238-sqoop-SQOOP_SERVER
CMF_CONF_DIR=/etc/cloudera-scm-agent
log4j:ERROR A "org.apache.log4j.RollingFileAppender" object is not assignable to a "org.apache.log4j.Appender" variable.
log4j:ERROR The class "org.apache.log4j.Appender" was loaded by 
log4j:ERROR [org.apache.catalina.loader.StandardClassLoader@52437b9a] whereas object of type
log4j:ERROR "org.apache.log4j.RollingFileAppender" was loaded by [WebappClassLoader
  context: /sqoop
  delegate: false
  repositories:
    /WEB-INF/classes/
----------> Parent Classloader:
org.apache.catalina.loader.StandardClassLoader@52437b9a
].
log4j:ERROR Could not instantiate appender named "RFA".
log4j:WARN No appenders could be found for logger (org.apache.hadoop.util.NativeCodeLoader).
log4j:WARN Please initialize the log4j system properly.
log4j:WARN See [url]http://logging.apache.org/log4j/1.2/faq.html#noconfig[/url] for more info.

* 解决
这个问题，是由于log的jar包冲突导致。可以直接删除log的jar包。
cd /opt/cloudera/parcels/CDH-5.10.0-1.cdh5.10.0.p0.41/lib/sqoop2/webapps/sqoop/WEB-INF/lib
mv log4j-1.2.16.jar log4j-1.2.16.jar.bak


刚开始我修改的/var/lib/sqoop2/tomcat-deployment/webapps/sqoop 目录下的pom文件和jar包，发现每次重启sqoop2，这个目录下的文件都会被覆盖，后来，通过 find / -name sqoop.war命令查找到/opt/cloudera/parcels/CDH-5.10.0-1.cdh5.10.0.p0.41/lib/sqoop2/ 这个目录下也存在，测试修改了下这个目录下的文件，生效了。OK，搞定。


# sqoop新建数据源报错：Could not create link.
* {"message":null,"cause":{"message":"The statement was aborted because it would have caused a duplicate key value in a unique or primary key constraint or unique index identified by 'FK_SQ_LNK_NAME_UNIQUE' defined on 'SQ_LINK'.","cause":{"message":"The statement was aborted because it would have caused a duplicate key value in a unique or primary key constraint or unique index identified by 'FK_SQ_LNK_NAME_UNIQUE' defined on 'SQ_LINK'."

*解决

sudo -u postgres psql

postgres=# CREATE ROLE sqoop LOGIN ENCRYPTED PASSWORD 'sqoop' NOSUPERUSER INHERIT CREATEDB NOCREATEROLE;

postgres=# CREATE DATABASE "sqoop" WITH OWNER = sqoop
 ENCODING = 'UTF-8'
 TABLESPACE = pg_default
 LC_COLLATE = 'en_US.UTF-8'
 LC_CTYPE = 'en_US.UTF-8'
 CONNECTION LIMIT = -1;
 
 1. 修改postgresql.conf

postgresql.conf存放位置在/etc/postgresql/9.x/main下，这里的x取决于你安装PostgreSQL的版本号，编辑或添加下面一行，使PostgreSQL可以接受来自任意IP的连接请求。

listen_addresses = '*'
2. 修改pg_hba.conf

pg_hba.conf，位置与postgresql.conf相同，虽然上面配置允许任意地址连接PostgreSQL，但是这在pg中还不够，我们还需在pg_hba.conf中配置服务端允许的认证方式。任意编辑器打开该文件，编辑或添加下面一行。

# TYPE  DATABASE  USER  CIDR-ADDRESS  METHOD
host  all  all 0.0.0.0/0 md5

service postgresql restart


https://www.cloudera.com/documentation/enterprise/5-9-x/topics/cm_ig_extrnl_pstgrs.html#cmig_topic_5_6
https://www.cloudera.com/documentation/enterprise/5-9-x/topics/cdh_ig_sqoop2_configure.html#topic_14_3
