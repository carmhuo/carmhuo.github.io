---
layout: post
title: Delta Lake深入分析(5)
subtitle: Delta Lake Delete操作源码分析
author: Carm
date: 2019-10-25 19:03:34 +0800
header-img: img/home-bg.jpg
categories: 大数据
tag:
  - spark
  - delta
---
# Delta Lake Delete操作

## Scala语法示例
```
import io.delta.tables._

val deltaTable = DeltaTable.forPath(spark, "/data/events/")

deltaTable.delete("date = '2017-01-01' AND id < 14")        // predicate using SQL formatted string
```
执行删除命令，Delta Lake会生成一个事物日志，内容大致如下：
```
{"commitInfo":{"timestamp":1566978478414,"operation":"DELETE","operationParameters":{"predicate":"[\"(`date` < \"2017-01-01\")\",\" (`id` < CAST('14' AS BIGINT))\ "]"},"readVersion":10,"isBlindAppend":false}}
{"remove":{"path":"date=2017-01-01/part-00000-ca73a0f4-fbeb-4ea8-9b9f-fa466a85724e.c000.snappy.parquet","deletionTimestamp":1566978478405,"dataChange":true}}
{"remove":{"path":"date=2017-01-01/part-00000-8e11f4cc-a7ac-47a1-8ce6-b9d87eaf6c51.c000.snappy.parquet","deletionTimestamp":1566978478405,"dataChange":true}}
{"add":{"path":"date=2017-01-01/part-00001-6ff11be3-22db-4ed2-bde3-a97d610fe11d.c000.snappy.parquet","partitionValues":{"date":"2017-01-01"},"size":429,"modificationTime":1566978478000,"dataChange":true}}
```
>重要：
 Delta Lake删除表中最新版本的数据时，不会删除物理存储数据，只将删除事件记录到事物日志里面，除非使用`vacuum`命令手动删除。

## 删除操作详细过程
Delta Lake删除实现主要分为三种情况：
1. 如果 Delete 时不指定删除条件，Delta Lake将删除全表数据。这种情况处理比较简单，只需要直接删除Delta Lake 表对应的所有文件即可；
2. 如果执行 Delete 的时候指定了相关删除条件，且删除条件只包含分区字段，比如 date 是 Delta Lake 表的分区字段，然后我们执行了 deltaTable.delete("date = '2017-01-01'") 这样相关的删除操作，那么我们可以直接从缓存在内存中的快照（snapshot）拿到需要删除的文件，直接删除即可，而且不需要执行数据重写操作。
3. 最后一种情况，用户删除的时候含有一些非分区字段的过滤条件，这时候我们就需要进行全表扫描，获取需要删除的数据在哪些文件里面，这又分两种情况：
    1. Delta Lake 表并不存在我们需要删除的数据，这时候不需要做任何操作，直接返回，就连事务日志都不用记录；
    2. 这种情况比较复杂，我们需要计算需要删除的数据在哪个文件里面（**非常耗时**），然后把对应的文件里面不需要删除的数据重写到新的文件里面（如果没有，就不生成新文件），最后记录事务日志。

>建议： 删除条件最好指定分区字段，避免全表扫描

**删除操作流程图如下所示。**

![](/img/delta/Delta-delete-progress.jpg)

## 源码分析
删除操作的实现类为`DeleteCommand`,集成Spark sql的`RunnableCommand`,下面主要看下其`run`方法：
```
final override def run(sparkSession: SparkSession): Seq[Row] = {
  recordDeltaOperation(tahoeFileIndex.deltaLog, "delta.dml.delete") {
    val deltaLog = tahoeFileIndex.deltaLog
    // 判断数据是否可删除
    deltaLog.assertRemovable()
    // 开始一个新的事务处理
    deltaLog.withNewTransaction { txn =>
      performDelete(sparkSession, deltaLog, txn)
    }
    // Re-cache all cached plans(including this relation itself, if it's cached) that refer to
    // this data source relation.
    sparkSession.sharedState.cacheManager.recacheByPlan(sparkSession, target)
  }

  Seq.empty[Row]
}
```
这段代码和我们之前的分析的写入过程差不多，详情请看[Delta Lake深入分析(4)--Delta Lake write操作+源码分析](###), 在这里我们就着重分析下`performDelete`方法。
```
private def performDelete(
      sparkSession: SparkSession, deltaLog: DeltaLog, txn: OptimisticTransaction) = {
    import sparkSession.implicits._

    var numTouchedFiles: Long = 0
    var numRewrittenFiles: Long = 0
    var scanTimeMs: Long = 0
    var rewriteTimeMs: Long = 0

    val startTime = System.nanoTime()
    val numFilesTotal = deltaLog.snapshot.numOfFiles

    val deleteActions: Seq[Action] = condition match {
      case None =>
        // Case 1: 无删除条件下，删除整张表
        val allFiles = txn.filterFiles(Nil)

        numTouchedFiles = allFiles.size
        scanTimeMs = (System.nanoTime() - startTime) / 1000 / 1000

        val operationTimestamp = System.currentTimeMillis()
        allFiles.map(_.removeWithTimestamp(operationTimestamp))
      case Some(cond) =>
        val (metadataPredicates, otherPredicates) =
          DeltaTableUtils.splitMetadataAndDataPredicates(
            cond, txn.metadata.partitionColumns, sparkSession)

        if (otherPredicates.isEmpty) {
          // Case 2: 删除条件中只有分区字段，删除文件集不需要扫描整个数据文件
          val operationTimestamp = System.currentTimeMillis()
          val candidateFiles = txn.filterFiles(metadataPredicates)

          scanTimeMs = (System.nanoTime() - startTime) / 1000 / 1000
          numTouchedFiles = candidateFiles.size

          candidateFiles.map(_.removeWithTimestamp(operationTimestamp))
        } else {
          // Case 3: 删除条件中存在非分区字段，需要进行全表扫描.
          val candidateFiles = txn.filterFiles(metadataPredicates ++ otherPredicates)

          numTouchedFiles = candidateFiles.size
          val nameToAddFileMap = generateCandidateFileMap(deltaLog.dataPath, candidateFiles)

          val fileIndex = new TahoeBatchFileIndex(
            sparkSession, "delete", candidateFiles, deltaLog, tahoeFileIndex.path, txn.snapshot)
          // Keep everything from the resolved target except a new TahoeFileIndex
          // that only involves the affected files instead of all files.
          val newTarget = DeltaTableUtils.replaceFileIndex(target, fileIndex)
          val data = Dataset.ofRows(sparkSession, newTarget)
          // 找到与删除条件匹配的文件集
          val filesToRewrite =
            withStatusCode("DELTA", s"Finding files to rewrite for DELETE operation") {
              if (numTouchedFiles == 0) {
                Array.empty[String]
              } else {
                data.filter(new Column(cond)).select(new Column(InputFileName())).distinct()
                  .as[String].collect()
              }
            }

          scanTimeMs = (System.nanoTime() - startTime) / 1000 / 1000
          if (filesToRewrite.isEmpty) {
            // Case 3.1: 匹配文件集为空，则不触发删除操作
            Nil
          } else {
            // Case 3.2: 重写文件
            val baseRelation = buildBaseRelation(
              sparkSession, txn, "delete", tahoeFileIndex.path, filesToRewrite, nameToAddFileMap)
            // Keep everything from the resolved target except a new TahoeFileIndex
            // that only involves the affected files instead of all files.
            val newTarget = DeltaTableUtils.replaceFileIndex(target, baseRelation.location)
            // 从待删除文件中获取非删除的数据，主要是将删除条件取反
            val targetDF = Dataset.ofRows(sparkSession, newTarget)
            val filterCond = Not(EqualNullSafe(cond, Literal(true, BooleanType)))
            val updatedDF = targetDF.filter(new Column(filterCond))
            // 写新文件
            val rewrittenFiles = withStatusCode(
              "DELTA", s"Rewriting ${filesToRewrite.size} files for DELETE operation") {
              txn.writeFiles(updatedDF)
            }
            numRewrittenFiles = rewrittenFiles.size
            rewriteTimeMs = (System.nanoTime() - startTime) / 1000 / 1000 - scanTimeMs
            val operationTimestamp = System.currentTimeMillis()
            //DELETE Action ++ Add Action
            removeFilesFromPaths(deltaLog, nameToAddFileMap, filesToRewrite, operationTimestamp) ++
              rewrittenFiles
          }
        }
    }
    if (deleteActions.nonEmpty) {
      // 如果有删除行为，则提交事务日志
      txn.commit(deleteActions, DeltaOperations.Delete(condition.map(_.sql).toSeq))
    }
    // Metrics信息，Delta Lake0.4暂时未使用该信息
    recordDeltaEvent(
      deltaLog,
      "delta.dml.delete.stats",
      data = DeleteMetric(
        condition = condition.map(_.sql).getOrElse("true"),
        numFilesTotal,
        numTouchedFiles,
        numRewrittenFiles,
        scanTimeMs,
        rewriteTimeMs)
    )
}
```

到这里， Delta Lake删除操作就分析完了，通过源码来看，其实现思想还是比较简单的。接下来，我将会对Delta Lake的Update操作进行源码分析，敬请期待。
