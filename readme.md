# Themis 

## Introduction

Themis provides cross-row/cross-table transaction on HBase based on [google's percolator](http://research.google.com/pubs/pub36726.html).

Themis guarantees the ACID characteristics of cross-row transaction by two-phase commit and conflict resolution, which is based on the single-row transaction of HBase. Themis depends on [chronos](https://github.com/XiaoMi/chronos) to provide global strictly incremental timestamp, which defines the global order for transactions and makes themis could read database snapshot before given timestamp. Themis adopts HBase coprocessor framework, which could serve after loading themis coprocessors without changing source code of HBase. The APIs of themis are similar with HBase's, including themisPut/themisDelete/themisGet/themisScan. We validate the correctness of themis for a few months, the performance of themis in current version is similar to the result reported in paper of [percolator](http://research.google.com/pubs/pub36726.html). 

## Example of Themis API

### themisPut 

    Configuration conf = HBaseConfiguration.create();
    HConnection connection = HConnectionManager.createConnection(conf);
    Transaction transaction = new Transaction(conf, connection);
    ThemisPut put = new ThemisPut(ROW).add(FAMILY, QUALIFIER, VALUE);
    transaction.put(TABLENAME, put);
    put = new ThemisPut(ANOTHER_ROW).add(FAMILY, QUALIFIER, VALUE);
    transaction.put(TABLENAME, put);
    transaction.commit();

### themisDelete 

     Transaction transaction = new Transaction(conf, connection);
     ThemisDelete delete = new ThemisDelete(ROW);
     delete.deleteColumn(FAMILY, QUALIFIER);
     transaction.delete(TABLENAME, delete);
     delete = new ThemisDelete(ANOTHER_ROW);
     delete.deleteColumn(FAMILY, QUALIFIER);
     transaction.delete(TABLENAME, delete);
     transaction.commit();

In above code, the mutations of two rows will both be applied to HBase and visiable to read after transaction.commit() finished. If transaction.commit() failed, neither of the two mutations will be visiable to read.

### themisGet 

    Transaction transaction = new Transaction(conf, connection);
    ThemisGet get = new ThemisGet(ROW).addColumn(FAMILY, QUALIFIER);
    Result resultA = transaction.get(TABLENAME, get);
    get = new ThemisGet(ANOTHER_ROW).addColumn(FAMILY, QUALIFIER);
    Result resultB = transaction.get(TABLENAME, get);
    // themisGet will return consistent results from ROW and ANOTHER_ROW 

### themisScan

    Transaction transaction = new Transaction(conf, connection);
    ThemisScan scan = new ThemisScan();
    scan.addColumn(FAMILY, QUALIFIER);
    ThemisScanner scanner = transaction.getScanner(TABLENAME, scan);
    Result result = null;
    while ((result = scanner.next()) != null) {
      // themisScan will return consistent state of database
      int value = Bytes.toInt(result.getValue(FAMILY, QUALIFIER));
    }
    scanner.close();

Themis will get a timestamp from chronos before transaction starts, and will promise to read the database snapshot before the timestamp.

For more example code, please see [Example.java](https://github.com/XiaoMi/themis/blob/master/themis-client/src/main/java/org/apache/hadoop/hbase/themis/example/Example.java)

## Principal and Implementation 

### Principal

**Themis Write:**

1. Select one column as primaryColumn and others as secondaryColumns from mutations of users. Themis will construct persistent lock for each column.
2. Prewrite-Phase: get timestamp from chronos(denoted as prewriteTs), write data and persistent lock to HBase with timestamp=prewriteTs when no write/write conflicts discovered.
3. After prewrite-phase finished, get timestamp from chronos(denoted as commitTs) and commit primaryColumn: erase the persistent lock and write the commit information with timestamp=commitTs if the persistent lock of primaryColumn is not deleted.
4. After primaryColumn committed, commit secondaryColumns: erase the persistent lock and write the commit information with timestamp=commitTs for each secondaryColumns.

Themis applies transaction mutations by two-phase write(prewrite/commit). The transaction will success and be visiable to read if primaryColumn is committed succesfully; otherwise, the transaction will fail and can't be read.

**Themis Read:**

1. Get timestamp from chronos(named startTs), check the read/write conflicts.
2. Read the snapshot of database before startTs when there are no read/write conflicts.

Themis provides the guarantee to read all transactions with commitTs smaller than startTs, which is the snapshot of database before startTs.

**Conflict Resolution:**

There might be write/write and read/write conflicts as described above. Themis will use the timestamp saved in persistent lock to judge whether the conflict transaction is expired. If the conflict transaction is expired, the current transaction will do rollback or commit for the conflict transaction according to whether the primaryColumn of conflict transcation is committed; otherwise, the current transaction will fail.

Please see [google's percolator](http://research.google.com/pubs/pub36726.html) for more details.

### Themis Implementation

The implementation of Themis adopts the HBase coprocessor framework, the following picture gives the modules of Themis:

![themis_architecture](https://raw.githubusercontent.com/XiaoMi/themis/master/themis_architecture.png)

**Themis Client:**

1. Transaction: provides APIs of themis, including themisPut/themisGet/themisDelete/themisScan/commit.
2. MutationCache: index the mutations of users by rows in client side.
3. ThemisCoprocessorClient: the client to access themis coprocessor.
4. TimestampOracle: the client to query [chronos](https://github.com/XiaoMi/chronos), which will cache the timestamp requests and issue batch request to chronos in one rpc.
5. LockCleaner: resovle write/write conflict and read/write conflict.

Themis client will manage the users's mutations by row and invoke methods of ThemisCoprocessorClient to do prewrite/commit for each row.

**Themis Coprocessor:**

1. ThemisProtocol/ThemisCoprocessorImpl: definition and implementation of the themis coprocessor interfaces. The major interfaces are prewrite/commit/themisGet.
2. ThemisServerScanner/ThemisScanObserver: implement themis scan logic.
3. ThemisRegionObserver: single-row write optimization, add data clean filter to the scanner of flush and compaction.
4. ThemisMasterObserver: automatically add lock family when users creating table, set timestamp for data clean periodly.

## Usage 

### Loads themis coprocessor in server side 

1. add themis-coprocessor dependency in the pom of HBase:

     ```
     <dependency>
       <groupId>com.xiaomi.infra</groupId>
       <artifactId>themis-coprocessor</artifactId>
       <version>1.0-SNAPSHOT</version>
     </dependency>
     ```
                                          
2. add configurations for themis coprocessor in hbase-site.xml:

     ```
     <property>
       <name>hbase.coprocessor.user.region.classes</name>
       <value>org.apache.hadoop.hbase.themis.cp.ThemisProtocolImpl,org.apache.hadoop.hbase.themis.cp.ThemisScanObserver,org.apache.hadoop.hbase.regionserver.ThemisRegionObserver</value>
     </property>
     <property>
        <name>hbase.coprocessor.master.classes</name>
        <value>org.apache.hadoop.hbase.master.ThemisMasterObserver</value>
     </property>

     ```

3. For familiy needs themis, set THEMIS_ENABLE to 'true' by adding "CONFIG => {'THEMIS_ENABLE', 'true'}" to the family descriptor in table create script. 

### Server Side Settings

1. Timeout of transaction. The timeout of read/write transaction could be set by 'themis.read.transaction.ttl' and 'themis.write.transaction.ttl' respectively.

2. Data clean. Old data which could not be read any more will be cleaned periodly if 'themis.expired.data.clean.enable' is enable, and the clean period could be specified by 'themis.expired.timestamp.calculator.period'.

These settings could be set in hbase-site.xml of server-side.

### depends themis-client

add the themis-client dependency to pom of project which needs cross-row transactions.

     <dependency>
       <groupId>com.xiaomi.infra</groupId>
       <artifactId>themis-client</artifactId>
       <version>1.0-SNAPSHOT</version>
     </dependency>

### run example code

1. the master branch depends on hbase 0.94.21 with hadoop.version=2.0.0-alpha. We need download source code of hbase 0.94.21 and install in maven local repository by(in the directory of hbase 0.94.21):
   
     mvn clean install -DskipTests -Dhadoop.profile=2.0

2. install themis in maven local repository(in the directory of themis):

     mvn clean install -DskipTests

3. start a standalone HBase cluster(0.94.21 with hadoop.version=2.0.0-alpha) and make sure themis-coprocessor is loaded as above steps.

4. run "org.apache.hadoop.hbase.themis.example.Example.java" by:
     
     mvn exec:java -Dexec.mainClass="org.apache.hadoop.hbase.themis.example.Example"
  
The result of themisPut/themisGet/themisDelete/themisScan will output to screen.

Themis will use a LocalTimestampOracle class to provide incremental timestamp for threads in the same process. To use the global incremental timestamp from Chronos, we need the following steps and config:

1. config and start a Chronos cluster, please see : https://github.com/XiaoMi/themis/.

2. add the following config to the hbase-site.xml which located under the classpath of themis-client side:

     ```
     <property>
       <name>themis.timestamp.oracle.class</name>
	     <value>org.apache.hadoop.hbase.themis.timestamp.RemoteTimestampOracleProxy</value>
     </property>
     ```

With this config, themis will connect Chronos cluster in local machine. Then, run the example code as introduced above, and themis will use the timestamp from Chronos to do transactions(The Chronos cluster address could be configed by 'themis.remote.timestamp.server.zk.quorum' and cluster name could be configed by 'themis.remote.timestamp.server.clustername').

## Test 

### Correctness Validation

We design an AccountTransfer simulation program to validate the correctness of implementation. This program will distribute initial values in different tables, rows and columns in HBase. Each column represents an account. Then, configured client threads will be concurrently started to read out a number of account values from different tables and rows by themisGet. After this, clients will randomly transfer values among these accounts while keeping the sum unchanged, which simulates concurrent cross-table/cross-row transactions. To check the correctness of transactions, a checker thread will periodically scan account values from all columns, make sure the current total value is the same as the initial total value. We run this validation program for a period when releasing a new version for themis.

### Performance Test 

**Percolator Result:**

[percolator](http://research.google.com/pubs/pub36726.html) tests the read/write performance for single-column transaction(represents the worst case of percolator) and gives the relative drop compared to BigTable as follow table.

|             | BigTable  | Percolator       | Relative            |
|-------------|-----------|------------------|---------------------|
| Read/s      | 15513     | 14590            | 0.94                |
| Write/s     | 31003     | 7232             | 0.23                |

**Themis Result:**
We evaluate the performance of themis under similar test conditions with percolator's and give the relative drop compared to HBase.

Evaluation of themisGet. Load 10g data into HBase before testing themisGet by reading loaded rows: 

| Client Thread | GetCount | Themis AvgLatency(us) | HBase AvgLatency(us) | Relative |
|-------------  |--------- |-----------------------|----------------------|----------|
| 1             | 1000000  | 846.08                | 783.08               | 0.90     |
| 5             | 5000000  | 1125.95               | 1016.54              | 0.90     |
| 10            | 5000000  | 1513.61               | 1348.58              | 0.89     |
| 20            | 5000000  | 2639.60               | 2427.78              | 0.92     |
| 50            | 5000000  | 6295.83               | 5935.88              | 0.94     |


Evaluation of themisPut. Load 3,000,000 rows data into HBase before testing themisPut. We config 256M cache size to buffer locks for transaction. 

| Client Thread | PutCount  | Themis AvgLatency(us) | HBase AvgLatency(us) | Relative |
|-------------  |---------- |-----------------------|----------------------|----------|
| 1             | 3000000   | 1620.69               | 818.62               | 0.51     |
| 5             | 10000000  | 1695.89               | 1074.13              | 0.63     |
| 10            | 20000000  | 2057.55               | 1309.12              | 0.64     |
| 20            | 20000000  | 2761.66               | 1902.79              | 0.69     |
| 50            | 30000000  | 5441.48               | 3702.04              | 0.68     |

The above tests are all done in a single region server. From the results, we can see the performance of themisGet is 90% of HBase's get and the performance of themisPut is about 60% of HBase's put. For themisGet, the result is similar to that reported in [percolator](http://research.google.com/pubs/pub36726.html) paper. The themisPut performance is much better compared to that reported in [percolator](http://research.google.com/pubs/pub36726.html) paper. We optimize the performance of single-column transaction by the following skills:

1. In prewrite phase, we only write the lock to MemStore;  

2. In commit phase, we erase corresponding lock if it exist, write data and commit information at the same time.

The aboving skills make prewrite phase not write HLog, so that improving the write performance a lot for single-column transaction. After applying the skills, if region server restarts after prewrite phase, the commit phase can't read the persistent and the transaction will fail, this won't break correctness of the algorithm.

**ConcurrentThemis Result:**
The prewrite of different rows could be implemented concurrently, which could do cross-row transaction more efficiently. We use 'ConcurrentThemis' to represent the concurrent way and 'RawThemis' to represent the original way, then get the efficiency comparsion(we don't pre-load data before this comparsion because we focus on the relative improvement):

| TransactionSize | PutCount | RawThemis AvgTime(us) | ConcurrentThemis AvgTime(us) | Relative Improve |
|-----------------|--------- |-----------------------|------------------------------|------------------|
| 2               | 1000000  | 1654.14               | 995.98                       | 1.66             |
| 4               | 1000000  | 3233.11               | 1297.49                      | 2.50             |
| 8               | 1000000  | 6470.30               | 1963.47                      | 3.30             |
| 16              | 1000000  | 13301.50              | 2941.81                      | 4.52             |
| 32              | 600000   | 28151.37              | 3384.17                      | 7.25             |
| 64              | 400000   | 51658.58              | 5765.08                      | 8.96             |
| 128             | 200000   | 103289.95             | 11282.95                     | 9.15             |

TransactionSize is number of rows in one transaction. The 'Relative Improve' is 'RawThemis AvgTime(us)' / 'ConcurrentThemis AvgTime(us)'. We can see ConcurrentThemis performs much better as the TransactionSize increases, however, there is slowdown of improvement when the TransactionSize is bigger than 32.

## Future Works

1. Optimize the memory usage of RegionServer. Persistent locks of committed transactions should be removed from memory so that only need to keep persistent locks of un-committed transactions in memory.
2. A normal way to clear expired data for thmeis.
3. When reading from a transaction, merge the the local mutation of the transaction with committed transactions from server-side.
4. Resolve lock conflict more efficiently. Each client could register a temporary lock in Zookeeper, and the client will lose the lock after it fails. Then, other clients could know the failure client and clean lock more quickly.

---


# Themis 

## 简介

Themis在HBase上实现跨行、跨表事务，原理基于google提出的[percolator](http://research.google.com/pubs/pub36726.html)算法。

Themis以HBase行级别事务为基础，通过两阶段写和冲突约定(写写冲突、读写冲突)保证跨行事务的ACID特性。Themis依赖[chronos](https://github.com/XiaoMi/chronos)提供的全局严格单调递增timestamp服务为事务全局定序，确保读取某个timestamp之前数据库的snapshot。Themis利用了HBase coprocessor框架，不需要修改HBase代码，在server端加载themis coprocessor后即可服务。Themis提供与HBase类似的数据读写接口：themisPut/themisDelete/themisGet/themisScan。经过了几个月的正确性验证和性能测试，目前性能与[percolator](http://research.google.com/pubs/pub36726.html)论文中报告的结果相近。

## Themis API使用示例

### themisPut

    Configuration conf = HBaseConfiguration.create();
    HConnection connection = HConnectionManager.createConnection(conf);
    Transaction transaction = new Transaction(conf, connection);
    ThemisPut put = new ThemisPut(ROW).add(FAMILY, QUALIFIER, VALUE);
    transaction.put(TABLENAME, put);
    put = new ThemisPut(ANOTHER_ROW).add(FAMILY, QUALIFIER, VALUE);
    transaction.put(TABLENAME, put);
    transaction.commit();

transaction.commit()成功，会保证ROW和ANOTHER_ROW的修改同时成功，并且对读同时可见；commit失败，会保证ROW和ANOTHER_ROW的修改都失败，对读都不可见。

### themisDelete

     Transaction transaction = new Transaction(conf, connection);
     ThemisDelete delete = new ThemisDelete(ROW);
     delete.deleteColumn(FAMILY, QUALIFIER);
     transaction.delete(TABLENAME, delete);
     delete = new ThemisDelete(ANOTHER_ROW);
     delete.deleteColumn(FAMILY, QUALIFIER);
     transaction.delete(TABLENAME, delete);
     transaction.commit();

与themisPut类似，themisDelete可以保证ROW和ANOTHER_ROW的删除同时成功或者均不成功。

### themisGet

    Transaction transaction = new Transaction(conf, connection);
    ThemisGet get = new ThemisGet(ROW).addColumn(FAMILY, QUALIFIER);
    Result resultA = transaction.get(TABLENAME, get);
    get = new ThemisGet(ANOTHER_ROW).addColumn(FAMILY, QUALIFIER);
    Result resultB = transaction.get(TABLENAME, get);
    // 对于同一个transaction的themisGet, 可以确保从ROW和ANOTHER_ROW读出完整的事务

### themisScan

    Transaction transaction = new Transaction(conf, connection);
    ThemisScan scan = new ThemisScan();
    scan.addColumn(FAMILY, QUALIFIER);
    ThemisScanner scanner = transaction.getScanner(TABLENAME, scan);
    Result result = null;
    while ((result = scanner.next()) != null) {
      // 对于同一个transaction的themisScan，可以确保scanner返回完整的事务
      int value = Bytes.toInt(result.getValue(FAMILY, QUALIFIER));
    }
    scanner.close();

Themis在事务开始之前会从chronos取一个startTs，themis的读可以确保读取到数据库在startTs之前所有提交的事务。

更多示例代码参见：[Example.java](https://github.com/XiaoMi/themis/blob/master/themis-client/src/main/java/org/apache/hadoop/hbase/themis/example/Example.java)

## 原理和实现

### Themis原理

**Themis的写步骤：**

1. 在用户写中选取一个column做为primaryColumn，其余的column为secondaryColumns。Themis会为primaryColumn和scondaryColumn构建对应的持久化锁(persistentLock)信息。
2. 从chronos取全局时间prewriteTs，进行prewrite：在没有写冲突的情况下，写入数据和持久化锁，时间戳为prewriteTs。
3. prewrite成功后，从chronos取全局时间commitTs，对primaryColumn进行commit：需要确保其persistentLock没有被删除的情况下删除persistentLock并写入commit信息(时间戳为commitTs)。
4. primaryColumn提交成功后，开始提交secondaryColumn：删除persistentLock并写入commit信息(时间戳为commitTs)。

Themis是通过prewrite/commit两阶段写来完成事务。primaryColumn的commit成功后，事务整体成功，对读可见；否则事务整体失败，对读不可见。

**Themis读步骤：**

1. 从chronos取一个startTs，首先判断是否有读写冲突。
2. 如果没有读写冲突，读取timestamp < startTs的最新提交的事务。

Themis可以确保读取commitTs < startTs的所有已提交事物，即数据库在startTs之前的snapshot。

**Themis冲突解决：**

Themis可能会遇到写写冲突和读写冲突。解决冲突是根据存储在persistentLock中的时间戳判断冲突事务是否过期。如果过期，根据冲突事务的primaryColumn是否提交，回滚或提交事务；否则，当前事务失败。

更多原理细节参考：[google's percolator](http://research.google.com/pubs/pub36726.html)

### Themis实现

Themis的实现利用了HBase的coprocessor框架，其模块图为：
![themis模块图](https://raw.githubusercontent.com/XiaoMi/themis/master/themis_architecture.png)

**ThemisClient主要模块为：**

1. Transaction。提供Themis的API：themisPut/themisGet/themisDelete/themisScan。
2. MutationCache。将用户的修改按照row索引在client端。
3. ThemisCoprocessorClient。访问themis coprocessor的客户端。
4. TimestampOracle。访问chronos的客户端，可以将客户端对chronos的请求做batch，然后批量取回timestamp。
5. LockCleaner。负责解决写写冲突和读写冲突。

对于写事务，Themis将用户的mutations按照row进行索引，然后利用ThemisCoprocessorClient的接口进行prewrite/commit和读操作。

**ThemisCoprocessor主要模块为：**

1. ThemisProtocol/ThemisCoprocessorImpl。定义和实现Themis coprocessor接口，主要接口是prewrite/commit/themisGet。
2. ThemisServerScanner/ThemisScanObserver。实现themisScan逻辑。
3. ThemisRegionObserver: 单行写性能优化逻辑，在flush和compaction的过程中通过添加Filter来实现数据清理。
4. ThemisMasterObserver: 用户创建表的时候自动添加lock family, 周期性设置过期数据的timestamp。

## Themis使用

### Themis服务端
1. 需要在hbase的pom中引入对themis-coprocessor的依赖：

     ```
     <dependency>
       <groupId>com.xiaomi.infra</groupId>
       <artifactId>themis-coprocessor</artifactId>
       <version>1.0-SNAPSHOT</version>
     </dependency>
     ```
                                          
2. hbase的配置文件hbase-site.xml中加入themis-coprocessor的配置项：

     ```
     <property>
       <name>hbase.coprocessor.user.region.classes</name>
       <value>org.apache.hadoop.hbase.themis.cp.ThemisProtocolImpl,org.apache.hadoop.hbase.themis.cp.ThemisScanObserver,org.apache.hadoop.hbase.regionserver.ThemisRegionObserver</value>
     </property>
     <property>
        <name>hbase.coprocessor.master.classes</name>
        <value>org.apache.hadoop.hbase.master.ThemisMasterObserver</value>
     </property>
     ```

3. 对于需要使用themis的family，需要设置THEMIS_ENABLE属性为true，建表的时候可以在family descriptor中加入"CONGIG => {'THEMIS_ENABLE', 'true'}"。

### 服务器端设置 

1. 事务timeout设置:读写事务的timeout可以通过'themis.read.transaction.ttl'和'themis.write.transaction.ttl'分别设置。

2. 数据清理:如果设置了'themis.expired.data.clean.enable', 不再会被其它事务读取的旧数据会被周期性的清理；清理周期可以通过'themis.expired.timestamp.calculator.period'来设置。

上面的属性可以在server端的hbase-site.xml中进行设置。

### Themis客户端
需要在使用Themis的项目的pom中引入themis-client的依赖：

     <dependency>
       <groupId>com.xiaomi.infra</groupId>
       <artifactId>themis-client</artifactId>
       <version>1.0-SNAPSHOT</version>
     </dependency>

### 运行Example代码
1. 按照上面的Themis服务端步骤，启动hbase单机版本的服务器，确保themis-coprocessor被加载。

2. 在themis-client目录下运行：mvn exec:java -Dexec.mainClass="org.apache.hadoop.hbase.themis.example.Example" 可以看到ThemisPut/ThemisGet/ThemisScan/ThemisDelete的运行结果。

默认情况下，themis将使用LocalTimestampOracle提供进程级别单调递增的timestamp。使用Chronos提供全局单调递增的timestamp，需要:

1. 配置和启动Chronos Server，参见：https://github.com/XiaoMi/themis/。

2. 在client端的hbase-site.xml配置文件中加入：

     ```
     <property>
       <name>themis.timestamp.oracle.class</name>
	     <value>org.apache.hadoop.hbase.themis.timestamp.RemoteTimestampOracleProxy</value>
     </property>
     ```

这样Themis会默认访问本机启动的Chronos Server。可以通过themis.remote.timestamp.server.zk.quorum和themis.remote.timestamp.server.clustername两个参数来指定需要访问的Chronos集群。运行Example代码，themis将使用Chronos提供的timestamp来完成事务。


## 测试

### 正确性验证

我们设计了一个AccountTransfer程序对themis正确性进行验证。AccountTransfer模拟多个用户，每个用户在HBase的某个column下初始一个value，记录验证开始前的initTotal。验证开始后，会启动多个线程在选定的column之间进行value transfter，修改value的值，但逻辑上保持total value不变，用以模拟事物的并发运行。同时，会有一个TotalChecker线程，不断读出当前所有column的value，求和得到currentTotal，检查currentTotal=initTotal。另外，会在themis的主要步骤上随机抛出异常，使事务失败，测试themis解决冲突的逻辑。每次更新themis的实现后，都会运行AccountTransfer一段时间，确保themis逻辑正确。

### 性能测试

**Percolator性能**

[google's percolator](http://research.google.com/pubs/pub36726.html)测试了在单column情况下读写性能相对于BigTable的降低百分比：

|             | BigTable | Percolator       | Relative            |
|-------------|----------|------------------|---------------------|
| Read/s      | 15513    | 14590            | 0.94                |
| Write/s     | 31003    | 7232             | 0.23                |

与percolator类似，themis也对比了单column情况下读写性能相对于HBase的降低，我们结论如下：

**Themis性能**

1. themisGet对比，预写入10g数据，然后读出写入的数据。

| Client Thread | GetCount | Themis AvgLatency(us) | HBase AvgLatency(us) | Relative |
|-------------  |--------- |-----------------------|----------------------|----------|
| 1             | 1000000  | 846.08                | 783.08               | 0.90     |
| 5             | 5000000  | 1125.95               | 1016.54              | 0.90     |
| 10            | 5000000  | 1513.61               | 1348.58              | 0.89     |
| 20            | 5000000  | 2639.60               | 2427.78              | 0.92     |
| 50            | 5000000  | 6295.83               | 5935.88              | 0.94     |


2. themisPut对比，预写入10g数据，然后对其中的row进行更新，对比写性能。

| Client Thread | PutCount  | Themis AvgLatency(us) | HBase AvgLatency(us) | Relative |
|-------------  |---------- |-----------------------|----------------------|----------|
| 1             | 3000000   | 1620.69               | 818.62               | 0.51     |
| 5             | 10000000  | 1695.89               | 1074.13              | 0.63     |
| 10            | 20000000  | 2057.55               | 1309.12              | 0.64     |
| 20            | 20000000  | 2761.66               | 1902.79              | 0.69     |
| 50            | 30000000  | 5441.48               | 3702.04              | 0.68     |

上面结论都是在单region server上得出的。可以看出，themis的读性能相当与HBase的90%，与percolator论文中的结果类似；写性能在HBase的60%左右，比percolator论文中的结果好很多。对于单column的写，我们做了以下优化：

1. 在prewrite阶段，只写锁信息到MemStore中；

2. 在commit阶段，如果能够读到读应的锁，删除锁并将数据和commit信息同时写入。

使用上面的步骤，在prewrite阶段不需要写HLog，优化了写性能。在这种情况下，如果region server在prewrite阶段之后重启，commit阶段将读不到对应锁信息，事务将失败，但这对算法的正确性并没有影响。

**跨行写事物并行优化后的性能**

对于跨行写事物，prewrite可以并发的执行，在commitPrimary之后，也可以并发的commitSecondary。并发有利于提高写事务的性能。我们将使用并发写优化的Themis称为ConcurrentThemis，未使用并发的Themis称为RawThemis，两者的性能对比如下：

1. 单线程，跨行事物的性能对比(我们关注相对性能的提升，没有预写数据)：

| TransactionSize | PutCount | RawThemis AvgTime(us) | ConcurrentThemis AvgTime(us) | Relative Improve |
|-----------------|--------- |-----------------------|------------------------------|------------------|
| 2               | 1000000  | 1654.14               | 995.98                       | 1.66             |
| 4               | 1000000  | 3233.11               | 1297.49                      | 2.50             |
| 8               | 1000000  | 6470.30               | 1963.47                      | 3.30             |
| 16              | 1000000  | 13301.50              | 2941.81                      | 4.52             |
| 32              | 600000   | 28151.37              | 3384.17                      | 7.25             |
| 64              | 400000   | 51658.58              | 5765.08                      | 8.96             |
| 128             | 200000   | 103289.95             | 11282.95                     | 9.15             |

TransactionSize是事务的行数，我们关注使用并发后的相对性能提升：RelativeImprove = ConcurrentThemisAvgTime / RawThemisAvgTime。可以看出，随着TransactionSize的增长，ConcurrentThemis相对于RawThemis的latency的提升逐渐增加。事物行数超过32后，提升逐渐放缓。


## 将来的工作

1. RegionServer内存优化。可以将已经删除的Lock信息从MemStore中清掉，确保RegionServer内存中只有当前正在执行的事务。
2. 清理过期数据。
3. 读出当前事务未提交的写。对于当前事务，会将server端已提交的事务与本事务还没有commit的写进行合并，提供更合理的Snapshot。
4. 更有效的解决锁冲突。client向zookeeper注册，通过是否在zookeeper丢锁判定client是否退出，帮助更快的清理锁。
