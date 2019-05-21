---
layout:     post
title:      "spark外部数据源之JDBC源码解读"
subtitle:   "九步拆解spark读取外部数据源"
date:       2019-05-21
author:     "Woods"
header-img: "img/post-bg-js-version.jpg"
tags:
    - Spark
    - SparkSql
---

@[toc]
## 环境：
    spark 2.4.0

本文通过源码中JDBC的读取数据库源码解读spark对于外部数据源的处理。

spark读取jdbc数据方法参照[官方网站](http://spark.apache.org/docs/latest/sql-data-sources-jdbc.html)：
```java
val jdbcDF = spark.read
  .format("jdbc")
  .option("url", "jdbc:postgresql:dbserver")
  .option("dbtable", "schema.tablename")
  .option("user", "username")
  .option("password", "password")
  .load()
```

## 第一步：DataFrameReader.format
```java
class DataFrameReader private[sql](sparkSession: SparkSession) extends Logging {
//...
def format(source: String): DataFrameReader = {
    this.source = source \\ "jdbc"
    this
  }
}
//...
```
指定source="jdbc"
## 第二步：DataFrameReader.option
```java
class DataFrameReader private[sql](sparkSession: SparkSession) extends Logging {
//...
def option(key: String, value: String): DataFrameReader = {
    this.extraOptions += (key -> value)
    this
  }
//...
}  
```
将option中的参数（url,user,pwd等）加入到`extraOptions：scala.collection.mutable.HashMap[String, String]` 中

## 第三步：DataFrameReader.load
```java
class DataFrameReader private[sql](sparkSession: SparkSession) extends Logging {
//...
  def load(): DataFrame = {
    load(Seq.empty: _*) // force invocation of `load(...varargs...)`
  }
//...java
def load(paths: String*): DataFrame = {
    if (source.toLowerCase(Locale.ROOT) == DDLUtils.HIVE_PROVIDER) {
      throw new AnalysisException("Hive data source can only be used with tables, you can not " +
        "read files of Hive data source directly.")
    }

    val cls = DataSource.lookupDataSource(source, sparkSession.sessionState.conf)
    if (classOf[DataSourceV2].isAssignableFrom(cls)) {
      val ds = cls.newInstance().asInstanceOf[DataSourceV2]
      if (ds.isInstanceOf[ReadSupport]) {
        val sessionOptions = DataSourceV2Utils.extractSessionConfigs(
          ds = ds, conf = sparkSession.sessionState.conf)
        val pathsOption = {
          val objectMapper = new ObjectMapper()
          DataSourceOptions.PATHS_KEY -> objectMapper.writeValueAsString(paths.toArray)
        }
        Dataset.ofRows(sparkSession, DataSourceV2Relation.create(
          ds, sessionOptions ++ extraOptions.toMap + pathsOption,
          userSpecifiedSchema = userSpecifiedSchema))
      } else {
        loadV1Source(paths: _*)
      }
    } else {
      loadV1Source(paths: _*)
    }
  }

}  
```
`load()`调用`load(paths: String*)`方法

cls通过`DataSource.lookupDataSource`方法传入一个source，这里是"jdbc"得到`"org.apache.spark.sql.execution.datasources.jdbc.JdbcRelationProvider"`

`if (classOf[DataSourceV2].isAssignableFrom(cls))`判断为false

跳转到最后的`loadV1Source(paths: _*)` 这是的paths="Nil$"

## 第四步：DataFrameReader.loadV1Source()
```java
class DataFrameReader private[sql](sparkSession: SparkSession) extends Logging {
//...
  private def loadV1Source(paths: String*) = {
    // Code path for data source v1.
    sparkSession.baseRelationToDataFrame(
      DataSource.apply(
        sparkSession,
        paths = paths,
        userSpecifiedSchema = userSpecifiedSchema,
        className = source,
        options = extraOptions.toMap).resolveRelation())
  }
//...
    
}
```
这里会构建一个`DataSource`对象
```java
case class DataSource(
    sparkSession: SparkSession,
    className: String, \\"jdbc"
    paths: Seq[String] = Nil,
    userSpecifiedSchema: Option[StructType] = None,
    partitionColumns: Seq[String] = Seq.empty,
    bucketSpec: Option[BucketSpec] = None,
    options: Map[String, String] = Map.empty,
    catalogTable: Option[CatalogTable] = None) extends Logging 
    //...
    private val caseInsensitiveOptions = CaseInsensitiveMap(options) \\实现序列化的Map
```
```java
DataSource对象属性值是：
className = "jdbc"
options = user,pwd，url,dbtale
userSpecifiedSchema = None
caseInsensitiveOptions(实现序列化的Map) = options
```

然后调用`DataSource.resolveRelation()`方法

## 第五步：DataSource.resolveRelation()
```java
case class DataSource(//...){
//...
def resolveRelation(checkFilesExist: Boolean = true): BaseRelation = {
    val relation = (providingClass.newInstance(), userSpecifiedSchema) match {
      // TODO: Throw when too much is given.
      case (dataSource: SchemaRelationProvider, Some(schema)) =>
        dataSource.createRelation(sparkSession.sqlContext, caseInsensitiveOptions, schema)
      case (dataSource: RelationProvider, None) =>
        dataSource.createRelation(sparkSession.sqlContext, caseInsensitiveOptions)
      case (_: SchemaRelationProvider, None) =>
        throw new AnalysisException(s"A schema needs to be specified when using $className.")
      case (dataSource: RelationProvider, Some(schema)) =>
        val baseRelation =
          dataSource.createRelation(sparkSession.sqlContext, caseInsensitiveOptions)
        if (baseRelation.schema != schema) {
          throw new AnalysisException(s"$className does not allow user-specified schemas.")
        }
        baseRelation

      // We are reading from the results of a streaming query. Load files from the metadata log
      // instead of listing them using HDFS APIs.
      case (format: FileFormat, _)
          if FileStreamSink.hasMetadata(
            caseInsensitiveOptions.get("path").toSeq ++ paths,
            sparkSession.sessionState.newHadoopConf()) =>
        val basePath = new Path((caseInsensitiveOptions.get("path").toSeq ++ paths).head)
        val fileCatalog = new MetadataLogFileIndex(sparkSession, basePath, userSpecifiedSchema)
        val dataSchema = userSpecifiedSchema.orElse {
          format.inferSchema(
            sparkSession,
            caseInsensitiveOptions,
            fileCatalog.allFiles())
        }.getOrElse {
          throw new AnalysisException(
            s"Unable to infer schema for $format at ${fileCatalog.allFiles().mkString(",")}. " +
                "It must be specified manually")
        }

        HadoopFsRelation(
          fileCatalog,
          partitionSchema = fileCatalog.partitionSchema,
          dataSchema = dataSchema,
          bucketSpec = None,
          format,
          caseInsensitiveOptions)(sparkSession)

      // This is a non-streaming file based datasource.
      case (format: FileFormat, _) =>
        val globbedPaths =
          checkAndGlobPathIfNecessary(checkEmptyGlobPath = true, checkFilesExist = checkFilesExist)
        val useCatalogFileIndex = sparkSession.sqlContext.conf.manageFilesourcePartitions &&
          catalogTable.isDefined && catalogTable.get.tracksPartitionsInCatalog &&
          catalogTable.get.partitionColumnNames.nonEmpty
        val (fileCatalog, dataSchema, partitionSchema) = if (useCatalogFileIndex) {
          val defaultTableSize = sparkSession.sessionState.conf.defaultSizeInBytes
          val index = new CatalogFileIndex(
            sparkSession,
            catalogTable.get,
            catalogTable.get.stats.map(_.sizeInBytes.toLong).getOrElse(defaultTableSize))
          (index, catalogTable.get.dataSchema, catalogTable.get.partitionSchema)
        } else {
          val index = createInMemoryFileIndex(globbedPaths)
          val (resultDataSchema, resultPartitionSchema) =
            getOrInferFileFormatSchema(format, Some(index))
          (index, resultDataSchema, resultPartitionSchema)
        }

        HadoopFsRelation(
          fileCatalog,
          partitionSchema = partitionSchema,
          dataSchema = dataSchema.asNullable,
          bucketSpec = bucketSpec,
          format,
          caseInsensitiveOptions)(sparkSession)

      case _ =>
        throw new AnalysisException(
          s"$className is not a valid Spark SQL Data Source.")
    }

    relation match {
      case hs: HadoopFsRelation =>
        SchemaUtils.checkColumnNameDuplication(
          hs.dataSchema.map(_.name),
          "in the data schema",
          equality)
        SchemaUtils.checkColumnNameDuplication(
          hs.partitionSchema.map(_.name),
          "in the partition schema",
          equality)
        DataSourceUtils.verifyReadSchema(hs.fileFormat, hs.dataSchema)
      case _ =>
        SchemaUtils.checkColumnNameDuplication(
          relation.schema.map(_.name),
          "in the data schema",
          equality)
    }

    relation
  }

    //...
}  
```
#### resolveRelation第一部分
```java
val relation = (providingClass.newInstance(), userSpecifiedSchema) match {
      // TODO: Throw when too much is given.
      case (dataSource: SchemaRelationProvider, Some(schema)) =>
        dataSource.createRelation(sparkSession.sqlContext, caseInsensitiveOptions, schema)
      case (dataSource: RelationProvider, None) =>
        dataSource.createRelation(sparkSession.sqlContext, caseInsensitiveOptions)
      case (_: SchemaRelationProvider, None) =>
        throw new AnalysisException(s"A schema needs to be specified when using $className.")
      case (dataSource: RelationProvider, Some(schema)) =>
        val baseRelation =
          dataSource.createRelation(sparkSession.sqlContext, caseInsensitiveOptions)
        if (baseRelation.schema != schema) {
          throw new AnalysisException(s"$className does not allow user-specified schemas.")
        }
        baseRelation
```
providingClass.newInstance()通过source="jdbc"得到的是JdbcRelationProvider,dataSource的类就是：`JdbcRelationProvider`

userSpecifiedSchema = NONE

所以匹配到第二个case
```java
case (dataSource: RelationProvider, None) =>
        dataSource.createRelation(sparkSession.sqlContext, caseInsensitiveOptions)
```

然后进入到`JdbcRelationProvider`的`createRelation`方法中。

## 第六步：==JdbcRelationProvider==.createRelation
```java
class JdbcRelationProvider extends CreatableRelationProvider
  with RelationProvider with DataSourceRegister {

  override def shortName(): String = "jdbc"

  override def createRelation(
      sqlContext: SQLContext,
      parameters: Map[String, String]): BaseRelation = {
    val jdbcOptions = new JDBCOptions(parameters)
    val resolver = sqlContext.conf.resolver
    val timeZoneId = sqlContext.conf.sessionLocalTimeZone
    val schema = JDBCRelation.getSchema(resolver, jdbcOptions)
    val parts = JDBCRelation.columnPartition(schema, resolver, timeZoneId, jdbcOptions)
    JDBCRelation(schema, parts, jdbcOptions)(sqlContext.sparkSession)
  }

      //...
  }
```
这一步是创建jdbc连接的重要部分。

> 1. jdbcOptions：jdbc连接等信息。
> 2. schema：获取要读的表的schema信息。
> 3. parts：获取分区信息。
> 4. 最后返回一个JDBCRelation对象。


来看看是怎么获得Schema的

getSchema
```java
def getSchema(resolver: Resolver, jdbcOptions: JDBCOptions): StructType = {
    val tableSchema = JDBCRDD.resolveTable(jdbcOptions)
    jdbcOptions.customSchema match {
      case Some(customSchema) => JdbcUtils.getCustomSchema(
        tableSchema, customSchema, resolver)
      case None => tableSchema
    }
  }
```
JDBCRDD.resolveTable
```java
def resolveTable(options: JDBCOptions): StructType = {
    val url = options.url
    val table = options.tableOrQuery
    val dialect = JdbcDialects.get(url)
    val conn: Connection = JdbcUtils.createConnectionFactory(options)()
    try {
      val statement = conn.prepareStatement(dialect.getSchemaQuery(table))
      try {
        statement.setQueryTimeout(options.queryTimeout)
        val rs = statement.executeQuery()
        try {
          JdbcUtils.getSchema(rs, dialect, alwaysNullable = true)
        } finally {
          rs.close()
        }
      } finally {
        statement.close()
      }
    } finally {
      conn.close()
    }
  }
```
通过option获取数据库连接
dialect.getSchemaQuery(table)
```java
 def getSchemaQuery(table: String): String = {
    s"SELECT * FROM $table WHERE 1=0"
  }
```

```java
def getSchema(
      resultSet: ResultSet,
      dialect: JdbcDialect,
      alwaysNullable: Boolean = false): StructType = {
    val rsmd = resultSet.getMetaData
    val ncols = rsmd.getColumnCount
    val fields = new Array[StructField](ncols)
    var i = 0
    while (i < ncols) {
      val columnName = rsmd.getColumnLabel(i + 1)
      val dataType = rsmd.getColumnType(i + 1)
      val typeName = rsmd.getColumnTypeName(i + 1)
      val fieldSize = rsmd.getPrecision(i + 1)
      val fieldScale = rsmd.getScale(i + 1)
      val isSigned = {
        try {
          rsmd.isSigned(i + 1)
        } catch {
          // Workaround for HIVE-14684:
          case e: SQLException if
          e.getMessage == "Method not supported" &&
            rsmd.getClass.getName == "org.apache.hive.jdbc.HiveResultSetMetaData" => true
        }
      }
      val nullable = if (alwaysNullable) {
        true
      } else {
        rsmd.isNullable(i + 1) != ResultSetMetaData.columnNoNulls
      }
      val metadata = new MetadataBuilder().putLong("scale", fieldScale)
      val columnType =
        dialect.getCatalystType(dataType, typeName, fieldSize, metadata).getOrElse(
          getCatalystType(dataType, fieldSize, fieldScale, isSigned))
      fields(i) = StructField(columnName, columnType, nullable)
      i = i + 1
    }
    new StructType(fields)
  }
```

使用`"SELECT * FROM $table WHERE 1=0"`获得`statement`，
然后使用`JdbcUtils.getSchema`来获取schema信息。

最后返回一个JDBCRelation
```java
private[sql] case class JDBCRelation(
    override val schema: StructType,
    parts: Array[Partition],
    jdbcOptions: JDBCOptions)(@transient val sparkSession: SparkSession)
  extends BaseRelation
  with PrunedFilteredScan
  with InsertableRelation {
  
  }
```

## 第七步：DataSource.resolveRelation
#### resolveRelation第二部分
```java
def resolveRelation(checkFilesExist: Boolean = true): BaseRelation = {
//...
   relation match {
      case hs: HadoopFsRelation =>
        SchemaUtils.checkColumnNameDuplication(
          hs.dataSchema.map(_.name),
          "in the data schema",
          equality)
        SchemaUtils.checkColumnNameDuplication(
          hs.partitionSchema.map(_.name),
          "in the partition schema",
          equality)
        DataSourceUtils.verifyReadSchema(hs.fileFormat, hs.dataSchema)
      case _ =>
        SchemaUtils.checkColumnNameDuplication(
          relation.schema.map(_.name),
          "in the data schema",
          equality)
    }
    
    relation
  }
```
之前得到relation = `JDBCRelation(user) [numPartitions=1]`

`SchemaUtils.checkColumnNameDuplication`检查是否有重复的列名称。

最后返回这个`relation`

## 第八步：DataFrameReader.loadV1Source返回一个DateFrame
第四步的loadV1Source
```java
class DataFrameReader private[sql](sparkSession: SparkSession) extends Logging {
//...
  private def loadV1Source(paths: String*) = {
    // Code path for data source v1.
    sparkSession.baseRelationToDataFrame(
      DataSource.apply(
        sparkSession,
        paths = paths,
        userSpecifiedSchema = userSpecifiedSchema,
        className = source,
        options = extraOptions.toMap).resolveRelation())
  }
//...
    
}
```

使用baseRelationToDataFrame返回一个了DataFrame
```java
def baseRelationToDataFrame(baseRelation: BaseRelation): DataFrame = {
    Dataset.ofRows(self, LogicalRelation(baseRelation))
  }
```

main代码中jdbcDF = 这个DataFrame

## 第九步：jdbcDF.show -> ==JDBCRelation==
show会生成LogicalPlan

```java
case class DataSourceAnalysis(conf: SQLConf) extends Rule[LogicalPlan] with CastSupport {

def apply(plan: LogicalPlan): Seq[execution.SparkPlan] = plan match {
    case PhysicalOperation(projects, filters, l @ LogicalRelation(t: CatalystScan, _, _, _)) =>
      pruneFilterProjectRaw(
        l,
        projects,
        filters,
        (requestedColumns, allPredicates, _) =>
          toCatalystRDD(l, requestedColumns, t.buildScan(requestedColumns, allPredicates))) :: Nil

    case PhysicalOperation(projects, filters,
                           l @ LogicalRelation(t: PrunedFilteredScan, _, _, _)) =>
      pruneFilterProject(
        l,
        projects,
        filters,
        (a, f) => toCatalystRDD(l, a, t.buildScan(a.map(_.name).toArray, f))) :: Nil
//...

val unhandledFilters = relation.unhandledFilters(translatedMap.values.toArray).toSet

//...
```
DataSourceAnalysis分别调用了`JDBCRelation`类中的`unhandledFilters`过滤
```java
override def unhandledFilters(filters: Array[Filter]): Array[Filter] = {
    if (jdbcOptions.pushDownPredicate) {
      filters.filter(JDBCRDD.compileFilter(_, JdbcDialects.get(jdbcOptions.url)).isEmpty)
    } else {
      filters
    }
  }
```
和`buildScan`创建查询
```java
override def buildScan(requiredColumns: Array[String], filters: Array[Filter]): RDD[Row] = {
    // Rely on a type erasure hack to pass RDD[InternalRow] back as RDD[Row]
    JDBCRDD.scanTable(
      sparkSession.sparkContext,
      schema,
      requiredColumns,
      filters,
      parts,
      jdbcOptions).asInstanceOf[RDD[Row]]
  }
```
JDBCRDD.scanTable
```java
def scanTable(
      sc: SparkContext,
      schema: StructType,
      requiredColumns: Array[String],
      filters: Array[Filter],
      parts: Array[Partition],
      options: JDBCOptions): RDD[InternalRow] = {
    val url = options.url
    val dialect = JdbcDialects.get(url)
    val quotedColumns = requiredColumns.map(colName => dialect.quoteIdentifier(colName))
    new JDBCRDD(
      sc,
      JdbcUtils.createConnectionFactory(options),
      pruneSchema(schema, requiredColumns),
      quotedColumns,
      filters,
      parts,
      url,
      options)
  }
```
new JDBCRDD
```java
 /**
   * `columns`, but as a String suitable for injection into a SQL query.
   */
  private val columnList: String = {
    val sb = new StringBuilder()
    columns.foreach(x => sb.append(",").append(x))
    if (sb.isEmpty) "1" else sb.substring(1)
  }

  /**
   * `filters`, but as a WHERE clause suitable for injection into a SQL query.
   */
  private val filterWhereClause: String =
    filters
      .flatMap(JDBCRDD.compileFilter(_, JdbcDialects.get(url)))
      .map(p => s"($p)").mkString(" AND ")

  /**
   * A WHERE clause representing both `filters`, if any, and the current partition.
   */
  private def getWhereClause(part: JDBCPartition): String = {
    if (part.whereClause != null && filterWhereClause.length > 0) {
      "WHERE " + s"($filterWhereClause)" + " AND " + s"(${part.whereClause})"
    } else if (part.whereClause != null) {
      "WHERE " + part.whereClause
    } else if (filterWhereClause.length > 0) {
      "WHERE " + filterWhereClause
    } else {
      ""
    }
  }
```
JDBCRDD中就是使用拼接字符串生成sql语句并通过connection来执行，获取数据的。



---
欢迎访问[个人博客](https://wsjwoods.github.io/)！