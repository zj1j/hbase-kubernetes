# hbase-on-k8s

## 一、概述

HBase 是一个面向列式存储的分布式数据库，其设计思想来源于 Google 的 BigTable 论文。HBase 底层存储基于 HDFS 实现，集群的管理基于 ZooKeeper 实现。HBase 良好的分布式架构设计为海量数据的快速存储、随机访问提供了可能，基于数据副本机制和分区机制可以轻松实现在线扩容、缩容和数据容灾，是大数据领域中 Key-Value 数据结构存储最常用的数据库方案。


#### 软件架构
软件架构说明

![输入图片说明](https://foruda.gitee.com/images/1667641771326453604/f433e0c8_1350539.png "屏幕截图")

## 二、开始编排部署（非高可用HDFS）
地址：[https://artifacthub.io/packages/helm/hbase/hbase](https://artifacthub.io/packages/helm/hbase/hbase)
###  1）下载chart 包
```bash
helm repo add hbase https://itboy87.github.io/bigdata-charts/

# hbase version 2.4.13
helm pull hbase/hbase --version 0.1.7
```
### 2）构建镜像
在下面连接hadoop高可用会重新构建镜像，这里就不重新构建镜像了，只是把远程的包推送到本地harbor仓库

```bash
docker pull ghcr.io/fleeksoft/hbase/hbase-base:2.4.13.2

# tag
docker tag ghcr.io/fleeksoft/hbase/hbase-base:2.4.13.2 myharbor.com/bigdata/hbase-base:2.4.13.2

# push
docker push myharbor.com/bigdata/hbase-base:2.4.13.2
```
### 3）修改yaml编排（非高可用HDFS）
- `hbase/values.yaml`
```bash
image:
  repository: myharbor.com/bigdata/hbase-base
  tag: 2.4.13.2
  pullPolicy: IfNotPresent

...

conf:
  hadoopUserName: admin
  hbaseSite:
    hbase.rootdir:  "hdfs://hadoop-hadoop-hdfs-nn.hadoop:9000/hbase"
    hbase.zookeeper.quorum:  "zookeeper.zookeeper:2181"

...

hbase:
  master:
    replicas: 2

  regionServer:
    replicas: 2

# 禁用内部的hadoop
hadoop:
  enabled: false

# 禁用内部的zookeeper
zookeeper:
  enabled: false
```

- `hbase/templates/hbase-configmap.yaml`

```bash
if [ {{ .Values.hadoop.enabled }} = true ];then
  NAMENODE_URL={{- printf "http://%s-hadoop-hdfs-nn:9870/index.html" .Release.Name }}
else
  hadoop_url={{ index .Values.conf.hbaseSite "hbase.rootdir" }}
  hadoop_url=`echo $hadoop_url|awk -F '/' '{print $3}'|awk -F':' '{print $1}'`
  NAMENODE_URL=http://${hadoop_url}:9870/index.html
fi
```

### 4）开始部署

```bash
# 先检查语法
helm lint ./hbase

# 开始安装
helm install hbase ./hbase -n hbase --create-namespace
```
NOTES

```bash
NAME: hbase
LAST DEPLOYED: Sat Nov  5 15:44:14 2022
NAMESPACE: hbase
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
1. You can get an HBASE Shell by running this command:
   kubectl exec -n hbase -it hbase-hbase-master-0 -- hbase shell

2. Inspect hbase master service ports with:
   kubectl exec -n hbase describe service hbase-hbase-master

3. Create a port-forward to the hbase manager UI:
   kubectl port-forward -n hbase svc/hbase-hbase-master 16010:16010

   Then open the ui in your browser:

   open http://localhost:16010

4. Create a port-forward to the hbase thrift manager UI:
   kubectl port-forward -n hbase svc/hbase-hbase-master 9095:9095

   Then open the ui in your browser:

   open http://localhost:9095
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/a0db46a859b44d95b888c9cf885ddaa6.png)
HDFS
![在这里插入图片描述](https://img-blog.csdnimg.cn/c67afd4a00004ef199b46e6ba256d84f.png)
查看

```bash
kubectl get pods,svc -n hbase -owide
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/29bdf788c4e041029a48c1e4d20457ac.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/5b6c9e780e214f6fa5b727dbd11d9af7.png)
### 5）测试验证
测试主备切换，重启当前active master pod

```bash
kubectl delete pod hbase-hbase-master-0 -n hbase
```
主备能正常切换
![在这里插入图片描述](https://img-blog.csdnimg.cn/c569b0af76c2417cbd972bab07b65c31.png)
### 6）卸载
```bash
helm uninstall hbase -n hbase
# delete ns 
kubectl delete ns hbase --force
```
## 三、开始编排部署（高可用 HDFS）
###  1）下载chart 包
```bash
helm repo add hbase https://itboy87.github.io/bigdata-charts/

# hbase version 2.4.13
helm pull hbase/hbase --version 0.1.7
```
### 2）构建镜像
这里是基于上面的镜像进行构建，只是把hadoop打包到镜像中，主要用的hadoop配置文件是`core-site.yaml`，`hdfs-site.yaml`

Dockerfile

```bash
FROM myharbor.com/bigdata/hbase-base:2.4.13.2

RUN mkdir -p /opt/apache

ENV HADOOP_VERSION=3.3.2

ADD hadoop-${HADOOP_VERSION}.tar.gz /opt/apache

ENV HADOOP_HOME=/opt/apache/hadoop

RUN ln -s /opt/apache/hadoop-${HADOOP_VERSION} $HADOOP_HOME

ENV HADOOP_CONF_DIR=${HADOOP_HOME}/et/hadoop

ENV PATH=${HADOOP_HOME}/bin:$PATH
```

开始构建
```bash
docker build -t myharbor.com/bigdata/hbase-hdfs-ha:2.4.13.2 . --no-cache

### 参数解释
# -t：指定镜像名称
# . ：当前目录Dockerfile
# -f：指定Dockerfile路径
#  --no-cache：不缓存

# 推送到harbor
docker push myharbor.com/bigdata/hbase-hdfs-ha:2.4.13.2
```

### 3）修改配置
- `hbase-hdfs-ha/values.yaml`
```bash
image:
  repository: myharbor.com/bigdata/hbase-hdfs-ha
  tag: 2.4.13.2
  pullPolicy: IfNotPresent

...

conf:
  hadoopUserName: admin
  hbaseSite:
    hbase.rootdir: "hdfs://myhdfs/hbase"
    hbase.zookeeper.quorum: "zookeeper.zookeeper:2181"
```
- `hbase-hdfs-ha/templates/hbase-configmap.yaml`
```bash
if [ {{ .Values.hadoop.enabled }} = true ];then
  NAMENODE_URL={{- printf "http://%s-hadoop-hdfs-nn:9870/index.html" .Release.Name }}
else
  NAMENODE_URL=http://hadoop-ha-hadoop-hdfs-nn-1.hadoop-ha:9870:9870/index.html
fi
```
```bash
# 先检查语法
helm lint ./hbase-hdfs-ha

# 开始安装
helm install hbase-hdfs-ha ./hbase-hdfs-ha -n hbase-hdfs-ha --create-namespace
```
NOTES

```bash
NAME: hbase-hdfs-ha
LAST DEPLOYED: Sat Nov  5 17:23:20 2022
NAMESPACE: hbase-hdfs-ha
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
1. You can get an HBASE Shell by running this command:
   kubectl exec -n hbase-hdfs-ha -it hbase-hdfs-ha-hbase-master-0 -- hbase shell

2. Inspect hbase master service ports with:
   kubectl exec -n hbase-hdfs-ha describe service hbase-hdfs-ha-hbase-master

3. Create a port-forward to the hbase manager UI:
   kubectl port-forward -n hbase-hdfs-ha svc/hbase-hdfs-ha-hbase-master 16010:16010

   Then open the ui in your browser:

   open http://localhost:16010

4. Create a port-forward to the hbase thrift manager UI:
   kubectl port-forward -n hbase-hdfs-ha svc/hbase-hdfs-ha-hbase-master 9095:9095

   Then open the ui in your browser:

   open http://localhost:9095

```
![在这里插入图片描述](https://img-blog.csdnimg.cn/d5bf1fa3cd6d44baac6150705480dbda.png)
HDFS
![在这里插入图片描述](https://img-blog.csdnimg.cn/99b2c3008b484e1ba83f8dd657e329e6.png)
查看

```bash
kubectl get pods,svc -n hbase-hdfs-ha
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/6249bbdcd57146e2b9c55e1093acac84.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/cdfd775a5a6d455db5555ece42b2b0b0.png)

### 5）测试验证
测试主备切换，重启当前active master pod

```bash
kubectl delete pod hbase-hbase-master-0 -n hbase
```
主备能正常切换
![在这里插入图片描述](https://img-blog.csdnimg.cn/c40d30898b8e4f9b98c773f8cd0dbade.png)
### 6）卸载
```bash
helm uninstall hbase-hdfs-ha -n hbase-hdfs-ha
# delete ns 
kubectl delete ns hbase-hdfs-ha --force
```