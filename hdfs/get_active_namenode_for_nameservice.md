## 如何获取ha模式下的hdfs active namenode?

### 可以通过hdfs rest api，具体命令如下:

```json
curl 'http://namenode1:50070/jmx?qry=Hadoop:service=NameNode,name=NameNodeStatus'
{
  "beans" : [ {
    "name" : "Hadoop:service=NameNode,name=NameNodeStatus",
    "modelerType" : "org.apache.hadoop.hdfs.server.namenode.NameNode",
    "State" : "active",
    "SecurityEnabled" : false,
    "NNRole" : "NameNode",
    "HostAndPort" : "namenode1:8020",
    "LastHATransitionTime" : 1543838691590
  } ]
}

curl 'http://namenode2:50070/jmx?qry=Hadoop:service=NameNode,name=NameNodeStatus'
{
  "beans" : [ {
    "name" : "Hadoop:service=NameNode,name=NameNodeStatus",
    "modelerType" : "org.apache.hadoop.hdfs.server.namenode.NameNode",
    "State" : "standby",
    "SecurityEnabled" : false,
    "NNRole" : "NameNode",
    "HostAndPort" : "namenode2:8020",
    "LastHATransitionTime" : 0
  } ]
  ```

### 获取active namenode有什么用处？


例如：ha模式下的hdfs集群间传输文件
具体命令如下：
>hadoop  distcp -overwrite ${INPUT_DATA_FILE_DIR} hdfs://${active_namenode}:8020/${INPUT_DATA_FILE_DIR}


