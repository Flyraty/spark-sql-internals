# FileScanRDD &mdash; Input RDD of FileSourceScanExec Physical Operator

`FileScanRDD` is the [input RDD](../physical-operators/FileSourceScanExec.md#inputRDD) of [FileSourceScanExec](../physical-operators/FileSourceScanExec.md) leaf physical operator (for [Whole-Stage Java Code Generation](../whole-stage-code-generation/index.md)).

!!! note "The Internals of Apache Spark"
    Find out more on `RDD` abstraction in [The Internals of Apache Spark](https://books.japila.pl/apache-spark-internals/apache-spark-internals/rdd/RDD.html).

## Creating Instance

`FileScanRDD` takes the following to be created:

* <span id="sparkSession"> [SparkSession](../SparkSession.md)
* <span id="readFunction"> Read Function that takes a [PartitionedFile](../datasources/PartitionedFile.md) and gives [internal binary rows](../InternalRow.md) back (`(PartitionedFile) => Iterator[InternalRow]`)
* <span id="filePartitions"> File Blocks as `FilePartition`s (`Seq[FilePartition]`)

`FileScanRDD` is created when [FileSourceScanExec](../physical-operators/FileSourceScanExec.md) physical operator is requested to [createBucketedReadRDD](../physical-operators/FileSourceScanExec.md#createBucketedReadRDD) and [createNonBucketedReadRDD](../physical-operators/FileSourceScanExec.md#createNonBucketedReadRDD) (when `FileSourceScanExec` operator is requested for the [input RDD](../physical-operators/FileSourceScanExec.md#inputRDD) when [WholeStageCodegenExec](../physical-operators/WholeStageCodegenExec.md) physical operator is executed).

## Configuration Properties

`FileScanRDD` uses the following properties (when requested to [compute a partition](#compute)):

* <span id="ignoreCorruptFiles"> [spark.sql.files.ignoreCorruptFiles](../configuration-properties.md#spark.sql.files.ignoreCorruptFiles)
* <span id="ignoreMissingFiles"> [spark.sql.files.ignoreMissingFiles](../configuration-properties.md#spark.sql.files.ignoreMissingFiles)

## <span id="FilePartition"><span id="files"><span id="index"> FilePartition

`FileScanRDD` is given [FilePartitions](#filePartitions) when [created](#creating-instance) that are custom RDD partitions with [PartitionedFiles](../datasources/PartitionedFile.md) (_file blocks_).

## <span id="getPreferredLocations"> Placement Preferences of Partition (Preferred Locations)

```scala
getPreferredLocations(
  split: RDDPartition): Seq[String]
```

`getPreferredLocations` assumes that the given `RDDPartition` is actually a [FilePartition](#FilePartition) and requests it for `preferredLocations`.

`getPreferredLocations` is part of Spark Core's `RDD` abstraction.

## <span id="getPartitions"> RDD Partitions

```scala
getPartitions: Array[RDDPartition]
```

`getPartitions` simply returns the [FilePartitions](#filePartitions).

`getPartitions` is part of Spark Core's `RDD` abstraction.

## <span id="compute"> Computing Partition

```scala
compute(
  split: RDDPartition,
  context: TaskContext): Iterator[InternalRow]
```

!!! note
    The `RDDPartition` given is actually a [FilePartition](#FilePartition) with one or more [PartitionedFiles](../datasources/PartitionedFile.md) (that [getPartitions](#getPartitions) returned).

`compute` is part of Spark Core's `RDD` abstraction.

### <span id="compute-next"> Retrieving Next Element

```scala
next(): Object
```

`next` takes the next element of the current iterator over elements of a file block ([PartitionedFile](../datasources/PartitionedFile.md)).

`next` increments the metrics of bytes and number of rows read (that could be the number of rows in a [ColumnarBatch](../ColumnarBatch.md) for vectorized reads).

`next` is part of Scala's [Iterator](https://www.scala-lang.org/api/2.12.x/scala/collection/Iterator.html#next) abstraction.

## Demo

```text
val q = spark.read.text("README.md")
val sparkPlan = q.queryExecution.executedPlan
scala> println(sparkPlan.numberedTreeString)
00 FileScan text [value#0] Batched: false, DataFilters: [], Format: Text, Location: InMemoryFileIndex[file:/Users/jacek/dev/oss/spark/README.md], PartitionFilters: [], PushedFilters: [], ReadSchema: struct<value:string>

import org.apache.spark.sql.execution.FileSourceScanExec
val scan = sparkPlan.collectFirst { case exec: FileSourceScanExec => exec }.get
val inputRDD = scan.inputRDDs.head

import org.apache.spark.sql.execution.datasources.FileScanRDD
assert(inputRDD.isInstanceOf[FileScanRDD])

val rdd = scan.execute
scala> println(rdd.toDebugString)
(1) MapPartitionsRDD[1] at execute at <console>:27 []
 |  FileScanRDD[0] at inputRDDs at <console>:26 []

val fileScanRDD = rdd.dependencies.head.rdd
assert(fileScanRDD.isInstanceOf[FileScanRDD])
```

## Logging

Enable `ALL` logging level for `org.apache.spark.sql.execution.datasources.FileScanRDD` logger to see what happens inside.

Add the following line to `conf/log4j.properties`:

```text
log4j.logger.org.apache.spark.sql.execution.datasources.FileScanRDD=ALL
```

Refer to [Logging](../spark-logging.md).
