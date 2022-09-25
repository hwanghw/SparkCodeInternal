[TOC]

# SparkPlan

## Spark's Planner
* 1st Phase: Transforms the logical plan to the physical plan using Strategies  
**QueryExecution.sparkPlan**  
**SparkPlanner.plan**

* 2nd Phase: use a Rule Executor to make the Physical Plan ready for execution  
**QueryExecution.prepareForExecution**

## Call stack

How is QueryExecution.sparkPlan and QueryExecution.prepareForExecution invoked?

QueryExecution.sparkPlan stack
```
	  at org.apache.spark.sql.execution.QueryExecution.sparkPlan$lzycompute(QueryExecution.scala:141)=====> lazy val sparkPlan: SparkPlan
	  - locked <0x328c> (a org.apache.spark.sql.execution.QueryExecution)
	  at org.apache.spark.sql.execution.QueryExecution.sparkPlan(QueryExecution.scala:138)
	  at org.apache.spark.sql.execution.QueryExecution.$anonfun$executedPlan$1(QueryExecution.scala:158)
	  at org.apache.spark.sql.execution.QueryExecution$$Lambda$1861.30604162.apply(Unknown Source:-1)
	  at org.apache.spark.sql.catalyst.QueryPlanningTracker.measurePhase(QueryPlanningTracker.scala:111)
	  at org.apache.spark.sql.execution.QueryExecution.$anonfun$executePhase$2(QueryExecution.scala:185)
	  at org.apache.spark.sql.execution.QueryExecution$$Lambda$1461.609375192.apply(Unknown Source:-1)
	  at org.apache.spark.sql.execution.QueryExecution$.withInternalError(QueryExecution.scala:512)
	  at org.apache.spark.sql.execution.QueryExecution.$anonfun$executePhase$1(QueryExecution.scala:185)
	  at org.apache.spark.sql.execution.QueryExecution$$Lambda$1460.911201454.apply(Unknown Source:-1)
	  at org.apache.spark.sql.SparkSession.withActive(SparkSession.scala:779)
	  at org.apache.spark.sql.execution.QueryExecution.executePhase(QueryExecution.scala:184)
	  at org.apache.spark.sql.execution.QueryExecution.executedPlan$lzycompute(QueryExecution.scala:158)
	  at org.apache.spark.sql.execution.QueryExecution.executedPlan(QueryExecution.scala:151)
	  at org.apache.spark.sql.execution.QueryExecution.simpleString(QueryExecution.scala:204)
	  at org.apache.spark.sql.execution.QueryExecution.org$apache$spark$sql$execution$QueryExecution$$explainString(QueryExecution.scala:251)
	  at org.apache.spark.sql.execution.QueryExecution.explainString(QueryExecution.scala:220)
	  at org.apache.spark.sql.execution.SQLExecution$.$anonfun$withNewExecutionId$6(SQLExecution.scala:103)
	  at org.apache.spark.sql.execution.SQLExecution$$$Lambda$1698.2050525584.apply(Unknown Source:-1)
	  at org.apache.spark.sql.execution.SQLExecution$.withSQLConfPropagated(SQLExecution.scala:171)
	  at org.apache.spark.sql.execution.SQLExecution$.$anonfun$withNewExecutionId$1(SQLExecution.scala:95)
	  at org.apache.spark.sql.execution.SQLExecution$$$Lambda$1682.2000856156.apply(Unknown Source:-1)
	  at org.apache.spark.sql.SparkSession.withActive(SparkSession.scala:779)
	  at org.apache.spark.sql.execution.SQLExecution$.withNewExecutionId(SQLExecution.scala:64)
	  at org.apache.spark.sql.Dataset.withAction(Dataset.scala:3924)
	  at org.apache.spark.sql.Dataset.collect(Dataset.scala:3188)
```

**org.apache.spark.sql.Dataset**
```
class Dataset[T] private[sql](
    @DeveloperApi @Unstable @transient val queryExecution: QueryExecution,
    @DeveloperApi @Unstable @transient val encoder: Encoder[T])
  extends Serializable {
  
  // Note for Spark contributors: if adding or updating any action in `Dataset`, please make sure
  // you wrap it with `withNewExecutionId` if this actions doesn't call other action.

  def this(sparkSession: SparkSession, logicalPlan: LogicalPlan, encoder: Encoder[T]) = {
    this(sparkSession.sessionState.executePlan(logicalPlan), encoder)  ====> executePlan() is to create the QueryExecution
  }

  def collect(): Array[T] = withAction("collect", queryExecution)(collectFromPlan)
 
  private def withAction[U](name: String, qe: QueryExecution)(action: SparkPlan => U) = {
    SQLExecution.withNewExecutionId(qe, Some(name)) {
      QueryExecution.withInternalError(s"""The "$name" action failed.""") {
        qe.executedPlan.resetMetrics()
        action(qe.executedPlan)
      }
    }
  }
```
**org.apache.spark.sql.execution.SQLExecution#withNewExecutionId**
```
  /**
   * Wrap an action that will execute "queryExecution" to track all Spark jobs in the body so that
   * we can connect them with an execution.
   */
  def withNewExecutionId[T](
      queryExecution: QueryExecution,
      name: Option[String] = None)(body: => T): T = queryExecution.sparkSession.withActive {
```

**org.apache.spark.sql.execution.QueryExecution#explainString**
```
  def explainString(
      mode: ExplainMode,
      maxFields: Int = SQLConf.get.maxToStringFields): String = {
    val concat = new PlanStringConcat()
    explainString(mode, maxFields, concat.append)
    withRedaction {
      concat.toString
    }
  }
```

**org.apache.spark.sql.execution.QueryExecution#executedPlan**
```
  // executedPlan should not be used to initialize any SparkPlan. It should be
  // only used for execution.  =====> ???
  lazy val executedPlan: SparkPlan = {
    // We need to materialize the optimizedPlan here, before tracking the planning phase, to ensure
    // that the optimization time is not counted as part of the planning phase.
    assertOptimized()
    executePhase(QueryPlanningTracker.PLANNING) {
      // clone the plan to avoid sharing the plan instance between different stages like analyzing,
      // optimizing and planning.
      QueryExecution.prepareForExecution(preparations, sparkPlan.clone())   =====> sparkPlan is initialized here and then call QueryExecution.prepareForExecution
    }
  }
```

## Code of SessionState to create QueryExecution
**org.apache.spark.sql.SparkSession#sessionState**
```
  /**
   * State isolated across sessions, including SQL configurations, temporary tables, registered
   * functions, and everything else that accepts a [[org.apache.spark.sql.internal.SQLConf]].
   * If `parentSessionState` is not null, the `SessionState` will be a copy of the parent.
   *
   * This is internal to Spark and there is no guarantee on interface stability.
   *
   * @since 2.2.0
   */
  @Unstable
  @transient
  lazy val sessionState: SessionState = {
    parentSessionState
      .map(_.clone(this))
      .getOrElse {
        val state = SparkSession.instantiateSessionState(
          SparkSession.sessionStateClassName(sharedState.conf),
          self)
        state
      }
  }
```

**org.apache.spark.sql.SparkSession#instantiateSessionState**
```
  /**
   * Helper method to create an instance of `SessionState` based on `className` from conf.
   * The result is either `SessionState` or a Hive based `SessionState`.
   */
  private def instantiateSessionState(
      className: String,
      sparkSession: SparkSession): SessionState = {
    try {
      // invoke new [Hive]SessionStateBuilder(
      //   SparkSession,
      //   Option[SessionState])
      val clazz = Utils.classForName(className)
      val ctor = clazz.getConstructors.head
      ctor.newInstance(sparkSession, None).asInstanceOf[BaseSessionStateBuilder].build()
    } catch {
      case NonFatal(e) =>
        throw new IllegalArgumentException(s"Error while instantiating '$className':", e)
    }
  }
```

**org.apache.spark.sql.SparkSession#sessionStateClassName**
```
  private val HIVE_SESSION_STATE_BUILDER_CLASS_NAME =
    "org.apache.spark.sql.hive.HiveSessionStateBuilder"
    
  private def sessionStateClassName(conf: SparkConf): String = {
    conf.get(CATALOG_IMPLEMENTATION) match {
      case "hive" => HIVE_SESSION_STATE_BUILDER_CLASS_NAME
      case "in-memory" => classOf[SessionStateBuilder].getCanonicalName
    }
  }
  
```


```
class SessionStateBuilder(
    session: SparkSession,
    parentState: Option[SessionState])
  extends BaseSessionStateBuilder(session, parentState) {
  override protected def newBuilder: NewBuilder = new SessionStateBuilder(_, _)
}
```

or

Hive
```
/**
 * Builder that produces a Hive-aware `SessionState`.
 */
class HiveSessionStateBuilder(
    session: SparkSession,
    parentState: Option[SessionState])
  extends BaseSessionStateBuilder(session, parentState) {
```

**org.apache.spark.sql.internal.BaseSessionStateBuilder#build**
```
  def build(): SessionState = {
    new SessionState(
      session.sharedState,
      conf,
      experimentalMethods,
      functionRegistry,
      tableFunctionRegistry,
      udfRegistration,
      () => catalog,
      sqlParser,
      () => analyzer,
      () => optimizer,
      planner,
      () => streamingQueryManager,
      listenerManager,
      () => resourceLoader,
      createQueryExecution,
      createClone,
      columnarRules,
      adaptiveRulesHolder)
  }
}
```

**org.apache.spark.sql.internal.BaseSessionStateBuilder#createQueryExecution**
```
  protected def createQueryExecution:
    (LogicalPlan, CommandExecutionMode.Value) => QueryExecution =
      (plan, mode) => new QueryExecution(session, plan, mode = mode)
```

**org.apache.spark.sql.execution.QueryExecution**
```

/**
 * The primary workflow for executing relational queries using Spark.  Designed to allow easy
 * access to the intermediate phases of query execution for developers.
 *
 * While this is not a public class, we should avoid changing the function names for the sake of
 * changing them, because a lot of developers use the feature for debugging.
 */
class QueryExecution(
    val sparkSession: SparkSession,
    val logical: LogicalPlan,
    val tracker: QueryPlanningTracker = new QueryPlanningTracker,
    val mode: CommandExecutionMode.Value = CommandExecutionMode.ALL) extends Logging {
```

## Code of SparkPlan generation

QueryExecution.sparkPlan  
    => QueryExecution.createSparkPlan

SparkPlanner.plan (strategies defined in SparkPlanner)  
    => SparkStrategies.plan  
    => QueryPlanner.plan

**org.apache.spark.sql.execution.QueryExecution**
```
/**
 * The primary workflow for executing relational queries using Spark.  Designed to allow easy
 * access to the intermediate phases of query execution for developers.
 *
 * While this is not a public class, we should avoid changing the function names for the sake of
 * changing them, because a lot of developers use the feature for debugging.
 */
class QueryExecution(
    val sparkSession: SparkSession,
    val logical: LogicalPlan,
    val tracker: QueryPlanningTracker = new QueryPlanningTracker,
    val mode: CommandExecutionMode.Value = CommandExecutionMode.ALL) extends Logging {
    
    
  lazy val sparkPlan: SparkPlan = {
    // We need to materialize the optimizedPlan here because sparkPlan is also tracked under
    // the planning phase
    assertOptimized()
    executePhase(QueryPlanningTracker.PLANNING) {
      // Clone the logical plan here, in case the planner rules change the states of the logical
      // plan.
      QueryExecution.createSparkPlan(sparkSession, planner, optimizedPlan.clone())
    }
  }
          
          
  /**
   * Transform a [[LogicalPlan]] into a [[SparkPlan]].
   *
   * Note that the returned physical plan still needs to be prepared for execution.
   */
  def createSparkPlan(
      sparkSession: SparkSession,
      planner: SparkPlanner,
      plan: LogicalPlan): SparkPlan = {
    // TODO: We use next(), i.e. take the first plan returned by the planner, here for now,
    //       but we will implement to choose the best plan.
    planner.plan(ReturnAnswer(plan)).next()
  }
  
```

**org.apache.spark.sql.execution.SparkPlanner**
```
class SparkPlanner(val session: SparkSession, val experimentalMethods: ExperimentalMethods)
  extends SparkStrategies with SQLConfHelper {
  
    override def strategies: Seq[Strategy] =
    experimentalMethods.extraStrategies ++
      extraPlanningStrategies ++ (
      LogicalQueryStageStrategy ::
      PythonEvals ::
      new DataSourceV2Strategy(session) ::
      FileSourceStrategy ::
      DataSourceStrategy ::
      SpecialLimits ::
      Aggregation ::
      Window ::
      JoinSelection ::
      InMemoryScans ::
      SparkScripts ::
      BasicOperators :: Nil)
```


```
abstract class SparkStrategies extends QueryPlanner[SparkPlan] {
  self: SparkPlanner =>

  override def plan(plan: LogicalPlan): Iterator[SparkPlan] = {
    super.plan(plan).map { p =>
      val logicalPlan = plan match {
        case ReturnAnswer(rootPlan) => rootPlan
        case _ => plan
      }
      p.setLogicalLink(logicalPlan)
      p
    }
  }
```

**org.apache.spark.sql.catalyst.planning.QueryPlanner**
```

/**
 * Abstract class for transforming [[LogicalPlan]]s into physical plans.
 * Child classes are responsible for specifying a list of [[GenericStrategy]] objects that
 * each of which can return a list of possible physical plan options.
 * If a given strategy is unable to plan all of the remaining operators in the tree,
 * it can call [[GenericStrategy#planLater planLater]], which returns a placeholder
 * object that will be [[collectPlaceholders collected]] and filled in
 * using other available strategies.
 *
 * TODO: RIGHT NOW ONLY ONE PLAN IS RETURNED EVER...
 *       PLAN SPACE EXPLORATION WILL BE IMPLEMENTED LATER.
 *
 * @tparam PhysicalPlan The type of physical plan produced by this [[QueryPlanner]]
 */
abstract class QueryPlanner[PhysicalPlan <: TreeNode[PhysicalPlan]] {
  /** A list of execution strategies that can be used by the planner */
  def strategies: Seq[GenericStrategy[PhysicalPlan]]

  def plan(plan: LogicalPlan): Iterator[PhysicalPlan] = {
    // Obviously a lot to do here still...

    // Collect physical plan candidates.
    val candidates = strategies.iterator.flatMap(_(plan))

    // The candidates may contain placeholders marked as [[planLater]],
    // so try to replace them by their child plans.
    val plans = candidates.flatMap { candidate =>
      val placeholders = collectPlaceholders(candidate)

      if (placeholders.isEmpty) {
        // Take the candidate as is because it does not contain placeholders.
        Iterator(candidate)
      } else {
        // Plan the logical plan marked as [[planLater]] and replace the placeholders.
        placeholders.iterator.foldLeft(Iterator(candidate)) {
          case (candidatesWithPlaceholders, (placeholder, logicalPlan)) =>
            // Plan the logical plan for the placeholder.
            val childPlans = this.plan(logicalPlan)  ====> if there is planLater, recursively apply all the strategies again

            candidatesWithPlaceholders.flatMap { candidateWithPlaceholders =>
              childPlans.map { childPlan =>
                // Replace the placeholder by the child plan
                candidateWithPlaceholders.transformUp {
                  case p if p.eq(placeholder) => childPlan
                }
              }
            }
        }
      }
    }

    val pruned = prunePlans(plans)
    assert(pruned.hasNext, s"No plan for $plan")
    pruned
```

## Code of prepared SparkPlan generation
In QueryExecution.prepareForExecution(), rules (Rule[SparkPlan]) are applied  
  => rules are defined in QueryExecution.preparations()

**org.apache.spark.sql.execution.QueryExecution#prepareForExecution**
```

  protected def preparations: Seq[Rule[SparkPlan]] = {
    QueryExecution.preparations(sparkSession,
      Option(InsertAdaptiveSparkPlan(AdaptiveExecutionContext(sparkSession, this))), false)
  }
  
  /**
   * Construct a sequence of rules that are used to prepare a planned [[SparkPlan]] for execution.
   * These rules will make sure subqueries are planned, make use the data partitioning and ordering
   * are correct, insert whole stage code gen, and try to reduce the work done by reusing exchanges
   * and subqueries.
   */
  private[execution] def preparations(
      sparkSession: SparkSession,
      adaptiveExecutionRule: Option[InsertAdaptiveSparkPlan] = None,
      subquery: Boolean): Seq[Rule[SparkPlan]] = {
    // `AdaptiveSparkPlanExec` is a leaf node. If inserted, all the following rules will be no-op
    // as the original plan is hidden behind `AdaptiveSparkPlanExec`.
    adaptiveExecutionRule.toSeq ++
    Seq(
      CoalesceBucketsInJoin,
      PlanDynamicPruningFilters(sparkSession),
      PlanSubqueries(sparkSession),
      RemoveRedundantProjects,
      EnsureRequirements(),
      // `ReplaceHashWithSortAgg` needs to be added after `EnsureRequirements` to guarantee the
      // sort order of each node is checked to be valid.
      ReplaceHashWithSortAgg,
      // `RemoveRedundantSorts` needs to be added after `EnsureRequirements` to guarantee the same
      // number of partitions when instantiating PartitioningCollection.
      RemoveRedundantSorts,
      DisableUnnecessaryBucketedScan,
      ApplyColumnarRulesAndInsertTransitions(
        sparkSession.sessionState.columnarRules, outputsColumnar = false),
      CollapseCodegenStages()) ++
      (if (subquery) {
        Nil
      } else {
        Seq(ReuseExchangeAndSubquery)
      })
  }
  
  /**
   * Prepares a planned [[SparkPlan]] for execution by inserting shuffle operations and internal
   * row format conversions as needed.
   */
  private[execution] def prepareForExecution(
      preparations: Seq[Rule[SparkPlan]],
      plan: SparkPlan): SparkPlan = {
    val planChangeLogger = new PlanChangeLogger[SparkPlan]()
    val preparedPlan = preparations.foldLeft(plan) { case (sp, rule) =>
      val result = rule.apply(sp)
      planChangeLogger.logRule(rule.ruleName, sp, result)
      result
    }
    planChangeLogger.logBatch("Preparations", plan, preparedPlan)
    preparedPlan
  }
```