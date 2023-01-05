---
layout: article
title: 修复Nexus数据库
date: 2020-06-05 12:38:00
author: 任怀林
categories:
- linux
thumb: linux.png
isCJKLanguage: true
---

昨天nexus所在主机的磁盘完全占满了，nexus无法正常运行了，也无法重启。
我的nexus是运行在docker里的，快一年了，日志很大，我删除了docker的日志，释放了一些空间。然后我尝试重启nexus，发现报错：

```
2020-06-05 03:02:00,645+0000 ERROR [FelixStartLevel] *SYSTEM com.orientechnologies.orient.core.storage.impl.local.paginated.OLocalPaginatedStorage - Exception `35D34EA4` in storage `plocal:/nexus-data/db/component`: 2.2.36 (build d3beb772c02098ceaea89779a7afd4b7305d3788, branch 2.2.x)
com.orientechnologies.orient.core.exception.OStorageException: Cannot open local storage '/nexus-data/db/component' with mode=rw^M
        DB name="component"
        at com.orientechnologies.orient.core.storage.impl.local.OAbstractPaginatedStorage.open(OAbstractPaginatedStorage.java:323)
        at com.orientechnologies.orient.core.db.document.ODatabaseDocumentTx.open(ODatabaseDocumentTx.java:259)
        at org.sonatype.nexus.orient.DatabaseManagerSupport.connect(DatabaseManagerSupport.java:142)
        at org.sonatype.nexus.orient.DatabaseManagerSupport.createInstance(DatabaseManagerSupport.java:276)
        at java.util.concurrent.ConcurrentHashMap.computeIfAbsent(ConcurrentHashMap.java:1660)
        at org.sonatype.nexus.orient.DatabaseManagerSupport.instance(DatabaseManagerSupport.java:253)
        at java.util.stream.ForEachOps$ForEachOp$OfRef.accept(ForEachOps.java:184)
        at java.util.Spliterators$ArraySpliterator.forEachRemaining(Spliterators.java:948)
        at java.util.stream.AbstractPipeline.copyInto(AbstractPipeline.java:481)
        at java.util.stream.ForEachOps$ForEachTask.compute(ForEachOps.java:291)
        at java.util.concurrent.CountedCompleter.exec(CountedCompleter.java:731)
        at java.util.concurrent.ForkJoinTask.doExec(ForkJoinTask.java:289)
        at java.util.concurrent.ForkJoinTask.doInvoke(ForkJoinTask.java:401)
        at java.util.concurrent.ForkJoinTask.invoke(ForkJoinTask.java:734)
        at java.util.stream.ForEachOps$ForEachOp.evaluateParallel(ForEachOps.java:160)
        at java.util.stream.ForEachOps$ForEachOp$OfRef.evaluateParallel(ForEachOps.java:174)
        at java.util.stream.AbstractPipeline.evaluate(AbstractPipeline.java:233)
        at java.util.stream.ReferencePipeline.forEach(ReferencePipeline.java:418)
        at java.util.stream.ReferencePipeline$Head.forEach(ReferencePipeline.java:583)

```

第一次遇到这种情况，我是懵的，于是赶紧Google。

找到了两篇文章：

1.  [https://groups.google.com/a/glists.sonatype.com/forum/#!topic/nexus-users/64W_y9fDGzQ](https://groups.google.com/a/glists.sonatype.com/forum/#!topic/nexus-users/64W_y9fDGzQ)

2. [https://issues.sonatype.org/browse/NEXUS-14174](https://issues.sonatype.org/browse/NEXUS-14174)

原理差不多，基本上是要把数据备份一下，然后用orientdb的命令导出，重建数据，再导回来。



我先进入nexus容器：

```
docker run -it \
    -p 8081:8081 \
    -p 8444:8444 \
    --name nexus \
    -- rm \
    -v /apps/nexus3//nexus-data:/nexus-data  \
    nexus:v3 bash
```

然后在容器里执行：

```
> cd /tmp  #一定要转到/tmp目录下。
> java -jar /opt/sonatype/nexus/lib/support/nexus-orient-console.jar
orientdb> connect plocal:/nexus-data/db/component admin admin
```

这时会报错的，跟docker日志的错误是一样的。

退出容器，把nexus的db目录备份一下。

然后转到db/component目录下。

```
$ cd /apps/nexus3/nexus-data/db/component
```

删除所有的`*.wal`。



```
rm *.wal
```

然后重新进入容器。

```
> cd /tmp  #一定要转到/tmp目录下。
> java -jar /opt/sonatype/nexus/lib/support/nexus-orient-console.jar
orientdb> connect plocal:/nexus-data/db/component admin admin
```

接下来运行命令把数据导出、重建并导回来。

```
orientdb> export database component-export
orientdb> drop database
orientdb> create database plocal:/nexus-data/db/component admin admin
orientdb> import database component-export.json.gz -preserveClusterIDs=true
orientdb> rebuild index *
orientdb> disconnect
```

这样数据库修复好了。如果有其它的数据库也坏了(非常有可能的)，逐一修复即可。

修复了这些数据库后，就可以重新运行docker来启动nexus了。
