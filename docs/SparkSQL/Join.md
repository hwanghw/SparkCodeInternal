[TOC]

# Join
##  Optimizer: re-order Join
### JIRA
[SPARK-12032 Filter can't be pushed down to correct Join because of bad order of Join](https://issues.apache.org/jira/browse/SPARK-12032)  
code: [PR-10073](https://github.com/apache/spark/pull/10073/files)  
ref code: [PR-10258](https://github.com/apache/spark/pull/10258/files)

For this query:
```
  select d.d_year, count(*) cnt
   FROM store_sales, date_dim d, customer c
   WHERE ss_customer_sk = c.c_customer_sk AND c.c_first_shipto_date_sk = d.d_date_sk
   group by d.d_year
```

Current optimized plan is
```
== Optimized Logical Plan ==
Aggregate [d_year#147], [d_year#147,(count(1),mode=Complete,isDistinct=false) AS cnt#425L]
 Project [d_year#147]
  Join Inner, Some(((ss_customer_sk#283 = c_customer_sk#101) && (c_first_shipto_date_sk#106 = d_date_sk#141)))
   Project [d_date_sk#141,d_year#147,ss_customer_sk#283]
    Join Inner, None
     Project [ss_customer_sk#283]
      Relation[] ParquetRelation[store_sales]
     Project [d_date_sk#141,d_year#147]
      Relation[] ParquetRelation[date_dim]
   Project [c_customer_sk#101,c_first_shipto_date_sk#106]
    Relation[] ParquetRelation[customer]
```
It will **join store_sales and date_dim together without any condition**, the condition c.c_first_shipto_date_sk = d.d_date_sk is not pushed to it because the bad order of joins.

The optimizer should re-order the joins, join date_dim after customer, then it can pushed down the condition correctly.

The plan should be
```
Aggregate [d_year#147], [d_year#147,(count(1),mode=Complete,isDistinct=false) AS cnt#425L]
 Project [d_year#147]
  Join Inner, Some((c_first_shipto_date_sk#106 = d_date_sk#141))
   Project [c_first_shipto_date_sk#106]
    Join Inner, Some((ss_customer_sk#283 = c_customer_sk#101))
     Project [ss_customer_sk#283]
      Relation[store_sales]
     Project [c_first_shipto_date_sk#106,c_customer_sk#101]
      Relation[customer]
   Project [d_year#147,d_date_sk#141]
    Relation[date_dim]
```

### Code
**org.apache.spark.sql.catalyst.optimizer.Optimizer#defaultBatches**
```
  /**
   * Defines the default rule batches in the Optimizer.
   *
   * Implementations of this class should override this method, and [[nonExcludableRules]] if
   * necessary, instead of [[batches]]. The rule batches that eventually run in the Optimizer,
   * i.e., returned by [[batches]], will be (defaultBatches - (excludedRules - nonExcludableRules)).
   */
  def defaultBatches: Seq[Batch] = {
    val operatorOptimizationRuleSet =
      Seq(
        // Operator push down
        PushProjectionThroughUnion,
        ReorderJoin,   ==> ReorderJoin rules
```
**org.apache.spark.sql.catalyst.optimizer.ReorderJoin**
```
/**
 * Reorder the joins and push all the conditions into join, so that the bottom ones have at least
 * one condition.
 *
 * The order of joins will not be changed if all of them already have at least one condition.
 *
 * If star schema detection is enabled, reorder the star join plans based on heuristics.
 */
object ReorderJoin extends Rule[LogicalPlan] with PredicateHelper {
  /**
   * Join a list of plans together and push down the conditions into them.
   *
   * The joined plan are picked from left to right, prefer those has at least one join condition.
   *
   * @param input a list of LogicalPlans to inner join and the type of inner join.
   * @param conditions a list of condition for join.
   */
  @tailrec
  final def createOrderedJoin(
      input: Seq[(LogicalPlan, InnerLike)],
      conditions: Seq[Expression]): LogicalPlan = {
      
      
  def apply(plan: LogicalPlan): LogicalPlan = plan.transformWithPruning(
    _.containsPattern(INNER_LIKE_JOIN), ruleId) {
    case p @ ExtractFiltersAndInnerJoins(input, conditions)
        if input.size > 2 && conditions.nonEmpty =>
      val reordered = if (conf.starSchemaDetection && !conf.cboEnabled) {
        val starJoinPlan = StarSchemaDetection.reorderStarJoins(input, conditions)
        if (starJoinPlan.nonEmpty) {
          val rest = input.filterNot(starJoinPlan.contains(_))
          createOrderedJoin(starJoinPlan ++ rest, conditions)
        } else {
          createOrderedJoin(input, conditions)
        }
      } else {
        createOrderedJoin(input, conditions)
      }

      if (p.sameOutput(reordered)) {
        reordered
      } else {
        // Reordering the joins have changed the order of the columns.
        // Inject a projection to make sure we restore to the expected ordering.
        Project(p.output, reordered)
      }
  }
```

**org.apache.spark.sql.catalyst.planning.ExtractFiltersAndInnerJoins**
```
/**
 * A pattern that collects the filter and inner joins.
 *
 *          Filter
 *            |
 *        inner Join
 *          /    \            ---->      (Seq(plan0, plan1, plan2), conditions)
 *      Filter   plan2
 *        |
 *  inner join
 *      /    \
 *   plan0    plan1
 *
 * Note: This pattern currently only works for left-deep trees.
 */
object ExtractFiltersAndInnerJoins extends PredicateHelper {

  /**
   * Flatten all inner joins, which are next to each other.
   * Return a list of logical plans to be joined with a boolean for each plan indicating if it
   * was involved in an explicit cross join. Also returns the entire list of join conditions for
   * the left-deep tree.
   */
  def flattenJoin(plan: LogicalPlan, parentJoinType: InnerLike = Inner)
      : (Seq[(LogicalPlan, InnerLike)], Seq[Expression]) = plan match {
    case Join(left, right, joinType: InnerLike, cond, hint) if hint == JoinHint.NONE =>
      val (plans, conditions) = flattenJoin(left, joinType)
      (plans ++ Seq((right, joinType)), conditions ++
        cond.toSeq.flatMap(splitConjunctivePredicates))
    case Filter(filterCondition, j @ Join(_, _, _: InnerLike, _, hint)) if hint == JoinHint.NONE =>
      val (plans, conditions) = flattenJoin(j)
      (plans, conditions ++ splitConjunctivePredicates(filterCondition))

    case _ => (Seq((plan, parentJoinType)), Seq.empty)
  }

  def unapply(plan: LogicalPlan)
      : Option[(Seq[(LogicalPlan, InnerLike)], Seq[Expression])]
      = plan match {
    case f @ Filter(filterCondition, j @ Join(_, _, joinType: InnerLike, _, hint))
        if hint == JoinHint.NONE =>
      Some(flattenJoin(f))
    case j @ Join(_, _, joinType, _, hint) if hint == JoinHint.NONE =>
      Some(flattenJoin(j))
    case _ => None
  }
}
```