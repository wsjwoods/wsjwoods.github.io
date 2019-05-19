---
layout:     post
title:      "Presto的mysql驱动问题"
subtitle:   "源码中的isWrapperFor和abort函数不支持较老的mysql驱动"
date:       2019-05-17
author:     "Woods"
header-img: "img/post-bg-js-version.jpg"
tags:
    - Presto
    - 大数据
---

由于公司数据主要存在MPP数据库中（底层MYSQL分布式数据库集群），决定使用Presto做OLAP查询引擎。因为Presto支持非HIVE的外部数据源。

版本
```
presto-server-0.216
```

安装部署好之后，配置mpp的catalog

`vim ./etc/catalog/mpp.properties`
```
connector.name=mysql
connection-url=jdbc:mysql://ip:5258
connection-user=xxx
connection-password=xxx
```



进入Presto查询数据时报错：
```
Caused by: java.sql.SQLException: Unknown system variable 'transaction_isolation'
	at com.mysql.jdbc.SQLError.createSQLException(SQLError.java:964)
	at com.mysql.jdbc.MysqlIO.checkErrorPacket(MysqlIO.java:3973)
	at com.mysql.jdbc.MysqlIO.checkErrorPacket(MysqlIO.java:3909)
	at com.mysql.jdbc.MysqlIO.sendCommand(MysqlIO.java:2527)
	at com.mysql.jdbc.MysqlIO.sqlQueryDirect(MysqlIO.java:2680)
	at com.mysql.jdbc.ConnectionImpl.execSQL(ConnectionImpl.java:2483)
	at com.mysql.jdbc.ConnectionImpl.execSQL(ConnectionImpl.java:2441)
	at com.mysql.jdbc.StatementImpl.executeQuery(StatementImpl.java:1381)
	at com.mysql.jdbc.ConnectionImpl.loadServerVariables(ConnectionImpl.java:3790)
	at com.mysql.jdbc.ConnectionImpl.initializePropsFromServer(ConnectionImpl.java:3227)
	at com.mysql.jdbc.ConnectionImpl.connectWithRetries(ConnectionImpl.java:2060)
	... 46 more
```
根据报错显示的问题`Unknown system variable`

这个问题是因为数据库版本问题导致的，由于我们的mpp数据的mysql版本较低，而Presto默认使用的jdbc驱动在
`./plugin/mysql/mysql-connector-java-5.1.44.jar`

所以替换Presto集群全部的mysql驱动为平时开发用的
`mysql-connector-java-5.0.5.jar`

重启集群。

再次查询数据，又报错。

```
java.lang.AbstractMethodError: Method com/mysql/jdbc/PreparedStatement.isWrapperFor(Ljava/lang/Class;)Z is abstract
	at com.mysql.jdbc.PreparedStatement.isWrapperFor(PreparedStatement.java)
	at com.facebook.presto.plugin.mysql.MySqlClient.getPreparedStatement(MySqlClient.java:115)
	at com.facebook.presto.plugin.jdbc.QueryBuilder.buildSql(QueryBuilder.java:129)
	at com.facebook.presto.plugin.jdbc.BaseJdbcClient.buildSql(BaseJdbcClient.java:261)
	at com.facebook.presto.plugin.jdbc.JdbcRecordCursor.<init>(JdbcRecordCursor.java:90)
	at com.facebook.presto.plugin.jdbc.JdbcRecordSet.cursor(JdbcRecordSet.java:59)
	at com.facebook.presto.spi.RecordPageSource.<init>(RecordPageSource.java:37)
	at com.facebook.presto.split.RecordPageSourceProvider.createPageSource(RecordPageSourceProvider.java:42)
	at com.facebook.presto.split.PageSourceManager.createPageSource(PageSourceManager.java:56)
	at com.facebook.presto.operator.ScanFilterAndProjectOperator.getOutput(ScanFilterAndProjectOperator.java:221)
	at com.facebook.presto.operator.Driver.processInternal(Driver.java:379)
	at com.facebook.presto.operator.Driver.lambda$processFor$8(Driver.java:283)
	at com.facebook.presto.operator.Driver.tryWithLock(Driver.java:675)
	at com.facebook.presto.operator.Driver.processFor(Driver.java:276)
	at com.facebook.presto.execution.SqlTaskExecution$DriverSplitRunner.processFor(SqlTaskExecution.java:1077)
	at com.facebook.presto.execution.executor.PrioritizedSplitRunner.process(PrioritizedSplitRunner.java:162)
	at com.facebook.presto.execution.executor.TaskExecutor$TaskRunner.run(TaskExecutor.java:483)
	at com.facebook.presto.$gen.Presto_0_216____20190516_093528_1.run(Unknown Source)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)
```

然后查看Presto的源码发现有这样一段话：
```
public class MySqlClient
        extends BaseJdbcClient
{
...
@Override
    public PreparedStatement getPreparedStatement(Connection connection, String sql)
            throws SQLException
    {
        PreparedStatement statement = connection.prepareStatement(sql);
        if (statement.isWrapperFor(Statement.class)) {
            statement.unwrap(Statement.class).enableStreamingResults();
        }
        return statement;
    }
...

{

```
这是使用了`statement.isWrapperFor`方法。

而报错信息显示这个发放是abstract的，无法使用。

推断应该是较新的mysql-connector-java实现了`statement.isWrapperFor`方法。

而我们使用的5.0.5版本未实现这个方法。

所以决定再次调整mysql驱动版本。

通过测试选用`mysql-connector-java-5.1.1.jar`版本

`isWrapperFor`错误没有了。

执行查询又出现了新的错误：
```
java.lang.AbstractMethodError: com.mysql.jdbc.JDBC4Connection.abort(Ljava/util/concurrent/Executor;)V
	at com.facebook.presto.plugin.mysql.MySqlClient.abortReadConnection(MySqlClient.java:107)
	at com.facebook.presto.plugin.jdbc.JdbcRecordCursor.close(JdbcRecordCursor.java:218)
	at com.facebook.presto.spi.RecordPageSource.close(RecordPageSource.java:74)
	at com.facebook.presto.operator.TableScanOperator.finish(TableScanOperator.java:185)
	at com.facebook.presto.operator.TableScanOperator.close(TableScanOperator.java:174)
	at com.facebook.presto.operator.Driver.closeAndDestroyOperators(Driver.java:546)
	at com.facebook.presto.operator.Driver.processInternal(Driver.java:406)
	at com.facebook.presto.operator.Driver.lambda$processFor$8(Driver.java:283)
	at com.facebook.presto.operator.Driver.tryWithLock(Driver.java:675)
	at com.facebook.presto.operator.Driver.processFor(Driver.java:276)
	at com.facebook.presto.execution.SqlTaskExecution$DriverSplitRunner.processFor(SqlTaskExecution.java:1077)
	at com.facebook.presto.execution.executor.PrioritizedSplitRunner.process(PrioritizedSplitRunner.java:162)
	at com.facebook.presto.execution.executor.TaskExecutor$TaskRunner.run(TaskExecutor.java:483)
	at com.facebook.presto.$gen.Presto_0_216____20190517_030318_1.run(Unknown Source)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)
	Suppressed: java.sql.SQLException: Connection is read-only. Queries leading to data modification are not allowed.
		at com.mysql.jdbc.SQLError.createSQLException(SQLError.java:1056)
		at com.mysql.jdbc.SQLError.createSQLException(SQLError.java:957)
		at com.mysql.jdbc.SQLError.createSQLException(SQLError.java:927)
		at com.mysql.jdbc.StatementImpl.executeUpdate(StatementImpl.java:1322)
		at com.mysql.jdbc.StatementImpl.executeUpdate(StatementImpl.java:1289)
		at com.mysql.jdbc.RowDataDynamic.close(RowDataDynamic.java:191)
		at com.mysql.jdbc.ResultSetImpl.realClose(ResultSetImpl.java:7422)
		at com.mysql.jdbc.ResultSetImpl.close(ResultSetImpl.java:861)
		at com.facebook.presto.plugin.jdbc.JdbcRecordCursor.close(JdbcRecordCursor.java:219)
		... 15 more
```

显示`JDBC4Connection.abort`也是一个未实现的方法。

所以继续更换mysql驱动。

最终选用`mysql-connector-java-5.1.1.jar`能解决上面所有问题。


附：如果没有现有的驱动能解决以上问题，就只能修改Presto-mysql源码了。


