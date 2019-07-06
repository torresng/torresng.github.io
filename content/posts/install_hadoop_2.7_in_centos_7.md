---
title: "Install Hadoop 2.7 in CentOS 7"
date: 2019-06-30T16:10:21+08:00
draft: false
---

## 配置環境

剛裝完 CentOS 7 先連網，然後安裝 wget, net-tools( ifconfig, netstat ) 和 vim 工具

```shell
$ sudo dhclient
$ sudo yum install -y wget net-tools vim
```

在 /opt/ 目錄下創建 module 目錄和 software 目錄，然後分配這兩個目錄的權限給自己

```sh
$ cd /opt/
$ sudo mkdir module software
$ sudo chown torres:torres module/ software/
$ ll
total 0
drwxr-xr-x. 2 torres torres 6 Jun 30 04:59 module
drwxr-xr-x. 2 torres torres 6 Jun 30 04:59 software
```

## 安裝Java JDK

在 software 目錄中下載 Java OpenJDK，解壓到 module 目錄下，然後在 /etc/profile 將 Java 添加到環境變量。需要 OpenJDK 其他版本可以到 [這里下載](https://openjdk.java.net/install/)

```shell
$ cd /opt/software
$ wget https://download.java.net/java/GA/jdk12.0.1/69cfe15208a647278a19ef0990eea691/12/GPL/openjdk-12.0.1_linux-x64_bin.tar.gz
$ tar -zxvf openjdk-12.0.1_linux-x64_bin.tar.gz -C /opt/module/
$ cd /opt/module/jdk-12.0.1/
$ sudo vim /etc/profile
export JAVA_HOME=/opt/module/jdk-12.0.1
export PATH=$PATH:$JAVA_HOME/bin
```

保存後退出，然後讓配置生效

```shell
$ source /etc/profile
$ java -version
openjdk version "12.0.1" 2019-04-16
OpenJDK Runtime Environment (build 12.0.1+12)
OpenJDK 64-Bit Server VM (build 12.0.1+12, mixed mode, sharing)
```

## 安裝Hadoop

在 software 目錄中下載 Hadoop，解壓到 module 目錄下，然後在 /etc/profile 將 Hadoop 添加到環境變量，這里用2.7.7版本。可以在 [這里下載](https://hadoop.apache.org/releases.html) 自己需要的Hadoop版本

```shell
$ cd /opt/software
$ wget https://www-us.apache.org/dist/hadoop/common/hadoop-2.7.7/hadoop-2.7.7.tar.gz
$ tar -zxvf hadoop-2.7.7.tar.gz -C /opt/module/
$ cd /opt/module/hadoop-2.7.7/
$ sudo vim /etc/profile
export HADOOP_HOME=/opt/module/hadoop-2.7.7
export PATH=$PATH:$HADOOP_HOME/bin
export PATH=$PATH:$HADOOP_HOME/sbin
export CLASSPATH=$($HADOOP_HOME/bin/hadoop classpath):$CLASSPATH
```

保存後退出，然後讓配置生效

```bash
$ source /etc/profile
$ hadoop version
Hadoop 2.7.7
Subversion Unknown -r c1aad84bd27cd79c3d1a7dd58202a8c3ee1ed3ac
Compiled by stevel on 2018-07-18T22:47Z
Compiled with protoc 2.5.0
From source with checksum 792e15d20b12c74bd6f19a1fb886490
This command was run using /opt/module/hadoop-2.7.7/share/hadoop/common/hadoop-common-2.7.7.jar
```

## 測試Hadoop

### Grep案例

運行grep測試並查看結果

```shell
$ mkdir input
$ cp etc/hadoop/*.xml input
$ hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.7.jar grep input output 'dfs[a-z.]+'
$ cat output/*
1	dfsadmin
```

### WordCount案例

創建 wc.input 文件

```shell
$ mkdir wcinput
$ cd wcinput
$ touch wc.input
$ vim wc.input
```

輸入如下內容

```
hadoop yarn
hadoop mapreduce
atguigu
atguigu
```

保存退出後回到 hadoop 目錄運行測試，並查看結果

```shell
$ cd ..
$ hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.7.jar wordcount wcinput/ wcoutput
$ cat wcoutput/part-r-00000
atguigu	2
hadoop	2
mapreduce	1
yarn	1
```

