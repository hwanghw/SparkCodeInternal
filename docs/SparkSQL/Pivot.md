[TOC]

# Pivot
## JIRA

[SPARK-8992 Add Pivot functionality to Spark SQL](https://issues.apache.org/jira/browse/SPARK-8992)

[SPARK-13749 Faster pivot implementation for many distinct values with two phase aggregation](https://issues.apache.org/jira/browse/SPARK-13749)

## code
**org.apache.spark.sql.catalyst.analysis.Analyzer#batches**
```
  override def batches: Seq[Batch] = Seq(
        ...
         Batch("Resolution", fixedPoint,
         ResolveTableValuedFunctions(v1SessionCatalog) ::
        ...
         ResolvePivot ::
         ResolveOrdinalInOrderByAndGroupBy ::
         ResolveAggAliasInGroupBy ::
         ResolveMissingReferences ::
         ExtractGenerator ::
         ResolveGenerate ::
         ResolveFunctions ::
         ResolveAliases ::
         ResolveSubquery ::
        ...
```

**org.apache.spark.sql.catalyst.analysis.Analyzer.ResolvePivot**
```
  object ResolvePivot extends Rule[LogicalPlan] with AliasHelper {
    def apply(plan: LogicalPlan): LogicalPlan = plan.resolveOperatorsWithPruning(
    
```

**org.apache.spark.sql.catalyst.analysis.Analyzer.ResolveAliases**
```
  object ResolveAliases extends Rule[LogicalPlan] {
    def apply(plan: LogicalPlan): LogicalPlan = plan.resolveOperatorsUpWithPruning(
      _.containsPattern(UNRESOLVED_ALIAS), ruleId) {
      case Aggregate(groups, aggs, child) if child.resolved && hasUnresolvedAlias(aggs) =>
        Aggregate(groups, assignAliases(aggs), child)

      case Pivot(groupByOpt, pivotColumn, pivotValues, aggregates, child)
        if child.resolved && groupByOpt.isDefined && hasUnresolvedAlias(groupByOpt.get) =>
        Pivot(Some(assignAliases(groupByOpt.get)), pivotColumn, pivotValues, aggregates, child)
```

**org.apache.spark.sql.catalyst.plans.logical.Pivot**
```
/**
 * A constructor for creating a pivot, which will later be converted to a [[Project]]
 * or an [[Aggregate]] during the query analysis.
 *
 * @param groupByExprsOpt A sequence of group by expressions. This field should be None if coming
 *                        from SQL, in which group by expressions are not explicitly specified.
 * @param pivotColumn     The pivot column.
 * @param pivotValues     A sequence of values for the pivot column.
 * @param aggregates      The aggregation expressions, each with or without an alias.
 * @param child           Child operator
 */
case class Pivot(
    groupByExprsOpt: Option[Seq[NamedExpression]],
    pivotColumn: Expression,
    pivotValues: Seq[Expression],
    aggregates: Seq[Expression],
    child: LogicalPlan) extends UnaryNode {
```

**org.apache.spark.sql.RelationalGroupedDataset**
```
/**
 * A set of methods for aggregations on a `DataFrame`, created by [[Dataset#groupBy groupBy]],
 * [[Dataset#cube cube]] or [[Dataset#rollup rollup]] (and also `pivot`).
 *
 * The main method is the `agg` function, which has multiple variants. This class also contains
 * some first-order statistics such as `mean`, `sum` for convenience.
 *
 * @note This class was named `GroupedData` in Spark 1.x.
 *
 * @since 2.0.0
 */
@Stable
class RelationalGroupedDataset protected[sql](
    private[sql] val df: DataFrame,
    private[sql] val groupingExprs: Seq[Expression],
    groupType: RelationalGroupedDataset.GroupType) {


  /**
   * Pivots a column of the current `DataFrame` and performs the specified aggregation.
   *
   * There are two versions of `pivot` function: one that requires the caller to specify the list
   * of distinct values to pivot on, and one that does not. The latter is more concise but less
   * efficient, because Spark needs to first compute the list of distinct values internally.
   *
   * {{{
   *   // Compute the sum of earnings for each year by course with each course as a separate column
   *   df.groupBy("year").pivot("course", Seq("dotNET", "Java")).sum("earnings")
   *
   *   // Or without specifying column values (less efficient)
   *   df.groupBy("year").pivot("course").sum("earnings")
   * }}}
   *
   * @param pivotColumn Name of the column to pivot.
   * @since 1.6.0
   */
  def pivot(pivotColumn: String): RelationalGroupedDataset = pivot(Column(pivotColumn))

  /**
   * Pivots a column of the current `DataFrame` and performs the specified aggregation.
   * There are two versions of pivot function: one that requires the caller to specify the list
   * of distinct values to pivot on, and one that does not. The latter is more concise but less
   * efficient, because Spark needs to first compute the list of distinct values internally.
   *
   * {{{
   *   // Compute the sum of earnings for each year by course with each course as a separate column
   *   df.groupBy("year").pivot("course", Seq("dotNET", "Java")).sum("earnings")
   *
   *   // Or without specifying column values (less efficient)
   *   df.groupBy("year").pivot("course").sum("earnings")
   * }}}
   *
   * From Spark 3.0.0, values can be literal columns, for instance, struct. For pivoting by
   * multiple columns, use the `struct` function to combine the columns and values:
   *
   * {{{
   *   df.groupBy("year")
   *     .pivot("trainingCourse", Seq(struct(lit("java"), lit("Experts"))))
   *     .agg(sum($"earnings"))
   * }}}
   *
   * @param pivotColumn Name of the column to pivot.
   * @param values List of values that will be translated to columns in the output DataFrame.
   * @since 1.6.0
   */
  def pivot(pivotColumn: String, values: Seq[Any]): RelationalGroupedDataset = {
    pivot(Column(pivotColumn), values)
  }

  /**
   * (Java-specific) Pivots a column of the current `DataFrame` and performs the specified
   * aggregation.
   *
   * There are two versions of pivot function: one that requires the caller to specify the list
   * of distinct values to pivot on, and one that does not. The latter is more concise but less
   * efficient, because Spark needs to first compute the list of distinct values internally.
   *
   * {{{
   *   // Compute the sum of earnings for each year by course with each course as a separate column
   *   df.groupBy("year").pivot("course", Arrays.<Object>asList("dotNET", "Java")).sum("earnings");
   *
   *   // Or without specifying column values (less efficient)
   *   df.groupBy("year").pivot("course").sum("earnings");
   * }}}
   *
   * @param pivotColumn Name of the column to pivot.
   * @param values List of values that will be translated to columns in the output DataFrame.
   * @since 1.6.0
   */
  def pivot(pivotColumn: String, values: java.util.List[Any]): RelationalGroupedDataset = {
    pivot(Column(pivotColumn), values)
  }

  /**
   * Pivots a column of the current `DataFrame` and performs the specified aggregation.
   * This is an overloaded version of the `pivot` method with `pivotColumn` of the `String` type.
   *
   * {{{
   *   // Or without specifying column values (less efficient)
   *   df.groupBy($"year").pivot($"course").sum($"earnings");
   * }}}
   *
   * @param pivotColumn he column to pivot.
   * @since 2.4.0
   */
  def pivot(pivotColumn: Column): RelationalGroupedDataset = {
    // This is to prevent unintended OOM errors when the number of distinct values is large
    val maxValues = df.sparkSession.sessionState.conf.dataFramePivotMaxValues
    // Get the distinct values of the column and sort them so its consistent
    val values = df.select(pivotColumn)
      .distinct()
      .limit(maxValues + 1)
      .sort(pivotColumn)  // ensure that the output columns are in a consistent logical order
      .collect()
      .map(_.get(0))
      .toSeq

    if (values.length > maxValues) {
      throw QueryCompilationErrors.aggregationFunctionAppliedOnNonNumericColumnError(
        pivotColumn.toString, maxValues)
    }

    pivot(pivotColumn, values)
  }

  /**
   * Pivots a column of the current `DataFrame` and performs the specified aggregation.
   * This is an overloaded version of the `pivot` method with `pivotColumn` of the `String` type.
   *
   * {{{
   *   // Compute the sum of earnings for each year by course with each course as a separate column
   *   df.groupBy($"year").pivot($"course", Seq("dotNET", "Java")).sum($"earnings")
   * }}}
   *
   * @param pivotColumn the column to pivot.
   * @param values List of values that will be translated to columns in the output DataFrame.
   * @since 2.4.0
   */
  def pivot(pivotColumn: Column, values: Seq[Any]): RelationalGroupedDataset = {
    groupType match {
      case RelationalGroupedDataset.GroupByType =>
        val valueExprs = values.map(_ match {
          case c: Column => c.expr
          case v =>
            try {
              Literal.apply(v)
            } catch {
              case _: SparkRuntimeException =>
                throw QueryExecutionErrors.pivotColumnUnsupportedError(v, pivotColumn.expr.dataType)
            }
        })
        new RelationalGroupedDataset(
          df,
          groupingExprs,
          RelationalGroupedDataset.PivotType(pivotColumn.expr, valueExprs))
      case _: RelationalGroupedDataset.PivotType =>
        throw QueryExecutionErrors.repeatedPivotsUnsupportedError()
      case _ =>
        throw QueryExecutionErrors.pivotNotAfterGroupByUnsupportedError()
    }
  }

  /**
   * (Java-specific) Pivots a column of the current `DataFrame` and performs the specified
   * aggregation. This is an overloaded version of the `pivot` method with `pivotColumn` of
   * the `String` type.
   *
   * @param pivotColumn the column to pivot.
   * @param values List of values that will be translated to columns in the output DataFrame.
   * @since 2.4.0
   */
  def pivot(pivotColumn: Column, values: java.util.List[Any]): RelationalGroupedDataset = {
    pivot(pivotColumn, values.asScala.toSeq)
  }
```

**org.apache.spark.sql.RelationalGroupedDataset.PivotType**
```
  private[sql] case class PivotType(pivotCol: Expression, values: Seq[Expression]) extends GroupType
```