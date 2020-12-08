---
layout: post
title: 整合Seata和TSF
date: 2020-06-23 10:38:00
author: 任怀林
categories:
- linux
  thumb: linux.png
---

# 1. 准备工作

## 系统配置

本机IP:  192.168.188.237

Docker 19.03

# 1.1 Consul

TSF使用consul做为配置和注册中心。

在本地运行时，需要自行下载consul，并运行如下命令启动：

```
./consul agent -dev -bind=0.0.0.0 -client=0.0.0.0
```

# 1.2 运行seata server

Seata server 负责维护全局和分支事务的状态，驱动全局事务提交或回滚。

使用docker来运行seata server

```
$ docker run --name seata-server --rm \
        -p 8091:8091 \
        -e SEATA_IP=192.168.188.237 \
        -e SEATA_CONFIG_NAME=file:/root/seata-config/registry \
        -v /Users/harley/Softwares/Developments/seata/conf:/root/seata-config  \
        seataio/seata-server
```

请注意`SEATA_IP`要配置为主机的内网IP，不能用`127.0.0.1`。

在目录`/Users/harley/Softwares/Developments/seata/conf`下要创建两个文件：`registry.conf`和 `file.conf`。

`registry.conf`告诉seata server 注册中心的信息和配置系统的信息。

```icon
registry {
  # file 、nacos 、eureka、redis、zk、consul、etcd3、sofa
  type = "consul"

  consul {
    cluster = "seata-server"
    serverAddr = "192.168.188.237:8500"
  }
}

config {
  # file、nacos 、apollo、zk、consul、etcd3
  type = "file"
  consul {
    serverAddr = "192.168.188.237:8500"
  }

  file {
    name = "file.conf"
  }
}
```

很明显，`registry`这一段是注册中心信息，config这一段是配置中心信息。

```icon
consul {
    cluster = "seata-server"
    serverAddr = "192.168.188.237:8500"
  }
```

这里的cluster是seata server 在consul里的名字，这个很重要，以后要用到。

`file.conf`的配置seata server的事务日志保存方式，有两种方式，一种是用文件来保存，另一种方式就是数据库。

```
## transaction log store, only used in seata-server
store {
  ## store mode: file、db
  mode = "file"

  ## file store property
  file {
    ## store location dir
    dir = "sessionStore"
    # branch session size , if exceeded first try compress lockkey, still exceeded throws exceptions
    maxBranchSessionSize = 16384
    # globe session size , if exceeded throws exceptions
    maxGlobalSessionSize = 512
    # file buffer size , if exceeded allocate new buffer
    fileWriteBufferCacheSize = 16384
    # when recover batch read size
    sessionReloadReadSize = 100
    # async, sync
    flushDiskMode = async
  }

  ## database store property
  db {
    ## the implement of javax.sql.DataSource, such as DruidDataSource(druid)/BasicDataSource(dbcp) etc.
    datasource = "druid"
    ## mysql/oracle/postgresql/h2/oceanbase etc.
    dbType = "mysql"
    driverClassName = "com.mysql.jdbc.Driver"
    url = "jdbc:mysql://127.0.0.1:3306/seata"
    user = "mysql"
    password = "mysql"
    minConn = 5
    maxConn = 30
    globalTable = "global_table"
    branchTable = "branch_table"
    lockTable = "lock_table"
    queryLimit = 100
    maxWait = 5000
  }
}
```

接下来，我们要把seata的配置写入到配置中心。

这里我们用的是consul，我们需要下载官方的导入脚本： https://raw.githubusercontent.com/seata/seata/develop/script/config-center/consul/consul-config.sh

这个脚本会把父目录里的config.txt中的内容导入到consul里。下面是`config.txt`的内容。

```
transport.type=TCP
transport.server=NIO
transport.heartbeat=true
transport.enableClientBatchSendRequest=false
transport.threadFactory.bossThreadPrefix=NettyBoss
transport.threadFactory.workerThreadPrefix=NettyServerNIOWorker
transport.threadFactory.serverExecutorThreadPrefix=NettyServerBizHandler
transport.threadFactory.shareBossWorker=false
transport.threadFactory.clientSelectorThreadPrefix=NettyClientSelector
transport.threadFactory.clientSelectorThreadSize=1
transport.threadFactory.clientWorkerThreadPrefix=NettyClientWorkerThread
transport.threadFactory.bossThreadSize=1
transport.threadFactory.workerThreadSize=default
transport.shutdown.wait=3
service.vgroupMapping.my_test_tx_group=seata-server
service.seata-server.grouplist=192.168.188.237:8091
service.enableDegrade=false
service.disableGlobalTransaction=false
client.rm.asyncCommitBufferLimit=10000
client.rm.lock.retryInterval=10
client.rm.lock.retryTimes=30
client.rm.lock.retryPolicyBranchRollbackOnConflict=true
client.rm.reportRetryCount=5
client.rm.tableMetaCheckEnable=false
client.rm.sqlParserType=druid
client.rm.reportSuccessEnable=false
client.rm.sagaBranchRegisterEnable=false
client.tm.commitRetryCount=5
client.tm.rollbackRetryCount=5
store.mode=file
store.file.dir=file_store/data
store.file.maxBranchSessionSize=16384
store.file.maxGlobalSessionSize=512
store.file.fileWriteBufferCacheSize=16384
store.file.flushDiskMode=async
store.file.sessionReloadReadSize=100
store.db.datasource=druid
store.db.dbType=mysql
store.db.driverClassName=com.mysql.jdbc.Driver
store.db.url=jdbc:mysql://127.0.0.1:3306/seata?useUnicode=true
store.db.user=username
store.db.password=password
store.db.minConn=5
store.db.maxConn=30
store.db.globalTable=global_table
store.db.branchTable=branch_table
store.db.queryLimit=100
store.db.lockTable=lock_table
store.db.maxWait=5000
server.recovery.committingRetryPeriod=1000
server.recovery.asynCommittingRetryPeriod=1000
server.recovery.rollbackingRetryPeriod=1000
server.recovery.timeoutRetryPeriod=1000
server.maxCommitRetryTimeout=-1
server.maxRollbackRetryTimeout=-1
server.rollbackRetryTimeoutUnlockEnable=false
client.undo.dataValidation=true
client.undo.logSerialization=jackson
server.undo.logSaveDays=7
server.undo.logDeletePeriod=86400000
client.undo.logTable=undo_log
client.log.exceptionRate=100
transport.serialization=seata
transport.compressor=none
metrics.enabled=false
metrics.registryType=compact
metrics.exporterList=prometheus
metrics.exporterPrometheusPort=9898
```

注意下面的两段：

```
service.vgroupMapping.my_test_tx_group=seata-server
service.seata-server.grouplist=192.168.188.237:8091
```

其中`service.vgroupMapping.my_test_tx_group`的`my_test_tx_group`是事务分组的名称。

为什么要引用事务分组的概念呢？官方的说法是：

```
为什么这么设计，不直接取服务名？
这里多了一层获取事务分组到映射集群的配置。这样设计后，事务分组可以作为资源的逻辑隔离单位，出现某集群故障时可以快速failover，只切换对应分组，可以把故障缩减到服务级别，但前提也是你有足够server集群。
```

这个事务分组是要配置在spring cloud的配置文件里的。

`service.vgroupMapping.my_test_tx_group=seata-server`这一行的配置的意思是`my_test_tx_group`这个事务分组对应的seaea集群为`seata-server`。这个集群的地址在配置的下一行里：

```
service.seata-server.grouplist=192.168.188.237:8091
```

这样通过事务分组的名字就可以找到后面的seata server了。

# 2. Spring Cloud 配置

```yaml
# 数据源配置
spring:
  datasource:
    type: com.alibaba.druid.pool.DruidDataSource
    url: jdbc:mysql://127.0.0.1:3306/seata-test?useUnicode=true&characterEncoding=UTF-8&serverTimezone=UTC
    driver-class-name: com.mysql.cj.jdbc.Driver
    username: root
    password: passw0rd
    max-wait: 60000
    max-active: 100
    min-idle: 10
    initial-size: 10

# seata的配置
seata:
  enabled: true
  application-id: account-service
  tx-service-group: my_test_tx_group
  enable-auto-data-source-proxy: true
  use-jdk-proxy: false
  excludes-for-auto-proxying: firstClassNameForExclude,secondClassNameForExclude
  client:
    rm:
      async-commit-buffer-limit: 1000
      report-retry-count: 5
      table-meta-check-enable: false
      report-success-enable: false
      saga-branch-register-enable: false
      lock:
        retry-interval: 10
        retry-times: 30
        retry-policy-branch-rollback-on-conflict: true
    tm:
      commit-retry-count: 5
      rollback-retry-count: 5
    undo:
      data-validation: true
      log-serialization: jackson
      log-table: undo_log
    log:
      exceptionRate: 100
  service:
    vgroup-mapping:
      my_test_tx_group: seata-server
    grouplist:
      seata-server: 192.168.188.237:8091
    enable-degrade: false
    disable-global-transaction: false
  transport:
    shutdown:
      wait: 3
    thread-factory:
      boss-thread-prefix: NettyBoss
      worker-thread-prefix: NettyServerNIOWorker
      server-executor-thread-prefix: NettyServerBizHandler
      share-boss-worker: false
      client-selector-thread-prefix: NettyClientSelector
      client-selector-thread-size: 1
      client-worker-thread-prefix: NettyClientWorkerThread
      worker-thread-size: default
      boss-thread-size: 1
    type: TCP
    server: NIO
    heartbeat: true
    serialization: seata
    compressor: none
    enable-client-batch-send-request: true
  config:
    type: consul
    consul:
      server-addr: 192.168.188.237:8500

  registry:
    type: consul
    consul:
      server-addr: 192.168.188.237:8500
```

注意这一段：

```yaml
 service:
    vgroup-mapping:
      my_test_tx_group: seata-server
    grouplist:
      seata-server: 192.168.188.237:8091
```

这里配置了本应用所使用的事务分组。

# 3. AT模式

## 3.1 前提

* 基于支持本地 ACID 事务的关系型数据库。

* Java 应用，通过 JDBC 访问数据库。

非关系型数据库是不支持的，这个很显然。

## 3.2 机制

两阶段提交协议的演变：

- 一阶段：业务数据和回滚日志记录在同一个本地事务中提交，释放本地锁和连接资源。

- 二阶段：
  
  - 提交异步化，非常快速地完成。
  - 回滚通过一阶段的回滚日志进行反向补偿。

![](https://user-gold-cdn.xitu.io/2019/11/28/16eb2b74f7b8fed1?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

概括来讲，`AT` 模式的工作流程分为**两阶段**。一阶段进行业务 `SQL` 执行，并通过 `SQL` 拦截、`SQL` 改写等过程生成修改数据前后的快照（`Image`），并作为 `UndoLog` 和业务修改**在同一个本地事务中提交**。

如果一阶段成功那么二阶段仅仅异步删除刚刚插入的 `UndoLog`；如果二阶段失败则通过 `UndoLog` 生成反向 `SQL` 语句回滚一阶段的数据修改。

示例代码：

```java
/**
     * AT分布式事务测试
     *
     * @return
     * @throws TransactionException
     */
    @GetMapping(value = "testCommit")
    @GlobalTransactional
    public Object testCommit() throws TransactionException {
        lock.lock();
        try {
            Product product = productService.getById(1);
            if (product.getStock() > 0) {
                LocalDateTime now = LocalDateTime.now();
                logger.info("seata分布式事务Id:{}", RootContext.getXID());
                Account account = accountService.getById(1);
                Orders orders = new Orders();
                orders.setCreateTime(now);
                orders.setProductId(product.getId());
                orders.setReplaceTime(now);
                orders.setSum(1);
                orders.setAmount(product.getPrice());
                orders.setAccountId(account.getId());
                product.setStock(product.getStock() - 1);
                account.setSum(account.getSum() != null ? account.getSum() + 1 : 1);
                account.setLastUpdateTime(now);
                productService.updateById(product);
                accountService.updateById(account);
                orderService.save(orders);
                return true;
            } else {
                return false;
            }
        } catch (Exception e) {
            logger.info("载入事务{}进行回滚" + e.getMessage(), RootContext.getXID());
            GlobalTransactionContext.reload(RootContext.getXID()).rollback();
            return false;
        } finally {
            lock.unlock();
        }
    }
```

要给业务代码加上`@GlobalTransactional`这个注解，所以seata的AT模式对业务没有侵入，使用起来非常方便。

# 4. TCC模式

TCC是模式是一种通用的分布式事务处理模式，用户通过定义 `try/confirm/cancel` 三个方法在应用层面模拟两阶段提交。提交和回滚的控制由应用来决定。

![Overview of a global transaction](http://seata.io/img/seata_tcc-1.png)

TCC 模式，不依赖于底层数据资源的事务支持：

- 一阶段 prepare 行为：调用 **自定义** 的 prepare 逻辑。
- 二阶段 commit 行为：调用 **自定义** 的 commit 逻辑。
- 二阶段 rollback 行为：调用 **自定义** 的 rollback 逻辑。

seata 的tcc对Dubbo的支持非常好，官方没有对Spring Cloud的支持，只能通过LocalTCC这个注解来实现。

代码实现：

```java
@LocalTCC
public interface TccAccountService {
    /**
     * 一阶段方法
     *
     * @param businessActionContext
     * @param accountId
     * @param amount
     */
    @TwoPhaseBusinessAction(name = "firstTccAction", commitMethod = "commit", rollbackMethod = "rollback")
    public boolean prepare(BusinessActionContext businessActionContext,
                           @BusinessActionContextParameter(paramName = "accountId") long accountId,
                           @BusinessActionContextParameter(paramName = "amount") int amount);

    /**
     * t
     * 二阶段提交
     *
     * @param businessActionContext
     * @return
     */
    public boolean commit(BusinessActionContext businessActionContext);

    /**
     * 二阶段回滚
     *
     * @param businessActionContext
     * @return
     */
    public boolean rollback(BusinessActionContext businessActionContext);
}

/////////////// TccAccountServiceImpl.java ///////////////


package com.renhl.service.impl;

import com.renhl.entity.Account;
import com.renhl.service.AccountDAO;
import com.renhl.service.TccAccountService;
import io.seata.rm.tcc.api.BusinessActionContext;
import io.seata.spring.annotation.GlobalTransactional;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import org.springframework.transaction.TransactionStatus;
import org.springframework.transaction.support.TransactionCallback;
import org.springframework.transaction.support.TransactionTemplate;

import java.util.Optional;

@Component
public class TccAccountServiceImpl implements TccAccountService {

    @Autowired
    private AccountDAO accountDAO;

    /**
     * 扣钱数据源事务模板
     */
    @Autowired
    private TransactionTemplate fromDsTransactionTemplate;


    @Override
    @GlobalTransactional
    public boolean deduct(long id, int amount) {
        return this.prepare(null, id, amount);
    }

    @Override
    public boolean prepare(BusinessActionContext businessActionContext, long accountId, int amount) {
        //分布式事务ID
        final String xid = businessActionContext.getXid();

        Boolean execute = false;
        execute = fromDsTransactionTemplate.execute(new TransactionCallback<Boolean>() {

            public Boolean doInTransaction(TransactionStatus status) {
                try {
                    Optional<Account> oa = accountDAO.findById(accountId);
                    if (oa.isPresent()) {
                        Account account = oa.get();
                        if (account.getSum() < amount) {
                            throw new RuntimeException("Account balance is insufficient.");
                        }
                        account.setFreezed(account.getFreezed() + amount);
                        accountDAO.save(account);
                    } else {
                        throw new RuntimeException("Account dose not exist.");
                    }


                    System.out.println(String.format("prepareMinus account[%s] amount[%d], dtx transaction id: %s.", accountId, amount, xid));
                    return true;
                } catch (Throwable t) {
                    t.printStackTrace();
                    status.setRollbackOnly();
                    return false;
                }
            }
        });
        return execute;
    }

    @Override
    public boolean commit(BusinessActionContext businessActionContext) {
        //分布式事务ID
        final String xid = businessActionContext.getXid();
        //账户ID
        final long accountId = Long.parseLong(String.valueOf(businessActionContext.getActionContext("accountId")));
        //转出金额
        final int amount = Integer.parseInt(String.valueOf(businessActionContext.getActionContext("amount")));

        Boolean execute = false;
        execute = fromDsTransactionTemplate.execute(new TransactionCallback<Boolean>() {

            @Override
            public Boolean doInTransaction(TransactionStatus status) {
                try {

                    Optional<Account> oa = accountDAO.findById(accountId);
                    if (oa.isPresent()) {
                        Account account = oa.get();
                        account.setFreezed(account.getFreezed() - amount);
                        account.setSum(account.getSum() - amount);
                        accountDAO.save(account);
                    } else {
                        throw new RuntimeException("Account dose not exist.");
                    }

                    System.out.println(String.format("minus account[%d] amount[%d], dtx transaction id: %s.", accountId, amount, xid));
                    return true;
                } catch (Throwable t) {
                    t.printStackTrace();
                    status.setRollbackOnly();
                    return false;
                }
            }
        });
        return execute;
    }

    @Override
    public boolean rollback(BusinessActionContext businessActionContext) {
        //分布式事务ID
        final String xid = businessActionContext.getXid();
        //账户ID
        final long accountId = Long.parseLong(String.valueOf(businessActionContext.getActionContext("accountId")));
        //转出金额
        final int amount = Integer.parseInt(String.valueOf(businessActionContext.getActionContext("amount")));

        Boolean execute = false;
        execute = fromDsTransactionTemplate.execute(new TransactionCallback<Boolean>() {

            @Override
            public Boolean doInTransaction(TransactionStatus status) {
                try {

                    Optional<Account> oa = accountDAO.findById(accountId);
                    if (oa.isPresent()) {
                        Account account = oa.get();
                        account.setFreezed(account.getFreezed() - amount);
                        accountDAO.save(account);
                    } else {
                        throw new RuntimeException("Account dose not exist.");
                    }

                    System.out.println(String.format("minus account[%d] amount[%d], dtx transaction id: %s.", accountId, amount, xid));
                    return true;
                } catch (Throwable t) {
                    t.printStackTrace();
                    status.setRollbackOnly();
                    return false;
                }
            }
        });
        return execute;
    }
}
```
