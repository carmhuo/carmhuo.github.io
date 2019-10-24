---
layout: post
title: Delta Lake深入分析(4)
subtitle: Delta Lake write操作+源码分析
author: Carm
date: 2019-10-24 19:42:34 +0800
header-img: img/home-bg.jpg
categories: 大数据
tag:
  - spark
  - delta
---
# Delta Lake write操作

## 用法说明
Delta lake写数据方式与Spark写parquet方式基本一样，仅需要将`format`格式由parquet改为delta。

Delta写数据分为两种模式，一种为append，另一种为overwrite。

1. 使用append模式可以自动向已存在的表中添加数据，使用如下：

    `df.write.format("delta").mode("append").save("/delta/events")`

2. 如果需要自动替换表中所有数据，可以使用override模式，使用如下：

  `df.write.format("delta").mode("overwrite").save("/delta/events")`

  或，使用谓词匹配分区列来**覆盖分区数据**，下面的命令会覆盖掉`df`中一月份的数据。

  ```
  df.write
    .format("delta")
    .mode("overwrite")
    .option("replaceWhere", "date >= '2017-01-01' AND date <= '2017-01-31'")
    .save("/delta/events")
  ```

## 写操作事务日志格式
Delta Lake事务日志采用JSON格式，下面我们将1.parquet和2.parquet两个文件添加到Delta(新)表中，其事务日志文件为***000000.json***，其内容如下：
```
{"commitInfo":{"timestamp":1571142770378,"operation":“WRITE","operationParameters":{"numFiles":4,"partitionedBy":"[\"date\"]","collectStats":false}}}
{"protocol":{"minReaderVersion":1,"minWriterVersion":2}}
{"metaData":{"id":"0d5cde4d-cf8d-4481-a02b-1069f82aa7b4","format":{"provider":"parquet","options":{}},"schemaString":"{\"type\":\"struct\",\"fields\":[{\"name\":\"name\",\"type\":\"string\",\"nullable\":true,\"metadata\":{}},{\"name\":\"age\",\"type\":\"integer\",\"nullable\":true,\"metadata\":{}},{\"name\":\"company\",\"type\":\"string\",\"nullable\":true,\"metadata\":{}},{\"name\":\"favorite_color\",\"type\":\"string\",\"nullable\":true,\"metadata\":{}},{\"name\":\"job\",\"type\":\"string\",\"nullable\":true,\"metadata\":{}},{\"name\":\"date\",\"type\":\"date\",\"nullable\":true,\"metadata\":{}}]}","partitionColumns":["date"],"configuration":{},"createdTime":1571142770363}}
{"add":{"path":"date=2015-07-02/1.parquet","partitionValues":{"date":"2015-07-02"},"size":1399,"modificationTime":1571141933048,"dataChange":true}}
{"add":{"path":"date=2018-03-21/2.parquet","partitionValues":{"date":"2018-03-21"},"size":1408,"modificationTime":1571141933077,"dataChange":true}}
```
其中，事务日志文件`000000.json`中的数字部分`000000`表示事务的版本号，单调递增；

- commitInfo: 包括事务提交的时间，事务操作类型、操作参数等信息
- protocol: 最小的读、写协议版本，Delta通过切换软件协议来启用新特性
- metaData: 包括表schema，分区列、数据存储格式等内容
- add： 写入新parquet文件的事务日志

> NOTE：
只有**初次写数据**到Delta表 或**修改元数据信息和协议**时才会在日志中记录***protocol*** 和***metaData***信息，其他情况下，事务日志中将不会再重复出现这个两个参数信息

# 源码分析
Delta Lake写数据操作具体实现类为`WriteIntoDelta`，其继承了sparksql的`RunnableCommand`接口，核心实现为`run`方法,代码如下：
```
override def run(sparkSession: SparkSession): Seq[Row] = {
  // 创建一个新的事务
  deltaLog.withNewTransaction { txn =>
    // 写数据，返回事务原子操作类型集，如metaData、add等原子操作信息,这些信息稍后会在事务提交时写入到事务
    // 日志中
    val actions = write(txn, sparkSession)
    // 创建一个操作对象，在提交事务时会将该信息写到commintInfo中
    val operation = DeltaOperations.Write(mode, Option(partitionColumns), options.replaceWhere)
    // 提交事务
    txn.commit(actions, operation)
  }
  Seq.empty
}

def withNewTransaction[T](thunk: OptimisticTransaction => T): T = {
   try {
     // 更新当前表的快照
     update()
     val txn = new OptimisticTransaction(this)
     OptimisticTransaction.setActive(txn)
     thunk(txn)
   } finally {
     OptimisticTransaction.clearActive()
   }
 }
```
从上面代码来看，整个写操作流程为
1. 更新表当前事务快照(该块代码分析见[Delta Lake深入分析(3)--Delta事务源码分析](###))
2. 创建一个乐观事务对象，并设置当前线程中事务对象为活动状态（一个线程仅可有一个事务处于活动状态）
3. 开始写数据
4. 提交事务日志
5. 设置事务状态为结束


接下来我们看下写数据操作具体是如何实现的，代码如下：

```
def write(txn: OptimisticTransaction, sparkSession: SparkSession): Seq[Action] = {
  import sparkSession.implicits._
  if (txn.readVersion > -1) {
    // 表存在，检查插入是否合法
    if (mode == SaveMode.ErrorIfExists) {
      throw DeltaErrors.pathAlreadyExistsException(deltaLog.dataPath)
    } else if (mode == SaveMode.Ignore) {
      return Nil
    } else if (mode == SaveMode.Overwrite) {
      deltaLog.assertRemovable()
    }
  }
  // 更新元数据，下面会单独分析
  updateMetadata(txn, data, partitionColumns, configuration, isOverwriteOperation)

  // Validate partition predicates
  val replaceWhere = options.replaceWhere
  val partitionFilters = if (replaceWhere.isDefined) {
    // 存在分区列条件
    val predicates = parsePartitionPredicates(sparkSession, replaceWhere.get)
    if (mode == SaveMode.Overwrite) {
      verifyPartitionPredicates(
        sparkSession, txn.metadata.partitionColumns, predicates)
    }
    Some(predicates)
  } else {
    None
  }

  if (txn.readVersion < 0) {
    // 初次写数据，创建日志目录
    deltaLog.fs.mkdirs(deltaLog.logPath)
  }
  // 将数据写到新parquet文件，并返回新增文件的事务日志（AddFile）
  // 内部调用Spark的FileFormatWriter接口写parquet文件
  val newFiles = txn.writeFiles(data, Some(options))
  // 删除被覆盖的文件（逻辑删除），并返回事务日志（DeleteFile）
  val deletedFiles = (mode, partitionFilters) match {
    case (SaveMode.Overwrite, None) =>
      // 删除表所有文件
      txn.filterFiles().map(_.remove)
    case (SaveMode.Overwrite, Some(predicates)) =>
      // 删除指定分区的数据文件
      // Check to make sure the files we wrote out were actually valid.
      val matchingFiles = DeltaLog.filterFileList(
        txn.metadata.partitionColumns, newFiles.toDF(), predicates).as[AddFile].collect()
      val invalidFiles = newFiles.toSet -- matchingFiles
      if (invalidFiles.nonEmpty) {
        val badPartitions = invalidFiles
          .map(_.partitionValues)
          .map { _.map { case (k, v) => s"$k=$v" }.mkString("/") }
          .mkString(", ")
        throw DeltaErrors.replaceWhereMismatchException(replaceWhere.get, badPartitions)
      }
      txn.filterFiles(predicates).map(_.remove)
    case _ => Nil
  }
  // 返回新增和删除文件事务日志
  newFiles ++ deletedFiles
}
```
元数据更新操作`updateMetadata`主要支持：
- schema更新
- schema + partition列更新


源码如下：
```
protected final def updateMetadata(
    txn: OptimisticTransaction,
    data: Dataset[_],
    partitionColumns: Seq[String],
    configuration: Map[String, String],
    isOverwriteMode: Boolean): Unit = {
  val dataSchema = data.schema.asNullable
  val mergedSchema = if (isOverwriteMode && canOverwriteSchema) {
    dataSchema
  } else {
  /* mergeSchemas（tableSchema, dataSchema），合并策略为：
   * 1、如果dataSchema中列名在tableschema中存在，但是数据类型不同，则根据向
   *   上类型转换规则：如(ShortType, ByteType) => ShortType; 如果一方类型为
   *   NullType，一方不为空，合并后使用非空类型。
   * 2、如果dataSchema中的列名在tableschema中不存在，则在tableschema最后添加新列
   */
    SchemaUtils.mergeSchemas(txn.metadata.schema, dataSchema)
  }
  // 标准化分区列，并检查非法分区列
  val normalizedPartitionCols =
    normalizePartitionColumns(data.sparkSession, partitionColumns, dataSchema)
  // 判断schema是否合并
  def isNewSchema: Boolean = txn.metadata.schema != mergedSchema
  // We need to make sure that the partitioning order and naming is consistent
  // if provided. Otherwise we follow existing partitioning
  def isNewPartitioning: Boolean = normalizedPartitionCols.nonEmpty &&
    txn.metadata.partitionColumns != normalizedPartitionCols
  // 分区列合法性校验
  PartitionUtils.validatePartitionColumn(
    mergedSchema,
    normalizedPartitionCols,
    // Delta is case insensitive regarding internal column naming
    caseSensitive = false)

  if (txn.readVersion == -1) {
    // 建表后第一次写入
    if (dataSchema.isEmpty) {
      throw DeltaErrors.emptyDataException
    }
    recordDeltaEvent(txn.deltaLog, "delta.ddl.initializeSchema")
    // If this is the first write, configure the metadata of the table.
    // 更新txn元数据信息
    txn.updateMetadata(
    // 创建一个新的Metadata
      Metadata(
        schemaString = dataSchema.json,
        partitionColumns = normalizedPartitionCols,
        configuration = configuration))
  } else if (isOverwriteMode && canOverwriteSchema && (isNewSchema || isNewPartitioning)) {
    // Can define new partitioning in overwrite mode
    // 更新Metadata中的schema和partition列
    val newMetadata = txn.metadata.copy(
      schemaString = dataSchema.json,
      partitionColumns = normalizedPartitionCols
    )
    recordDeltaEvent(txn.deltaLog, "delta.ddl.overwriteSchema")
    txn.updateMetadata(newMetadata)
  } else if (isNewSchema && canMergeSchema && !isNewPartitioning) {
    logInfo(s"New merged schema: ${mergedSchema.treeString}")
    recordDeltaEvent(txn.deltaLog, "delta.ddl.mergeSchema")
    // 仅更新Metadata中的schema
    txn.updateMetadata(txn.metadata.copy(schemaString = mergedSchema.json))
  } else if (isNewSchema || isNewPartitioning) {
    // 异常处理
    recordDeltaEvent(txn.deltaLog, "delta.schemaValidation.failure")
    val errorBuilder = new MetadataMismatchErrorBuilder
    if (isNewSchema) {
      errorBuilder.addSchemaMismatch(txn.metadata.schema, dataSchema)
    }
    if (isNewPartitioning) {
      errorBuilder.addPartitioningMismatch(txn.metadata.partitionColumns, normalizedPartitionCols)
    }
    if (isOverwriteMode) {
      errorBuilder.addOverwriteBit()
    }
    errorBuilder.finalizeAndThrow()
  }
}

```
Delta Lake调用Spark的`FileFormatWriter`接口写parquet文件,源码如下：
```
def writeFiles(
    data: Dataset[_],
    writeOptions: Option[DeltaOptions],
    isOptimize: Boolean): Seq[AddFile] = {
  hasWritten = true

  val spark = data.sparkSession
  val partitionSchema = metadata.partitionSchema
  val outputPath = deltaLog.dataPath

  val (queryExecution, output) = normalizeData(data, metadata.partitionColumns)
  val partitioningColumns =
    getPartitioningColumns(partitionSchema, output, output.length < data.schema.size)

  val committer = getCommitter(outputPath)

  val invariants = Invariants.getFromSchema(metadata.schema, spark)
  // 设置一个新execution id
  SQLExecution.withNewExecutionId(spark, queryExecution) {
    val outputSpec = FileFormatWriter.OutputSpec(
      outputPath.toString,
      Map.empty,
      output)

    val physicalPlan = DeltaInvariantCheckerExec(queryExecution.executedPlan, invariants)

    FileFormatWriter.write(
      sparkSession = spark,
      plan = physicalPlan,
      fileFormat = snapshot.fileFormat, // ParquetFileFormat
      committer = committer,  //文件提交协议
      outputSpec = outputSpec,
      hadoopConf= spark.sessionState.newHadoopConfWithOptions(metadata.configuration),
      partitionColumns = partitioningColumns,
      bucketSpec = None,
      statsTrackers = Nil,
      options = Map.empty)
  }

  committer.addedStatuses
}
```
至此，Delta Lake写数据操作的所有源码都分析完了，总结一下：

**Delta Lake主要是在Spark写parquet数据上封装了一层事务操作，写数据操作会被记录到事务日志中，并进行持久化，以便后续Delta可以根据事务日志来构建表的状态。**

顺便说一句，在Delta中事务日志是非常重要的，表的一切操作都是基于事务日志，所以事务日志不能损坏或丢失，否则表中可能会出现脏数据。
