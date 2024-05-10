[TOC]

# TaskSchedulerImpl
## **resourceOffers**
&nbsp;org.apache.spark.scheduler.TaskSchedulerImpl#resourceOffers=>  
&nbsp;&nbsp; org.apache.spark.scheduler.TaskSchedulerImpl#resourceOfferSingleTaskSet=>  
&nbsp;&nbsp;&nbsp;    org.apache.spark.scheduler.TaskSetManager#resourceOffer

org.apache.spark.scheduler.TaskSchedulerImpl#resourceOffers
```scala
  /**
   * Called by cluster manager to offer resources on workers. We respond by asking our active task
   * sets for tasks in order of priority. We fill each node with tasks in a round-robin manner so
   * that tasks are balanced across the cluster.
   */
  def resourceOffers(
      offers: IndexedSeq[WorkerOffer],
      isAllFreeResources: Boolean = true): Seq[Seq[TaskDescription]] = synchronized {
      
      ...
      
    val shuffledOffers = shuffleOffers(filteredOffers) ===> shuffle WorkerOffers to avoid always placing tasks on the same workers
    // Build a list of tasks to assign to each worker.
    // Note the size estimate here might be off with different ResourceProfiles but should be
    // close estimate
    val tasks = shuffledOffers.map(o => new ArrayBuffer[TaskDescription](o.cores / CPUS_PER_TASK))
    val availableResources = shuffledOffers.map(_.resources).toArray
    val availableCpus = shuffledOffers.map(o => o.cores).toArray
    val resourceProfileIds = shuffledOffers.map(o => o.resourceProfileId).toArray
    val sortedTaskSets = rootPool.getSortedTaskSetQueue
    for (taskSet <- sortedTaskSets) {
      logDebug("parentName: %s, name: %s, runningTasks: %s".format(
        taskSet.parent.name, taskSet.name, taskSet.runningTasks))
      if (newExecAvail) {
        taskSet.executorAdded()
      }
     }

    // Take each TaskSet in our scheduling order, and then offer it to each node in increasing order
    // of locality levels so that it gets a chance to launch local tasks on all of them.
    // NOTE: the preferredLocality order: PROCESS_LOCAL, NODE_LOCAL, NO_PREF, RACK_LOCAL, ANY
    for (taskSet <- sortedTaskSets) {
      ...
        } else {
        var launchedAnyTask = false
        var noDelaySchedulingRejects = true
        var globalMinLocality: Option[TaskLocality] = None
        for (currentMaxLocality <- taskSet.myLocalityLevels) {
          var launchedTaskAtCurrentMaxLocality = false
          do {
            val (noDelayScheduleReject, minLocality) = resourceOfferSingleTaskSet(
              taskSet, currentMaxLocality, shuffledOffers, availableCpus,
              availableResources, tasks)
            launchedTaskAtCurrentMaxLocality = minLocality.isDefined
            launchedAnyTask |= launchedTaskAtCurrentMaxLocality
            noDelaySchedulingRejects &= noDelayScheduleReject
            globalMinLocality = minTaskLocality(globalMinLocality, minLocality)
          } while (launchedTaskAtCurrentMaxLocality)
        }

      
  /**
   * Shuffle offers around to avoid always placing tasks on the same workers.  Exposed to allow
   * overriding in tests, so it can be deterministic.
   */
  protected def shuffleOffers(offers: IndexedSeq[WorkerOffer]): IndexedSeq[WorkerOffer] = {
    Random.shuffle(offers)
  }
```

org.apache.spark.scheduler.TaskSchedulerImpl#resourceOfferSingleTaskSet
```scala
  /**
   * Offers resources to a single [[TaskSetManager]] at a given max allowed [[TaskLocality]].
   *
   * @param taskSet task set manager to offer resources to
   * @param maxLocality max locality to allow when scheduling
   * @param shuffledOffers shuffled resource offers to use for scheduling,
   *                       remaining resources are tracked by below fields as tasks are scheduled
   * @param availableCpus  remaining cpus per offer,
   *                       value at index 'i' corresponds to shuffledOffers[i]
   * @param availableResources remaining resources per offer,
   *                           value at index 'i' corresponds to shuffledOffers[i]
   * @param tasks tasks scheduled per offer, value at index 'i' corresponds to shuffledOffers[i]
   * @return tuple of (no delay schedule rejects?, option of min locality of launched task)
   */
  private def resourceOfferSingleTaskSet(
      taskSet: TaskSetManager,
      maxLocality: TaskLocality,
      shuffledOffers: Seq[WorkerOffer],
      availableCpus: Array[Int],
      availableResources: Array[Map[String, Buffer[String]]],
      tasks: IndexedSeq[ArrayBuffer[TaskDescription]])
    : (Boolean, Option[TaskLocality]) = {
    var noDelayScheduleRejects = true
    var minLaunchedLocality: Option[TaskLocality] = None
    // nodes and executors that are excluded for the entire application have already been
    // filtered out by this point
    for (i <- shuffledOffers.indices) { ======> round robin to assign a task per offer/executor
      val execId = shuffledOffers(i).executorId
      val host = shuffledOffers(i).host
      val taskSetRpID = taskSet.taskSet.resourceProfileId

      // check whether the task can be scheduled to the executor base on resource profile.
      if (sc.resourceProfileManager
        .canBeScheduled(taskSetRpID, shuffledOffers(i).resourceProfileId)) {
        val taskResAssignmentsOpt = resourcesMeetTaskRequirements(taskSet, availableCpus(i),
          availableResources(i)) ===> Check whether the resources from the WorkerOffer are enough to run at least one task.
        taskResAssignmentsOpt.foreach { taskResAssignments =>
          try {
            val prof = sc.resourceProfileManager.resourceProfileFromId(taskSetRpID)
            val taskCpus = ResourceProfile.getTaskCpusOrDefaultForProfile(prof, conf)
            val (taskDescOption, didReject, index) =
              taskSet.resourceOffer(execId, host, maxLocality, taskCpus, taskResAssignments) ===> Respond to an offer of a single executor from the scheduler by finding a task
            noDelayScheduleRejects &= !didReject
            for (task <- taskDescOption) {
              val (locality, resources) = if (task != null) {
                tasks(i) += task
                addRunningTask(task.taskId, execId, taskSet)
                (taskSet.taskInfos(task.taskId).taskLocality, task.resources)
              } else {
                assert(taskSet.isBarrier, "TaskDescription can only be null for barrier task")
                val barrierTask = taskSet.barrierPendingLaunchTasks(index)
                barrierTask.assignedOfferIndex = i
                barrierTask.assignedCores = taskCpus
                (barrierTask.taskLocality, barrierTask.assignedResources)
              }

              minLaunchedLocality = minTaskLocality(minLaunchedLocality, Some(locality))
              availableCpus(i) -= taskCpus
              assert(availableCpus(i) >= 0)
              resources.foreach { case (rName, rInfo) =>
                // Remove the first n elements from availableResources addresses, these removed
                // addresses are the same as that we allocated in taskResourceAssignments since it's
                // synchronized. We don't remove the exact addresses allocated because the current
                // approach produces the identical result with less time complexity.
                availableResources(i)(rName).remove(0, rInfo.addresses.size)
              }
            }
          } catch {
            case e: TaskNotSerializableException =>
              logError(s"Resource offer failed, task set ${taskSet.name} was not serializable")
              // Do not offer resources for this task, but don't throw an error to allow other
              // task sets to be submitted.
              return (noDelayScheduleRejects, minLaunchedLocality)
          }
        }
      }
    }
    (noDelayScheduleRejects, minLaunchedLocality)
  }
```

org.apache.spark.scheduler.TaskSetManager#resourceOffer
```scala
  /**
   * Respond to an offer of a single executor from the scheduler by finding a task
   *
   * NOTE: this function is either called with a maxLocality which
   * would be adjusted by delay scheduling algorithm or it will be with a special
   * NO_PREF locality which will be not modified
   *
   * @param execId the executor Id of the offered resource
   * @param host  the host Id of the offered resource
   * @param maxLocality the maximum locality we want to schedule the tasks at
   * @param taskCpus the number of CPUs for the task
   * @param taskResourceAssignments the resource assignments for the task
   *
   * @return Triple containing:
   *         (TaskDescription of launched task if any,
   *         rejected resource due to delay scheduling?,
   *         dequeued task index)
   */
  @throws[TaskNotSerializableException]
  def resourceOffer(
      execId: String,
      host: String,
      maxLocality: TaskLocality.TaskLocality,
      taskCpus: Int = sched.CPUS_PER_TASK,
      taskResourceAssignments: Map[String, ResourceInformation] = Map.empty)
    : (Option[TaskDescription], Boolean, Int)
```


org.apache.spark.scheduler.TaskDescription
```scala
/**
 * Description of a task that gets passed onto executors to be executed, usually created by
 * `TaskSetManager.resourceOffer`.
 *
 * TaskDescriptions and the associated Task need to be serialized carefully for two reasons:
 *
 *     (1) When a TaskDescription is received by an Executor, the Executor needs to first get the
 *         list of JARs and files and add these to the classpath, and set the properties, before
 *         deserializing the Task object (serializedTask). This is why the Properties are included
 *         in the TaskDescription, even though they're also in the serialized task.
 *     (2) Because a TaskDescription is serialized and sent to an executor for each task, efficient
 *         serialization (both in terms of serialization time and serialized buffer size) is
 *         important. For this reason, we serialize TaskDescriptions ourselves with the
 *         TaskDescription.encode and TaskDescription.decode methods.  This results in a smaller
 *         serialized size because it avoids serializing unnecessary fields in the Map objects
 *         (which can introduce significant overhead when the maps are small).
 */
private[spark] class TaskDescription(
    val taskId: Long,
    val attemptNumber: Int, ===> how many times this task has been attempted (0 for the first attempt)
    val executorId: String,
    val name: String,
    val index: Int,    // Index within this task's TaskSet
    val partitionId: Int,
    val addedFiles: Map[String, Long],
    val addedJars: Map[String, Long],
    val addedArchives: Map[String, Long],
    val properties: Properties,
    val cpus: Int,
    val resources: immutable.Map[String, ResourceInformation],
    val serializedTask: ByteBuffer) {

  assert(cpus > 0, "CPUs per task should be > 0")

  override def toString: String = s"TaskDescription($name)"
}
```



## **handle failed task**

&nbsp;org.apache.spark.scheduler.TaskSchedulerImpl#handleFailedTask=>  
&nbsp;&nbsp; org.apache.spark.scheduler.TaskSetManager#handleFailedTask=>  
&nbsp;&nbsp;&nbsp;  org.apache.spark.scheduler.DAGScheduler#taskEnded

org.apache.spark.scheduler.TaskSetManager#handleFailedTask
```scala
  
  /**
   * Marks the task as failed, re-adds it to the list of pending tasks, and notifies the
   * DAG Scheduler.
   */
  def handleFailedTask(tid: Long, state: TaskState, reason: TaskFailedReason): Unit = {
    val info = taskInfos(tid)
    // SPARK-37300: when the task was already finished state, just ignore it,
    // so that there won't cause copiesRunning wrong result.
    if (info.finished) {
      return
    }
    removeRunningTask(tid)
    info.markFinished(state, clock.getTimeMillis())
    val index = info.index
    copiesRunning(index) -= 1
    var accumUpdates: Seq[AccumulatorV2[_, _]] = Seq.empty
    var metricPeaks: Array[Long] = Array.empty
    val failureReason = s"Lost ${taskName(tid)} (${info.host} " +
      s"executor ${info.executorId}): ${reason.toErrorString}"
    val failureException: Option[Throwable] = reason match {
      case fetchFailed: FetchFailed =>
        logWarning(failureReason)
        if (!successful(index)) {
          successful(index) = true
          tasksSuccessful += 1
        }
        isZombie = true

        if (fetchFailed.bmAddress != null) {
          healthTracker.foreach(_.updateExcludedForFetchFailure(
            fetchFailed.bmAddress.host, fetchFailed.bmAddress.executorId))
        }

        None

      case ef: ExceptionFailure =>
        // ExceptionFailure's might have accumulator updates
        accumUpdates = ef.accums
        metricPeaks = ef.metricPeaks.toArray
        val task = taskName(tid)
        if (ef.className == classOf[NotSerializableException].getName) {
          // If the task result wasn't serializable, there's no point in trying to re-execute it.
          logError(s"$task had a not serializable result: ${ef.description}; not retrying")
          sched.dagScheduler.taskEnded(tasks(index), reason, null, accumUpdates, metricPeaks, info)
          abort(s"$task had a not serializable result: ${ef.description}")
          return
        }
        if (ef.className == classOf[TaskOutputFileAlreadyExistException].getName) {
          // If we can not write to output file in the task, there's no point in trying to
          // re-execute it.
          logError("Task %s in stage %s (TID %d) can not write to output file: %s; not retrying"
            .format(info.id, taskSet.id, tid, ef.description))
          sched.dagScheduler.taskEnded(tasks(index), reason, null, accumUpdates, metricPeaks, info)
          abort("Task %s in stage %s (TID %d) can not write to output file: %s".format(
            info.id, taskSet.id, tid, ef.description))
          return
        }
        val key = ef.description
        val now = clock.getTimeMillis()
        val (printFull, dupCount) = {
          if (recentExceptions.contains(key)) {
            val (dupCount, printTime) = recentExceptions(key)
            if (now - printTime > EXCEPTION_PRINT_INTERVAL) {
              recentExceptions(key) = (0, now)
              (true, 0)
            } else {
              recentExceptions(key) = (dupCount + 1, printTime)
              (false, dupCount + 1)
            }
          } else {
            recentExceptions(key) = (0, now)
            (true, 0)
          }
        }
        if (printFull) {
          logWarning(failureReason)
        } else {
          logInfo(
            s"Lost $task on ${info.host}, executor ${info.executorId}: " +
              s"${ef.className} (${ef.description}) [duplicate $dupCount]")
        }
        ef.exception

      case tk: TaskKilled =>
        // TaskKilled might have accumulator updates
        accumUpdates = tk.accums
        metricPeaks = tk.metricPeaks.toArray
        logWarning(failureReason)
        None

      case e: ExecutorLostFailure if !e.exitCausedByApp =>
        logInfo(s"${taskName(tid)} failed because while it was being computed, its executor " +
          "exited for a reason unrelated to the task. Not counting this failure towards the " +
          "maximum number of failures for the task.")
        None

      case e: TaskFailedReason =>  // TaskResultLost and others
        logWarning(failureReason)
        None
    }

    if (tasks(index).isBarrier) {
      isZombie = true
    }

    sched.dagScheduler.taskEnded(tasks(index), reason, null, accumUpdates, metricPeaks, info)

    if (!isZombie && reason.countTowardsTaskFailures) {
      assert (null != failureReason)
      taskSetExcludelistHelperOpt.foreach(_.updateExcludedForFailedTask(
        info.host, info.executorId, index, failureReason))
      numFailures(index) += 1
      if (numFailures(index) >= maxTaskFailures) {
        logError("Task %d in stage %s failed %d times; aborting job".format(
          index, taskSet.id, maxTaskFailures))
        abort("Task %d in stage %s failed %d times, most recent failure: %s\nDriver stacktrace:"
          .format(index, taskSet.id, maxTaskFailures, failureReason), failureException)
        return
      }
    }

    if (successful(index)) {
      logInfo(s"${taskName(info.taskId)} failed, but the task will not" +
        " be re-executed (either because the task failed with a shuffle data fetch failure," +
        " so the previous stage needs to be re-run, or because a different copy of the task" +
        " has already succeeded).")
    } else {
      addPendingTask(index)
    }

    maybeFinishTaskSet()
  }
```