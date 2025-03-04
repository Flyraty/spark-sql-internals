# QueryStageExec Leaf Physical Operators

`QueryStageExec` is an [extension](#contract) of the [LeafExecNode](SparkPlan.md#LeafExecNode) abstraction for [leaf physical operators](#implementations) for [Adaptive Query Execution](../adaptive-query-execution/index.md).

## Contract

### <span id="cancel"> Cancelling

```scala
cancel(): Unit
```

Cancels the stage materialization if in progress; otherwise does nothing.

Used when:

* `AdaptiveSparkPlanExec` physical operator is requested to [cleanUpAndThrowException](AdaptiveSparkPlanExec.md#cleanUpAndThrowException)

### <span id="doMaterialize"> Materializing

```scala
doMaterialize(): Future[Any]
```

Used when:

* `QueryStageExec` is requested to [materialize](#materialize)

### <span id="getRuntimeStatistics"> Runtime Statistics

```scala
getRuntimeStatistics: Statistics
```

[Statistics](../logical-operators/Statistics.md) after stage materialization

Used when:

* `QueryStageExec` is requested to [computeStats](#computeStats)

### <span id="id"> Unique ID

```scala
id: Int
```

Used when:

* [CoalesceShufflePartitions](../physical-optimizations/CoalesceShufflePartitions.md) adaptive physical optimization is executed

### <span id="newReuseInstance"> newReuseInstance

```scala
newReuseInstance(
  newStageId: Int,
  newOutput: Seq[Attribute]): QueryStageExec
```

Used when:

* `AdaptiveSparkPlanExec` physical operator is requested to [reuseQueryStage](AdaptiveSparkPlanExec.md#reuseQueryStage)

### <span id="plan"> Physical Query Plan

```scala
plan: SparkPlan
```

The [sub-tree](SparkPlan.md) of the main query plan of this query stage (that acts like a child operator, but `QueryStageExec` is a [LeafExecNode](SparkPlan.md#LeafExecNode) and has no children)

## Implementations

* <span id="BroadcastQueryStageExec"> [BroadcastQueryStageExec](BroadcastQueryStageExec.md)
* <span id="ShuffleQueryStageExec"> [ShuffleQueryStageExec](ShuffleQueryStageExec.md)

## <span id="resultOption"><span id="_resultOption"> Result

```scala
_resultOption: AtomicReference[Option[Any]]
```

`QueryStageExec` uses a `_resultOption` transient volatile internal variable (of type [AtomicReference]({{ java.api }}/java/util/concurrent/atomic/AtomicReference.html)) for the result of a successful [materialization](#materialize) of the `QueryStageExec` operator (when preparing for query execution):

* [Broadcast variable](BroadcastQueryStageExec.md#materializeWithTimeout) (_broadcasting data_) for [BroadcastQueryStageExec](BroadcastQueryStageExec.md)
* [MapOutputStatistics](ShuffleQueryStageExec.md#mapStats) (_submitting map stages_) for [ShuffleQueryStageExec](ShuffleQueryStageExec.md)

As `AtomicReference` is mutable that is enough to change the value.

`_resultOption` is set when `AdaptiveSparkPlanExec` physical operator is requested for the [final physical plan](AdaptiveSparkPlanExec.md#getFinalPhysicalPlan).

`_resultOption` is available using [resultOption](#resultOption).

### <span id="resultOption"> resultOption

```scala
resultOption: AtomicReference[Option[Any]]
```

`resultOption` returns the current value of the [_resultOption](#_resultOption) registry.

`resultOption` is used when:

* `QueryStageExec` is requested to [isMaterialized](#isMaterialized)
* `ShuffleQueryStageExec` is requested for the [MapOutputStatistics](ShuffleQueryStageExec.md#mapStats)

## <span id="computeStats"> Statistics

```scala
computeStats(): Option[Statistics]
```

If this `QueryStageExec` has been [materialized](#isMaterialized), `computeStats` gives a new [Statistics](../logical-operators/Statistics.md) with the [runtime statistics](#getRuntimeStatistics). Otherwise, `computeStats` returns no statistics (`None`).

`computeStats` is used when:

* `LogicalQueryStage` logical operator is requested for the [Statistics](../logical-operators/LogicalQueryStage.md#computeStats)

## <span id="materialize"> Materializing Query Stage

```scala
materialize(): Future[Any]
```

`materialize` prints out the following DEBUG message to the logs (with the [id](#id)):

```text
Materialize query stage [simpleName]: [id]
```

`materialize` [doMaterialize](#doMaterialize).

??? note "Final Method"
    `materialize` is a Scala **final method** and may not be overridden in [subclasses](#implementations).

    Learn more in the [Scala Language Specification]({{ scala.spec }}/05-classes-and-objects.html#final).

`materialize` is used when:

* `AdaptiveSparkPlanExec` physical operator is requested to [getFinalPhysicalPlan](AdaptiveSparkPlanExec.md#getFinalPhysicalPlan)

## <span id="generateTreeString"> Text Representation

```scala
generateTreeString(
  depth: Int,
  lastChildren: Seq[Boolean],
  append: String => Unit,
  verbose: Boolean,
  prefix: String = "",
  addSuffix: Boolean = false,
  maxFields: Int,
  printNodeId: Boolean,
  indent: Int = 0): Unit
```

`generateTreeString` is part of the [TreeNode](../catalyst/TreeNode.md#generateTreeString) abstraction.

`generateTreeString` [generateTreeString](../catalyst/TreeNode.md#generateTreeString) (the default) followed by another [generateTreeString](../catalyst/TreeNode.md#generateTreeString) (with the depth incremented).
