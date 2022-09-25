# Shuffle Service

[TOC]


## BlockTranserService
### Code


```
/**
 * The BlockTransferService that used for fetching a set of blocks at time. Each instance of
 * BlockTransferService contains both client and server inside.
 */
private[spark]
abstract class BlockTransferService extends BlockStoreClient {

```


```
/**
 * A BlockTransferService that uses Netty to fetch a set of blocks at time.
 */
private[spark] class NettyBlockTransferService(
    conf: SparkConf,
    securityManager: SecurityManager,
    bindAddress: String,
    override val hostName: String,
    _port: Int,
    numCores: Int,
    driverEndPointRef: RpcEndpointRef = null)
  extends BlockTransferService {
  
    override def init(blockDataManager: BlockDataManager): Unit = {
    val rpcHandler = new NettyBlockRpcServer(conf.getAppId, serializer, blockDataManager)
    var serverBootstrap: Option[TransportServerBootstrap] = None
    var clientBootstrap: Option[TransportClientBootstrap] = None
    this.transportConf = SparkTransportConf.fromSparkConf(conf, "shuffle", numCores)
    if (authEnabled) {
      serverBootstrap = Some(new AuthServerBootstrap(transportConf, securityManager))
      clientBootstrap = Some(new AuthClientBootstrap(transportConf, conf.getAppId, securityManager))
    }
    transportContext = new TransportContext(transportConf, rpcHandler)
    clientFactory = transportContext.createClientFactory(clientBootstrap.toSeq.asJava)
    server = createServer(serverBootstrap.toList)
    appId = conf.getAppId

    if (hostName.equals(bindAddress)) {
      logger.info(s"Server created on $hostName:${server.getPort}")
    } else {
      logger.info(s"Server created on $hostName $bindAddress:${server.getPort}")
    }
  }
```

## Improve Spark shuffle server responsiveness to non-ChunkFetch requests
[SPARK-24355 Improve Spark shuffle server responsiveness to non-ChunkFetch requests](https://issues.apache.org/jira/browse/SPARK-24355)

[SPARK-30623 Spark external shuffle allow disable of separate event loop group](https://issues.apache.org/jira/browse/SPARK-30623)  
What changes were proposed in this pull request? Fix the regression caused by PR #22173.
The original PR changes the logic of handling `ChunkFetchReqeust` from async to sync, that's causes the shuffle benchmark regression.
This PR fixes the regression back to the async mode by reusing the config `spark.shuffle.server.chunkFetchHandlerThreadsPercent`.
When the user sets the config, ChunkFetchReqeust will be processed in a separate event loop group, otherwise, the code path is exactly the same as before.
Performance regression described in [comment](https://github.com/apache/spark/pull/22173#issuecomment-572459561

**org.apache.spark.network.server.ChunkFetchRequestHandler#respond**
```
  /**
   * The invocation to channel.writeAndFlush is async, and the actual I/O on the
   * channel will be handled by the EventLoop the channel is registered to. So even
   * though we are processing the ChunkFetchRequest in a separate thread pool, the actual I/O,
   * which is the potentially blocking call that could deplete server handler threads, is still
   * being processed by TransportServer's default EventLoopGroup.
   *
   * When syncModeEnabled is true, Spark will throttle the max number of threads that channel I/O
   * for sending response to ChunkFetchRequest, the thread calling channel.writeAndFlush will wait
   * for the completion of sending response back to client by invoking await(). This will throttle
   * the rate at which threads from ChunkFetchRequest dedicated EventLoopGroup submit channel I/O
   * requests to TransportServer's default EventLoopGroup, thus making sure that we can reserve
   * some threads in TransportServer's default EventLoopGroup for handling other RPC messages.
   */
  private ChannelFuture respond(
      final Channel channel,
      final Encodable result) throws InterruptedException {
    final SocketAddress remoteAddress = channel.remoteAddress();
    ChannelFuture channelFuture;
    if (syncModeEnabled) {
      channelFuture = channel.writeAndFlush(result).await();
    } else {
      channelFuture = channel.writeAndFlush(result);
    }
    return channelFuture.addListener((ChannelFutureListener) future -> {
      if (future.isSuccess()) {
        logger.trace("Sent result {} to client {}", result, remoteAddress);
      } else {
        logger.error(String.format("Error sending result %s to %s; closing connection",
          result, remoteAddress), future.cause());
        channel.close();
      }
    });
  }   
```

Q: Why await() needed?

I think await does't provide any benefit and could be removed.
When the chunk fetch event loop runs
```
channel.writeAndFlush(result)
```
This adds a WriteAndFlushTask in the pendingQueue of the default server-IO thread registered with that channel.

The code in NioEventLoop.run() itself throttles the number of tasks that can be run at a time from its pending queue.
Here is the code:
```
                    final long ioStartTime = System.nanoTime();
                    try {
                        processSelectedKeys();
                    } finally {
                        // Ensure we always run tasks.
                        final long ioTime = System.nanoTime() - ioStartTime;
                        runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                    }
                }
```
Here it records how much time it took to perform the IO operations, that is, execute processSelectedKeys(). runAllTasks, which is the method that processes the tasks from pendingQueue, will be performed for the same amount of time.

runAllTasks() does process 64 tasks and then checks the time.
```
        // Check timeout every 64 tasks because nanoTime() is relatively expensive.
            // XXX: Hard-coded value - will make it configurable if it is really a problem.
            if ((runTasks & 0x3F) == 0) {
                lastExecutionTime = ScheduledFutureTask.nanoTime();
                if (lastExecutionTime >= deadline) {
                    break;
                }
            } 
```
This ensures that the default server-IO thread always gets time to process the ready channels. Its not always busy processing WriteAndFlushTask


Answer:
I removed the await and tested with our internal stress testing framework. I started seeing SASL requests timing out. In this test, I observed more than 2 minutes delay between channel registration and when the first bytes are read from the channel.
```
2020-01-24 22:53:34,019 DEBUG org.spark_project.io.netty.handler.logging.LoggingHandler: [id: 0xd475f5ff, L:/10.150.16.27:7337 - R:/10.150.16.44:11388] REGISTERED
2020-01-24 22:53:34,019 DEBUG org.spark_project.io.netty.handler.logging.LoggingHandler: [id: 0xd475f5ff, L:/10.150.16.27:7337 - R:/10.150.16.44:11388] ACTIVE

2020-01-24 22:55:05,207 DEBUG org.spark_project.io.netty.handler.logging.LoggingHandler: [id: 0xd475f5ff, L:/10.150.16.27:7337 - R:/10.150.16.44:11388] READ: 48B
2020-01-24 22:55:05,207 DEBUG org.spark_project.io.netty.handler.logging.LoggingHandler: [id: 0xd475f5ff, L:/10.150.16.27:7337 - R:/10.150.16.44:11388] WRITE: org.apache.spark.network.protocol.MessageWithHeader@27e59ee9
2020-01-24 22:55:05,207 DEBUG org.spark_project.io.netty.handler.logging.LoggingHandler: [id: 0xd475f5ff, L:/10.150.16.27:7337 - R:/10.150.16.44:11388] FLUSH
2020-01-24 22:55:05,207 INFO org.apache.spark.network.server.OutgoingChannelHandler: OUTPUT request 5929104419960968526 channel d475f5ff request_rec 1579906505207 transport_rec 1579906505207 flush 1579906505207  receive-transport 0 transport-flush 0 total 0
```
Since there is a delay in reading the channel, I suspect this is because the hardcoding in netty code
SingleThreadEventExecutor.runAllTask() that checks time only after 64 tasks. WriteAndFlush tasks are bulky tasks. With await there will be just 1 WriteAndFlushTask per channel in the IO thread's pending queue and the rest of the tasks will be smaller tasks.
However, without await there are more WriteAndFlush tasks per channel in the IO thread's queue. Since it processes 64 tasks and then checks time, this time increases with more WriteAndFlush tasks.

```
/ Check timeout every 64 tasks because nanoTime() is relatively expensive.
            // XXX: Hard-coded value - will make it configurable if it is really a problem.
            if ((runTasks & 0x3F) == 0) {
                lastExecutionTime = ScheduledFutureTask.nanoTime();
                if (lastExecutionTime >= deadline) {
                    break;
                }
            }
```
I can test this theory by lowering this number in a fork of netty and building spark against it. However, for now we can't remove await().

Note: This test was with a dedicated boss event loop group which is why we don't see any delay in channel registration.




## push-based shuffle
[SPARK-30602 SPIP: Support push-based shuffle to improve shuffle efficiency](https://issues.apache.org/jira/browse/SPARK-30602)

[Consolidated reference PR for Push-based shuffle](https://github.com/apache/spark/pull/29808)


## Use remote storage for persisting shuffle data
[architecture discussion - Use remote storage for persisting shuffle data](https://issues.apache.org/jira/browse/SPARK-25299)

[SPIP: `SPARK-25299 - An API For Writing Shuffle Data To Remote Storage](https://docs.google.com/document/d/1d6egnL6WHOwWZe8MWv3m8n4PToNacdx7n_0iMSWwhCQ/edit)
