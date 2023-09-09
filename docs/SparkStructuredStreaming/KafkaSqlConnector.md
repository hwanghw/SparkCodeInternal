[TOC]

# InternalKafkaProducerPool

org.apache.spark.sql.kafka010.producer.InternalKafkaProducerPool#InternalKafkaProducerPool(org.apache.spark.SparkConf)



```scala
/**
 * Provides object pool for [[CachedKafkaProducer]] which is grouped by
 * [[org.apache.spark.sql.kafka010.producer.InternalKafkaProducerPool.CacheKey]].
 */
private[producer] class InternalKafkaProducerPool(
    executorService: ScheduledExecutorService,
    val clock: Clock,
    conf: SparkConf) extends Logging {
  ...
  ShutdownHookManager.addShutdownHook { () =>
    try {
      pool.shutdown()
    } catch {
      case e: Throwable =>
        logWarning("Ignoring Exception while shutting down pools from shutdown hook", e)
    }
  }
```



# KafkaOffsetReaderAdmin



org.apache.spark.sql.kafka010.KafkaOffsetReaderAdmin#withRetries

```scala
  /**
   * Helper function that does multiple retries on a body of code that returns offsets.
   * Retries are needed to handle transient failures. For e.g. race conditions between getting
   * assignment and getting position while topics/partitions are deleted can cause NPEs.
   */
  private def withRetries(body: => Map[TopicPartition, Long]): Map[TopicPartition, Long] = {
    synchronized {
      var result: Option[Map[TopicPartition, Long]] = None
      var attempt = 1
      var lastException: Throwable = null
      while (result.isEmpty && attempt <= maxOffsetFetchAttempts
        && !Thread.currentThread().isInterrupted) {
        try {
          result = Some(body)
        } catch {
          case NonFatal(e) =>
            lastException = e
            logWarning(s"Error in attempt $attempt getting Kafka offsets: ", e)
            attempt += 1
            Thread.sleep(offsetFetchAttemptIntervalMs)
            resetAdmin()
        }
      }
      if (Thread.interrupted()) {
        throw new InterruptedException()
      }
      if (result.isEmpty) {
        assert(attempt > maxOffsetFetchAttempts)
        assert(lastException != null)
        throw lastException
      }
      result.get
    }
  }
```

