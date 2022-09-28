---
title:基于docker构建hadoop平台
data:2022-09-26 10:55:51
author:CHENSHONE
tags:[hadoop,hbase,spark,docker]
categories:[环境]
---

# 基于docker构建hadoop平台

## 准备

### 构建虚拟网络

```sh
docker network create --driver=bridge hadoop
```

查看是否创建成功

```sh
docker network ls
NETWORK ID     NAME      DRIVER    SCOPE
87bf5f082af9   bridge    bridge    local
65e355efabcd   hadoop    bridge    local
5e7b0f6300d1   host      host      local
ec6c87608c88   none      null      local
```

看到列表中有hadoop即代表创建虚拟网络成功

### 安装ubuntu

#### 拉取镜像

```sh
docker pull ubuntu:16.04
```

查看是否拉取成功

```sh
❯ docker image ls
REPOSITORY   TAG       IMAGE ID       CREATED         SIZE
ubuntu       16.04     b6f507652425   13 months ago   135MB
```

看到列表中有ubuntu即代表拉取成功

#### 构建容器

```sh
docker run -itd ubuntu:16.04
```

查看是否构建成功

```sh
docker ps -a
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS                   PORTS     NAMES
1cfae773d0b7   ubuntu:16.04   "/bin/bash"              4 seconds ago   Up 3 seconds                       hardcore_nobel
```

可以看到容器已经构建且处于运行状态

### 配置环境

首先进入刚刚创建的ubuntu容器

```sh
docker exec -it 1cfa /bin/bash
```

#### 修改apt源

因为众所周知的原因，需要将apt源替换成国内镜像

首先先将原来的源备份一下

```bash
cp /etc/apt/sources.list /etc/apt/sources.list.bak
```

删除源

```bash
rm /etc/apt/sources.list
```

写入镜像源

```bash
echo "deb http://mirrors.aliyun.com/ubuntu/ xenial main
> deb-src http://mirrors.aliyun.com/ubuntu/ xenial main
>
> deb http://mirrors.aliyun.com/ubuntu/ xenial-updates main
> deb-src http://mirrors.aliyun.com/ubuntu/ xenial-updates main
>
> deb http://mirrors.aliyun.com/ubuntu/ xenial universe
> deb-src http://mirrors.aliyun.com/ubuntu/ xenial universe
> deb http://mirrors.aliyun.com/ubuntu/ xenial-updates universe
> deb-src http://mirrors.aliyun.com/ubuntu/ xenial-updates universe
>
> deb http://mirrors.aliyun.com/ubuntu/ xenial-security main
> deb-src http://mirrors.aliyun.com/ubuntu/ xenial-security main
> deb http://mirrors.aliyun.com/ubuntu/ xenial-security universe
> deb-src http://mirrors.aliyun.com/ubuntu/ xenial-security universe" > /etc/apt/sources.list
```

要写入的内容如下，复制粘贴即可

```
deb http://mirrors.aliyun.com/ubuntu/ xenial main
deb-src http://mirrors.aliyun.com/ubuntu/ xenial main

deb http://mirrors.aliyun.com/ubuntu/ xenial-updates main
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-updates main

deb http://mirrors.aliyun.com/ubuntu/ xenial universe
deb-src http://mirrors.aliyun.com/ubuntu/ xenial universe
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates universe
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-updates universe

deb http://mirrors.aliyun.com/ubuntu/ xenial-security main
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-security main
deb http://mirrors.aliyun.com/ubuntu/ xenial-security universe
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-security universe
```

最后，输入`apt update`来更新一下即可

#### 安装jdk与scala

##### jdk

```bash
apt install openjdk-8-jdk
```

查看是否安装成功

```bash
root@1cfae773d0b7:/# java -version
openjdk version "1.8.0_292"
OpenJDK Runtime Environment (build 1.8.0_292-8u292-b10-0ubuntu1~16.04.1-b10)
OpenJDK 64-Bit Server VM (build 25.292-b10, mixed mode)
```

##### scala

```bash
apt install scala
```

查看是否安装成功

```bash
root@1cfae773d0b7:/# scala
Welcome to Scala version 2.11.6 (OpenJDK 64-Bit Server VM, Java 1.8.0_292).
Type in expressions to have them evaluated.
Type :help for more information.

scala>
```

输入`:quit`即可退出scala交互

#### vim

```bash
apt install vim
```

#### net-tools

```bash
apt install net-tools
```

#### ssh

安装ssh服务端与客户端

```bash
apt-get install openssh-server
```

```bash
apt-get install openssh-client
```

启动ssh服务

```bash
root@1cfae773d0b7:~# service ssh start
 * Starting OpenBSD Secure Shell server sshd
   ...done.
```

将该命令写入到.bashrc中方便自启动

```bash
echo "service ssh start" >> .bashrc
```

##### 配置免密登录

1.   生成密钥

     ```bash
     ssh-keygen
     ```

     然后一路回车即可

2.   将公钥追加到 authorized_keys 文件中

     ```bash
     cat .ssh/id_rsa.pub >> .ssh/authorized_keys
     ```

3.   测试一下能否登录本机

     ```bash
     ssh 127.0.0.1
     ```

     初次登录会出现如下提示

     `Are you sure you want to continue connecting (yes/no)?`

     输入`yes`即可

     ```bash
     Welcome to Ubuntu 16.04.7 LTS (GNU/Linux 5.10.124-linuxkit x86_64)
     
      * Documentation:  https://help.ubuntu.com
      * Management:     https://landscape.canonical.com
      * Support:        https://ubuntu.com/advantage
     Last login: Mon Sep 26 04:22:06 2022 from 127.0.0.1
      * Starting OpenBSD Secure Shell server sshd
        ...done.
     root@1cfae773d0b7:~#
     ```

     出现如上信息即表示成功登录本机（127.0.0.1）

     输入`exit`即可退出登录

     ```bash
     root@1cfae773d0b7:~# exit
     logout
     Connection to 127.0.0.1 closed.
     ```

## hadoop安装与配置

首先，先进入用户根目录

```bash
cd ~
```

然后下载hadoop

```bash
wget https://mirrors.tuna.tsinghua.edu.cn/apache/hadoop/common/hadoop-3.2.3/hadoop-3.2.3.tar.gz
```

解压到`/usr/local/`目录并重命名文件夹

```bash
root@1cfae773d0b7:~#tar -zxvf hadoop-3.2.0.tar.gz -C /usr/local/
root@1cfae773d0b7:~#cd /usr/local
root@1cfae773d0b7:/usr/local# mv hadoop-3.2.3 hadoop
```

将要配置的环境变量添加到 `/etc/profile`文件中

>   使用`update-alternatives --config java`命令查看jdk的安装目录
>
>   ```bash
>   root@1cfae773d0b7:/usr/local# update-alternatives --config java
>   There is only one alternative in link group java (providing /usr/bin/java): /usr/lib/jvm/java-8-openjdk-amd64/jre/bin/java
>   Nothing to configure.
>   ```

用vim打开`/etc/profile`

```bash
vim /etc/profile
```

将如下内容写入到文件末尾

```bash
#java
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
export JRE_HOME=${JAVA_HOME}/jre    
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib    
export PATH=${JAVA_HOME}/bin:$PATH
#hadoop
export HADOOP_HOME=/usr/local/hadoop
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
export HADOOP_COMMON_HOME=$HADOOP_HOME 
export HADOOP_HDFS_HOME=$HADOOP_HOME 
export HADOOP_MAPRED_HOME=$HADOOP_HOME
export HADOOP_YARN_HOME=$HADOOP_HOME 
export HADOOP_INSTALL=$HADOOP_HOME 
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native 
export HADOOP_CONF_DIR=$HADOOP_HOME 
export HADOOP_LIBEXEC_DIR=$HADOOP_HOME/libexec 
export JAVA_LIBRARY_PATH=$HADOOP_HOME/lib/native:$JAVA_LIBRARY_PATH
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
export HDFS_DATANODE_USER=root
export HDFS_DATANODE_SECURE_USER=root
export HDFS_SECONDARYNAMENODE_USER=root
export HDFS_NAMENODE_USER=root
export YARN_RESOURCEMANAGER_USER=root
export YARN_NODEMANAGER_USER=root
```

然后执行`source /etc/profile`命令使环境变量生效

进入`/usr/local/hadoop/etc/hadoop`目录

在 hadoop-env.sh 文件末尾添加如下信息

```bash
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
export HDFS_NAMENODE_USER=root
export HDFS_DATANODE_USER=root
export HDFS_SECONDARYNAMENODE_USER=root
export YARN_RESOURCEMANAGER_USER=root
export YARN_NODEMANAGER_USER=root
```

将 core-site.xml中的configuration修改为如下内容

```xml
<configuration>
    <property>
        <name>fs.default.name</name>
        <value>hdfs://h01:9000</value>
    </property>
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/home/hadoop3/hadoop/tmp</value>
    </property>
</configuration>
```

将 hdfs-site.xml中的configuration修改为如下内容

```xml
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>2</value>
    </property>
    <property>
        <name>dfs.namenode.name.dir</name>
        <value>/home/hadoop3/hadoop/hdfs/name</value>
    </property>
    <property>
        <name>dfs.namenode.data.dir</name>
        <value>/home/hadoop3/hadoop/hdfs/data</value>
    </property>
</configuration>
```

将 mapred-site.xml中的configuration修改为如下内容

```xml
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
    <property>
        <name>mapreduce.application.classpath</name>
        <value>
            /usr/local/hadoop/etc/hadoop,
            /usr/local/hadoop/share/hadoop/common/*,
            /usr/local/hadoop/share/hadoop/common/lib/*,
            /usr/local/hadoop/share/hadoop/hdfs/*,
            /usr/local/hadoop/share/hadoop/hdfs/lib/*,
            /usr/local/hadoop/share/hadoop/mapreduce/*,
            /usr/local/hadoop/share/hadoop/mapreduce/lib/*,
            /usr/local/hadoop/share/hadoop/yarn/*,
            /usr/local/hadoop/share/hadoop/yarn/lib/*
        </value>
    </property>
</configuration>
```

将 yarn-site.xml中的configuration修改为如下内容

```xml
<configuration>
    <property>
        <name>yarn.resourcemanager.hostname</name>
        <value>h01</value>
    </property>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
</configuration>
```

将 workers中的内容修改为如下内容

```text
h01
h02
h03
h04
h05
```

如此 hadoop就配置完成了

## 使用docker配置集群

将当前容器导出为镜像

```sh
❯ docker commit -a "chenshone" -m "hadoop setting image" 1cfa hadoop_server:1.0
sha256:992587967c3a7dfe01db7034e4ed5c50d2b2350d957f090d15f8ec654a6021a0
```

查看当前有哪些镜像

```sh
❯ docker images
REPOSITORY      TAG       IMAGE ID       CREATED              SIZE
hadoop_server   1.0       992587967c3a   About a minute ago   2.29GB
ubuntu          16.04     b6f507652425   13 months ago        135MB
```

启动5个终端来配置集群

用刚刚导出的image构建5个容器，将它们都加入到最开始创建的名为`hadoop`的虚拟网络中，并将其主机名分别修改为`h01`、`h02`、`h03`、`h04`、`h05`。

其中`h01`作为master节点，需要暴露端口以供外界访问。

```sh
docker run  -it --network=hadoop -h h01 --name=h01 -p 9870:9870 -p 8088:8088 hadoop_server:1.0 /bin/bash
```

其他四个节点只需内部直接相互访问，无需向外暴露端口

```sh
❯ docker run  -it --network=hadoop -h h02 --name=h02 hadoop_server:1.0 /bin/bash

❯ docker run  -it --network=hadoop -h h03 --name=h03 hadoop_server:1.0 /bin/bash

❯ docker run  -it --network=hadoop -h h04 --name=h04 hadoop_server:1.0 /bin/bash

❯ docker run  -it --network=hadoop -h h05 --name=h05 hadoop_server:1.0 /bin/bash
```

然后在h01中启动集群

先进行格式化操作（不格式化的话，hdfs可能起不来）

```bash
root@h01:/# /usr/local/hadoop/bin/hadoop namenode -format
```

启动

```bash
root@h01:/# /usr/local/hadoop/sbin/start-all.sh
Starting namenodes on [h01]
h01: Warning: Permanently added 'h01,172.18.0.2' (ECDSA) to the list of known hosts.
Starting datanodes
h04: Warning: Permanently added 'h04,172.18.0.5' (ECDSA) to the list of known hosts.
h05: Warning: Permanently added 'h05,172.18.0.6' (ECDSA) to the list of known hosts.
h02: Warning: Permanently added 'h02,172.18.0.3' (ECDSA) to the list of known hosts.
h03: Warning: Permanently added 'h03,172.18.0.4' (ECDSA) to the list of known hosts.
h05: WARNING: /usr/local/hadoop/logs does not exist. Creating.
h04: WARNING: /usr/local/hadoop/logs does not exist. Creating.
h02: WARNING: /usr/local/hadoop/logs does not exist. Creating.
h03: WARNING: /usr/local/hadoop/logs does not exist. Creating.
Starting secondary namenodes [h01]
Starting resourcemanager
Starting nodemanagers
```

然后在浏览器中访问`localhost:8088`和`localhost:9870`即可查看监控信息

使用如下命令即可查看分布式文件系统状态

```bash
root@h01:/# cd /usr/local/hadoop/bin/
root@h01:/usr/local/hadoop/bin# ./hadoop dfsadmin -report
```

这样，hadoop集群就构建好了。

### WordCount例子

集群构建好之后，可以试着跑一下WordCount的例子。

我们可以选择 `LICENSE.txt` 作为需要统计的文件

可以先看一下该文件的大小

```bash
root@h01:/usr/local/hadoop# cat LICENSE.txt | wc
   2814   21903  150571
```

可以发现，该文件有2814行，21903个单词，数据量还是很大的。

然后将其内容写入到 `file1.txt` 中

```bash
root@h01:/usr/local/hadoop# cat LICENSE.txt > file1.txt
```

在HDFS中创建input文件夹

```bash
root@h01:/usr/local/hadoop/bin# ./hadoop fs -mkdir /input
```

将刚刚创建的`file1.txt`上传到HDFS中

```bash
root@h01:/usr/local/hadoop/bin# ./hadoop fs -put ../file1.txt /input
```

检查`file1.txt`是否上传成功

```bash
root@h01:/usr/local/hadoop/bin# ./hadoop fs -ls /input
Found 1 items
-rw-r--r--   2 root supergroup     150571 2022-09-26 06:29 /input/file1.txt
```

运行 wordcount 例子程序

```bash
root@h01:/usr/local/hadoop/bin# ./hadoop jar ../share/hadoop/mapreduce/hadoop-mapreduce-examples-3.2.3.jar wordcount /input /output
2022-09-26 06:33:06,459 INFO client.RMProxy: Connecting to ResourceManager at h01/172.18.0.2:8032
2022-09-26 06:33:06,877 INFO mapreduce.JobResourceUploader: Disabling Erasure Coding for path: /tmp/hadoop-yarn/staging/root/.staging/job_1664172811516_0001
2022-09-26 06:33:07,334 INFO input.FileInputFormat: Total input files to process : 1
2022-09-26 06:33:07,804 INFO mapreduce.JobSubmitter: number of splits:1
2022-09-26 06:33:07,936 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_1664172811516_0001
2022-09-26 06:33:07,938 INFO mapreduce.JobSubmitter: Executing with tokens: []
2022-09-26 06:33:08,104 INFO conf.Configuration: resource-types.xml not found
2022-09-26 06:33:08,105 INFO resource.ResourceUtils: Unable to find 'resource-types.xml'.
2022-09-26 06:33:08,519 INFO impl.YarnClientImpl: Submitted application application_1664172811516_0001
2022-09-26 06:33:08,557 INFO mapreduce.Job: The url to track the job: http://h01:8088/proxy/application_1664172811516_0001/
2022-09-26 06:33:08,558 INFO mapreduce.Job: Running job: job_1664172811516_0001
2022-09-26 06:33:15,645 INFO mapreduce.Job: Job job_1664172811516_0001 running in uber mode : false
2022-09-26 06:33:15,646 INFO mapreduce.Job:  map 0% reduce 0%
2022-09-26 06:33:20,701 INFO mapreduce.Job:  map 100% reduce 0%
2022-09-26 06:33:27,875 INFO mapreduce.Job:  map 100% reduce 100%
2022-09-26 06:33:27,892 INFO mapreduce.Job: Job job_1664172811516_0001 completed successfully
2022-09-26 06:33:27,980 INFO mapreduce.Job: Counters: 54
        File System Counters
                FILE: Number of bytes read=46854
                FILE: Number of bytes written=567357
                FILE: Number of read operations=0
                FILE: Number of large read operations=0
                FILE: Number of write operations=0
                HDFS: Number of bytes read=150667
                HDFS: Number of bytes written=35326
                HDFS: Number of read operations=8
                HDFS: Number of large read operations=0
                HDFS: Number of write operations=2
                HDFS: Number of bytes read erasure-coded=0
        Job Counters
                Launched map tasks=1
                Launched reduce tasks=1
                Rack-local map tasks=1
                Total time spent by all maps in occupied slots (ms)=3026
                Total time spent by all reduces in occupied slots (ms)=3726
                Total time spent by all map tasks (ms)=3026
                Total time spent by all reduce tasks (ms)=3726
                Total vcore-milliseconds taken by all map tasks=3026
                Total vcore-milliseconds taken by all reduce tasks=3726
                Total megabyte-milliseconds taken by all map tasks=3098624
                Total megabyte-milliseconds taken by all reduce tasks=3815424
        Map-Reduce Framework
                Map input records=2814
                Map output records=21904
                Map output bytes=234037
                Map output materialized bytes=46854
                Input split bytes=96
                Combine input records=21904
                Combine output records=2981
                Reduce input groups=2981
                Reduce shuffle bytes=46854
                Reduce input records=2981
                Reduce output records=2981
                Spilled Records=5962
                Shuffled Maps =1
                Failed Shuffles=0
                Merged Map outputs=1
                GC time elapsed (ms)=91
                CPU time spent (ms)=1550
                Physical memory (bytes) snapshot=480907264
                Virtual memory (bytes) snapshot=5207310336
                Total committed heap usage (bytes)=387448832
                Peak Map Physical memory (bytes)=311922688
                Peak Map Virtual memory (bytes)=2603814912
                Peak Reduce Physical memory (bytes)=168984576
                Peak Reduce Virtual memory (bytes)=2603495424
        Shuffle Errors
                BAD_ID=0
                CONNECTION=0
                IO_ERROR=0
                WRONG_LENGTH=0
                WRONG_MAP=0
                WRONG_REDUCE=0
        File Input Format Counters
                Bytes Read=150571
        File Output Format Counters
                Bytes Written=35326
```

查看HDFS的output文件中的内容

```bash
root@h01:/usr/local/hadoop/bin# ./hadoop fs -ls /output
Found 2 items
-rw-r--r--   2 root supergroup          0 2022-09-26 06:33 /output/_SUCCESS
-rw-r--r--   2 root supergroup      35326 2022-09-26 06:33 /output/part-r-00000
```

查看下`part-r-00000`的内容

```bash
root@h01:/usr/local/hadoop/bin# ./hadoop fs -cat /output/part-r-00000
```

## Hbase的安装与配置

下载Hbase

```bash
wget https://archive.apache.org/dist/hbase/2.3.7/hbase-2.3.7-bin.tar.gz
```

解压到`/usr/local/`文件夹

```bash
root@h01:~# tar -zxvf hbase-2.3.7-bin.tar.gz -C /usr/local/
```

配置下环境变量，将如下内容添加到`/etc/profile`中

```bash
#hbase
export HBASE_HOME=/usr/local/hbase-2.3.7
export PATH=$PATH:$HBASE_HOME/bin
```

使新配置的环境变量生效

```bash
root@h01:/usr/local# source /etc/profile
```

同时，其余四个节点都需要在其各自的`/etc/profile`中添加上面的内容

使用ssh（或者打开四个终端）进入到其余四个节点，然后依次修改即可。



进入`/usr/local/hbase-2.3.7/conf`文件下修改相应文件

在`hbase-env.sh`中加入如下内容

```bash
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
export HBASE_MANAGES_ZK=true
```

在`hbase-site.xml`中将configuration修改为如下内容

```xml
<configuration>
        <property>
                <name>hbase.rootdir</name>
                <value>hdfs://h01:9000/hbase</value>
        </property>
        <property>
                <name>hbase.cluster.distributed</name>
                <value>true</value>
        </property>
        <property>
                <name>hbase.master</name>
                <value>h01:60000</value>
        </property>
        <property>
                <name>hbase.zookeeper.quorum</name>
                <value>h01,h02,h03,h04,h05</value>
        </property>
        <property>
                <name>hbase.zookeeper.property.dataDir</name>
                <value>/home/hadoop/zoodata</value>
        </property>
</configuration>
```

将`regionservers`修改为如下内容

```text
h01
h02
h03
h04
h05
```

使用 `scp` 命令将配置好的 Hbase 复制到其他 4 个容器中

```bash
root@h01:~# scp -r /usr/local/hbase-2.3.7 root@h02:/usr/local/
root@h01:~# scp -r /usr/local/hbase-2.3.7 root@h03:/usr/local/
root@h01:~# scp -r /usr/local/hbase-2.3.7 root@h04:/usr/local/
root@h01:~# scp -r /usr/local/hbase-2.3.7 root@h05:/usr/local/
```

启动hbase

```bash
root@h01:/usr/local/hbase-2.3.7/bin# start-hbase.sh
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/usr/local/hadoop/share/hadoop/common/lib/slf4j-log4j12-1.7.25.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/usr/local/hbase-2.3.7/lib/client-facing-thirdparty/slf4j-log4j12-1.7.30.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
SLF4J: Actual binding is of type [org.slf4j.impl.Log4jLoggerFactory]
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/usr/local/hadoop/share/hadoop/common/lib/slf4j-log4j12-1.7.25.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/usr/local/hbase-2.3.7/lib/client-facing-thirdparty/slf4j-log4j12-1.7.30.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
SLF4J: Actual binding is of type [org.slf4j.impl.Log4jLoggerFactory]
h03: running zookeeper, logging to /usr/local/hbase-2.3.7/bin/../logs/hbase-root-zookeeper-h03.out
h01: running zookeeper, logging to /usr/local/hbase-2.3.7/bin/../logs/hbase-root-zookeeper-h01.out
h02: running zookeeper, logging to /usr/local/hbase-2.3.7/bin/../logs/hbase-root-zookeeper-h02.out
h04: running zookeeper, logging to /usr/local/hbase-2.3.7/bin/../logs/hbase-root-zookeeper-h04.out
h05: running zookeeper, logging to /usr/local/hbase-2.3.7/bin/../logs/hbase-root-zookeeper-h05.out
running master, logging to /usr/local/hbase-2.3.7/logs/hbase--master-h01.out
h05: running regionserver, logging to /usr/local/hbase-2.3.7/bin/../logs/hbase-root-regionserver-h05.out
h01: running regionserver, logging to /usr/local/hbase-2.3.7/bin/../logs/hbase-root-regionserver-h01.out
h04: running regionserver, logging to /usr/local/hbase-2.3.7/bin/../logs/hbase-root-regionserver-h04.out
h03: running regionserver, logging to /usr/local/hbase-2.3.7/bin/../logs/hbase-root-regionserver-h03.out
h02: running regionserver, logging to /usr/local/hbase-2.3.7/bin/../logs/hbase-root-regionserver-h02.out
```

打开hbase的shell，并做一下测试

```bash
root@h01:/usr/local/hbase-2.3.7/bin# hbase shell
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/usr/local/hadoop/share/hadoop/common/lib/slf4j-log4j12-1.7.25.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/usr/local/hbase-2.3.7/lib/client-facing-thirdparty/slf4j-log4j12-1.7.30.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
SLF4J: Actual binding is of type [org.slf4j.impl.Log4jLoggerFactory]
HBase Shell
Use "help" to get list of supported commands.
Use "exit" to quit this interactive shell.
For Reference, please visit: http://hbase.apache.org/2.0/book.html#shell
Version 2.3.7, r8b2f5141e900c851a2b351fccd54b13bcac5e2ed, Tue Oct 12 16:38:55 UTC 2021
Took 0.0008 seconds
hbase(main):001:0>
hbase(main):002:0*
hbase(main):003:0* whoami
root (auth:SIMPLE)
    groups: root
Took 0.0243 seconds
hbase(main):004:0> create 'member','id','address','info'
Created table member
Took 1.7968 seconds
=> Hbase::Table - member
hbase(main):005:0> put 'member', 'debugo','id','11'
Took 0.1932 seconds
hbase(main):006:0> put 'member', 'debugo','info:age','27'
Took 0.0074 seconds
hbase(main):007:0> count 'member'
1 row(s)
Took 0.0709 seconds
=> 1
hbase(main):008:0>
```

## spark的安装与配置

下载spark压缩包到本地

```bash
wget https://archive.apache.org/dist/spark/spark-2.4.8/spark-2.4.8-bin-hadoop2.7.tgz
```

将其解压到 `/usr/local` 目录下面

```bash
root@h01:~# tar -zxvf spark-2.4.8-bin-hadoop2.7.tgz -C /usr/local/
```

进入`/usr/local/`文件夹，将刚刚解压的文件夹改个名字

```bash
root@h01:/usr/local# mv spark-2.4.8-bin-hadoop2.7/ spark-2.4.8
```

添加环境变量

```bash
vim /etc/profile
```

将如下内容加入到环境变量中

```bash
#spark
export SPARK_HOME=/usr/local/spark-2.4.8
export PATH=$PATH:$SPARK_HOME/bin
```

运行如下命令使得刚刚配置的环境变量生效

```bash
source /etc/profile
```

同样的，将其他四个节点也添加如上的环境变量



在`/usr/local/spark-2.4.8/conf`目录中修改配置

修改文件名

```bash
mv spark-env.sh.template spark-env.sh
mv slaves.template slaves
```

在`spark-env.sh`文件中添加如下内容

```sh
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
export HADOOP_HOME=/usr/local/hadoop
export HADOOP_CONF_DIR=/usr/local/hadoop/etc/hadoop
export SCALA_HOME=/usr/share/scala

export SPARK_MASTER_HOST=h01
export SPARK_MASTER_IP=h01
export SPARK_WORKER_MEMORY=4g
```

将`slaves`的内容修改为如下内容

```text
h01
h02
h03
h04
h05
```

将配置好的spark使用`scp`命令上传到其他四个节点中

```bash
scp -r /usr/local/spark-2.4.8 root@h02:/usr/local/
scp -r /usr/local/spark-2.4.8 root@h03:/usr/local/
scp -r /usr/local/spark-2.4.8 root@h04:/usr/local/
scp -r /usr/local/spark-2.4.8 root@h05:/usr/local/
```

启动spark

```bash
root@h01:/usr/local/spark-2.4.8/sbin# start-all.sh
Starting namenodes on [h01]
Starting datanodes
Starting secondary namenodes [h01]
Starting resourcemanager
Starting nodemanagers
```

进入spark shell交互界面

```bash
root@h01:/usr/local/spark-2.4.8/bin# spark-shell
2022-09-26 09:59:32,002 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
Setting default log level to "WARN".
To adjust logging level use sc.setLogLevel(newLevel). For SparkR, use setLogLevel(newLevel).
Spark context Web UI available at http://h01:4040
Spark context available as 'sc' (master = local[*], app id = local-1664186379024).
Spark session available as 'spark'.
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 2.4.8
      /_/

Using Scala version 2.11.12 (OpenJDK 64-Bit Server VM, Java 1.8.0_292)
Type in expressions to have them evaluated.
Type :help for more information.

scala>
```

跑一个小demo测试一下

```shell
scala> val file = sc.textFile("/input/file1.txt")
file: org.apache.spark.rdd.RDD[String] = /input/file1.txt MapPartitionsRDD[1] at textFile at <console>:24

scala> val count = file.flatMap(line=>line.split(" ")).map(word=>(word,1)).reduceByKey((a,b)=>a+b)
count: org.apache.spark.rdd.RDD[(String, Int)] = ShuffledRDD[4] at reduceByKey at <console>:25

scala> count.collect()
res0: Array[(String, Int)] = Array((Unless,5), (works,,4), (order,2), (MERCHANTABILITY,,10), (Warranties,2), (been,8), (infringement.,1), (users,,1), (appropriateness,2), (appears,,1), (Corporation,1), (offering,2), (file,24), (map,,1), (payment,1), (agreement,,4), (responsibilities,1), (are,91), (Version.,2), (2.,11), ((a)?rename,1), (reproduction,,4), (consists,3), (IS,43), (grant,10), (DATA,,16), (offer.,4), (nature;,1), (Under,3), ("[]",1), (3(b),2), (General,3), (SESAC),,1), (entirety,2), (contributors,12), (B.,2), (JLine,1), ("printed,1), ("COPYRIGHTS,1), (synchronization,2), (1.9.,2), (chosen,1), (relating,5), (technically,2), (initiation,1), (US,2), (EXTENT,5), (Contribution.,4), (org.apache.hadoop.util.bloom.*,1), (i),2), (?Covered,2), (conclusions,1), (Authors,1), (associated,...
scala>
```

