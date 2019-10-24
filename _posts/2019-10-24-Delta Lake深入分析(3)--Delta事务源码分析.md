---
layout: post
title: Delta Lake深入分析(3)
subtitle: Delta Lake事务源码分析
author: Carm
date: 2019-10-24 10:34:30 +0800
header-img: img/home-bg.jpg
categories: 大数据
tag:
  - spark
  - delta
---
# Delta事物日志源码分析
之前，我们讲了Delta Lake事务实现原理，可以点击[这里](??????)查看。接下来，我们具体看看Delta Lake源码实现，本次使用当前最新版本***Delta Lake 0.4***进行分析。

Delta Lake事务入口类为：**OptimisticTransaction**，事务的提交主要由`commit`方法完成，接下来我们主要看下该方法是如何实现的。

```
def commit(actions: Seq[Action], op: DeltaOperations.Operation): Long = recordDeltaOperation(
    deltaLog, "delta.commit") {
  val version = try {
    // Try to commit at the next version.
    // 事务提交合法性检查
    var finalActions = prepareCommit(actions, op)
    // 提交的事务是否全部为AddFile
    val isBlindAppend = {
      val onlyAddFiles =
        finalActions.collect { case f: FileAction => f }.forall(_.isInstanceOf[AddFile])
      onlyAddFiles && !dependsOnFiles
    }
    //是否需要提交CommintInfo日志
    if (spark.sessionState.conf.getConf(DeltaSQLConf.DELTA_COMMIT_INFO_ENABLED)) {
      commitInfo = CommitInfo(
        clock.getTimeMillis(),
        op.name,
        op.jsonEncodedValues,
        Map.empty,
        Some(readVersion).filter(_ >= 0),
        None,
        Some(isBlindAppend))
      finalActions = commitInfo +: finalActions
    }
    // 提交事务，返回事务最新版本号
    val commitVersion = doCommit(snapshot.version + 1, finalActions, 0)
    logInfo(s"Committed delta #$commitVersion to ${deltaLog.logPath}")
    // 检查是否进行checkpoint
    postCommit(commitVersion, finalActions)
    commitVersion
  } catch {
    case e: DeltaConcurrentModificationException =>
      recordDeltaEvent(deltaLog, "delta.commit.conflict." + e.conflictType)
      throw e
    case NonFatal(e) =>
      recordDeltaEvent(
        deltaLog, "delta.commit.failure", data = Map("exception" -> Utils.exceptionString(e)))
      throw e
  }
  version
}
```
Delta Lake事务提交主要分三个步骤：

1. **prepareCommit**： 主要做一些提交前的准备工作，验证事务日志的合法性等
2. **doCommit**: 使用乐观方式提交事务，如果发现写入冲突，会进行版本更新并重试。重试的代码逻辑见checkAndRetry方法。
3. **postCommit**: 收尾操作，主要检查是否需要进行checkpoint操作

下面具体分析下这三个步骤。


prepareCommit代码如下：
```
protected def prepareCommit(actions: Seq[Action], op: DeltaOperations.Operation): Seq[Action] = {

  assert(!committed, "Transaction already committed.")

  // If the metadata has changed, add that to the set of actions
  var finalActions = newMetadata.toSeq ++ actions
// 确保一个事务中元数据变化不能超过一次
  val metadataChanges = finalActions.collect { case m: Metadata => m }
  assert(
    metadataChanges.length <= 1,
    "Cannot change the metadata more than once in a transaction.")
  metadataChanges.foreach(m => verifyNewMetadata(m))

  // If this is the first commit and no protocol is specified, initialize the protocol version.
  if (snapshot.version == -1) {
    // 第一次提交时，确保日志目录存在
    deltaLog.ensureLogDirectoryExist()
    // 第一次提交时，初始化Protocol,并添加到事务列表中
    if (!finalActions.exists(_.isInstanceOf[Protocol])) {
      finalActions = Protocol() +: finalActions
    }
  }
  // 第一次提交时，将全局配置合并到Metadata中
  finalActions = finalActions.map {
    // Fetch global config defaults for the first commit
    case m: Metadata if snapshot.version == -1 =>
      val updatedConf = DeltaConfigs.mergeGlobalConfigs(
        spark.sessionState.conf, m.configuration, Protocol())
      m.copy(configuration = updatedConf)
    case other => other
  }
  // 检查协议版本
  deltaLog.protocolWrite(
    snapshot.protocol,
    logUpgradeMessage = !actions.headOption.exists(_.isInstanceOf[Protocol]))

  // We make sure that this isn't an appendOnly table as we check if we need to delete
  // files.
// 检查RemoveFile文件是否可删除
  val removes = actions.collect { case r: RemoveFile => r }
  if (removes.exists(_.dataChange)) deltaLog.assertRemovable()

  finalActions
}
```
doCommit代码如下：
```
private def doCommit(
    attemptVersion: Long,
    actions: Seq[Action],
    attemptNumber: Int): Long = deltaLog.lockInterruptibly {
  try {
    logDebug(s"Attempting to commit version $attemptVersion with ${actions.size} actions")
    // 事务日志持久化，默认将事务日志文件写到HDFS，如果写入冲突（并发写问题），则抛出异常，统一由catch
    // 中checkAndRetry方法进行冲突处理。
    deltaLog.store.write(
      deltaFile(deltaLog.logPath, attemptVersion),
      actions.map(_.json).toIterator)
    val commitTime = System.nanoTime()
    // 更新当前快照（因为刚写了的新的delta日志文件，需更新内存日志快照）
    val postCommitSnapshot = deltaLog.update()
    if (postCommitSnapshot.version < attemptVersion) {
      throw new IllegalStateException(
        s"The committed version is $attemptVersion " +
          s"but the current version is ${postCommitSnapshot.version}.")
    }

    // 一些指标信息
    var numAbsolutePaths = 0
    var pathHolder: Path = null
    val distinctPartitions = new mutable.HashSet[Map[String, String]]
    val adds = actions.collect {
      case a: AddFile =>
        pathHolder = new Path(new URI(a.path))
        if (pathHolder.isAbsolute) numAbsolutePaths += 1
        distinctPartitions += a.partitionValues
        a
    }
    val stats = CommitStats(
      startVersion = snapshot.version,
      commitVersion = attemptVersion,
      readVersion = postCommitSnapshot.version,
      txnDurationMs = NANOSECONDS.toMillis(commitTime - txnStartNano),
      commitDurationMs = NANOSECONDS.toMillis(commitTime - commitStartNano),
      numAdd = adds.size,
      numRemove = actions.collect { case r: RemoveFile => r }.size,
      bytesNew = adds.filter(_.dataChange).map(_.size).sum,
      numFilesTotal = postCommitSnapshot.numOfFiles,
      sizeInBytesTotal = postCommitSnapshot.sizeInBytes,
      protocol = postCommitSnapshot.protocol,
      info = Option(commitInfo).map(_.copy(readVersion = None, isolationLevel = None)).orNull,
      newMetadata = newMetadata,
      numAbsolutePaths,
      numDistinctPartitionsInAdd = distinctPartitions.size,
      isolationLevel = null)
    recordDeltaEvent(deltaLog, "delta.commit.stats", data = stats)
    // 返回提交版本
    attemptVersion
  } catch {
    case e: java.nio.file.FileAlreadyExistsException =>
      // 主要解决事务提交版本冲突问题，Delta Lake并发写事务日志采用MVCC机制，如果写文件时遇到版本冲突
      // (即写入的文件已存在,DeltaLake已版本号作为文件名)，则更新事务日志后重新进行提交
      checkAndRetry(attemptVersion, actions, attemptNumber)
  }
}
```
下面我们看下checkAndRetry方法的代码：
```
protected def checkAndRetry(checkVersion: Long,actions: Seq[Action],attemptNumber: Int): Long =  recordDeltaOperation(
      deltaLog,
      "delta.commit.retry",
      tags = Map(TAG_LOG_STORE_CLASS -> deltaLog.store.getClass.getName)) {
  // 当前提交版本落后事务日志库版本，需要更新事务日志
  deltaLog.update()
  // 重新设置提交版本号，最新版本+1
  val nextAttempt = deltaLog.snapshot.version + 1
  // 更新版本后，检查是否有逻辑上的冲突。若有冲突，则直接抛出异常，事务提交失败
  (checkVersion until nextAttempt).foreach { version =>
    val winningCommitActions =
      deltaLog.store.read(deltaFile(deltaLog.logPath, version)).map(Action.fromJson)
    val metadataUpdates = winningCommitActions.collect { case a: Metadata => a }
    val txns = winningCommitActions.collect { case a: SetTransaction => a }
    val protocol = winningCommitActions.collect { case a: Protocol => a }
    val commitInfo = winningCommitActions.collectFirst { case a: CommitInfo => a }.map(
      ci => ci.copy(version = Some(version)))
    val fileActions = winningCommitActions.collect { case f: FileAction => f }
    // If the log protocol version was upgraded, make sure we are still okay.
    // Fail the transaction if we're trying to upgrade protocol ourselves.
    if (protocol.nonEmpty) {
      protocol.foreach { p =>
        deltaLog.protocolRead(p)
        deltaLog.protocolWrite(p)
      }
      actions.foreach {
        case Protocol(_, _) => throw new ProtocolChangedException(commitInfo)
        case _ =>
      }
    }
    // Fail if the metadata is different than what the txn read.
    if (metadataUpdates.nonEmpty) {
      throw new MetadataChangedException(commitInfo)
    }
    // Fail if the data is different than what the txn read.
    if (dependsOnFiles && fileActions.nonEmpty) {
      throw new ConcurrentWriteException(commitInfo)
    }
    // Fail if idempotent transactions have conflicted.
    val txnOverlap = txns.map(_.appId).toSet intersect readTxn.toSet
    if (txnOverlap.nonEmpty) {
      throw new ConcurrentTransactionException(commitInfo)
    }
  }
  logInfo(s"No logical conflicts with deltas [$checkVersion, $nextAttempt), retrying.")
  // 如果没有发生逻辑冲突，则重新提交事务
  doCommit(nextAttempt, actions, attemptNumber + 1)
}
```

最后，我们看下postCommit方法代码：
```
protected def postCommit(commitVersion: Long, commitActions: Seq[Action]): Unit = {
  committed = true
  // checkpointInterval系统默认为10（可修改），也就是每提交十次事务进行一次checkpoint操作
  if (commitVersion != 0 && commitVersion % deltaLog.checkpointInterval == 0) {
    try {
      deltaLog.checkpoint()
    } catch {
      case e: IllegalStateException =>
        logWarning("Failed to checkpoint table state.", e)
    }
  }
}
```
到这里，Delta Lake的事务提交的一个完整流程就结束了。


不过，通过阅读上面的代码发现，还有一个比较重要的内容没有分析，那就是***内存快照更新***（`deltaLog.update()`），在之前的[Delta Lake事务实现原理](????)中我们分析了快照更新操作原理，接下来我主要看下这段代码是如何实现。
```
/*事务日志更新*/
def update(stalenessAcceptable: Boolean = false): Snapshot = {
    val doAsync = stalenessAcceptable && !isSnapshotStale
    if (!doAsync) {
      lockInterruptibly {
        // 同步更新
        updateInternal(isAsync = false)
      }
    } else {
      if (asyncUpdateTask == null || asyncUpdateTask.isCompleted) {
        val jobGroup = spark.sparkContext.getLocalProperty(SparkContext.SPARK_JOB_GROUP_ID)
        asyncUpdateTask = Future[Unit] {
          spark.sparkContext.setLocalProperty("spark.scheduler.pool", "deltaStateUpdatePool")
          spark.sparkContext.setJobGroup(
            jobGroup,
            s"Updating state of Delta table at ${currentSnapshot.path}",
            interruptOnCancel = true)
          tryUpdate(isAsync = true)
        }(DeltaLog.deltaLogAsyncUpdateThreadPool)
      }
      currentSnapshot
    }
  }
```
事务提交过程中使用同步更新日志，具体代码如下：
```
private def updateInternal(isAsync: Boolean): Snapshot =
  recordDeltaOperation(this, "delta.log.update", Map(TAG_ASYNC -> isAsync.toString)) {
  withStatusCode("DELTA", "Updating the Delta table's state") {
    try {
      // 查找版本号大于当前版本的checkpoint文件和Delta文件
      val newFiles = store
        // List from the current version since we want to get the checkpoint file for the current
        // version
        .listFrom(checkpointPrefix(logPath, math.max(currentSnapshot.version, 0L)))
        // Pick up checkpoint files not older than the current version and delta files newer than
        // the current version
        .filter { file =>
          isCheckpointFile(file.getPath) ||
            (isDeltaFile(file.getPath) && deltaVersion(file.getPath) > currentSnapshot.version)
      }.toArray
      // 分离checkpoint文件和delta文件
      val (checkpoints, deltas) = newFiles.partition(f => isCheckpointFile(f.getPath))
      // 如果delta文件为空，则当前版本就是最新版本，不需要更新，直接返回。
      if (deltas.isEmpty) {
        lastUpdateTimestamp = clock.getTimeMillis()
        return currentSnapshot
      }
      val deltaVersions = deltas.map(f => deltaVersion(f.getPath))
      // 确保版本号连续
      verifyDeltaVersions(deltaVersions)
      val lastChkpoint = lastCheckpoint.map(CheckpointInstance.apply)
          .getOrElse(CheckpointInstance.MaxValue)
      val checkpointFiles = checkpoints.map(f => CheckpointInstance(f.getPath))
      // 获取最新的checkpoint实例
      val newCheckpoint = getLatestCompleteCheckpointFromList(checkpointFiles, lastChkpoint)
      val newSnapshot = if (newCheckpoint.isDefined) {
        // 如果存在新的checkpoint，则从checkpoit点开始构建新的快照
        val newCheckpointVersion = newCheckpoint.get.version
        assert(
          newCheckpointVersion >= currentSnapshot.version,
          s"Attempting to load a checkpoint($newCheckpointVersion) " +
              s"older than current version (${currentSnapshot.version})")
        val newCheckpointFiles = newCheckpoint.get.getCorrespondingFiles(logPath)

        val newVersion = deltaVersions.last
        val deltaFiles =
          ((newCheckpointVersion + 1) to newVersion).map(deltaFile(logPath, _))

        logInfo(s"Loading version $newVersion starting from checkpoint $newCheckpointVersion")

        new Snapshot(
          logPath,
          newVersion,
          None,
          newCheckpointFiles ++ deltaFiles,
          minFileRetentionTimestamp,
          this,
          deltas.last.getModificationTime)
      } else {
        // 如果不存在新的checkpoint，则将新的delta日志文件应用到当前快照
        assert(currentSnapshot.version + 1 == deltaVersions.head,
          s"versions in [${currentSnapshot.version + 1}, ${deltaVersions.head}) are missing")
        if (currentSnapshot.lineageLength >= maxSnapshotLineageLength) {
          // Load Snapshot from scratch to avoid StackOverflowError
          // 如果当前snapshot血统长度大于50（默认值）,则清除之前的血统，重新从版本0开始构建一个snapshot
          getSnapshotAt(deltaVersions.last, Some(deltas.last.getModificationTime))
        } else {
          // 否则，使用当前的snapshot构造一个新的snapshot，并将血统长度+1
          new Snapshot(
            logPath,
            deltaVersions.last,
            Some(currentSnapshot.state),
            deltas.map(_.getPath),
            minFileRetentionTimestamp,
            this,
            deltas.last.getModificationTime,
            lineageLength = currentSnapshot.lineageLength + 1)
        }
      }
      validateChecksum(newSnapshot)
      // 清除之前快照缓存
      currentSnapshot.uncache()
      currentSnapshot = newSnapshot
    } catch {
      case f: FileNotFoundException =>
        val message = s"No delta log found for the Delta table at $logPath"
        logInfo(message)
        // When the state is empty, this is expected. The log will be lazily created when needed.
        // When the state is not empty, it's a real issue and we can't continue to execution.
        if (currentSnapshot.version != -1) {
          val e = new FileNotFoundException(message)
          e.setStackTrace(f.getStackTrace())
          throw e
        }
    }
    lastUpdateTimestamp = clock.getTimeMillis()
    currentSnapshot
  }
}
```

到此，Delta Lake事务相关实现已全部讲完。接下来，我会继续分析其DML(Update/Delete/Merge)实现源码。
