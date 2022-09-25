[TOC]

# Yarn Allocator
## Code

### **YarnAllocator**

org.apache.spark.deploy.yarn.YarnAllocator

```
/**
 * YarnAllocator is charged with requesting containers from the YARN ResourceManager and deciding
 * what to do with containers when YARN fulfills these requests.
 *
 * This class makes use of YARN's AMRMClient APIs. We interact with the AMRMClient in three ways:
 * * Making our resource needs known, which updates local bookkeeping about containers requested.
 * * Calling "allocate", which syncs our local container requests with the RM, and returns any
 *   containers that YARN has granted to us.  This also functions as a heartbeat.
 * * Processing the containers granted to us to possibly launch executors inside of them.
 *
 * The public methods of this class are thread-safe.  All methods that mutate state are
 * synchronized.
 */
private[yarn] class YarnAllocator(
    driverUrl: String,
    driverRef: RpcEndpointRef,
    conf: YarnConfiguration,
    sparkConf: SparkConf,
    amClient: AMRMClient[ContainerRequest],
    appAttemptId: ApplicationAttemptId,
    securityMgr: SecurityManager,
    localResources: Map[String, LocalResource],
    resolver: SparkRackResolver,
    clock: Clock = new SystemClock)
  extends Logging {
```