[TOC]



# StreamingQueryManager

SparkSession -> SessionState -> StreamingQueryManager



```scala
private[sql] class SessionState(
  ......
  // The streamingQueryManager is lazy to avoid creating a StreamingQueryManager for each session
  // when connecting to ThriftServer.
  lazy val streamingQueryManager: StreamingQueryManager = streamingQueryManagerBuilder()
```



=>

org.apache.spark.sql.streaming.DataStreamWriter#startQuery

```scala
  private def startQuery(
      sink: Table,
      newOptions: CaseInsensitiveMap[String],
      recoverFromCheckpoint: Boolean = true,
      catalogAndIdent: Option[(TableCatalog, Identifier)] = None,
      catalogTable: Option[CatalogTable] = None): StreamingQuery = {
    val useTempCheckpointLocation = SOURCES_ALLOW_ONE_TIME_QUERY.contains(source)    

    df.sparkSession.sessionState.streamingQueryManager.startQuery(
      newOptions.get("queryName"),
      newOptions.get("checkpointLocation"),
      df,
      newOptions.originalMap,
      sink,
      outputMode,
      useTempCheckpointLocation = useTempCheckpointLocation,
      recoverFromCheckpointLocation = recoverFromCheckpoint,
      trigger = trigger,
      catalogAndIdent = catalogAndIdent,
      catalogTable = catalogTable)
  }
```



org.apache.spark.sql.streaming.StreamingQueryManager#startQuery

```scala
  /**
   * Start a [[StreamingQuery]].
   *
   * @param userSpecifiedName Query name optionally specified by the user.
   * @param userSpecifiedCheckpointLocation  Checkpoint location optionally specified by the user.
   * @param df Streaming DataFrame.
   * @param sink  Sink to write the streaming outputs.
   * @param outputMode  Output mode for the sink.
   * @param useTempCheckpointLocation  Whether to use a temporary checkpoint location when the user
   *                                   has not specified one. If false, then error will be thrown.
   * @param recoverFromCheckpointLocation  Whether to recover query from the checkpoint location.
   *                                       If false and the checkpoint location exists, then error
   *                                       will be thrown.
   * @param trigger [[Trigger]] for the query.
   * @param triggerClock [[Clock]] to use for the triggering.
   * @param catalogAndIdent Catalog and identifier for the sink, set when it is a V2 catalog table
   */
  @throws[TimeoutException]
  private[sql] def startQuery(
      userSpecifiedName: Option[String],
      userSpecifiedCheckpointLocation: Option[String],
      df: DataFrame,
      extraOptions: Map[String, String],
      sink: Table,
      outputMode: OutputMode,
      useTempCheckpointLocation: Boolean = false,
      recoverFromCheckpointLocation: Boolean = true,
      trigger: Trigger = Trigger.ProcessingTime(0),
      triggerClock: Clock = new SystemClock(),
      catalogAndIdent: Option[(TableCatalog, Identifier)] = None,
      catalogTable: Option[CatalogTable] = None): StreamingQuery = {
    val query = createQuery(
      userSpecifiedName,
      userSpecifiedCheckpointLocation,
      df,
      extraOptions,
      sink,
      outputMode,
      useTempCheckpointLocation,
      recoverFromCheckpointLocation,
      trigger,
      triggerClock,
      catalogAndIdent,
      catalogTable)
    // scalastyle:on argcount

    // The following code block checks if a stream with the same name or id is running. Then it
    // returns an Option of an already active stream to stop outside of the lock
    // to avoid a deadlock.
    val activeRunOpt = activeQueriesSharedLock.synchronized {
      // Make sure no other query with same name is active
      userSpecifiedName.foreach { name =>
        if (activeQueries.values.exists(_.name == name)) {
          throw new IllegalArgumentException(s"Cannot start query with name $name as a query " +
            s"with that name is already active in this SparkSession")
        }
      }

      // Make sure no other query with same id is active across all sessions
      val activeOption = Option(sparkSession.sharedState.activeStreamingQueries.get(query.id))
        .orElse(activeQueries.get(query.id)) // shouldn't be needed but paranoia ...

      val shouldStopActiveRun =
        sparkSession.conf.get(SQLConf.STREAMING_STOP_ACTIVE_RUN_ON_RESTART)
      if (activeOption.isDefined) {
        if (shouldStopActiveRun) {
          val oldQuery = activeOption.get
          logWarning(s"Stopping existing streaming query [id=${query.id}, " +
            s"runId=${oldQuery.runId}], as a new run is being started.")
          Some(oldQuery)
        } else {
          throw new IllegalStateException(
            s"Cannot start query with id ${query.id} as another query with same id is " +
              s"already active. Perhaps you are attempting to restart a query from checkpoint " +
              s"that is already active. You may stop the old query by setting the SQL " +
              "configuration: " +
              s"""spark.conf.set("${SQLConf.STREAMING_STOP_ACTIVE_RUN_ON_RESTART.key}", true) """ +
              "and retry.")
        }
      } else {
        // nothing to stop so, no-op
        None
      }
    }

    // stop() will clear the queryId from activeStreamingQueries as well as activeQueries
    activeRunOpt.foreach(_.stop())

    activeQueriesSharedLock.synchronized {
      // We still can have a race condition when two concurrent instances try to start the same
      // stream, while a third one was already active and stopped above. In this case, we throw a
      // ConcurrentModificationException.
      val oldActiveQuery = sparkSession.sharedState.activeStreamingQueries.put(
        query.id, query.streamingQuery) // we need to put the StreamExecution, not the wrapper
      if (oldActiveQuery != null) {
        throw QueryExecutionErrors.concurrentQueryInstanceError()
      }
      activeQueries.put(query.id, query)
    }

    try {
      // When starting a query, it will call `StreamingQueryListener.onQueryStarted` synchronously.
      // As it's provided by the user and can run arbitrary codes, we must not hold any lock here.
      // Otherwise, it's easy to cause dead-lock, or block too long if the user codes take a long
      // time to finish.
      query.streamingQuery.start()   ===> org.apache.spark.sql.execution.streaming.StreamExecution#start
    } catch {
      case e: Throwable =>
        unregisterTerminatedStream(query)
        throw e
    }
    query
  }
```





org.apache.spark.sql.execution.streaming.StreamExecution#start

```scala
/**
 * Manages the execution of a streaming Spark SQL query that is occurring in a separate thread.
 * Unlike a standard query, a streaming query executes repeatedly each time new data arrives at any
 * [[Source]] present in the query plan. Whenever new data arrives, a [[QueryExecution]] is created
 * and the results are committed transactionally to the given [[Sink]].
 *
 * @param deleteCheckpointOnStop whether to delete the checkpoint if the query is stopped without
 *                               errors. Checkpoint deletion can be forced with the appropriate
 *                               Spark configuration.
 */
abstract class StreamExecution(
    override val sparkSession: SparkSession,
    override val name: String,
    val resolvedCheckpointRoot: String,
    val analyzedPlan: LogicalPlan,
    val sink: Table,
    val trigger: Trigger,
    val triggerClock: Clock,
    val outputMode: OutputMode,
    deleteCheckpointOnStop: Boolean)
  extends StreamingQuery with ProgressReporter with Logging {
    
  /**
   * Starts the execution. This returns only after the thread has started and [[QueryStartedEvent]]
   * has been posted to all the listeners.
   */
  def start(): Unit = {
    logInfo(s"Starting $prettyIdString. Use $resolvedCheckpointRoot to store the query checkpoint.")
    queryExecutionThread.setDaemon(true)
    queryExecutionThread.start()
    startLatch.await()  // Wait until thread started and QueryStart event has been posted
  }
    
    
  /**
   * The thread that runs the micro-batches of this stream. Note that this thread must be
   * [[org.apache.spark.util.UninterruptibleThread]] to workaround KAFKA-1894: interrupting a
   * running `KafkaConsumer` may cause endless loop.
   */
  val queryExecutionThread: QueryExecutionThread =
    new QueryExecutionThread(s"stream execution thread for $prettyIdString") {
      override def run(): Unit = {
        // To fix call site like "run at <unknown>:0", we bridge the call site from the caller
        // thread to this micro batch thread
        sparkSession.sparkContext.setCallSite(callSite)
        runStream()
      }
    }
    
  /**
   * Activate the stream and then wrap a callout to runActivatedStream, handling start and stop.
   *
   * Note that this method ensures that [[QueryStartedEvent]] and [[QueryTerminatedEvent]] are
   * posted such that listeners are guaranteed to get a start event before a termination.
   * Furthermore, this method also ensures that [[QueryStartedEvent]] event is posted before the
   * `start()` method returns.
   */
  private def   runStream(): Unit = {
    try {
          ...
        if (state.compareAndSet(INITIALIZING, ACTIVE)) {
          // Unblock `awaitInitialization`
          initializationLatch.countDown()
          runActivatedStream(sparkSessionForStream)  ===> implemented by MicroBatchExecution or ContinuousExecution
          updateStatusMessage("Stopped")
        } else {
          // `stop()` is already called. Let `finally` finish the cleanup.
        }
     ......
     
    } catch {
      case e if isInterruptedByStop(e, sparkSession.sparkContext) =>
        // interrupted by stop()
        updateStatusMessage("Stopped")
      case e: IOException if e.getMessage != null
        && e.getMessage.startsWith(classOf[InterruptedException].getName)
        && state.get == TERMINATED =>
        // This is a workaround for HADOOP-12074: `Shell.runCommand` converts `InterruptedException`
        // to `new IOException(ie.toString())` before Hadoop 2.8.
        updateStatusMessage("Stopped")
      case e: Throwable =>
        val message = if (e.getMessage == null) "" else e.getMessage
        streamDeathCause = new StreamingQueryException(
          toDebugString(includeLogicalPlan = isInitialized),
          cause = e,
          committedOffsets.toOffsetSeq(sources, offsetSeqMetadata).toString,
          availableOffsets.toOffsetSeq(sources, offsetSeqMetadata).toString,
          errorClass = "STREAM_FAILED",
          messageParameters = Map(
            "id" -> id.toString,
            "runId" -> runId.toString,
            "message" -> message))
        logError(s"Query $prettyIdString terminated with error", e)
        updateStatusMessage(s"Terminated with exception: $message")
        // Rethrow the fatal errors to allow the user using `Thread.UncaughtExceptionHandler` to
        // handle them
        if (!NonFatal(e)) {
          throw e
        }
    } finally queryExecutionThread.runUninterruptibly {
      // The whole `finally` block must run inside `runUninterruptibly` to avoid being interrupted
      // when a query is stopped by the user. We need to make sure the following codes finish
      // otherwise it may throw `InterruptedException` to `UncaughtExceptionHandler` (SPARK-21248).

      // Release latches to unblock the user codes since exception can happen in any place and we
      // may not get a chance to release them
      startLatch.countDown()
      initializationLatch.countDown()

      try {
        stopSources()
        cleanup()
        state.set(TERMINATED)
        currentStatus = status.copy(isTriggerActive = false, isDataAvailable = false)

        // Update metrics and status
        sparkSession.sparkContext.env.metricsSystem.removeSource(streamMetrics)

        // Notify others
        sparkSession.streams.notifyQueryTermination(StreamExecution.this)
        postEvent(
          new QueryTerminatedEvent(id, runId, exception.map(_.cause).map(Utils.exceptionString)))

        // Delete the temp checkpoint when either force delete enabled or the query didn't fail
        if (deleteCheckpointOnStop &&
            (sparkSession.sessionState.conf
              .getConf(SQLConf.FORCE_DELETE_TEMP_CHECKPOINT_LOCATION) || exception.isEmpty)) {
          val checkpointPath = new Path(resolvedCheckpointRoot)
          try {
            logInfo(s"Deleting checkpoint $checkpointPath.")
            fileManager.delete(checkpointPath)
          } catch {
            case NonFatal(e) =>
              // Deleting temp checkpoint folder is best effort, don't throw non fatal exceptions
              // when we cannot delete them.
              logWarning(s"Cannot delete $checkpointPath", e)
          }
        }
      } finally {
        awaitProgressLock.lock()
        try {
          // Wake up any threads that are waiting for the stream to progress.
          awaitProgressLockCondition.signalAll()
        } finally {
          awaitProgressLock.unlock()
        }
        terminationLatch.countDown()
      }
    }
  }
```

# MicroBatchExecution



```scala
class MicroBatchExecution(
    sparkSession: SparkSession,
    trigger: Trigger,
    triggerClock: Clock,
    extraOptions: Map[String, String],
    plan: WriteToStream)
  extends StreamExecution(
    sparkSession, plan.name, plan.resolvedCheckpointLocation, plan.inputQuery, plan.sink, trigger,
    triggerClock, plan.outputMode, plan.deleteCheckpointOnStop) with AsyncLogPurge {


  /**
   * Repeatedly attempts to run batches as data arrives.
   */
  protected def runActivatedStream(sparkSessionForStream: SparkSession): Unit = {

    val noDataBatchesEnabled =
      sparkSessionForStream.sessionState.conf.streamingNoDataMicroBatchesEnabled

    triggerExecutor.execute(() => {
      if (isActive) {

        // check if there are any previous errors and bubble up any existing async operations
        errorNotifier.throwErrorIfExists

        var currentBatchHasNewData = false // Whether the current batch had new data

        startTrigger()

        reportTimeTaken("triggerExecution") {
          // We'll do this initialization only once every start / restart
          if (currentBatchId < 0) {
            AcceptsLatestSeenOffsetHandler.setLatestSeenOffsetOnSources(
              offsetLog.getLatest().map(_._2), sources)
            populateStartOffsets(sparkSessionForStream)
            logInfo(s"Stream started from $committedOffsets")
          }

          // Set this before calling constructNextBatch() so any Spark jobs executed by sources
          // while getting new data have the correct description
          sparkSession.sparkContext.setJobDescription(getBatchDescriptionString)

          // Try to construct the next batch. This will return true only if the next batch is
          // ready and runnable. Note that the current batch may be runnable even without
          // new data to process as `constructNextBatch` may decide to run a batch for
          // state cleanup, etc. `isNewDataAvailable` will be updated to reflect whether new data
          // is available or not.
          if (!isCurrentBatchConstructed) {
            isCurrentBatchConstructed = constructNextBatch(noDataBatchesEnabled)
          }

          // Record the trigger offset range for progress reporting *before* processing the batch
          recordTriggerOffsets(
            from = committedOffsets,
            to = availableOffsets,
            latest = latestOffsets)

          // Remember whether the current batch has data or not. This will be required later
          // for bookkeeping after running the batch, when `isNewDataAvailable` will have changed
          // to false as the batch would have already processed the available data.
          currentBatchHasNewData = isNewDataAvailable

          currentStatus = currentStatus.copy(isDataAvailable = isNewDataAvailable)
          if (isCurrentBatchConstructed) {
            if (currentBatchHasNewData) updateStatusMessage("Processing new data")
            else updateStatusMessage("No new data but cleaning up state")
            runBatch(sparkSessionForStream)
          } else {
            updateStatusMessage("Waiting for data to arrive")
          }
        }

        // Must be outside reportTimeTaken so it is recorded
        finishTrigger(currentBatchHasNewData, isCurrentBatchConstructed)

        // Signal waiting threads. Note this must be after finishTrigger() to ensure all
        // activities (progress generation, etc.) have completed before signaling.
        withProgressLocked { awaitProgressLockCondition.signalAll() }

        // If the current batch has been executed, then increment the batch id and reset flag.
        // Otherwise, there was no data to execute the batch and sleep for some time
        if (isCurrentBatchConstructed) {
          currentBatchId += 1
          isCurrentBatchConstructed = false
        } else if (triggerExecutor.isInstanceOf[MultiBatchExecutor]) {
          logInfo("Finished processing all available data for the trigger, terminating this " +
            "Trigger.AvailableNow query")
          state.set(TERMINATED)
        } else Thread.sleep(pollingDelayMs)
      }
      updateStatusMessage("Waiting for next trigger")
      isActive
    })
  }

```



# Watermark Support

## removeKeysOlderThanWatermark

**org.apache.spark.sql.execution.streaming.WatermarkSupport#removeKeysOlderThanWatermark**

```scala
  protected def removeKeysOlderThanWatermark(store: StateStore): Unit = {
    if (watermarkPredicateForKeysForEviction.nonEmpty) {
      val numRemovedStateRows = longMetric("numRemovedStateRows")
      store.iterator().foreach { rowPair =>
        if (watermarkPredicateForKeysForEviction.get.eval(rowPair.key)) {
          store.remove(rowPair.key)
          numRemovedStateRows += 1
        }
      }
    }
  }

protected def removeKeysOlderThanWatermark(
      storeManager: StreamingAggregationStateManager,
      store: StateStore): Unit = {
    if (watermarkPredicateForKeysForEviction.nonEmpty) {
      val numRemovedStateRows = longMetric("numRemovedStateRows")
      storeManager.keys(store).foreach { keyRow =>
        if (watermarkPredicateForKeysForEviction.get.eval(keyRow)) {
          storeManager.remove(store, keyRow)
          numRemovedStateRows += 1
        }
      }
    }
  }
```

**org.apache.spark.sql.execution.streaming.state.RocksDBStateStoreProvider.RocksDBStateStore#iterator**

```scala
    override def iterator(): Iterator[UnsafeRowPair] = {
      rocksDB.iterator().map { kv =>
        val rowPair = encoder.decode(kv)
        if (!isValidated && rowPair.value != null) {
          StateStoreProvider.validateStateRowFormat(
            rowPair.key, keySchema, rowPair.value, valueSchema, storeConf)
          isValidated = true
        }
        rowPair
      }
    }
```



**org.apache.spark.sql.execution.streaming.state.RocksDB#iterator**

```scala
  def iterator(): Iterator[ByteArrayPair] = {
    val iter = writeBatch.newIteratorWithBase(db.newIterator())
    logInfo(s"Getting iterator from version $loadedVersion")
    iter.seekToFirst()

    // Attempt to close this iterator if there is a task failure, or a task interruption.
    // This is a hack because it assumes that the RocksDB is running inside a task.
    Option(TaskContext.get()).foreach { tc =>
      tc.addTaskCompletionListener[Unit] { _ => iter.close() }
    }

    new NextIterator[ByteArrayPair] {
      override protected def getNext(): ByteArrayPair = {
        if (iter.isValid) {
          byteArrayPair.set(iter.key, iter.value)
          iter.next()
          byteArrayPair
        } else {
          finished = true
          iter.close()
          null
        }
      }
      override protected def close(): Unit = { iter.close() }
    }
  }
```

