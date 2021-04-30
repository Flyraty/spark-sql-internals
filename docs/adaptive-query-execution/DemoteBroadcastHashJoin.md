# DemoteBroadcastHashJoin Logical Optimization

`DemoteBroadcastHashJoin` is a logical optimization in [Adaptive Query Execution](../adaptive-query-execution/index.md) to [transform Join logical operators](#apply) (with no [join hints](../JoinStrategyHint.md)).

Quoting [What's new in Apache Spark 3.0 - demote broadcast hash join](https://www.waitingforcode.com/apache-spark-sql/whats-new-apache-spark-3-demote-broadcast-hash-join/read) article:

> This rule checks whether the nodes involved in the join have a lot of empty partitions. If it's the case, the rule adds a no broadcast hash join hint to prevent the broadcast strategy to be applied.

`DemoteBroadcastHashJoin` uses [spark.sql.adaptive.nonEmptyPartitionRatioForBroadcastJoin](../configuration-properties.md#spark.sql.adaptive.nonEmptyPartitionRatioForBroadcastJoin) configuration property for the threshold to [demote a broadcast hash join](#shouldDemote).

`DemoteBroadcastHashJoin` is a [Catalyst rule](../catalyst/Rule.md) for transforming [logical plans](../logical-operators/LogicalPlan.md) (`Rule[LogicalPlan]`).

## Creating Instance

`DemoteBroadcastHashJoin` takes the following to be created:

* <span id="conf"> [SQLConf](../SQLConf.md)

`DemoteBroadcastHashJoin` is created when:

* [AdaptiveSparkPlanExec](../adaptive-query-execution/AdaptiveSparkPlanExec.md) physical operator is requested for the [logical optimizer](../adaptive-query-execution/AdaptiveSparkPlanExec.md#optimizer)

## <span id="apply"> Executing Rule

```scala
apply(
  plan: LogicalPlan): LogicalPlan
```

`apply` is part of the [Rule](../catalyst/Rule.md#apply) abstraction.

`apply` transforms [Join](../logical-operators/Join.md) logical operators with no [JoinStrategyHint](../JoinStrategyHint.md) hints.

`apply` checks whether to [demote or not](#shouldDemote) for the left first and then for the right side of the join operator. If so, `apply` registers [NO_BROADCAST_HASH](../JoinStrategyHint.md#NO_BROADCAST_HASH) join hint with the join operator.

### <span id="shouldDemote"> shouldDemote

```scala
shouldDemote(
  plan: LogicalPlan): Boolean
```

`shouldDemote` supports [LogicalQueryStage](../adaptive-query-execution/LogicalQueryStage.md) logical operators with [ShuffleQueryStageExec](../adaptive-query-execution/ShuffleQueryStageExec.md) physical operators only. For any other [logical operators](../logical-operators/LogicalPlan.md) `shouldDemote` is `false`.

`shouldDemote` makes sure that the [result](../adaptive-query-execution/QueryStageExec.md#resultOption) and [MapOutputStatistics](../adaptive-query-execution/ShuffleQueryStageExec.md#mapStats) of the `ShuffleQueryStageExec` operator are available. Otherwise, `shouldDemote` is `false`.

`shouldDemote` is `true` when the ratio of the non-empty partitions to all the partitions (based on the `MapOutputStatistics`) is below [spark.sql.adaptive.nonEmptyPartitionRatioForBroadcastJoin](../configuration-properties.md#spark.sql.adaptive.nonEmptyPartitionRatioForBroadcastJoin) configuration property.