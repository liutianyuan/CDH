* sqoop 报错：Could not start job.
[09/Oct/2017 16:26:38 +0800] exception    ERROR    {"message":null,"error-code-class":"org.apache.sqoop.error.code.GenericJdbcConnectorError","stack-trace":[{"file":"GenericJdbcFromInitializer.java","line":147,"class":"org.apache.sqoop.connector.jdbc.GenericJdbcFromInitializer"

解决办法：
cd /opt/cloudera/parcels/CDH/lib/hadoop/client
sudo ln -s ../../hadoop-hdfs/lib/jackson-mapper-asl-1.8.8.jar .
sudo ln -s ../../hadoop-hdfs/lib/jackson-core-asl-1.8.8.jar .
