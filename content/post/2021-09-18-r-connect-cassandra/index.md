---
title: R connect Cassandra
author: xilin
date: '2021-09-18'
slug: r-connect-cassandra
categories:
  - R
tags: []
---

2020-04-15：最新的方法参考“使用datastax的JDBC Driver”部分

Cassandra数据库[在维基百科的简介](https://zh.wikipedia.org/wiki/Cassandra)：

> Apache Cassandra（社区内一般简称为C*）是一套开源分布式NoSQL数据库系统。它最初由Facebook开发，用于储存收件箱等简单格式数据，集Google BigTable的数据模型与Amazon Dynamo的完全分布式架构于一身。Facebook于2008将 Cassandra 开源，此后，由于Cassandra良好的可扩展性和性能，被 Apple[1], Comcast[2],Instagram[3], Spotify[4], eBay[5], Rackspace[6], Netflix[7]等知名网站所采用，成为了一种流行的分布式结构化数据存储方案。  
在数据库排行榜“DB-Engines Ranking”中，Cassandra排在第七位，是非关系型数据库中排名第二高的（仅次于MongoDB）[8]。

R作为一门偏统计学使用的科学计算语言，并不像Java、Python等语言能够及时更新与这些前沿数据库的连接方式，在踩了许多坑之后，终于研究出较为靠谱的方法，在这里记录下来。

本文介绍了三种连接方法，第一种是通过Spark连接Cassandra的方法，这是较为靠谱的方式，但连接速度较慢，其次是JDBC的连接方法，在有了合适的driver后是最合适的方法，最后简单提一下RCassandra包。

### 通过Spark连接Cassandra
连接Cassandra的主要困难在于没有合适的用来连接数据库的R语言driver。而使用通过R先连接Spark，然后在Spark里连接Cassandra的原因在于，连接Spark的sparklyr包，以及Spark连接Cassandra的driver都能保持更新，这样不容易存在无法连接最新版Cassandra的问题（其他方法会提到）。同时也可以借用Spark的计算框架，先对数据进行处理，然后加载到R环境里。

这个方法主要来自于[sparklyr 0.6版本的更新说明](https://blog.rstudio.com/2017/07/31/sparklyr-0-6/)。本文不会介绍sparklyr的使用方法，相关介绍可以看[官方文档](http://spark.rstudio.com)。

R连接Spark除了可以使用RStudio出品的sparklyr外，还可以使用Spark社区自家的[SparkR包](https://spark.apache.org/docs/latest/sparkr.html)。有关sparklyr和SparkR的区别可以看[这篇文章](https://cosx.org/2018/05/sparkr-vs-sparklyr/)。

简单介绍一下Spark。[Spark](https://zh.wikipedia.org/wiki/Apache_Spark)是Apache开源的分布式计算框架，在机器学习领域非常火热：

> Apache Spark是一个开源集群运算框架，最初是由加州大学柏克莱分校AMPLab所开发。相对于Hadoop的MapReduce会在运行完工作后将中介数据存放到磁盘中，Spark使用了存储器内运算技术，能在数据尚未写入硬盘时即在存储器内分析运算。Spark在存储器内运行程序的运算速度能做到比Hadoop MapReduce的运算速度快上100倍，即便是运行程序于硬盘时，Spark也能快上10倍速度。[1]Spark允许用户将数据加载至集群存储器，并多次对其进行查询，非常适合用于机器学习算法。[2]  
使用Spark需要搭配集群管理员和分布式存储系统。Spark支持独立模式（本地Spark集群）、Hadoop YARN或Apache Mesos的集群管理。[3] 在分布式存储方面，Spark可以和HDFS[4]、 Cassandra[5] 、OpenStack Swift和Amazon S3等接口搭载。 Spark也支持伪分布式（pseudo-distributed）本地模式，不过通常只用于开发或测试时以本机文件系统取代分布式存储系统。在这样的情况下，Spark仅在一台机器上使用每个CPU核心运行程序。  
在2014年有超过465位贡献家投入Spark开发[6]，让其成为Apache软件基金会以及大数据众多开源项目中最为活跃的项目。

R通过Spark连接Cassandra的话，首先需要一个适合Spark连接Cassandra的driver，[datastax开源的driver](https://github.com/datastax/spark-cassandra-connector)使用非常广泛，[在这里](https://github.com/datastax/spark-cassandra-connector#version-compatibility)确定好机器上的Spark和Cassandra版本后，下载合适的driver。笔者的spark是2.3，Cassandra是3.7，因此使用2.3的driver。

安装好sparklyr后，使用如下方法连接Cassandra
```r
library(sparklyr)

# 复制默认参数
config <- spark_config()

# 加载Spark连接Cassandra的driver
config[["sparklyr.defaultPackages"]] <- c("datastax:spark-cassandra-connector:2.3.0-s_2.11")
# Cassandra的IP地址，本机为localhost，在其他服务器上输入对应IP地址
config[["spark.cassandra.connection.host"]] <- c("localhost")
# 连接Cassandra的用户名和密码
config[["spark.cassandra.auth.username"]] <- c("<username>")
config[["spark.cassandra.auth.password"]] <- c("<password>")
# 扩大内存
config[["sparklyr.shell.driver-memory"]] <- "8G"
config[["sparklyr.shell.executor-memory"]] <- "8G"

spark_sc <- spark_connect(master="local", config=config)
```
运行后没有报错并且显示出如下信息，就已经连接上Spark以及Cassandra了。
```r
* Using Spark: 2.3.0
```

从Cassandra中读取数据到R环境的方式有两种，第一种是使用DBI包里面的SQL语句，这种方式较为简单，并且可以使用where语句来进行数据筛选，但是处理较大的数据时容易报错。
```r
library(DBI)

# 加载数据到Spark环境中
spark_read_source(sc=spark_sc, name="index_returns",
                  source="org.apache.spark.sql.cassandra",
                  options=list(keyspace="x", table="y"),
                  memory=F)
# 使用DBI包中的dbGetQuery函数把数据加载到R环境里
index_returns <- dbGetQuery(spark_sc, paste0("select * from index_returns where id in ('", paste(index_id[, "id"], collapse="', '"), "')"))

```
第二种方式是使用[collect函数加载到R环境里](https://spark.rstudio.com/dplyr/#collecting-to-r)，操作方式和第一种有点区别。
```r
# 赋值为handle
spark_h <- spark_read_source(sc=spark_sc, name="index_returns",
                             source="org.apache.spark.sql.cassandra",
                             options=list(keyspace="x", table="y"),
                             memory=F)
# 使用dplyr包的filter函数进行处理
spark_h <- dplyr::filter(spark_h, id %in% index_id[, "id"])
# 将handle执行并加载数据到R环境里
dt <- collect(spark_h)                             
```
这里使用了[dplyr包的函数在spark环境中对数据进行了处理](https://spark.rstudio.com/dplyr/)，实际效果和第一种方法一样。也可以直接`collect(spark_h)`将数据提取到R环境中。

`collect()`函数在执行之前，写多次`filter()`函数都不会运行，只是将运行步骤储存了起来，最后一起运行。

### 通过JDBC连接Cassandra
这个方法来自[这个回答](https://stackoverflow.com/questions/21994077/how-to-read-data-from-cassandra-with-r)，通过R的RJDBC包，再利用Cassandra的JDBC driver也可以连接上Cassandra。在实际操作中，原文提到的driver已经无法在3.x版本的Cassandra上使用。尝试多种JDBC driver后，终于得到了一个靠谱的方式，请看下方第二个driver的介绍。

第一个driver的操作步骤如下：前往[这里](https://github.com/adejanovski/cassandra-jdbc-wrapper)下载相应的JDBC driver的jar包，然后去[这里](https://github.com/datastax/java-driver)这里下载Java driver的jar包，放到一个目录下。下载链接在作者提供的Maven代码的下方，是提供了所有依赖包的jar文件。如果会使用Maven的话，也可以使用Maven来安装包。

以现在（2019年3月）为例，下载完成后目录下有“cassandra-jdbc-wrapper-3.1.0-SNAPSHOT.jar”和“cassandra-java-driver-3.7.1”的文件夹。然后把R的工作目录设定到这个文件夹里，运行以下代码
```r
library(RJDBC)

# 加载driver
drv <- JDBC(driverClass="com.github.adejanovski.cassandra.jdbc.CassandraDriver", 
            classPath=list.files("./", pattern="jar$", full.names=T, recursive=T))
# 参数，数据库的地址，用户名和密码
conn <- dbConnect(drv, "jdbc:cassandra://localhost:9042/<keyspace>", user="", password="")
```
若没有报错的话，就已经连接上了。然后使用SQL语句进行查询
```r
# 使用SQL语句读取数据
> dbGetQuery(conn, "select x,y from z limit 1;")
                                 x             y
1 6fba75f5-0fd3-11e9-8d27-4a000747d560     0.02969529

# 查询date类型数据时报错
> dbGetQuery(conn, "select date from z limit 1;")
Error in .jcall(rp, "I", "fetch", stride, block) : 
  java.lang.NullPointerException
```
发现无法读取date类型的数据，存在问题。

后续笔者使用了探究了其他的driver，发现[Cassandra JDBC Driver from DbSchema](https://www.dbschema.com/cassandra-jdbc-driver.html)可以使用，但这个driver只支持Java 11的运行，后续将源码用Java 8编译后（感谢jungle的热情帮助:)），终于可以使用了。使用方式和上一种一样：
```r
library(RJDBC)

cassdrv <- JDBC(driverClass="com.dbschema.CassandraJdbcDriver",
                classPath=list.files("./Cassandra JDBC Driver from DbSchema", pattern="jar$", full.names=T, recursive=T))
cassconn <- dbConnect(drv, "jdbc:cassandra://localhost:9042/<keyspace>", user="", password="")

> dbGetQuery(casscon, "select * from z limit 1;")
                                    a        b                         c       d
1 6fba75f5-0fd3-11e9-8d27-4a000747d560   2009-07-01                   0     0.02969529

```
类似的，应该也可以通过ODBC的方式连接Cassandra，有兴趣的读者可以自己尝试。

## 使用datastax的JDBC Driver

dbschema的driver实际使用中还是有问题，例如`dbListTables()`无法使用，后面又换成了datastax的driver。

首先前往datastax的[网站](https://downloads.datastax.com/#odbc-jdbc-drivers)下载driver文件，在“Drivers - ODBC/JDBC Drivers”下，选择“Simba JDBC Driver for Apache Cassandra”。

下载解压后放到一个目录里，例如`~/tmp/SimbaCassandraJDBC42/`，使用如下代码连接即可

```R
cass_drv <- JDBC(driverClass="com.simba.cassandra.jdbc42.Driver",classPath=list.files(file.path("~/tmp/SimbaCassandraJDBC42/"), pattern="jar$", full.names=TRUE, recursive=TRUE))
cass_conn <- dbConnect(cass_drv, "jdbc:cassandra://localhost:9042;DefaultKeyspace=test;AuthMech=1;UID=UID;PWD=PWD")
```

URL里面的参数可以参考[文档](https://downloads.datastax.com/jdbc/cql/2.0.3.1003/SimbaCassandraJDBCGuide.pdf)第12、28页

注：需要把机器的java升至java 11版本。判断当前rJava的版本：

```R
library(rJava)
.jinit()
.jcall("java/lang/System", "S", "getProperty", "java.runtime.version") 

> [1] "11.0.5+10-post-Ubuntu-0ubuntu1.118.04"
```

升级包括：安装、选择默认的java版本，配置`JAVA_HOME`参数，Rstudio Server可能需要配置`/etc/rstudio/rserver.conf`下的`rsession-ld-library-path`，可以参考《Ubuntu 18.04环境下RStudio Server安装rJava包》这篇文章。

注：如果出现内存不足的报错

```
java.lang.OutOfMemoryError: Java heap space
```

添加参数，[增加jvm的内存](https://stackoverflow.com/questions/24691603/r-rjdbc-java-lang-outofmemoryerror)，下面代码是增加到8GB

```R
options(java.parameters="-Xmx8g")
```



### 通过RCassandra包
这种方式请参考[《R利剑NoSQL系列文章之Cassandra》](http://blog.fens.me/nosql-r-cassandra/)这篇文章。

由于RCassandra包已经多年没有更新，无法连接3.7版本的Cassandra，因而无法使用。


