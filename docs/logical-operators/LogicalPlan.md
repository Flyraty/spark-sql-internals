# LogicalPlan &mdash; Logical Relational Operators of Structured Query

`LogicalPlan` is an extension of the [QueryPlan](../catalyst/QueryPlan.md) abstraction for [logical operators](#implementations) to build a **logical query plan** (as a tree of logical operators).

`LogicalPlan` is eventually resolved (_transformed_) to a [physical operator](../physical-operators/SparkPlan.md).

## Implementations

### <span id="BinaryNode"> BinaryNode

Logical operators with two [child](../catalyst/TreeNode.md#children) logical operators

### <span id="Command"> Command

[Command](Command.md)

### <span id="LeafNode"> LeafNode

[LeafNode](LeafNode.md) is a logical operator with no [child](../catalyst/TreeNode.md#children) operators

### <span id="UnaryNode"> UnaryNode

Logical operators with a single [child](../catalyst/TreeNode.md#children) logical operator

### Other Logical Operators

* [CreateTable](CreateTable.md)
* [IgnoreCachedData](IgnoreCachedData.md)
* [NamedRelation](NamedRelation.md)
* [ParsedStatement](ParsedStatement.md)
* [SupportsSubquery](SupportsSubquery.md)
* [V2CreateTablePlan](V2CreateTablePlan.md)
* [View](View.md)
* _others_

## <span id="statsCache"> Statistics Cache

Cached plan statistics (as `Statistics`) of the `LogicalPlan`

Computed and cached in [stats](#stats)

Used in [stats](#stats) and [verboseStringWithSuffix](#verboseStringWithSuffix)

Reset in [invalidateStatsCache](#invalidateStatsCache)

## <span id="stats"> Estimated Statistics

```scala
stats(
  conf: CatalystConf): Statistics
```

`stats` returns the <<statsCache, cached plan statistics>> or <<computeStats, computes a new one>> (and caches it as <<statsCache, statsCache>>).

`stats` is used when:

* A `LogicalPlan` <<computeStats, computes `Statistics`>>
* `QueryExecution` is requested to [build a complete text representation](../QueryExecution.md#completeString)
* `JoinSelection` [checks whether a plan can be broadcast](../execution-planning-strategies/JoinSelection.md#canBroadcast) et al
* CostBasedJoinReorder.md[CostBasedJoinReorder] attempts to reorder inner joins
* `LimitPushDown` is executed (for [FullOuter](../joins.md#FullOuter) join)
* `AggregateEstimation` estimates `Statistics`
* `FilterEstimation` estimates child `Statistics`
* `InnerOuterEstimation` estimates `Statistics` of the left and right sides of a join
* `LeftSemiAntiEstimation` estimates `Statistics`
* `ProjectEstimation` estimates `Statistics`

## <span id="refresh"> Refreshing Child Logical Operators

```scala
refresh(): Unit
```

`refresh` calls itself recursively for every [child](../catalyst/TreeNode.md#children) logical operator.

!!! note
    `refresh` is overriden by [LogicalRelation](LogicalRelation.md#refresh) only (that refreshes the location of `HadoopFsRelation` relations only).

`refresh` is used when:

* `SessionCatalog` is requested to [refresh a table](../SessionCatalog.md#refreshTable)

* `CatalogImpl` is requested to [refresh a table](../CatalogImpl.md#refreshTable)

## <span id="resolve"> Resolving Column Attributes to References in Query Plan

```scala
resolve(
  nameParts: Seq[String],
  resolver: Resolver): Option[NamedExpression]
resolve(
  schema: StructType,
  resolver: Resolver): Seq[Attribute]
```

`resolve` requests the [outputAttributes](#outputAttributes) to [resolve](../expressions/AttributeSeq.md#resolve) and then the [outputMetadataAttributes](#outputMetadataAttributes) if the first [resolve](../expressions/AttributeSeq.md#resolve) did not give a [NamedExpression](../expressions/NamedExpression.md).

## Accessing Logical Query Plan of Structured Query

In order to get the [logical plan](../QueryExecution.md#logical) of a structured query you should use the <<Dataset.md#queryExecution, QueryExecution>>.

```text
scala> :type q
org.apache.spark.sql.Dataset[Long]

val plan = q.queryExecution.logical
scala> :type plan
org.apache.spark.sql.catalyst.plans.logical.LogicalPlan
```

`LogicalPlan` goes through [execution stages](../QueryExecution.md#execution-pipeline) (as a [QueryExecution](../QueryExecution.md)). In order to convert a `LogicalPlan` to a `QueryExecution` you should use `SessionState` and request it to ["execute" the plan](../SessionState.md#executePlan).

```text
scala> :type spark
org.apache.spark.sql.SparkSession

// You could use Catalyst DSL to create a logical query plan
scala> :type plan
org.apache.spark.sql.catalyst.plans.logical.LogicalPlan

val qe = spark.sessionState.executePlan(plan)
scala> :type qe
org.apache.spark.sql.execution.QueryExecution
```

## <span id="maxRows"> Maximum Number of Records

```scala
maxRows: Option[Long]
```

`maxRows` is undefined by default (`None`).

`maxRows` is used when `LogicalPlan` is requested for [maxRowsPerPartition](#maxRowsPerPartition).

## <span id="maxRowsPerPartition"> Maximum Number of Records per Partition

```scala
maxRowsPerPartition: Option[Long]
```

`maxRowsPerPartition` is exactly the [maximum number of records](#maxRows) by default.

`maxRowsPerPartition` is used when [LimitPushDown](../logical-optimizations/LimitPushDown.md) logical optimization is executed.

## Executing Logical Plan

A common idiom in Spark SQL to make sure that a logical plan can be analyzed is to request a `SparkSession` for the [SessionState](../SparkSession.md#sessionState) that is in turn requested to ["execute"](../SessionState.md#executePlan) the logical plan (which simply creates a [QueryExecution](../QueryExecution.md)).

```text
scala> :type plan
org.apache.spark.sql.catalyst.plans.logical.LogicalPlan

val qe = sparkSession.sessionState.executePlan(plan)
qe.assertAnalyzed()
// the following gives the analyzed logical plan
// no exceptions are expected since analysis went fine
val analyzedPlan = qe.analyzed
```

## Converting Logical Plan to Dataset

Another common idiom in Spark SQL to convert a `LogicalPlan` into a `Dataset` is to use [Dataset.ofRows](../Dataset.md#ofRows) internal method that ["executes"](../SessionState.md#executePlan) the logical plan followed by creating a [Dataset](../Dataset.md) with the [QueryExecution](../QueryExecution.md) and [RowEncoder](../RowEncoder.md).

## <span id="childrenResolved"> childrenResolved

```scala
childrenResolved: Boolean
```

A logical operator is considered **partially resolved** when its [child operators](../catalyst/TreeNode.md#children) are resolved (aka _children resolved_).

## <span id="resolved"> resolved

```scala
resolved: Boolean
```

`resolved` is `true` for all [expressions](../catalyst/QueryPlan.md#expressions) and the [children](childrenResolved) resolved.

??? note "Lazy Value"
    `resolved` is a Scala **lazy value** to guarantee that the code to initialize it is executed once only (when accessed for the first time) and the computed value never changes afterwards.

## <span id="metadataOutput"> Metadata Output Attributes

```scala
metadataOutput: Seq[Attribute]
```

`metadataOutput` requests the [children](#children) for the `metadataOutput` (recursively).

is used when:

* `SubqueryAlias` is requested for the [metadataOutput](SubqueryAlias.md#metadataOutput)
* `LogicalPlan` is requested for the [childAttributes](#childAttributes)
* `DataSourceV2Relation` is requested to [include metadata columns](DataSourceV2Relation.md#withMetadataColumns)
