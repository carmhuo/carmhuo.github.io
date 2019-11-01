---
layout: post
title: Delta Lake深入分析(6)
subtitle: Delta Lake 删除操作源码分析.md
author: Carm
date: 2019-11-01 09:25:30 +0800
header-img: img/home-bg.jpg
categories: 大数据
tag:
  - spark
  - delta
---
# Delta Lake Update操作
## Scala语法示例
```
import io.delta.tables._

val deltaTable = DeltaTable.forPath(spark, "/data/events/")

deltaTable.updateExpr(            
  "eventType = 'clck'",           // predicate
  Map("eventType" -> "'click'")   // update expressions
)
```

## Update 操作流程

1. 扫描文件，找出与更新`predicate`条件匹配地候选文件集
2. 读取候选文件集，使用`update expressions`替换候选集中的旧值，并生成一个新的DataFrame
3. 使用Delta协议原子性的将新的DataFrame写入新文件，并删除步骤1中产生的候选文件集

## 源码分析
```
private def performUpdate(
    sparkSession: SparkSession, deltaLog: DeltaLog, txn: OptimisticTransaction): Unit = {
  import sparkSession.implicits._

  var numTouchedFiles: Long = 0
  var numRewrittenFiles: Long = 0
  var scanTimeMs: Long = 0
  var rewriteTimeMs: Long = 0

  val startTime = System.nanoTime()
  val numFilesTotal = deltaLog.snapshot.numOfFiles

  val updateCondition = condition.getOrElse(Literal(true, BooleanType))
  // 分离Partition条件和数据条件
  val (metadataPredicates, dataPredicates) =
    DeltaTableUtils.splitMetadataAndDataPredicates(
      updateCondition, txn.metadata.partitionColumns, sparkSession)
  // 扫描Delta表，返回与条件相匹配的候选文件集
  val candidateFiles = txn.filterFiles(metadataPredicates ++ dataPredicates)
  // 物理上数据文件路径与文件相对应操作（AddFile）的映射
  val nameToAddFile = generateCandidateFileMap(deltaLog.dataPath, candidateFiles)

  scanTimeMs = (System.nanoTime() - startTime) / 1000 / 1000

  val actions: Seq[Action] = if (candidateFiles.isEmpty) {
    // Case 1: Do nothing if no row qualifies the partition predicates
    // that are part of Update condition
    // 候选文件集为空，不进行任何事务操作
    Nil
  } else if (dataPredicates.isEmpty) {
    // Case 2: Update all the rows from the files that are in the specified partitions
    // when the data filter is empty
    // 仅Partition分区更新
    numTouchedFiles = candidateFiles.length
    // 缓存待重写文件路径
    val filesToRewrite = candidateFiles.map(_.path)
    val operationTimestamp = System.currentTimeMillis()
    // 标记待候选文件为删除状态，并返回删除事件RemoveFile
    val deleteActions = candidateFiles.map(_.removeWithTimestamp(operationTimestamp))
    // 扫描所有候选文件，将更新后的数据写到新文件中
    val rewrittenFiles = rewriteFiles(sparkSession, txn, tahoeFileIndex.path,
      filesToRewrite, nameToAddFile, updateCondition)
    // 记录新生成文件的数量
    numRewrittenFiles = rewrittenFiles.size
    rewriteTimeMs = (System.nanoTime() - startTime) / 1000 / 1000 - scanTimeMs
    // 删除和新建文件事件集合
    deleteActions ++ rewrittenFiles
  } else {
    // Case 3: Find all the affected files using the user-specified condition
    // 创建一个FileIndex对象，保存与条件相匹配的文件路径
    // 使用非Partition过滤字段，需进行全表扫描
    val fileIndex = new TahoeBatchFileIndex(
      sparkSession, "update", candidateFiles, deltaLog, tahoeFileIndex.path, txn.snapshot)
    // Keep everything from the resolved target except a new TahoeFileIndex
    // that only involves the affected files instead of all files.
    // 使用TahoeBatchFileIndex对象替换掉HadoopFsRelation中的FileIndex对象
    // ，并生成新的逻辑计划
    val newTarget = DeltaTableUtils.replaceFileIndex(target, fileIndex)
    // 生成一个DataFrame
    val data = Dataset.ofRows(sparkSession, newTarget)
    // 获取待重写文件
    val filesToRewrite =
      withStatusCode("DELTA", s"Finding files to rewrite for UPDATE operation") {
        data.filter(new Column(updateCondition)).select(input_file_name())
          .distinct().as[String].collect()
      }

    scanTimeMs = (System.nanoTime() - startTime) / 1000 / 1000
    // 待重写文件数量
    numTouchedFiles = filesToRewrite.length

    if (filesToRewrite.isEmpty) {
      // Case 3.1: Do nothing if no row qualifies the UPDATE condition
      Nil
    } else {
      // Case 3.2: Delete the old files and generate the new files containing the updated
      // values
      val operationTimestamp = System.currentTimeMillis()
      // 将待重写文件标记为已删除状态
      val deleteActions =
        removeFilesFromPaths(deltaLog, nameToAddFile, filesToRewrite, operationTimestamp)
      val rewrittenFiles =
        withStatusCode("DELTA", s"Rewriting ${filesToRewrite.size} files for UPDATE operation") {
          // 将更新后的数据重写到新文件
          rewriteFiles(sparkSession, txn, tahoeFileIndex.path,
            filesToRewrite, nameToAddFile, updateCondition)
        }

      numRewrittenFiles = rewrittenFiles.size
      rewriteTimeMs = (System.nanoTime() - startTime) / 1000 / 1000 - scanTimeMs
      // 返回事务操作
      deleteActions ++ rewrittenFiles
    }
  }

  if (actions.nonEmpty) {
    // 提交事务，并将事务保存到文件，详细过程可以参考事务日志分析部分
    txn.commit(actions, DeltaOperations.Update(condition.map(_.toString)))
  }
  // 统计信息
  recordDeltaEvent(
    deltaLog,
    "delta.dml.update.stats",
    data = UpdateMetric(
      condition = condition.map(_.sql).getOrElse("true"),
      numFilesTotal,
      numTouchedFiles,
      numRewrittenFiles,
      scanTimeMs,
      rewriteTimeMs)
  )
}
```
我们接着分析下，写新文件的源码。
```
private def rewriteFiles(
    spark: SparkSession,
    txn: OptimisticTransaction,
    rootPath: Path,  // 数据根目录
    inputLeafFiles: Seq[String],  //待重写文件
    nameToAddFileMap: Map[String, AddFile],  // 候选文件-->AddFile映射
    // 更新条件
    condition: Expression): Seq[AddFile] = {

  // Containing the map from the relative file path to AddFile
  // 构造一个BaseRelation--- HadoopFsRelation
  val baseRelation = buildBaseRelation(
    spark, txn, "update", rootPath, inputLeafFiles, nameToAddFileMap)
  // 使用TahoeBatchFileIndex替换target逻辑计划中的FileIndex，该文件索引包含所有候选文件的位置
  val newTarget = DeltaTableUtils.replaceFileIndex(target, baseRelation.location)
  // 将候选文件集加载到Saprk DF中
  val targetDf = Dataset.ofRows(spark, newTarget)
  // 构造一个更新后的df
  val updatedDataFrame = {
    // 使用更新表达式创建新的project
    val updatedColumns = buildUpdatedColumns(condition)
    // 获得更新后的df
    targetDf.select(updatedColumns: _*)
  }
  // 将更新后的df写到新文件中
  txn.writeFiles(updatedDataFrame)
}

```
上面代码中创建了一个`BaseRelation`对象，该对象中最重要的就是构建一个`TahoeBatchFileIndex`对象，并替换原有spark逻辑计划中的`FileIndex`，用来过滤文件。
```
protected def buildBaseRelation(
      spark: SparkSession,
      txn: OptimisticTransaction,
      actionType: String,
      rootPath: Path,
      inputLeafFiles: Seq[String],
      nameToAddFileMap: Map[String, AddFile]): HadoopFsRelation = {
    val deltaLog = txn.deltaLog
    val scannedFiles = inputLeafFiles.map(f => getTouchedFile(rootPath, f, nameToAddFileMap))
    // 重要！！！  构建一个FileIndex对象，包含数据过滤条件
    val fileIndex = new TahoeBatchFileIndex(
      spark, actionType, scannedFiles, deltaLog, rootPath, txn.snapshot)
    HadoopFsRelation(
      fileIndex,
      partitionSchema = txn.metadata.partitionSchema,
      dataSchema = txn.metadata.schema,
      bucketSpec = None,
      deltaLog.snapshot.fileFormat,
      txn.metadata.format.options)(spark)
  }
```

使用更新表达式替换旧值的代码比较简单，主要是使用条件表达式构造一个新列，判断记录是否需要更新，源码如下：
```
private def buildUpdatedColumns(condition: Expression): Seq[Column] = {
    updateExpressions.zip(target.output).map { case (update, original) =>
      // 构造一个`If`表达式, "_FUNC_(expr1, expr2, expr3)" -
      // If `expr1` evaluates to true, then returns `expr2`;
      // otherwise returns `expr3`.
      val updated = If(condition, update, original)
      // 构造一个新列
      new Column(Alias(updated, original.name)())
    }
}
```

总的来说， Delta Lake Update操作流程还是比较清晰的，大家好好阅读应该没有什么技术障碍问题。哈哈，后期我将会分析下Delta Lake的Merge操作，敬请期待！
