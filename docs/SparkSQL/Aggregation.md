[TOC]

# Aggregation

## code

###  test case test("groupBy")

```scala
class DataFrameAggregateSuite extends QueryTest
  with SharedSparkSession
  with AdaptiveSparkPlanHelper {
  import testImplicits._

  val absTol = 1e-8

  test("groupBy1") {
    checkAnswer(
      testData2.groupBy("a").agg(sum($"b")),
      Seq(Row(1, 3), Row(2, 3), Row(3, 3))
    )
  }

  test("count withSQLConf") {
    for ((wholeStage, useObjectHashAgg) <-
           Seq(
//             (true, true),
//             (true, false),
             (false, true),
//             (false, false)
           )) {
      withSQLConf(
        (SQLConf.WHOLESTAGE_CODEGEN_ENABLED.key, wholeStage.toString),
        (SQLConf.USE_OBJECT_HASH_AGG.key, useObjectHashAgg.toString)) {
        checkAnswer(
          testData2.groupBy("a").agg(count("*")),
          Row(1, 2) :: Row(2, 2) :: Row(3, 2) :: Nil
        )
      }
    }
  }

```

### HashAggregateExec
**org.apache.spark.sql.execution.aggregate.HashAggregateExec#doExecute** is not used if whole code gen enabled

org.apache.spark.sql.execution.aggregate.AggregateCodegenSupport#doProduce
```
  protected override def doProduce(ctx: CodegenContext): String = {
    if (groupingExpressions.isEmpty) {
      doProduceWithoutKeys(ctx)
    } else {
      doProduceWithKeys(ctx)
    }
  }
```
=>
org.apache.spark.sql.execution.aggregate.HashAggregateExec#doProduceWithKeys



org.apache.spark.sql.execution.InputAdapter ====> Scan[obj#2]
=>
org.apache.spark.sql.execution.InputRDDCodegen#doProduce
=>
org.apache.spark.sql.execution.InputAdapter (call InputAdapter's consume which is in its parent CodegenSupport)
=>
org.apache.spark.sql.execution.CodegenSupport#consume
=>
org.apache.spark.sql.execution.CodegenSupport#constructDoConsumeFunction
=>
org.apache.spark.sql.execution.SerializeFromObjectExec#doConsume
=>
org.apache.spark.sql.execution.aggregate.HashAggregateExec (call HashAggregateExec's doConsume which is in its parent AggregateCodegenSupport)
=>
org.apache.spark.sql.execution.aggregate.AggregateCodegenSupport#doConsume
=>
org.apache.spark.sql.execution.aggregate.HashAggregateExec#doConsumeWithKeys
### BufferedRowIterator
```scala
/**
 * An iterator interface used to pull the output from generated function for multiple operators
 * (whole stage codegen).
 */
public abstract class BufferedRowIterator {
  protected LinkedList<InternalRow> currentRows = new LinkedList<>();
  // used when there is no column in output
  protected UnsafeRow unsafeRow = new UnsafeRow(0);
  private long startTimeNs = System.nanoTime();

  protected int partitionIndex = -1;

  public boolean hasNext() throws IOException {
    if (currentRows.isEmpty()) {
      processNext();
    }
    return !currentRows.isEmpty();
  }
  
  public void append(InternalRow row) { ===> called by GeneratedIteratorForCodegenStage1.hashAgg_doAggregateWithKeysOutput_0
    currentRows.add(row);
  }

  /**
   * Returns whether `processNext()` should stop processing next row from `input` or not.
   *
   * If it returns true, the caller should exit the loop (return from processNext()).
   */
  public boolean shouldStop() {
    return !currentRows.isEmpty();
  }
  
```


### GeneratedIteratorForCodegenStage1
org.apache.spark.sql.execution.WholeStageCodegenExec#doCodeGen => 'cleanedSource'

final class GeneratedIteratorForCodegenStage1 extends org.apache.spark.sql.execution.BufferedRowIterator {
          =>      processNext()

```java
public Object generate(Object[] references) {
	return new GeneratedIteratorForCodegenStage1(references);
}

/*wsc_codegenStageId*/
final class GeneratedIteratorForCodegenStage1 extends org.apache.spark.sql.execution.BufferedRowIterator {
	private Object[] references;
	private scala.collection.Iterator[] inputs;
	private boolean hashAgg_initAgg_0;
	private boolean hashAgg_bufIsNull_0;
	private long hashAgg_bufValue_0;
	private hashAgg_FastHashMap_0 hashAgg_fastHashMap_0;
	private org.apache.spark.unsafe.KVIterator < UnsafeRow, UnsafeRow > hashAgg_fastHashMapIter_0;
	private org.apache.spark.unsafe.KVIterator hashAgg_mapIter_0;
	private org.apache.spark.sql.execution.UnsafeFixedWidthAggregationMap hashAgg_hashMap_0;
	private org.apache.spark.sql.execution.UnsafeKVExternalSorter hashAgg_sorter_0;
	private scala.collection.Iterator inputadapter_input_0;
	private boolean hashAgg_hashAgg_isNull_6_0;
	private org.apache.spark.sql.catalyst.expressions.codegen.UnsafeRowWriter[] serializefromobject_mutableStateArray_0 = new org.apache.spark.sql.catalyst.expressions.codegen.UnsafeRowWriter[4];

	public GeneratedIteratorForCodegenStage1(Object[] references) {
		this.references = references;
	}

	public void init(int index, scala.collection.Iterator[] inputs) {
		partitionIndex = index;
		this.inputs = inputs;

		inputadapter_input_0 = inputs[0];
		serializefromobject_mutableStateArray_0[0] = new org.apache.spark.sql.catalyst.expressions.codegen.UnsafeRowWriter(2, 0);
		serializefromobject_mutableStateArray_0[1] = new org.apache.spark.sql.catalyst.expressions.codegen.UnsafeRowWriter(2, 0);
		serializefromobject_mutableStateArray_0[2] = new org.apache.spark.sql.catalyst.expressions.codegen.UnsafeRowWriter(1, 0);
		serializefromobject_mutableStateArray_0[3] = new org.apache.spark.sql.catalyst.expressions.codegen.UnsafeRowWriter(2, 0);

	}

	public class hashAgg_FastHashMap_0 {
		private org.apache.spark.sql.catalyst.expressions.RowBasedKeyValueBatch batch;
		private int[] buckets;
		private int capacity = 1 << 16;
		private double loadFactor = 0.5;
		private int numBuckets = (int)(capacity / loadFactor);
		private int maxSteps = 2;
		private int numRows = 0;
		private Object emptyVBase;
		private long emptyVOff;
		private int emptyVLen;
		private boolean isBatchFull = false;
		private org.apache.spark.sql.catalyst.expressions.codegen.UnsafeRowWriter agg_rowWriter;

		public hashAgg_FastHashMap_0(
			org.apache.spark.memory.TaskMemoryManager taskMemoryManager,
			InternalRow emptyAggregationBuffer) {
			batch = org.apache.spark.sql.catalyst.expressions.RowBasedKeyValueBatch
				.allocate(((org.apache.spark.sql.types.StructType) references[1] /* keySchemaTerm */ ), ((org.apache.spark.sql.types.StructType) references[2] /* valueSchemaTerm */ ), taskMemoryManager, capacity);

			final UnsafeProjection valueProjection = UnsafeProjection.create(((org.apache.spark.sql.types.StructType) references[2] /* valueSchemaTerm */ ));
			final byte[] emptyBuffer = valueProjection.apply(emptyAggregationBuffer).getBytes();

			emptyVBase = emptyBuffer;
			emptyVOff = Platform.BYTE_ARRAY_OFFSET;
			emptyVLen = emptyBuffer.length;

			agg_rowWriter = new org.apache.spark.sql.catalyst.expressions.codegen.UnsafeRowWriter(
				1, 0);

			buckets = new int[numBuckets];
			java.util.Arrays.fill(buckets, -1);
		}

		public org.apache.spark.sql.catalyst.expressions.UnsafeRow findOrInsert(int hashAgg_key_0) {
			long h = hash(hashAgg_key_0);
			int step = 0;
			int idx = (int) h & (numBuckets - 1);
			while (step < maxSteps) {
				// Return bucket index if it's either an empty slot or already contains the key
				if (buckets[idx] == -1) {
					if (numRows < capacity && !isBatchFull) {
						agg_rowWriter.reset();

						agg_rowWriter.write(0, hashAgg_key_0);
						org.apache.spark.sql.catalyst.expressions.UnsafeRow agg_result = agg_rowWriter.getRow();
						Object kbase = agg_result.getBaseObject();
						long koff = agg_result.getBaseOffset();
						int klen = agg_result.getSizeInBytes();

						UnsafeRow vRow
							= batch.appendRow(kbase, koff, klen, emptyVBase, emptyVOff, emptyVLen);
						if (vRow == null) {
							isBatchFull = true;
						} else {
							buckets[idx] = numRows++;
						}
						return vRow;
					} else {
						// No more space
						return null;
					}
				} else if (equals(idx, hashAgg_key_0)) {
					return batch.getValueRow(buckets[idx]);
				}
				idx = (idx + 1) & (numBuckets - 1);
				step++;
			}
			// Didn't find it
			return null;
		}

		private boolean equals(int idx, int hashAgg_key_0) {
			UnsafeRow row = batch.getKeyRow(buckets[idx]);
			return (row.getInt(0) == hashAgg_key_0);
		}

		private long hash(int hashAgg_key_0) {
			long hashAgg_hash_0 = 0;

			int hashAgg_result_0 = hashAgg_key_0;
			hashAgg_hash_0 = (hashAgg_hash_0 ^ (0x9e3779b9)) + hashAgg_result_0 + (hashAgg_hash_0 << 6) + (hashAgg_hash_0 >>> 2);

			return hashAgg_hash_0;
		}

		public org.apache.spark.unsafe.KVIterator < UnsafeRow, UnsafeRow > rowIterator() {
			return batch.rowIterator();
		}

		public void close() {
			batch.close();
		}

	}

	private void hashAgg_doAggregateWithKeys_0() throws java.io.IOException {
		while (inputadapter_input_0.hasNext()) {
			InternalRow inputadapter_row_0 = (InternalRow) inputadapter_input_0.next();

			boolean inputadapter_isNull_0 = inputadapter_row_0.isNullAt(0);
			org.apache.spark.sql.test.SQLTestData$TestData2 inputadapter_value_0 = inputadapter_isNull_0 ?
				null : ((org.apache.spark.sql.test.SQLTestData$TestData2) inputadapter_row_0.get(0, null));

			serializefromobject_doConsume_0(inputadapter_row_0, inputadapter_value_0, inputadapter_isNull_0);
			// shouldStop check is eliminated
		}

		hashAgg_fastHashMapIter_0 = hashAgg_fastHashMap_0.rowIterator();
		hashAgg_mapIter_0 = ((org.apache.spark.sql.execution.aggregate.HashAggregateExec) references[0] /* plan */ ).finishAggregate(hashAgg_hashMap_0, hashAgg_sorter_0, ((org.apache.spark.sql.execution.metric.SQLMetric) references[3] /* peakMemory */ ), ((org.apache.spark.sql.execution.metric.SQLMetric) references[4] /* spillSize */ ), ((org.apache.spark.sql.execution.metric.SQLMetric) references[5] /* avgHashProbe */ ), ((org.apache.spark.sql.execution.metric.SQLMetric) references[6] /* numTasksFallBacked */ ));

	}

	private void hashAgg_doConsume_0(int hashAgg_expr_0_0, int hashAgg_expr_1_0) throws java.io.IOException {
		UnsafeRow hashAgg_unsafeRowAggBuffer_0 = null;
		UnsafeRow hashAgg_fastAggBuffer_0 = null;

		if (!false) {
			hashAgg_fastAggBuffer_0 = hashAgg_fastHashMap_0.findOrInsert(
				hashAgg_expr_0_0);
		}
		// Cannot find the key in fast hash map, try regular hash map.
		if (hashAgg_fastAggBuffer_0 == null) {
			// generate grouping key
			serializefromobject_mutableStateArray_0[2].reset();

			serializefromobject_mutableStateArray_0[2].write(0, hashAgg_expr_0_0);
			int hashAgg_unsafeRowKeyHash_0 = (serializefromobject_mutableStateArray_0[2].getRow()).hashCode();
			if (true) {
				// try to get the buffer from hash map
				hashAgg_unsafeRowAggBuffer_0 =
					hashAgg_hashMap_0.getAggregationBufferFromUnsafeRow((serializefromobject_mutableStateArray_0[2].getRow()), hashAgg_unsafeRowKeyHash_0);
			}
			// Can't allocate buffer from the hash map. Spill the map and fallback to sort-based
			// aggregation after processing all input rows.
			if (hashAgg_unsafeRowAggBuffer_0 == null) {
				if (hashAgg_sorter_0 == null) {
					hashAgg_sorter_0 = hashAgg_hashMap_0.destructAndCreateExternalSorter();
				} else {
					hashAgg_sorter_0.merge(hashAgg_hashMap_0.destructAndCreateExternalSorter());
				}

				// the hash map had be spilled, it should have enough memory now,
				// try to allocate buffer again.
				hashAgg_unsafeRowAggBuffer_0 = hashAgg_hashMap_0.getAggregationBufferFromUnsafeRow(
					(serializefromobject_mutableStateArray_0[2].getRow()), hashAgg_unsafeRowKeyHash_0);
				if (hashAgg_unsafeRowAggBuffer_0 == null) {
					// failed to allocate the first page
					throw new org.apache.spark.memory.SparkOutOfMemoryError("No enough memory for aggregation");
				}
			}

		}

		// Updates the proper row buffer
		if (hashAgg_fastAggBuffer_0 != null) {
			hashAgg_unsafeRowAggBuffer_0 = hashAgg_fastAggBuffer_0;
		}

		// common sub-expressions

		// evaluate aggregate functions and update aggregation buffers

		hashAgg_hashAgg_isNull_6_0 = true;
		long hashAgg_value_7 = -1 L;
		do {
			boolean hashAgg_isNull_7 = hashAgg_unsafeRowAggBuffer_0.isNullAt(0);
			long hashAgg_value_8 = hashAgg_isNull_7 ?
				-1 L : (hashAgg_unsafeRowAggBuffer_0.getLong(0));
			if (!hashAgg_isNull_7) {
				hashAgg_hashAgg_isNull_6_0 = false;
				hashAgg_value_7 = hashAgg_value_8;
				continue;
			}

			if (!false) {
				hashAgg_hashAgg_isNull_6_0 = false;
				hashAgg_value_7 = 0 L;
				continue;
			}

		} while (false);
		boolean hashAgg_isNull_9 = false;
		long hashAgg_value_10 = -1 L;
		if (!false) {
			hashAgg_value_10 = (long) hashAgg_expr_1_0;
		}
		long hashAgg_value_6 = -1 L;

		hashAgg_value_6 = hashAgg_value_7 + hashAgg_value_10;

		hashAgg_unsafeRowAggBuffer_0.setLong(0, hashAgg_value_6);

	}

	private void serializefromobject_doConsume_0(InternalRow inputadapter_row_0, org.apache.spark.sql.test.SQLTestData$TestData2 serializefromobject_expr_0_0, boolean serializefromobject_exprIsNull_0_0) throws java.io.IOException {
		if (serializefromobject_exprIsNull_0_0) {
			throw new NullPointerException(((java.lang.String) references[7] /* errMsg */ ));
		}
		boolean serializefromobject_isNull_0 = true;
		int serializefromobject_value_0 = -1;
		serializefromobject_isNull_0 = false;
		if (!serializefromobject_isNull_0) {
			serializefromobject_value_0 = serializefromobject_expr_0_0.a();
		}
		if (serializefromobject_exprIsNull_0_0) {
			throw new NullPointerException(((java.lang.String) references[8] /* errMsg */ ));
		}
		boolean serializefromobject_isNull_4 = true;
		int serializefromobject_value_4 = -1;
		serializefromobject_isNull_4 = false;
		if (!serializefromobject_isNull_4) {
			serializefromobject_value_4 = serializefromobject_expr_0_0.b();
		}

		hashAgg_doConsume_0(serializefromobject_value_0, serializefromobject_value_4);

	}

	private void hashAgg_doAggregateWithKeysOutput_0(UnsafeRow hashAgg_keyTerm_0, UnsafeRow hashAgg_bufferTerm_0)
	throws java.io.IOException {
		((org.apache.spark.sql.execution.metric.SQLMetric) references[9] /* numOutputRows */ ).add(1);

		int hashAgg_value_12 = hashAgg_keyTerm_0.getInt(0);
		boolean hashAgg_isNull_12 = hashAgg_bufferTerm_0.isNullAt(0);
		long hashAgg_value_13 = hashAgg_isNull_12 ?
			-1 L : (hashAgg_bufferTerm_0.getLong(0));

		serializefromobject_mutableStateArray_0[3].reset();

		serializefromobject_mutableStateArray_0[3].zeroOutNullBytes();

		serializefromobject_mutableStateArray_0[3].write(0, hashAgg_value_12);

		if (hashAgg_isNull_12) {
			serializefromobject_mutableStateArray_0[3].setNullAt(1);
		} else {
			serializefromobject_mutableStateArray_0[3].write(1, hashAgg_value_13);
		}
		append((serializefromobject_mutableStateArray_0[3].getRow()));  ===> append() is in 'BufferedRowIterator'

	}

	protected void processNext() throws java.io.IOException {
		if (!hashAgg_initAgg_0) {
			hashAgg_initAgg_0 = true;
			hashAgg_fastHashMap_0 = new hashAgg_FastHashMap_0(((org.apache.spark.sql.execution.aggregate.HashAggregateExec) references[0] /* plan */ ).getTaskContext().taskMemoryManager(), ((org.apache.spark.sql.execution.aggregate.HashAggregateExec) references[0] /* plan */ ).getEmptyAggregationBuffer());

			((org.apache.spark.sql.execution.aggregate.HashAggregateExec) references[0] /* plan */ ).getTaskContext().addTaskCompletionListener(
				new org.apache.spark.util.TaskCompletionListener() {
					@Override
					public void onTaskCompletion(org.apache.spark.TaskContext context) {
						hashAgg_fastHashMap_0.close();
					}
				});

			hashAgg_hashMap_0 = ((org.apache.spark.sql.execution.aggregate.HashAggregateExec) references[0] /* plan */ ).createHashMap();
			long wholestagecodegen_beforeAgg_0 = System.nanoTime();
			hashAgg_doAggregateWithKeys_0();
			((org.apache.spark.sql.execution.metric.SQLMetric) references[10] /* aggTime */ ).add((System.nanoTime() - wholestagecodegen_beforeAgg_0) / 1000000);
		}
		// output the result

		while (hashAgg_fastHashMapIter_0.next()) {
			UnsafeRow hashAgg_aggKey_0 = (UnsafeRow) hashAgg_fastHashMapIter_0.getKey();
			UnsafeRow hashAgg_aggBuffer_0 = (UnsafeRow) hashAgg_fastHashMapIter_0.getValue();
			hashAgg_doAggregateWithKeysOutput_0(hashAgg_aggKey_0, hashAgg_aggBuffer_0);

			if (shouldStop()) return;
		}
		hashAgg_fastHashMap_0.close();

		while (hashAgg_mapIter_0.next()) {
			UnsafeRow hashAgg_aggKey_0 = (UnsafeRow) hashAgg_mapIter_0.getKey();
			UnsafeRow hashAgg_aggBuffer_0 = (UnsafeRow) hashAgg_mapIter_0.getValue();
			hashAgg_doAggregateWithKeysOutput_0(hashAgg_aggKey_0, hashAgg_aggBuffer_0);
			if (shouldStop()) return;
		}
		hashAgg_mapIter_0.close();
		if (hashAgg_sorter_0 == null) {
			hashAgg_hashMap_0.free();
		}
	}

}

```

### inputRDDs
plan
```
*(1) HashAggregate(keys=[a#3], functions=[partial_sum(b#4)], output=[a#3, sum#17L])
+- *(1) SerializeFromObject [knownnotnull(assertnotnull(input[0, org.apache.spark.sql.test.SQLTestData$TestData2, true])).a AS a#3, knownnotnull(assertnotnull(input[0, org.apache.spark.sql.test.SQLTestData$TestData2, true])).b AS b#4]
   +- Scan[obj#2]
```

code

``` scala
org.apache.spark.sql.execution.WholeStageCodegenExec#doExecute

  override def doExecute(): RDD[InternalRow] = {
    val (ctx, cleanedSource) = doCodeGen()
    // try to compile and fallback if it failed
    val (_, compiledCodeStats) = try {
      CodeGenerator.compile(cleanedSource)
    } catch {
      case NonFatal(_) if !Utils.isTesting && conf.codegenFallback =>
        // We should already saw the error message
        logWarning(s"Whole-stage codegen disabled for plan (id=$codegenStageId):\n $treeString")
        return child.execute()
    }

    // Check if compiled code has a too large function
    if (compiledCodeStats.maxMethodCodeSize > conf.hugeMethodLimit) {
      logInfo(s"Found too long generated codes and JIT optimization might not work: " +
        s"the bytecode size (${compiledCodeStats.maxMethodCodeSize}) is above the limit " +
        s"${conf.hugeMethodLimit}, and the whole-stage codegen was disabled " +
        s"for this plan (id=$codegenStageId). To avoid this, you can raise the limit " +
        s"`${SQLConf.WHOLESTAGE_HUGE_METHOD_LIMIT.key}`:\n$treeString")
      return child.execute()
    }

    val references = ctx.references.toArray

    val durationMs = longMetric("pipelineTime")

    // Even though rdds is an RDD[InternalRow] it may actually be an RDD[ColumnarBatch] with
    // type erasure hiding that. This allows for the input to a code gen stage to be columnar,
    // but the output must be rows.
    val rdds = child.asInstanceOf[CodegenSupport].inputRDDs()
    assert(rdds.size <= 2, "Up to two input RDDs can be supported")
    val evaluatorFactory = new WholeStageCodegenEvaluatorFactory(
      cleanedSource, durationMs, references)
    if (rdds.length == 1) {
      if (conf.usePartitionEvaluator) {
        rdds.head.mapPartitionsWithEvaluator(evaluatorFactory) ===> Return a new RDD by applying an evaluator to each partition of this RDD. The given evaluator factory will be serialized and sent to executors, and each task will create an evaluator with the factory, and use the evaluator to transform the data of the input partition.
      } else {
        rdds.head.mapPartitionsWithIndex { (index, iter) =>
          val evaluator = evaluatorFactory.createEvaluator()
          evaluator.eval(index, iter)  ===> run the generated code
        }
      }
    } else {
      // Right now, we support up to two input RDDs.
      if (conf.usePartitionEvaluator) {
        rdds.head.zipPartitionsWithEvaluator(rdds(1), evaluatorFactory)
      } else {
        rdds.head.zipPartitions(rdds(1)) { (leftIter, rightIter) =>
          Iterator((leftIter, rightIter))
          // a small hack to obtain the correct partition index
        }.mapPartitionsWithIndex { (index, zippedIter) =>
          val (leftIter, rightIter) = zippedIter.next()
          val evaluator = evaluatorFactory.createEvaluator()
          evaluator.eval(index, leftIter, rightIter)
        }
      }
    }
  }
```
=> org.apache.spark.sql.execution.aggregate.AggregateCodegenSupport#inputRDDs,  org.apache.spark.sql.execution.SerializeFromObjectExec#inputRDDs
```
  override def inputRDDs(): Seq[RDD[InternalRow]] = {
    child.asInstanceOf[CodegenSupport].inputRDDs()
  }
```
=> org.apache.spark.sql.execution.InputAdapter
```scala
/**
 * InputAdapter is used to hide a SparkPlan from a subtree that supports codegen.
 *
 * This is the leaf node of a tree with WholeStageCodegen that is used to generate code
 * that consumes an RDD iterator of InternalRow.
 */
case class InputAdapter(child: SparkPlan) extends UnaryExecNode with InputRDDCodegen {
  // `InputAdapter` can only generate code to process the rows from its child. If the child produces
  // columnar batches, there must be a `ColumnarToRowExec` above `InputAdapter` to handle it by
  // overriding `inputRDDs` and calling `InputAdapter#executeColumnar` directly.
  override def inputRDD: RDD[InternalRow] = child.execute()
```
=> org.apache.spark.sql.execution.InputRDDCodegen

```

/**
 * Leaf codegen node reading from a single RDD.
 */
trait InputRDDCodegen extends CodegenSupport {

  def inputRDD: RDD[InternalRow]
  
  ...
  
  override def inputRDDs(): Seq[RDD[InternalRow]] = {
    inputRDD :: Nil
  }
```

=> inputRDD is in InputAdapter (override def inputRDD: RDD[InternalRow] = child.execute())  whose 'child' is ExternalRDDScanExec
org.apache.spark.sql.execution.ExternalRDDScanExec
```
/** Physical plan node for scanning data from an RDD. */
case class ExternalRDDScanExec[T](
    outputObjAttr: Attribute,
    rdd: RDD[T]) extends LeafExecNode with ObjectProducerExec {

```


ExternalRDDScanExec is initialized at
```
  object BasicOperators extends Strategy {

        case ExternalRDD(outputObjAttr, rdd) => ExternalRDDScanExec(outputObjAttr, rdd) :: Nil

```
