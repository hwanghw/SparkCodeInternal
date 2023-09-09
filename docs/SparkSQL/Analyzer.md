[TOC]

# Analyzer

## Logical query plan analyzer
```scala
abstract class BaseSessionStateBuilder(
    val session: SparkSession,
    val parentState: Option[SessionState]) {
  /**
   * Logical query plan analyzer for resolving unresolved attributes and relations.
   *
   * Note: this depends on the `conf` and `catalog` fields.
   */
  protected def analyzer: Analyzer = new Analyzer(catalogManager) {
    override val extendedResolutionRules: Seq[Rule[LogicalPlan]] =
      new FindDataSourceTable(session) +:
        new ResolveSQLOnFile(session) +:
        new FallBackFileSourceV2(session) +:
        ResolveEncodersInScalaAgg +:
        new ResolveSessionCatalog(catalogManager) +:      =====> ResolveSessionCatalog Rule
        ResolveWriteToStream +:
        new EvalSubqueriesForTimeTravel +:
        customResolutionRules
        
```

## ResolveSessionCatalog

```scala

/**
 * Resolves catalogs from the multi-part identifiers in SQL statements, and convert the statements
 * to the corresponding v1 or v2 commands if the resolved catalog is the session catalog.
 *
 * We can remove this rule once we implement all the catalog functionality in `V2SessionCatalog`.
 */
class ResolveSessionCatalog(val catalogManager: CatalogManager)
  extends Rule[LogicalPlan] with LookupCatalog {
  
    override def apply(plan: LogicalPlan): LogicalPlan = plan.resolveOperatorsUp {
  
      case AnalyzeTables(DatabaseInSessionCatalog(db), noScan) =>
      AnalyzeTablesCommand(Some(db), noScan)
```

```

/**
 * Analyzes all tables in the given database to generate statistics.
 */
case class AnalyzeTablesCommand(
    databaseName: Option[String],
    noScan: Boolean) extends LeafRunnableCommand {

  override def run(sparkSession: SparkSession): Seq[Row] = {
    val catalog = sparkSession.sessionState.catalog
    val db = databaseName.getOrElse(catalog.getCurrentDatabase)
    catalog.listTables(db).foreach { tbl =>
      try {
        CommandUtils.analyzeTable(sparkSession, tbl, noScan)
      } catch {
        case NonFatal(e) =>
          logWarning(s"Failed to analyze table ${tbl.table} in the " +
            s"database $db because of ${e.toString}", e)
      }
    }
    Seq.empty[Row]
  }
}
```

## analyzeTable
org.apache.spark.sql.execution.command.CommandUtils#analyzeTable

```
  def analyzeTable(
      sparkSession: SparkSession,
      tableIdent: TableIdentifier,
      noScan: Boolean): Unit = {
    val sessionState = sparkSession.sessionState
    val db = tableIdent.database.getOrElse(sessionState.catalog.getCurrentDatabase)
    val tableIdentWithDB = TableIdentifier(tableIdent.table, Some(db))
    val tableMeta = sessionState.catalog.getTableMetadata(tableIdentWithDB)
    if (tableMeta.tableType == CatalogTableType.VIEW) {
      // Analyzes a catalog view if the view is cached
      val table = sparkSession.table(tableIdent.quotedString)
      val cacheManager = sparkSession.sharedState.cacheManager
      if (cacheManager.lookupCachedData(table.logicalPlan).isDefined) {
        if (!noScan) {
          // To collect table stats, materializes an underlying columnar RDD
          table.count()
        }
      } else {
        throw QueryCompilationErrors.analyzeTableNotSupportedOnViewsError()
      }
    } else {
      // Compute stats for the whole table
      val (newTotalSize, _) = CommandUtils.calculateTotalSize(sparkSession, tableMeta)  =====> calculateTotalSize of the table
      val newRowCount =
        if (noScan) None else Some(BigInt(sparkSession.table(tableIdentWithDB).count()))

      // Update the metastore if the above statistics of the table are different from those
      // recorded in the metastore.
      val newStats = CommandUtils.compareAndGetNewStats(tableMeta.stats, newTotalSize, newRowCount)
      if (newStats.isDefined) {
        sessionState.catalog.alterTableStats(tableIdentWithDB, newStats)
      }
    }
  }
  
  def calculateTotalSize(
      spark: SparkSession,
      catalogTable: CatalogTable): (BigInt, Seq[CatalogTablePartition]) = {
    val sessionState = spark.sessionState
    val startTime = System.nanoTime()
    val (totalSize, newPartitions) = if (catalogTable.partitionColumnNames.isEmpty) {
      (calculateSingleLocationSize(sessionState, catalogTable.identifier,                 =====> calculateSingleLocationSize
        catalogTable.storage.locationUri), Seq())
    } else {
      // Calculate table size as a sum of the visible partitions. See SPARK-21079
      val partitions = sessionState.catalog.listPartitions(catalogTable.identifier)       =====> listPartitions
      logInfo(s"Starting to calculate sizes for ${partitions.length} partitions.")
      val paths = partitions.map(_.storage.locationUri)
      val sizes = calculateMultipleLocationSizes(spark, catalogTable.identifier, paths)   =====> calculateMultipleLocationSizes
      val newPartitions = partitions.zipWithIndex.flatMap { case (p, idx) =>
        val newStats = CommandUtils.compareAndGetNewStats(p.stats, sizes(idx), None)
        newStats.map(_ => p.copy(stats = newStats))
      }
      (sizes.sum, newPartitions)
    }
    logInfo(s"It took ${(System.nanoTime() - startTime) / (1000 * 1000)} ms to calculate" +
      s" the total size for table ${catalogTable.identifier}.")
    (totalSize, newPartitions)
  }


  def calculateMultipleLocationSizes(
      sparkSession: SparkSession,
      tid: TableIdentifier,
      paths: Seq[Option[URI]]): Seq[Long] = {
    if (sparkSession.sessionState.conf.parallelFileListingInStatsComputation) {
      calculateMultipleLocationSizesInParallel(sparkSession, paths.map(_.map(new Path(_))))
    } else {
      paths.map(p => calculateSingleLocationSize(sparkSession.sessionState, tid, p))
    }
  }
  

  def calculateSingleLocationSize(
      sessionState: SessionState,
      identifier: TableIdentifier,
      locationUri: Option[URI]): Long = {
    // This method is mainly based on
    // org.apache.hadoop.hive.ql.stats.StatsUtils.getFileSizeForTable(HiveConf, Table)
    // in Hive 0.13 (except that we do not use fs.getContentSummary).
    // TODO: Generalize statistics collection.
    // TODO: Why fs.getContentSummary returns wrong size on Jenkins?
    // Can we use fs.getContentSummary in future?
    // Seems fs.getContentSummary returns wrong table size on Jenkins. So we use
    // countFileSize to count the table size.
    val stagingDir = sessionState.conf.getConfString("hive.exec.stagingdir", ".hive-staging")

    def getPathSize(fs: FileSystem, path: Path): Long = {
      val fileStatus = fs.getFileStatus(path)
      val size = if (fileStatus.isDirectory) {
        fs.listStatus(path)
          .map { status =>
            if (isDataPath(status.getPath, stagingDir)) {
              getPathSize(fs, status.getPath)
            } else {
              0L
            }
          }.sum
      } else {
        fileStatus.getLen
      }

      size
    }

    val startTime = System.nanoTime()
    val size = locationUri.map { p =>
      val path = new Path(p)
      try {
        val fs = path.getFileSystem(sessionState.newHadoopConf())
        getPathSize(fs, path)
      } catch {
        case NonFatal(e) =>
          logWarning(
            s"Failed to get the size of table ${identifier.table} in the " +
              s"database ${identifier.database} because of ${e.toString}", e)
          0L
      }
    }.getOrElse(0L)
    val durationInMs = (System.nanoTime() - startTime) / (1000 * 1000)
    logDebug(s"It took $durationInMs ms to calculate the total file size under path $locationUri.")

    size
  }

```

```
private[spark] object HadoopFSUtils extends Logging {
  /**
   * Lists a collection of paths recursively. Picks the listing strategy adaptively depending
   * on the number of paths to list.
   *
   * This may only be called on the driver.
   *
   * @param sc Spark context used to run parallel listing.
   * @param paths Input paths to list
   * @param hadoopConf Hadoop configuration
   * @param filter Path filter used to exclude leaf files from result
   * @param ignoreMissingFiles Ignore missing files that happen during recursive listing
   *                           (e.g., due to race conditions)
   * @param ignoreLocality Whether to fetch data locality info when listing leaf files. If false,
   *                       this will return `FileStatus` without `BlockLocation` info.
   * @param parallelismThreshold The threshold to enable parallelism. If the number of input paths
   *                             is smaller than this value, this will fallback to use
   *                             sequential listing.
   * @param parallelismMax The maximum parallelism for listing. If the number of input paths is
   *                       larger than this value, parallelism will be throttled to this value
   *                       to avoid generating too many tasks.
   * @return for each input path, the set of discovered files for the path
   */
  def parallelListLeafFiles(
    sc: SparkContext,
    paths: Seq[Path],
    hadoopConf: Configuration,
    filter: PathFilter,
    ignoreMissingFiles: Boolean,
    ignoreLocality: Boolean,
    parallelismThreshold: Int,
    parallelismMax: Int): Seq[(Path, Seq[FileStatus])] = {
    parallelListLeafFilesInternal(sc, paths, hadoopConf, filter, isRootLevel = true,
      ignoreMissingFiles, ignoreLocality, parallelismThreshold, parallelismMax)
  }
```