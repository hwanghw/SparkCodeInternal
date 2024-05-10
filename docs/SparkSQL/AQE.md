[TOC]

# Adaptive execution in Spark
## Jira
[SPARK-31412 Feature requirement (with subtasks list)](https://issues.apache.org/jira/browse/SPARK-31412)  
[Design Doc](https://docs.google.com/document/d/1mpVjvQZRAkD-Ggy6-hcjXtBPiQoVbZGe3dLnAKgtJ4k/edit?usp=sharing)

[SPARK-23128 The basic framework for the new Adaptive Query Execution](https://issues.apache.org/jira/browse/SPARK-23128)

[SPARK-28177 Adjust post shuffle partition number in adaptive execution](https://issues.apache.org/jira/browse/SPARK-28177)

[SPARK-29544 Optimize skewed join at runtime with new Adaptive Execution](https://issues.apache.org/jira/browse/SPARK-29544)

[SPARK-31865 Fix complex AQE query stage not reused](https://issues.apache.org/jira/browse/SPARK-31865)

[SPARK-35552 Make query stage materialized more readable](https://issues.apache.org/jira/browse/SPARK-35552)

[SPARK-9850 Adaptive execution in Spark (original idea)](https://issues.apache.org/jira/browse/SPARK-9850)  
[Design Doc](https://issues.apache.org/jira/secure/attachment/12749984/AdaptiveExecutionInSpark.pdf)

[SPARK-9851 Support submitting map stages individually in DAGScheduler](https://issues.apache.org/jira/browse/SPARK-9851)

[SPARK document Adaptive Query Execution](https://spark.apache.org/docs/latest/sql-performance-tuning.html#adaptive-query-execution)
## **QueryExecution#preparations**

org.apache.spark.sql.execution.QueryExecution#preparations
```
private[execution] def preparations(
   sparkSession: SparkSession,
   adaptiveExecutionRule: Option[InsertAdaptiveSparkPlan] = None,
   subquery: Boolean): Seq[Rule[SparkPlan]] = {
 // `AdaptiveSparkPlanExec` is a leaf node. If inserted, all the following rules will be no-op
 // as the original plan is hidden behind `AdaptiveSparkPlanExec`.
 adaptiveExecutionRule.toSeq ++
```

## **InsertAdaptiveSparkPlan**
org.apache.spark.sql.execution.adaptive.InsertAdaptiveSparkPlan
```
/**
* This rule wraps the query plan with an [[AdaptiveSparkPlanExec]], which executes the query plan
* and re-optimize the plan during execution based on runtime data statistics.
*
* Note that this rule is stateful and thus should not be reused across query executions.
*/
case class InsertAdaptiveSparkPlan(
   adaptiveExecutionContext: AdaptiveExecutionContext) extends Rule[SparkPlan] {

  override def apply(plan: SparkPlan): SparkPlan = applyInternal(plan, false)

  private def applyInternal(plan: SparkPlan, isSubquery: Boolean): SparkPlan = plan match {
    case _ if !conf.adaptiveExecutionEnabled => plan
    case _: ExecutedCommandExec => plan
    case _: CommandResultExec => plan
    case c: DataWritingCommandExec => c.copy(child = apply(c.child))
    case c: V2CommandExec => c.withNewChildren(c.children.map(apply))
    case _ if shouldApplyAQE(plan, isSubquery) =>
      if (supportAdaptive(plan)) {
        try {
          // Plan sub-queries recursively and pass in the shared stage cache for exchange reuse.
          // Fall back to non-AQE mode if AQE is not supported in any of the sub-queries.
          val subqueryMap = buildSubqueryMap(plan)
          val planSubqueriesRule = PlanAdaptiveSubqueries(subqueryMap)
          val preprocessingRules = Seq(
            planSubqueriesRule)
          // Run pre-processing rules.
          val newPlan = AdaptiveSparkPlanExec.applyPhysicalRules(plan, preprocessingRules)
          logDebug(s"Adaptive execution enabled for plan: $plan")
          AdaptiveSparkPlanExec(newPlan, adaptiveExecutionContext, preprocessingRules, isSubquery)
        } catch {
          case SubqueryAdaptiveNotSupportedException(subquery) =>
            logWarning(s"${SQLConf.ADAPTIVE_EXECUTION_ENABLED.key} is enabled " +
              s"but is not supported for sub-query: $subquery.")
            plan
        }
      } else {
        logDebug(s"${SQLConf.ADAPTIVE_EXECUTION_ENABLED.key} is enabled " +
          s"but is not supported for query: $plan.")
        plan
      }

    case _ => plan
  }

  // AQE is only useful when the query has exchanges or sub-queries. This method returns true if
  // one of the following conditions is satisfied:
  //   - The config ADAPTIVE_EXECUTION_FORCE_APPLY is true.
  //   - The input query is from a sub-query. When this happens, it means we've already decided to
  //     apply AQE for the main query and we must continue to do it.
  //   - The query contains exchanges.
  //   - The query may need to add exchanges. It's an overkill to run `EnsureRequirements` here, so
  //     we just check `SparkPlan.requiredChildDistribution` and see if it's possible that the
  //     the query needs to add exchanges later.
  //   - The query contains sub-query.
  private def shouldApplyAQE(plan: SparkPlan, isSubquery: Boolean): Boolean = {
    conf.getConf(SQLConf.ADAPTIVE_EXECUTION_FORCE_APPLY) || isSubquery || {
      plan.exists {
        case _: Exchange => true
        case p if !p.requiredChildDistribution.forall(_ == UnspecifiedDistribution) => true
        case p => p.expressions.exists(_.exists {
          case _: SubqueryExpression => true
          case _ => false
        })
      }
    }
  } 
```

## **AdaptiveSparkPlanExec**

org.apache.spark.sql.execution.adaptive.AdaptiveSparkPlanExec
```

/**
 * A root node to execute the query plan adaptively. It splits the query plan into independent
 * stages and executes them in order according to their dependencies. The query stage
 * materializes its output at the end. When one stage completes, the data statistics of the
 * materialized output will be used to optimize the remainder of the query.
 *
 * To create query stages, we traverse the query tree bottom up. When we hit an exchange node,
 * and if all the child query stages of this exchange node are materialized, we create a new
 * query stage for this exchange node. The new stage is then materialized asynchronously once it
 * is created.
 *
 * When one query stage finishes materialization, the rest query is re-optimized and planned based
 * on the latest statistics provided by all materialized stages. Then we traverse the query plan
 * again and create more stages if possible. After all stages have been materialized, we execute
 * the rest of the plan.
 */
case class AdaptiveSparkPlanExec(
    inputPlan: SparkPlan,
    @transient context: AdaptiveExecutionContext,
    @transient preprocessingRules: Seq[Rule[SparkPlan]],
    @transient isSubquery: Boolean,
    @transient override val supportsColumnar: Boolean = false)
  extends LeafExecNode {

  override def doExecute(): RDD[InternalRow] = {
    withFinalPlanUpdate(_.execute())
  }
  
  private def withFinalPlanUpdate[T](fun: SparkPlan => T): T = {
    val plan = getFinalPhysicalPlan()
    val result = fun(plan)
    finalPlanUpdate
    result
  }

 ====================  getFinalPhysicalPlan =========================
  private def getFinalPhysicalPlan(): SparkPlan = lock.synchronized {
    if (isFinalPlan) return currentPhysicalPlan

    // In case of this adaptive plan being executed out of `withActive` scoped functions, e.g.,
    // `plan.queryExecution.rdd`, we need to set active session here as new plan nodes can be
    // created in the middle of the execution.
    context.session.withActive {
      val executionId = getExecutionId
      // Use inputPlan logicalLink here in case some top level physical nodes may be removed
      // during `initialPlan`
      var currentLogicalPlan = inputPlan.logicalLink.get
      var result = createQueryStages(currentPhysicalPlan) ===> createQueryStages
      val events = new LinkedBlockingQueue[StageMaterializationEvent]()
      val errors = new mutable.ArrayBuffer[Throwable]()
      var stagesToReplace = Seq.empty[QueryStageExec]
      while (!result.allChildStagesMaterialized) { ===> wait till all child stages materialized
        currentPhysicalPlan = result.newPlan
        if (result.newStages.nonEmpty) {
          stagesToReplace = result.newStages ++ stagesToReplace
          executionId.foreach(onUpdatePlan(_, result.newStages.map(_.plan)))

          // SPARK-33933: we should submit tasks of broadcast stages first, to avoid waiting
          // for tasks to be scheduled and leading to broadcast timeout.
          // This partial fix only guarantees the start of materialization for BroadcastQueryStage
          // is prior to others, but because the submission of collect job for broadcasting is
          // running in another thread, the issue is not completely resolved.
          val reorderedNewStages = result.newStages
            .sortWith {
              case (_: BroadcastQueryStageExec, _: BroadcastQueryStageExec) => false
              case (_: BroadcastQueryStageExec, _) => true
              case _ => false
            }


          ====================  stage.materialize() is run as Future async =========================
          // Start materialization of all new stages and fail fast if any stages failed eagerly
          reorderedNewStages.foreach { stage =>
            try {
              stage.materialize().onComplete { res => ===> materialize() is done async
                if (res.isSuccess) {
                  events.offer(StageSuccess(stage, res.get)) ===> put into events queue
                } else {
                  events.offer(StageFailure(stage, res.failed.get))
                }
              }(AdaptiveSparkPlanExec.executionContext)
            } catch {
              case e: Throwable =>
                cleanUpAndThrowException(Seq(e), Some(stage.id))
            }
          }
          ==========================================================================================
          
        }

        ====== Wait on the next completed stage ======
        // Wait on the next completed stage, which indicates new stats are available and probably
        // new stages can be created. There might be other stages that finish at around the same
        // time, so we process those stages too in order to reduce re-planning.
        val nextMsg = events.take()  ===> block wait on event queue
        val rem = new util.ArrayList[StageMaterializationEvent]()
        events.drainTo(rem)
        (Seq(nextMsg) ++ rem.asScala).foreach {
          case StageSuccess(stage, res) =>
            stage.resultOption.set(Some(res))
          case StageFailure(stage, ex) =>
            errors.append(ex)
        }

        // In case of errors, we cancel all running stages and throw exception.
        if (errors.nonEmpty) {
          cleanUpAndThrowException(errors.toSeq, None)
        }

        ======= re-optimizing and re-planning ======
        // Try re-optimizing and re-planning. Adopt the new plan if its cost is equal to or less
        // than that of the current plan; otherwise keep the current physical plan together with
        // the current logical plan since the physical plan's logical links point to the logical
        // plan it has originated from.
        // Meanwhile, we keep a list of the query stages that have been created since last plan
        // update, which stands for the "semantic gap" between the current logical and physical
        // plans. And each time before re-planning, we replace the corresponding nodes in the
        // current logical plan with logical query stages to make it semantically in sync with
        // the current physical plan. Once a new plan is adopted and both logical and physical
        // plans are updated, we can clear the query stage list because at this point the two plans
        // are semantically and physically in sync again.
        val logicalPlan = replaceWithQueryStagesInLogicalPlan(currentLogicalPlan, stagesToReplace)
        val afterReOptimize = reOptimize(logicalPlan)
        if (afterReOptimize.isDefined) {
          val (newPhysicalPlan, newLogicalPlan) = afterReOptimize.get
          val origCost = costEvaluator.evaluateCost(currentPhysicalPlan)
          val newCost = costEvaluator.evaluateCost(newPhysicalPlan)
          if (newCost < origCost ||
            (newCost == origCost && currentPhysicalPlan != newPhysicalPlan)) {
            logOnLevel("Plan changed:\n" +
              sideBySide(currentPhysicalPlan.treeString, newPhysicalPlan.treeString).mkString("\n"))
            cleanUpTempTags(newPhysicalPlan)
            currentPhysicalPlan = newPhysicalPlan
            currentLogicalPlan = newLogicalPlan
            stagesToReplace = Seq.empty[QueryStageExec]
          }
        }
        // Now that some stages have finished, we can try creating new stages.
        result = createQueryStages(currentPhysicalPlan)
      }

      // Run the final plan when there's no more unfinished stages.
      currentPhysicalPlan = applyPhysicalRules(
        optimizeQueryStage(result.newPlan, isFinalStage = true),
        postStageCreationRules(supportsColumnar),
        Some((planChangeLogger, "AQE Post Stage Creation")))
      isFinalPlan = true
      executionId.foreach(onUpdatePlan(_, Seq(currentPhysicalPlan)))
      currentPhysicalPlan
    }
  }


  /**
   * This method is called recursively to traverse the plan tree bottom-up and create a new query
   * stage or try reusing an existing stage if the current node is an [[Exchange]] node and all of
   * its child stages have been materialized.
   *
   * With each call, it returns:
   * 1) The new plan replaced with [[QueryStageExec]] nodes where new stages are created.
   * 2) Whether the child query stages (if any) of the current node have all been materialized.
   * 3) A list of the new query stages that have been created.
   */
  private def createQueryStages(plan: SparkPlan): CreateStageResult = plan match {
    case e: Exchange =>
      // First have a quick check in the `stageCache` without having to traverse down the node.
      context.stageCache.get(e.canonicalized) match {
        case Some(existingStage) if conf.exchangeReuseEnabled =>
          val stage = reuseQueryStage(existingStage, e)
          val isMaterialized = stage.isMaterialized
          CreateStageResult(
            newPlan = stage,
            allChildStagesMaterialized = isMaterialized,
            newStages = if (isMaterialized) Seq.empty else Seq(stage))

        case _ =>
          val result = createQueryStages(e.child)
          val newPlan = e.withNewChildren(Seq(result.newPlan)).asInstanceOf[Exchange]
          // Create a query stage only when all the child query stages are ready.
          if (result.allChildStagesMaterialized) {
            var newStage = newQueryStage(newPlan)
            if (conf.exchangeReuseEnabled) {
              // Check the `stageCache` again for reuse. If a match is found, ditch the new stage
              // and reuse the existing stage found in the `stageCache`, otherwise update the
              // `stageCache` with the new stage.
              val queryStage = context.stageCache.getOrElseUpdate(
                newStage.plan.canonicalized, newStage)
              if (queryStage.ne(newStage)) {
                newStage = reuseQueryStage(queryStage, e)
              }
            }
            val isMaterialized = newStage.isMaterialized
            CreateStageResult(
              newPlan = newStage,
              allChildStagesMaterialized = isMaterialized,
              newStages = if (isMaterialized) Seq.empty else Seq(newStage))
          } else {
            CreateStageResult(newPlan = newPlan,
              allChildStagesMaterialized = false, newStages = result.newStages)
          }
      }

    case q: QueryStageExec =>
      CreateStageResult(newPlan = q,
        allChildStagesMaterialized = q.isMaterialized, newStages = Seq.empty)

    case _ =>
      if (plan.children.isEmpty) {
        CreateStageResult(newPlan = plan, allChildStagesMaterialized = true, newStages = Seq.empty)
      } else {
        val results = plan.children.map(createQueryStages)
        CreateStageResult(
          newPlan = plan.withNewChildren(results.map(_.newPlan)),
          allChildStagesMaterialized = results.forall(_.allChildStagesMaterialized),
          newStages = results.flatMap(_.newStages))
      }
  }
  
  private def newQueryStage(e: Exchange): QueryStageExec = {
    val optimizedPlan = optimizeQueryStage(e.child, isFinalStage = false)
    val queryStage = e match {
      case s: ShuffleExchangeLike =>
        val newShuffle = applyPhysicalRules(
          s.withNewChildren(Seq(optimizedPlan)),
          postStageCreationRules(outputsColumnar = s.supportsColumnar),
          Some((planChangeLogger, "AQE Post Stage Creation")))
        if (!newShuffle.isInstanceOf[ShuffleExchangeLike]) {
          throw new IllegalStateException(
            "Custom columnar rules cannot transform shuffle node to something else.")
        }
        ShuffleQueryStageExec(currentStageId, newShuffle, s.canonicalized)
      case b: BroadcastExchangeLike =>
        val newBroadcast = applyPhysicalRules(
          b.withNewChildren(Seq(optimizedPlan)),
          postStageCreationRules(outputsColumnar = b.supportsColumnar),
          Some((planChangeLogger, "AQE Post Stage Creation")))
        if (!newBroadcast.isInstanceOf[BroadcastExchangeLike]) {
          throw new IllegalStateException(
            "Custom columnar rules cannot transform broadcast node to something else.")
        }
        BroadcastQueryStageExec(currentStageId, newBroadcast, b.canonicalized)
    }
    currentStageId += 1
    setLogicalLinkForNewQueryStage(queryStage, e)
    queryStage
  }
  
  

    
 
```

**rules**
```
  @transient private val costEvaluator =
    conf.getConf(SQLConf.ADAPTIVE_CUSTOM_COST_EVALUATOR_CLASS) match {
      case Some(className) => CostEvaluator.instantiate(className, session.sparkContext.getConf)
      case _ => SimpleCostEvaluator(conf.getConf(SQLConf.ADAPTIVE_FORCE_OPTIMIZE_SKEWED_JOIN))
    }

  // A list of physical plan rules to be applied before creation of query stages. The physical
  // plan should reach a final status of query stages (i.e., no more addition or removal of
  // Exchange nodes) after running these rules.
  @transient private val queryStagePreparationRules: Seq[Rule[SparkPlan]] = {
    // For cases like `df.repartition(a, b).select(c)`, there is no distribution requirement for
    // the final plan, but we do need to respect the user-specified repartition. Here we ask
    // `EnsureRequirements` to not optimize out the user-specified repartition-by-col to work
    // around this case.
    val ensureRequirements =
      EnsureRequirements(requiredDistribution.isDefined, requiredDistribution)
    Seq(
      RemoveRedundantProjects,
      ensureRequirements,
      ValidateSparkPlan,
      ReplaceHashWithSortAgg,
      RemoveRedundantSorts,
      DisableUnnecessaryBucketedScan,
      OptimizeSkewedJoin(ensureRequirements)
    ) ++ context.session.sessionState.adaptiveRulesHolder.queryStagePrepRules
  }

  // A list of physical optimizer rules to be applied to a new stage before its execution. These
  // optimizations should be stage-independent.
  @transient private val queryStageOptimizerRules: Seq[Rule[SparkPlan]] = Seq(
    PlanAdaptiveDynamicPruningFilters(this),
    ReuseAdaptiveSubquery(context.subqueryCache),
    OptimizeSkewInRebalancePartitions,
    CoalesceShufflePartitions(context.session),
    // `OptimizeShuffleWithLocalRead` needs to make use of 'AQEShuffleReadExec.partitionSpecs'
    // added by `CoalesceShufflePartitions`, and must be executed after it.
    OptimizeShuffleWithLocalRead
  )

  // This rule is stateful as it maintains the codegen stage ID. We can't create a fresh one every
  // time and need to keep it in a variable.
  @transient private val collapseCodegenStagesRule: Rule[SparkPlan] =
    CollapseCodegenStages()

  // A list of physical optimizer rules to be applied right after a new stage is created. The input
  // plan to these rules has exchange as its root node.
  private def postStageCreationRules(outputsColumnar: Boolean) = Seq(
    ApplyColumnarRulesAndInsertTransitions(
      context.session.sessionState.columnarRules, outputsColumnar),
    collapseCodegenStagesRule
  )
  
  @transient val initialPlan = context.session.withActive {
    applyPhysicalRules(
      inputPlan, queryStagePreparationRules, Some((planChangeLogger, "AQE Preparations")))
  }

  @volatile private var currentPhysicalPlan = initialPlan
  
  // The logical plan optimizer for re-optimizing the current logical plan.
  @transient private val optimizer = new AQEOptimizer(conf,
    session.sessionState.adaptiveRulesHolder.runtimeOptimizerRules)
    
  private def optimizeQueryStage(plan: SparkPlan, isFinalStage: Boolean): SparkPlan = {
    val optimized = queryStageOptimizerRules.foldLeft(plan) { case (latestPlan, rule) =>
      val applied = rule.apply(latestPlan)
      val result = rule match {
        case _: AQEShuffleReadRule if !applied.fastEquals(latestPlan) =>
          val distribution = if (isFinalStage) {
            // If `requiredDistribution` is None, it means `EnsureRequirements` will not optimize
            // out the user-specified repartition, thus we don't have a distribution requirement
            // for the final plan.
            requiredDistribution.getOrElse(UnspecifiedDistribution)
          } else {
            UnspecifiedDistribution
          }
          if (ValidateRequirements.validate(applied, distribution)) {
            applied
          } else {
            logDebug(s"Rule ${rule.ruleName} is not applied as it breaks the " +
              "distribution requirement of the query plan.")
            latestPlan
          }
        case _ => applied
      }
      planChangeLogger.logRule(rule.ruleName, latestPlan, result)
      result
    }
    planChangeLogger.logBatch("AQE Query Stage Optimization", plan, optimized)
    optimized
  }
  
  /**
   * Re-optimize and run physical planning on the current logical plan based on the latest stats.
   */
  private def reOptimize(logicalPlan: LogicalPlan): Option[(SparkPlan, LogicalPlan)] = {
    try {
      logicalPlan.invalidateStatsCache()
      val optimized = optimizer.execute(logicalPlan)
      val sparkPlan = context.session.sessionState.planner.plan(ReturnAnswer(optimized)).next()
      val newPlan = applyPhysicalRules(
        sparkPlan,
        preprocessingRules ++ queryStagePreparationRules,
        Some((planChangeLogger, "AQE Replanning")))

      // When both enabling AQE and DPP, `PlanAdaptiveDynamicPruningFilters` rule will
      // add the `BroadcastExchangeExec` node manually in the DPP subquery,
      // not through `EnsureRequirements` rule. Therefore, when the DPP subquery is complicated
      // and need to be re-optimized, AQE also need to manually insert the `BroadcastExchangeExec`
      // node to prevent the loss of the `BroadcastExchangeExec` node in DPP subquery.
      // Here, we also need to avoid to insert the `BroadcastExchangeExec` node when the newPlan is
      // already the `BroadcastExchangeExec` plan after apply the `LogicalQueryStageStrategy` rule.
      val finalPlan = inputPlan match {
        case b: BroadcastExchangeLike
          if (!newPlan.isInstanceOf[BroadcastExchangeLike]) => b.withNewChildren(Seq(newPlan))
        case _ => newPlan
      }

      Some((finalPlan, optimized))
    } catch {
      case e: InvalidAQEPlanException[_] =>
        logOnLevel(s"Re-optimize - ${e.getMessage()}:\n${e.plan}")
        None
    }
  }
  
```

## **QueryStageExec**
org.apache.spark.sql.execution.adaptive.QueryStageExec
```

/**
 * A query stage is an independent subgraph of the query plan. Query stage materializes its output
 * before proceeding with further operators of the query plan. The data statistics of the
 * materialized output can be used to optimize subsequent query stages.
 *
 * There are 2 kinds of query stages:
 *   1. Shuffle query stage. This stage materializes its output to shuffle files, and Spark launches
 *      another job to execute the further operators.
 *   2. Broadcast query stage. This stage materializes its output to an array in driver JVM. Spark
 *      broadcasts the array before executing the further operators.
 */
abstract class QueryStageExec extends LeafExecNode {

  @transient
  @volatile
  protected var _resultOption = new AtomicReference[Option[Any]](None)

  private[adaptive] def resultOption: AtomicReference[Option[Any]] = _resultOption
  def isMaterialized: Boolean = resultOption.get().isDefined

  /**
   * Compute the statistics of the query stage if executed, otherwise None.
   */
  def computeStats(): Option[Statistics] = if (isMaterialized) {
    val runtimeStats = getRuntimeStatistics
    val dataSize = runtimeStats.sizeInBytes.max(0)
    val numOutputRows = runtimeStats.rowCount.map(_.max(0))
    Some(Statistics(dataSize, numOutputRows, isRuntime = true))
  } else {
    None
  }
  
  
  /**
   * Materialize this query stage, to prepare for the execution, like submitting map stages,
   * broadcasting data, etc. The caller side can use the returned [[Future]] to wait until this
   * stage is ready.
   */
  def doMaterialize(): Future[Any]

====== materialize() is called by org.apache.spark.sql.execution.adaptive.AdaptiveSparkPlanExec#getFinalPhysicalPlan  ===
  /**
   * Materialize this query stage, to prepare for the execution, like submitting map stages,
   * broadcasting data, etc. The caller side can use the returned [[Future]] to wait until this
   * stage is ready.
   */
  final def materialize(): Future[Any] = {  
    logDebug(s"Materialize query stage ${this.getClass.getSimpleName}: $id")
    doMaterialize()
  }
=========================================================================================================  
```

## **ShuffleQueryStageExec**
&nbsp;org.apache.spark.sql.execution.adaptive.AdaptiveSparkPlanExec#getFinalPhysicalPlan =>  
&nbsp;&nbsp;    org.apache.spark.sql.execution.adaptive.QueryStageExec#materialize =>  
&nbsp;&nbsp;&nbsp;        org.apache.spark.sql.execution.adaptive.ShuffleQueryStageExec#doMaterialize=>  
&nbsp;&nbsp;&nbsp;&nbsp;            org.apache.spark.sql.execution.exchange.ShuffleExchangeLike#submitShuffleJob=>  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;                org.apache.spark.sql.execution.SparkPlan#executeQuery=>  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;                    org.apache.spark.sql.execution.exchange.ShuffleExchangeExec#mapOutputStatisticsFuture=>  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;                        org.apache.spark.SparkContext#submitMapStage=>  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;                         org.apache.spark.scheduler.DAGScheduler#submitMapStage=>  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;                            org.apache.spark.scheduler.JobWaiter

run test `test("SPARK-37063: OptimizeSkewInRebalancePartitions support optimize non-root node")`

```
/**
 * A shuffle query stage whose child is a [[ShuffleExchangeLike]] or [[ReusedExchangeExec]].
 *
 * @param id the query stage id.
 * @param plan the underlying plan.
 * @param _canonicalized the canonicalized plan before applying query stage optimizer rules.
 */
case class ShuffleQueryStageExec(
    override val id: Int,
    override val plan: SparkPlan,
    override val _canonicalized: SparkPlan) extends QueryStageExec {

  @transient val shuffle = plan match {
    case s: ShuffleExchangeLike => s
    case ReusedExchangeExec(_, s: ShuffleExchangeLike) => s
    case _ =>
      throw new IllegalStateException(s"wrong plan for shuffle stage:\n ${plan.treeString}")
  }

  @transient private lazy val shuffleFuture = shuffle.submitShuffleJob  ===> org.apache.spark.sql.execution.exchange.ShuffleExchangeLike#submitShuffleJob

  override def doMaterialize(): Future[Any] = shuffleFuture
  
  
  override def getRuntimeStatistics: Statistics = shuffle.runtimeStatistics
```

org.apache.spark.sql.execution.exchange.ShuffleExchangeLike#submitShuffleJob

```
  /**
   * The asynchronous job that materializes the shuffle. It also does the preparations work,
   * such as waiting for the subqueries.
   */
  final def submitShuffleJob: Future[MapOutputStatistics] = executeQuery {
    mapOutputStatisticsFuture
  }

```
===> org.apache.spark.sql.execution.exchange.ShuffleExchangeExec#mapOutputStatisticsFuture
```
  // 'mapOutputStatisticsFuture' is only needed when enable AQE.
  @transient
  override lazy val mapOutputStatisticsFuture: Future[MapOutputStatistics] = {
    if (inputRDD.getNumPartitions == 0) {
      Future.successful(null)
    } else {
      sparkContext.submitMapStage(shuffleDependency)
    }
  }
```

===> org.apache.spark.sql.execution.SparkPlan#executeQuery
```
  /**
   * Executes a query after preparing the query and adding query plan information to created RDDs
   * for visualization.
   */
  protected final def executeQuery[T](query: => T): T = {
    RDDOperationScope.withScope(sparkContext, nodeName, false, true) {
      prepare()
      waitForSubqueries()
      query
    }
  }

```
===> org.apache.spark.SparkContext#submitMapStage
```
  /**
   * Submit a map stage for execution. This is currently an internal API only, but might be
   * promoted to DeveloperApi in the future.
   */
  private[spark] def submitMapStage[K, V, C](dependency: ShuffleDependency[K, V, C])
      : SimpleFutureAction[MapOutputStatistics] = {
    assertNotStopped()
    val callSite = getCallSite()
    var result: MapOutputStatistics = null
    val waiter = dagScheduler.submitMapStage(
      dependency,
      (r: MapOutputStatistics) => { result = r },
      callSite,
      localProperties.get)
    new SimpleFutureAction[MapOutputStatistics](waiter, result)
  }
```
===> org.apache.spark.scheduler.DAGScheduler#submitMapStage
```
  /**
   * Submit a shuffle map stage to run independently and get a JobWaiter object back. The waiter
   * can be used to block until the job finishes executing or can be used to cancel the job.
   * This method is used for adaptive query planning, to run map stages and look at statistics
   * about their outputs before submitting downstream stages.
   *
   * @param dependency the ShuffleDependency to run a map stage for
   * @param callback function called with the result of the job, which in this case will be a
   *   single MapOutputStatistics object showing how much data was produced for each partition
   * @param callSite where in the user program this job was submitted
   * @param properties scheduler properties to attach to this job, e.g. fair scheduler pool name
   */
  def submitMapStage[K, V, C](
      dependency: ShuffleDependency[K, V, C],
      callback: MapOutputStatistics => Unit,
      callSite: CallSite,
      properties: Properties): JobWaiter[MapOutputStatistics] = {

    val rdd = dependency.rdd
    val jobId = nextJobId.getAndIncrement()
    if (rdd.partitions.length == 0) {
      throw SparkCoreErrors.cannotRunSubmitMapStageOnZeroPartitionRDDError()
    }

    // SPARK-23626: `RDD.getPartitions()` can be slow, so we eagerly compute
    // `.partitions` on every RDD in the DAG to ensure that `getPartitions()`
    // is evaluated outside of the DAGScheduler's single-threaded event loop:
    eagerlyComputePartitionsForRddAndAncestors(rdd)

    // We create a JobWaiter with only one "task", which will be marked as complete when the whole
    // map stage has completed, and will be passed the MapOutputStatistics for that stage.
    // This makes it easier to avoid race conditions between the user code and the map output
    // tracker that might result if we told the user the stage had finished, but then they queries
    // the map output tracker and some node failures had caused the output statistics to be lost.
    val waiter = new JobWaiter[MapOutputStatistics](
      this, jobId, 1,
      (_: Int, r: MapOutputStatistics) => callback(r))
    eventProcessLoop.post(MapStageSubmitted(
      jobId, dependency, callSite, waiter, Utils.cloneProperties(properties)))
    waiter
  }
```
===> org.apache.spark.scheduler.JobWaiter
```
/**
 * An object that waits for a DAGScheduler job to complete. As tasks finish, it passes their
 * results to the given handler function.
 */
private[spark] class JobWaiter[T](
    dagScheduler: DAGScheduler,
    val jobId: Int,
    totalTasks: Int,
    resultHandler: (Int, T) => Unit)
  extends JobListener with Logging {
```


stack in `test("SPARK-37063: OptimizeSkewInRebalancePartitions support optimize non-root node")`
```
	  at org.apache.spark.sql.execution.SparkPlan.prepare(SparkPlan.scala:291)
	  at org.apache.spark.sql.execution.SparkPlan.$anonfun$executeQuery$1(SparkPlan.scala:244)
	  at org.apache.spark.sql.execution.SparkPlan$$Lambda$2339.1114370802.apply(Unknown Source:-1)
	  at org.apache.spark.rdd.RDDOperationScope$.withScope(RDDOperationScope.scala:151)
	  at org.apache.spark.sql.execution.SparkPlan.executeQuery(SparkPlan.scala:243)
	  at org.apache.spark.sql.execution.exchange.ShuffleExchangeLike.submitShuffleJob(ShuffleExchangeExec.scala:73)
	  at org.apache.spark.sql.execution.exchange.ShuffleExchangeLike.submitShuffleJob$(ShuffleExchangeExec.scala:72)
	  at org.apache.spark.sql.execution.exchange.ShuffleExchangeExec.submitShuffleJob(ShuffleExchangeExec.scala:120)
	  at org.apache.spark.sql.execution.adaptive.ShuffleQueryStageExec.shuffleFuture$lzycompute(QueryStageExec.scala:185)
	  - locked <0x362c> (a org.apache.spark.sql.execution.adaptive.ShuffleQueryStageExec)
	  at org.apache.spark.sql.execution.adaptive.ShuffleQueryStageExec.shuffleFuture(QueryStageExec.scala:185)
	  at org.apache.spark.sql.execution.adaptive.ShuffleQueryStageExec.doMaterialize(QueryStageExec.scala:187)
	  at org.apache.spark.sql.execution.adaptive.QueryStageExec.materialize(QueryStageExec.scala:61)
	  at org.apache.spark.sql.execution.adaptive.AdaptiveSparkPlanExec.$anonfun$getFinalPhysicalPlan$5(AdaptiveSparkPlanExec.scala:286)
	  at org.apache.spark.sql.execution.adaptive.AdaptiveSparkPlanExec.$anonfun$getFinalPhysicalPlan$5$adapted(AdaptiveSparkPlanExec.scala:284)
	  at org.apache.spark.sql.execution.adaptive.AdaptiveSparkPlanExec$$Lambda$2336.1706125628.apply(Unknown Source:-1)
	  at scala.collection.Iterator.foreach(Iterator.scala:943)
	  at scala.collection.Iterator.foreach$(Iterator.scala:943)
	  at scala.collection.AbstractIterator.foreach(Iterator.scala:1431)
	  at scala.collection.IterableLike.foreach(IterableLike.scala:74)
	  at scala.collection.IterableLike.foreach$(IterableLike.scala:73)
	  at scala.collection.AbstractIterable.foreach(Iterable.scala:56)
	  at org.apache.spark.sql.execution.adaptive.AdaptiveSparkPlanExec.$anonfun$getFinalPhysicalPlan$1(AdaptiveSparkPlanExec.scala:284)
	  at org.apache.spark.sql.execution.adaptive.AdaptiveSparkPlanExec$$Lambda$2295.312068212.apply(Unknown Source:-1)
	  at org.apache.spark.sql.SparkSession.withActive(SparkSession.scala:827)
	  at org.apache.spark.sql.execution.adaptive.AdaptiveSparkPlanExec.getFinalPhysicalPlan(AdaptiveSparkPlanExec.scala:256)
	  - locked <0x389c> (a java.lang.Object)
	  at org.apache.spark.sql.execution.adaptive.AdaptiveSparkPlanExec.withFinalPlanUpdate(AdaptiveSparkPlanExec.scala:401)
	  at org.apache.spark.sql.execution.adaptive.AdaptiveSparkPlanExec.executeCollect(AdaptiveSparkPlanExec.scala:374)
	  at org.apache.spark.sql.Dataset.collectFromPlan(Dataset.scala:4384)
	  at org.apache.spark.sql.Dataset.$anonfun$collect$1(Dataset.scala:3625)
	  at org.apache.spark.sql.Dataset$$Lambda$2283.289194317.apply(Unknown Source:-1)
	  at org.apache.spark.sql.Dataset.$anonfun$withAction$2(Dataset.scala:4374)
	  at org.apache.spark.sql.Dataset$$Lambda$2290.1482151715.apply(Unknown Source:-1)
	  at org.apache.spark.sql.execution.QueryExecution$.withInternalError(QueryExecution.scala:529)
	  at org.apache.spark.sql.Dataset.$anonfun$withAction$1(Dataset.scala:4372)
	  at org.apache.spark.sql.Dataset$$Lambda$2284.280804333.apply(Unknown Source:-1)
	  at org.apache.spark.sql.execution.SQLExecution$.$anonfun$withNewExecutionId$6(SQLExecution.scala:118)
	  at org.apache.spark.sql.execution.SQLExecution$$$Lambda$1713.806842585.apply(Unknown Source:-1)
	  at org.apache.spark.sql.execution.SQLExecution$.withSQLConfPropagated(SQLExecution.scala:195)
	  at org.apache.spark.sql.execution.SQLExecution$.$anonfun$withNewExecutionId$1(SQLExecution.scala:103)
	  at org.apache.spark.sql.execution.SQLExecution$$$Lambda$1703.1020198427.apply(Unknown Source:-1)
	  at org.apache.spark.sql.SparkSession.withActive(SparkSession.scala:827)
	  at org.apache.spark.sql.execution.SQLExecution$.withNewExecutionId(SQLExecution.scala:65)
	  at org.apache.spark.sql.Dataset.withAction(Dataset.scala:4372)
	  at org.apache.spark.sql.Dataset.collect(Dataset.scala:3625)
	  at org.apache.spark.sql.execution.adaptive.AdaptiveQueryExecSuite.runAdaptiveAndVerifyResult(AdaptiveQueryExecSuite.scala:83)
	  at org.apache.spark.sql.execution.adaptive.AdaptiveQueryExecSuite.checkRebalance$1(AdaptiveQueryExecSuite.scala:2417)
	  at org.apache.spark.sql.execution.adaptive.AdaptiveQueryExecSuite.$anonfun$new$273(AdaptiveQueryExecSuite.scala:2430)
	  at org.apache.spark.sql.execution.adaptive.AdaptiveQueryExecSuite$$Lambda$2074.1374673778.apply$mcV$sp(Unknown Source:-1)
	  at org.apache.spark.sql.catalyst.plans.SQLHelper.withSQLConf(SQLHelper.scala:54)
	  at org.apache.spark.sql.catalyst.plans.SQLHelper.withSQLConf$(SQLHelper.scala:38)
	  at org.apache.spark.sql.execution.adaptive.AdaptiveQueryExecSuite.org$apache$spark$sql$test$SQLTestUtilsBase$$super$withSQLConf(AdaptiveQueryExecSuite.scala:51)
	  at org.apache.spark.sql.test.SQLTestUtilsBase.withSQLConf(SQLTestUtils.scala:247)
	  at org.apache.spark.sql.test.SQLTestUtilsBase.withSQLConf$(SQLTestUtils.scala:245)
	  at org.apache.spark.sql.execution.adaptive.AdaptiveQueryExecSuite.withSQLConf(AdaptiveQueryExecSuite.scala:51)
	  at org.apache.spark.sql.execution.adaptive.AdaptiveQueryExecSuite.$anonfun$new$270(AdaptiveQueryExecSuite.scala:2429)
	  at org.apache.spark.sql.execution.adaptive.AdaptiveQueryExecSuite$$Lambda$2046.860717660.apply$mcV$sp(Unknown Source:-1)
	  at org.apache.spark.sql.catalyst.plans.SQLHelper.withSQLConf(SQLHelper.scala:54)
	  at org.apache.spark.sql.catalyst.plans.SQLHelper.withSQLConf$(SQLHelper.scala:38)
	  at org.apache.spark.sql.execution.adaptive.AdaptiveQueryExecSuite.org$apache$spark$sql$test$SQLTestUtilsBase$$super$withSQLConf(AdaptiveQueryExecSuite.scala:51)
	  at org.apache.spark.sql.test.SQLTestUtilsBase.withSQLConf(SQLTestUtils.scala:247)
	  at org.apache.spark.sql.test.SQLTestUtilsBase.withSQLConf$(SQLTestUtils.scala:245)
	  at org.apache.spark.sql.execution.adaptive.AdaptiveQueryExecSuite.withSQLConf(AdaptiveQueryExecSuite.scala:51)
	  at org.apache.spark.sql.execution.adaptive.AdaptiveQueryExecSuite.$anonfun$new$269(AdaptiveQueryExecSuite.scala:2411)
	  at org.apache.spark.sql.execution.adaptive.AdaptiveQueryExecSuite$$Lambda$2044.1413677222.apply$mcV$sp(Unknown Source:-1)
```

## **BroadcastQueryStageExec**

```
/**
 * A broadcast query stage whose child is a [[BroadcastExchangeLike]] or [[ReusedExchangeExec]].
 *
 * @param id the query stage id.
 * @param plan the underlying plan.
 * @param _canonicalized the canonicalized plan before applying query stage optimizer rules.
 */
case class BroadcastQueryStageExec(
    override val id: Int,
    override val plan: SparkPlan,
    override val _canonicalized: SparkPlan) extends QueryStageExec {

  @transient val broadcast = plan match {
    case b: BroadcastExchangeLike => b
    case ReusedExchangeExec(_, b: BroadcastExchangeLike) => b
    case _ =>
      throw new IllegalStateException(s"wrong plan for broadcast stage:\n ${plan.treeString}")
  }

  override def doMaterialize(): Future[Any] = {
    broadcast.submitBroadcastJob
  }
  
  override def getRuntimeStatistics: Statistics = broadcast.runtimeStatistics
```

## **reuseQueryStage**

org.apache.spark.sql.execution.adaptive.AdaptiveSparkPlanExec#createQueryStages => reuseQueryStage  

org.apache.spark.sql.execution.adaptive.AdaptiveSparkPlanExec#reuseQueryStage
```
  private def reuseQueryStage(existing: QueryStageExec, exchange: Exchange): QueryStageExec = {
    val queryStage = existing.newReuseInstance(currentStageId, exchange.output)
    currentStageId += 1
    setLogicalLinkForNewQueryStage(queryStage, exchange)
    queryStage
  }
```

org.apache.spark.sql.execution.adaptive.BroadcastQueryStageExec#newReuseInstance
```
  override def newReuseInstance(newStageId: Int, newOutput: Seq[Attribute]): QueryStageExec = {
    val reuse = BroadcastQueryStageExec(
      newStageId,
      ReusedExchangeExec(newOutput, broadcast),
      _canonicalized)
    reuse._resultOption = this._resultOption
    reuse
  }
```

org.apache.spark.sql.execution.adaptive.ShuffleQueryStageExec#newReuseInstance
```
  override def newReuseInstance(newStageId: Int, newOutput: Seq[Attribute]): QueryStageExec = {
    val reuse = ShuffleQueryStageExec(
      newStageId,
      ReusedExchangeExec(newOutput, shuffle),
      _canonicalized)
    reuse._resultOption = this._resultOption
    reuse
  }
```

## Adaptive coalesce partitions
SQLConf
```scala
  val COALESCE_PARTITIONS_ENABLED =
    buildConf("spark.sql.adaptive.coalescePartitions.enabled")
      .doc(s"When true and '${ADAPTIVE_EXECUTION_ENABLED.key}' is true, Spark will coalesce " +
        "contiguous shuffle partitions according to the target size (specified by " +
        s"'${ADVISORY_PARTITION_SIZE_IN_BYTES.key}'), to avoid too many small tasks.")
      .version("3.0.0")
      .booleanConf
      .createWithDefault(true)

  val COALESCE_PARTITIONS_PARALLELISM_FIRST =
    buildConf("spark.sql.adaptive.coalescePartitions.parallelismFirst")
      .doc("When true, Spark does not respect the target size specified by " +
        s"'${ADVISORY_PARTITION_SIZE_IN_BYTES.key}' (default 64MB) when coalescing contiguous " +
        "shuffle partitions, but adaptively calculate the target size according to the default " +
        "parallelism of the Spark cluster. The calculated size is usually smaller than the " +
        "configured target size. This is to maximize the parallelism and avoid performance " +
        "regression when enabling adaptive query execution. It's recommended to set this config " +
        "to false and respect the configured target size.")
      .version("3.2.0")
      .booleanConf
      .createWithDefault(true)

  val COALESCE_PARTITIONS_MIN_PARTITION_SIZE =
    buildConf("spark.sql.adaptive.coalescePartitions.minPartitionSize")
      .doc("The minimum size of shuffle partitions after coalescing. This is useful when the " +
        "adaptively calculated target size is too small during partition coalescing.")
      .version("3.2.0")
      .bytesConf(ByteUnit.BYTE)
      .checkValue(_ > 0, "minPartitionSize must be positive")
      .createWithDefaultString("1MB")

  val COALESCE_PARTITIONS_MIN_PARTITION_NUM =
    buildConf("spark.sql.adaptive.coalescePartitions.minPartitionNum")
      .internal()
      .doc("(deprecated) The suggested (not guaranteed) minimum number of shuffle partitions " +
        "after coalescing. If not set, the default value is the default parallelism of the " +
        "Spark cluster. This configuration only has an effect when " +
        s"'${ADAPTIVE_EXECUTION_ENABLED.key}' and " +
        s"'${COALESCE_PARTITIONS_ENABLED.key}' are both true.")
      .version("3.0.0")
      .intConf
      .checkValue(_ > 0, "The minimum number of partitions must be positive.")
      .createOptional

  val COALESCE_PARTITIONS_INITIAL_PARTITION_NUM =
    buildConf("spark.sql.adaptive.coalescePartitions.initialPartitionNum")
      .doc("The initial number of shuffle partitions before coalescing. If not set, it equals to " +
        s"${SHUFFLE_PARTITIONS.key}. This configuration only has an effect when " +
        s"'${ADAPTIVE_EXECUTION_ENABLED.key}' and '${COALESCE_PARTITIONS_ENABLED.key}' " +
        "are both true.")
      .version("3.0.0")
      .intConf
      .checkValue(_ > 0, "The initial number of partitions must be positive.")
      .createOptional
```
org.apache.spark.sql.execution.adaptive.AdaptiveSparkPlanExec#queryStageOptimizerRules
```
  // A list of physical optimizer rules to be applied to a new stage before its execution. These
  // optimizations should be stage-independent.
  @transient private val queryStageOptimizerRules: Seq[Rule[SparkPlan]] = Seq(
    PlanAdaptiveDynamicPruningFilters(this),
    ReuseAdaptiveSubquery(context.subqueryCache),
    OptimizeSkewInRebalancePartitions,
    
    ====== rule for coalesce partitions ==========
    CoalesceShufflePartitions(context.session),
    ==============================================
    // `OptimizeShuffleWithLocalRead` needs to make use of 'AQEShuffleReadExec.partitionSpecs'
    // added by `CoalesceShufflePartitions`, and must be executed after it.
    OptimizeShuffleWithLocalRead
  )
```

org.apache.spark.sql.execution.adaptive.CoalesceShufflePartitions
```
/**
 * A rule to coalesce the shuffle partitions based on the map output statistics, which can
 * avoid many small reduce tasks that hurt performance.
 */
case class CoalesceShufflePartitions(session: SparkSession) extends AQEShuffleReadRule {
  override def apply(plan: SparkPlan): SparkPlan = {
    if (!conf.coalesceShufflePartitionsEnabled) {
      return plan
    }

    // Ideally, this rule should simply coalesce partitions w.r.t. the target size specified by
    // ADVISORY_PARTITION_SIZE_IN_BYTES (default 64MB). To avoid perf regression in AQE, this
    // rule by default tries to maximize the parallelism and set the target size to
    // `total shuffle size / Spark default parallelism`. In case the `Spark default parallelism`
    // is too big, this rule also respect the minimum partition size specified by
    // COALESCE_PARTITIONS_MIN_PARTITION_SIZE (default 1MB).
    // For history reason, this rule also need to support the config
    // COALESCE_PARTITIONS_MIN_PARTITION_NUM. We should remove this config in the future.
    val minNumPartitions = conf.getConf(SQLConf.COALESCE_PARTITIONS_MIN_PARTITION_NUM).getOrElse {
      if (conf.getConf(SQLConf.COALESCE_PARTITIONS_PARALLELISM_FIRST)) {
        // We fall back to Spark default parallelism if the minimum number of coalesced partitions
        // is not set, so to avoid perf regressions compared to no coalescing.
        session.sparkContext.defaultParallelism
      } else {
        // If we don't need to maximize the parallelism, we set `minPartitionNum` to 1, so that
        // the specified advisory partition size will be respected.
        1
      }
    }
    val advisoryTargetSize = conf.getConf(SQLConf.ADVISORY_PARTITION_SIZE_IN_BYTES)
    val minPartitionSize = if (Utils.isTesting) {
      // In the tests, we usually set the target size to a very small value that is even smaller
      // than the default value of the min partition size. Here we also adjust the min partition
      // size to be not larger than 20% of the target size, so that the tests don't need to set
      // both configs all the time to check the coalescing behavior.
      conf.getConf(SQLConf.COALESCE_PARTITIONS_MIN_PARTITION_SIZE).min(advisoryTargetSize / 5)
    } else {
      conf.getConf(SQLConf.COALESCE_PARTITIONS_MIN_PARTITION_SIZE)
    }

    // Sub-plans under the Union operator can be coalesced independently, so we can divide them
    // into independent "coalesce groups", and all shuffle stages within each group have to be
    // coalesced together.
    val coalesceGroups = collectCoalesceGroups(plan)

    // Divide minimum task parallelism among coalesce groups according to their data sizes.
    val minNumPartitionsByGroup = if (coalesceGroups.length == 1) {
      Seq(math.max(minNumPartitions, 1))
    } else {
      val sizes =
        coalesceGroups.map(_.flatMap(_.shuffleStage.mapStats.map(_.bytesByPartitionId.sum)).sum)
      val totalSize = sizes.sum
      sizes.map { size =>
        val num = if (totalSize > 0) {
          math.round(minNumPartitions * 1.0 * size / totalSize)
        } else {
          minNumPartitions
        }
        math.max(num.toInt, 1)
      }
    }

    val specsMap = mutable.HashMap.empty[Int, Seq[ShufflePartitionSpec]]
    // Coalesce partitions for each coalesce group independently.
    coalesceGroups.zip(minNumPartitionsByGroup).foreach { case (shuffleStages, minNumPartitions) =>
      val newPartitionSpecs = ShufflePartitionsUtil.coalescePartitions(
        shuffleStages.map(_.shuffleStage.mapStats),
        shuffleStages.map(_.partitionSpecs),
        advisoryTargetSize = advisoryTargetSize,
        minNumPartitions = minNumPartitions,
        minPartitionSize = minPartitionSize)

      if (newPartitionSpecs.nonEmpty) {
        shuffleStages.zip(newPartitionSpecs).map { case (stageInfo, partSpecs) =>
          specsMap.put(stageInfo.shuffleStage.id, partSpecs)
        }
      }
    }

    if (specsMap.nonEmpty) {
      updateShuffleReads(plan, specsMap.toMap)
    } else {
      plan
    }
  }

  private def updateShuffleReads(
      plan: SparkPlan, specsMap: Map[Int, Seq[ShufflePartitionSpec]]): SparkPlan = plan match {
    // Even for shuffle exchange whose input RDD has 0 partition, we should still update its
    // `partitionStartIndices`, so that all the leaf shuffles in a stage have the same
    // number of output partitions.
    case ShuffleStageInfo(stage, _) =>
      specsMap.get(stage.id).map { specs =>
        AQEShuffleReadExec(stage, specs)
      }.getOrElse(plan)
    case other => other.mapChildren(updateShuffleReads(_, specsMap))
  }

```

org.apache.spark.sql.execution.adaptive.AQEShuffleReadRule
```

/**
 * A rule that may create [[AQEShuffleReadExec]] on top of [[ShuffleQueryStageExec]] and change the
 * plan output partitioning. The AQE framework will skip the rule if it leads to extra shuffles.
 */
trait AQEShuffleReadRule extends Rule[SparkPlan] {
  /**
   * Returns the list of [[ShuffleOrigin]]s supported by this rule.
   */
  protected def supportedShuffleOrigins: Seq[ShuffleOrigin]

  protected def isSupported(shuffle: ShuffleExchangeLike): Boolean = {
    supportedShuffleOrigins.contains(shuffle.shuffleOrigin)
  }
}
```

org.apache.spark.sql.execution.adaptive.AQEShuffleReadExec
```
/**
 * A wrapper of shuffle query stage, which follows the given partition arrangement.
 *
 * @param child           It is usually `ShuffleQueryStageExec`, but can be the shuffle exchange
 *                        node during canonicalization.
 * @param partitionSpecs  The partition specs that defines the arrangement, requires at least one
 *                        partition.
 */
case class AQEShuffleReadExec private(
    child: SparkPlan,
    partitionSpecs: Seq[ShufflePartitionSpec]) extends UnaryExecNode {

  private def shuffleStage = child match {
    case stage: ShuffleQueryStageExec => Some(stage)
    case _ => None
  }
      
  private lazy val shuffleRDD: RDD[_] = {
    shuffleStage match {
      case Some(stage) =>
        sendDriverMetrics()
        stage.shuffle.getShuffleRDD(partitionSpecs.toArray)
      case _ =>
        throw new IllegalStateException("operating on canonicalized plan")
    }
  }

  override protected def doExecute(): RDD[InternalRow] = {
    shuffleRDD.asInstanceOf[RDD[InternalRow]]
  }
```

org.apache.spark.sql.execution.exchange.ShuffleExchangeExec#getShuffleRDD
```
  override def getShuffleRDD(partitionSpecs: Array[ShufflePartitionSpec]): RDD[InternalRow] = {
    new ShuffledRowRDD(shuffleDependency, readMetrics, partitionSpecs)
  }
```