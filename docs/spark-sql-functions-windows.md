# Standard Functions for Window Aggregation (Window Functions)

**Window aggregate functions** (aka **window functions** or **windowed aggregates**) are functions that perform a calculation over a group of records called **window** that are in _some_ relation to the current record (i.e. can be in the same partition or frame as the current row).

In other words, when executed, a window function computes a value for each and every row in a window (per [window specification](window-functions/WindowSpec.md)).

Window functions are also called **over functions** due to how they are applied using [over](Column.md#over) operator.

Spark SQL supports three kinds of window functions:

* *ranking* functions
* *analytic* functions
* *aggregate* functions

.Window Aggregate Functions in Spark SQL
[align="center",cols="1,1,2",width="80%",options="header"]
|===
|
| Function
| Purpose

.5+^.^|*Ranking functions*

| <<rank, rank>>
|

| <<dense_rank, dense_rank>>
|

| <<percent_rank, percent_rank>>
|

| <<ntile, ntile>>
|

| <<row_number, row_number>>
|

.3+^.^| *Analytic functions*

| <<cume_dist, cume_dist>>
a|

| <<lag, lag>>
|

| <<lead, lead>>
|
|===

For aggregate functions, you can use the existing [aggregate functions](basic-aggregation/index.md) as window functions, e.g. `sum`, `avg`, `min`, `max` and `count`.

```text
// Borrowed from 3.5. Window Functions in PostgreSQL documentation
// Example of window functions using Scala API
//
case class Salary(depName: String, empNo: Long, salary: Long)
val empsalary = Seq(
  Salary("sales", 1, 5000),
  Salary("personnel", 2, 3900),
  Salary("sales", 3, 4800),
  Salary("sales", 4, 4800),
  Salary("personnel", 5, 3500),
  Salary("develop", 7, 4200),
  Salary("develop", 8, 6000),
  Salary("develop", 9, 4500),
  Salary("develop", 10, 5200),
  Salary("develop", 11, 5200)).toDS

import org.apache.spark.sql.expressions.Window
// Windows are partitions of deptName
scala> val byDepName = Window.partitionBy('depName)
byDepName: org.apache.spark.sql.expressions.WindowSpec = org.apache.spark.sql.expressions.WindowSpec@1a711314

scala> empsalary.withColumn("avg", avg('salary) over byDepName).show
+---------+-----+------+-----------------+
|  depName|empNo|salary|              avg|
+---------+-----+------+-----------------+
|  develop|    7|  4200|           5020.0|
|  develop|    8|  6000|           5020.0|
|  develop|    9|  4500|           5020.0|
|  develop|   10|  5200|           5020.0|
|  develop|   11|  5200|           5020.0|
|    sales|    1|  5000|4866.666666666667|
|    sales|    3|  4800|4866.666666666667|
|    sales|    4|  4800|4866.666666666667|
|personnel|    2|  3900|           3700.0|
|personnel|    5|  3500|           3700.0|
+---------+-----+------+-----------------+
```

You describe a window using the convenient factory methods in <<Window-object, Window object>> that create a <<window-functions/WindowSpec.md#, window specification>> that you can further refine with *partitioning*, *ordering*, and *frame boundaries*.

After you describe a window you can apply <<functions, window aggregate functions>> like _ranking_ functions (e.g. `RANK`), _analytic_ functions (e.g. `LAG`), and the regular [aggregate functions](basic-aggregation/index.md), e.g. `sum`, `avg`, `max`.

NOTE: Window functions are supported in structured queries using <<sql, SQL>> and [Column](Column.md)-based expressions.

Although similar to [aggregate functions](basic-aggregation/index.md), a window function does not group rows into a single output row and retains their separate identities. A window function can access rows that are linked to the current row.

NOTE: The main difference between window aggregate functions and spark-sql-functions.md#aggregate-functions[aggregate functions] with [grouping operators](basic-aggregation/index.md) is that the former calculate values for every row in a window while the latter gives you at most the number of input rows, one value per group.

TIP: See <<examples, Examples>> section in this document.

You can mark a function _window_ by `OVER` clause after a function in SQL, e.g. `avg(revenue) OVER (...)` or [over method](Column.md#over) on a function in the Dataset API, e.g. `rank().over(...)`.

!!! NOTE
    Window functions belong to [Window functions group](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.functions$) in Spark's Scala API.

=== [[Window-object]] Window object

`Window` object provides functions to define windows (as [WindowSpec](window-functions/WindowSpec.md) instances).

`Window` object lives in `org.apache.spark.sql.expressions` package. Import it to use `Window` functions.

```scala
import org.apache.spark.sql.expressions.Window
```

There are two families of the functions available in `Window` object that create [WindowSpec](window-functions/WindowSpec.md) instance for one or many [Column](Column.md) instances:

* [partitionBy](#partitionBy)
* [orderBy](#orderBy)

==== [[partitionBy]] Partitioning Records -- `partitionBy` Methods

[source, scala]
----
partitionBy(colName: String, colNames: String*): WindowSpec
partitionBy(cols: Column*): WindowSpec
----

`partitionBy` creates an instance of `WindowSpec` with partition expression(s) defined for one or more columns.

[source, scala]
----
// partition records into two groups
// * tokens starting with "h"
// * others
val byHTokens = Window.partitionBy('token startsWith "h")

// count the sum of ids in each group
val result = tokens.select('*, sum('id) over byHTokens as "sum over h tokens").orderBy('id)

scala> .show
+---+-----+-----------------+
| id|token|sum over h tokens|
+---+-----+-----------------+
|  0|hello|                4|
|  1|henry|                4|
|  2|  and|                2|
|  3|harry|                4|
+---+-----+-----------------+
----

==== [[orderBy]] Ordering in Windows -- `orderBy` Methods

[source, scala]
----
orderBy(colName: String, colNames: String*): WindowSpec
orderBy(cols: Column*): WindowSpec
----

`orderBy` allows you to control the order of records in a window.

[source, scala]
----
import org.apache.spark.sql.expressions.Window
val byDepnameSalaryDesc = Window.partitionBy('depname).orderBy('salary desc)

// a numerical rank within the current row's partition for each distinct ORDER BY value
scala> val rankByDepname = rank().over(byDepnameSalaryDesc)
rankByDepname: org.apache.spark.sql.Column = RANK() OVER (PARTITION BY depname ORDER BY salary DESC UnspecifiedFrame)

scala> empsalary.select('*, rankByDepname as 'rank).show
+---------+-----+------+----+
|  depName|empNo|salary|rank|
+---------+-----+------+----+
|  develop|    8|  6000|   1|
|  develop|   10|  5200|   2|
|  develop|   11|  5200|   2|
|  develop|    9|  4500|   4|
|  develop|    7|  4200|   5|
|    sales|    1|  5000|   1|
|    sales|    3|  4800|   2|
|    sales|    4|  4800|   2|
|personnel|    2|  3900|   1|
|personnel|    5|  3500|   2|
+---------+-----+------+----+
----

==== [[rangeBetween]] `rangeBetween` Method

[source, scala]
----
rangeBetween(start: Long, end: Long): WindowSpec
----

`rangeBetween` creates a <<window-functions/WindowSpec.md#, WindowSpec>> with the frame boundaries from `start` (inclusive) to `end` (inclusive).

NOTE: It is recommended to use `Window.unboundedPreceding`, `Window.unboundedFollowing` and `Window.currentRow` to describe the frame boundaries when a frame is unbounded preceding, unbounded following and at current row, respectively.

[source, scala]
----
import org.apache.spark.sql.expressions.Window
import org.apache.spark.sql.expressions.WindowSpec
val spec: WindowSpec = Window.rangeBetween(Window.unboundedPreceding, Window.currentRow)
----

Internally, `rangeBetween` creates a `WindowSpec` with `SpecifiedWindowFrame` and `RangeFrame` type.

=== [[frame]] Frame

At its core, a window function calculates a return value for every input row of a table based on a group of rows, called the *frame*. Every input row can have a unique frame associated with it.

When you define a frame you have to specify three components of a frame specification - the *start and end boundaries*, and the *type*.

Types of boundaries (two positions and three offsets):

* `UNBOUNDED PRECEDING` - the first row of the partition
* `UNBOUNDED FOLLOWING` - the last row of the partition
* `CURRENT ROW`
* `<value> PRECEDING`
* `<value> FOLLOWING`

Offsets specify the offset from the current input row.

Types of frames:

* `ROW` - based on _physical offsets_ from the position of the current input row
* `RANGE` - based on _logical offsets_ from the position of the current input row

In the current implementation of <<window-functions/WindowSpec.md#, WindowSpec>> you can use two methods to define a frame:

* `rowsBetween`
* `rangeBetween`

See <<window-functions/WindowSpec.md#, WindowSpec>> for their coverage.

=== [[sql]] Window Operators in SQL Queries

The grammar of windows operators in SQL accepts the following:

1. `CLUSTER BY` or `PARTITION BY` or `DISTRIBUTE BY` for partitions,

2. `ORDER BY` or `SORT BY` for sorting order,

3. `RANGE`, `ROWS`, `RANGE BETWEEN`, and `ROWS BETWEEN` for window frame types,

4. `UNBOUNDED PRECEDING`, `UNBOUNDED FOLLOWING`, `CURRENT ROW` for frame bounds.

TIP: Consult sql/AstBuilder.md#withWindows[withWindows] helper in `AstBuilder`.

=== [[examples]] Examples

==== [[example-top-n]] Top N per Group

Top N per Group is useful when you need to compute the first and second best-sellers in category.

NOTE: This example is borrowed from an _excellent_ article  https://databricks.com/blog/2015/07/15/introducing-window-functions-in-spark-sql.html[Introducing Window Functions in Spark SQL].

.Table PRODUCT_REVENUE
[align="center",width="80%",options="header"]
|===
|product |category |revenue
|      Thin|cell phone|   6000
|    Normal|    tablet|   1500
|      Mini|    tablet|   5500
|Ultra thin|cell phone|   5000
| Very thin|cell phone|   6000
|       Big|    tablet|   2500
|  Bendable|cell phone|   3000
|  Foldable|cell phone|   3000
|       Pro|    tablet|   4500
|      Pro2|    tablet|   6500
|===

Question: What are the best-selling and the second best-selling products in every category?

```
val dataset = Seq(
  ("Thin",       "cell phone", 6000),
  ("Normal",     "tablet",     1500),
  ("Mini",       "tablet",     5500),
  ("Ultra thin", "cell phone", 5000),
  ("Very thin",  "cell phone", 6000),
  ("Big",        "tablet",     2500),
  ("Bendable",   "cell phone", 3000),
  ("Foldable",   "cell phone", 3000),
  ("Pro",        "tablet",     4500),
  ("Pro2",       "tablet",     6500))
  .toDF("product", "category", "revenue")

scala> dataset.show
+----------+----------+-------+
|   product|  category|revenue|
+----------+----------+-------+
|      Thin|cell phone|   6000|
|    Normal|    tablet|   1500|
|      Mini|    tablet|   5500|
|Ultra thin|cell phone|   5000|
| Very thin|cell phone|   6000|
|       Big|    tablet|   2500|
|  Bendable|cell phone|   3000|
|  Foldable|cell phone|   3000|
|       Pro|    tablet|   4500|
|      Pro2|    tablet|   6500|
+----------+----------+-------+

scala> data.where('category === "tablet").show
+-------+--------+-------+
|product|category|revenue|
+-------+--------+-------+
| Normal|  tablet|   1500|
|   Mini|  tablet|   5500|
|    Big|  tablet|   2500|
|    Pro|  tablet|   4500|
|   Pro2|  tablet|   6500|
+-------+--------+-------+
```

The question boils down to ranking products in a category based on their revenue, and to pick the best selling and the second best-selling products based the ranking.

```
import org.apache.spark.sql.expressions.Window
val overCategory = Window.partitionBy('category).orderBy('revenue.desc)

val ranked = data.withColumn("rank", dense_rank.over(overCategory))

scala> ranked.show
+----------+----------+-------+----+
|   product|  category|revenue|rank|
+----------+----------+-------+----+
|      Pro2|    tablet|   6500|   1|
|      Mini|    tablet|   5500|   2|
|       Pro|    tablet|   4500|   3|
|       Big|    tablet|   2500|   4|
|    Normal|    tablet|   1500|   5|
|      Thin|cell phone|   6000|   1|
| Very thin|cell phone|   6000|   1|
|Ultra thin|cell phone|   5000|   2|
|  Bendable|cell phone|   3000|   3|
|  Foldable|cell phone|   3000|   3|
+----------+----------+-------+----+

scala> ranked.where('rank <= 2).show
+----------+----------+-------+----+
|   product|  category|revenue|rank|
+----------+----------+-------+----+
|      Pro2|    tablet|   6500|   1|
|      Mini|    tablet|   5500|   2|
|      Thin|cell phone|   6000|   1|
| Very thin|cell phone|   6000|   1|
|Ultra thin|cell phone|   5000|   2|
+----------+----------+-------+----+
```

==== Revenue Difference per Category

NOTE: This example is the 2nd example from an _excellent_ article  https://databricks.com/blog/2015/07/15/introducing-window-functions-in-spark-sql.html[Introducing Window Functions in Spark SQL].

```
import org.apache.spark.sql.expressions.Window
val reveDesc = Window.partitionBy('category).orderBy('revenue.desc)
val reveDiff = max('revenue).over(reveDesc) - 'revenue

scala> data.select('*, reveDiff as 'revenue_diff).show
+----------+----------+-------+------------+
|   product|  category|revenue|revenue_diff|
+----------+----------+-------+------------+
|      Pro2|    tablet|   6500|           0|
|      Mini|    tablet|   5500|        1000|
|       Pro|    tablet|   4500|        2000|
|       Big|    tablet|   2500|        4000|
|    Normal|    tablet|   1500|        5000|
|      Thin|cell phone|   6000|           0|
| Very thin|cell phone|   6000|           0|
|Ultra thin|cell phone|   5000|        1000|
|  Bendable|cell phone|   3000|        3000|
|  Foldable|cell phone|   3000|        3000|
+----------+----------+-------+------------+
```

==== Difference on Column

Compute a difference between values in rows in a column.

```
val pairs = for {
  x <- 1 to 5
  y <- 1 to 2
} yield (x, 10 * x * y)
val ds = pairs.toDF("ns", "tens")

scala> ds.show
+---+----+
| ns|tens|
+---+----+
|  1|  10|
|  1|  20|
|  2|  20|
|  2|  40|
|  3|  30|
|  3|  60|
|  4|  40|
|  4|  80|
|  5|  50|
|  5| 100|
+---+----+

import org.apache.spark.sql.expressions.Window
val overNs = Window.partitionBy('ns).orderBy('tens)
val diff = lead('tens, 1).over(overNs)

scala> ds.withColumn("diff", diff - 'tens).show
+---+----+----+
| ns|tens|diff|
+---+----+----+
|  1|  10|  10|
|  1|  20|null|
|  3|  30|  30|
|  3|  60|null|
|  5|  50|  50|
|  5| 100|null|
|  4|  40|  40|
|  4|  80|null|
|  2|  20|  20|
|  2|  40|null|
+---+----+----+
```

Please note that http://stackoverflow.com/a/32379437/1305344[Why do Window functions fail with "Window function X does not take a frame specification"?]

The key here is to remember that DataFrames are RDDs under the covers and hence aggregation like grouping by a key in DataFrames is RDD's `groupBy` (or worse, `reduceByKey` or `aggregateByKey` transformations).

==== [[example-running-total]] Running Total

The *running total* is the sum of all previous lines including the current one.

[source, scala]
----
val sales = Seq(
  (0, 0, 0, 5),
  (1, 0, 1, 3),
  (2, 0, 2, 1),
  (3, 1, 0, 2),
  (4, 2, 0, 8),
  (5, 2, 2, 8))
  .toDF("id", "orderID", "prodID", "orderQty")

scala> sales.show
+---+-------+------+--------+
| id|orderID|prodID|orderQty|
+---+-------+------+--------+
|  0|      0|     0|       5|
|  1|      0|     1|       3|
|  2|      0|     2|       1|
|  3|      1|     0|       2|
|  4|      2|     0|       8|
|  5|      2|     2|       8|
+---+-------+------+--------+

val orderedByID = Window.orderBy('id)

val totalQty = sum('orderQty).over(orderedByID).as('running_total)
val salesTotalQty = sales.select('*, totalQty).orderBy('id)

scala> salesTotalQty.show
16/04/10 23:01:52 WARN Window: No Partition Defined for Window operation! Moving all data to a single partition, this can cause serious performance degradation.
+---+-------+------+--------+-------------+
| id|orderID|prodID|orderQty|running_total|
+---+-------+------+--------+-------------+
|  0|      0|     0|       5|            5|
|  1|      0|     1|       3|            8|
|  2|      0|     2|       1|            9|
|  3|      1|     0|       2|           11|
|  4|      2|     0|       8|           19|
|  5|      2|     2|       8|           27|
+---+-------+------+--------+-------------+

val byOrderId = orderedByID.partitionBy('orderID)
val totalQtyPerOrder = sum('orderQty).over(byOrderId).as('running_total_per_order)
val salesTotalQtyPerOrder = sales.select('*, totalQtyPerOrder).orderBy('id)

scala> salesTotalQtyPerOrder.show
+---+-------+------+--------+-----------------------+
| id|orderID|prodID|orderQty|running_total_per_order|
+---+-------+------+--------+-----------------------+
|  0|      0|     0|       5|                      5|
|  1|      0|     1|       3|                      8|
|  2|      0|     2|       1|                      9|
|  3|      1|     0|       2|                      2|
|  4|      2|     0|       8|                      8|
|  5|      2|     2|       8|                     16|
+---+-------+------+--------+-----------------------+
----

==== [[example-rank]] Calculate rank of row

See <<explain-windows, "Explaining" Query Plans of Windows>> for an elaborate example.

=== Interval data type for Date and Timestamp types

See https://issues.apache.org/jira/browse/SPARK-8943[[SPARK-8943\] CalendarIntervalType for time intervals].

With the Interval data type, you could use intervals as values specified in `<value> PRECEDING` and `<value> FOLLOWING` for `RANGE` frame. It is specifically suited for time-series analysis with window functions.

==== Accessing values of earlier rows

FIXME What's the value of rows before current one?

==== [[example-moving-average]] Moving Average

==== [[example-cumulative-aggregates]] Cumulative Aggregates

Eg. cumulative sum

=== User-defined aggregate functions

See https://issues.apache.org/jira/browse/SPARK-3947[[SPARK-3947\] Support Scala/Java UDAF].

With the window function support, you could use user-defined aggregate functions as window functions.

=== [[explain-windows]] "Explaining" Query Plans of Windows

```
import org.apache.spark.sql.expressions.Window
val byDepnameSalaryDesc = Window.partitionBy('depname).orderBy('salary desc)

scala> val rankByDepname = rank().over(byDepnameSalaryDesc)
rankByDepname: org.apache.spark.sql.Column = RANK() OVER (PARTITION BY depname ORDER BY salary DESC UnspecifiedFrame)

// empsalary defined at the top of the page
scala> empsalary.select('*, rankByDepname as 'rank).explain(extended = true)
== Parsed Logical Plan ==
'Project [*, rank() windowspecdefinition('depname, 'salary DESC, UnspecifiedFrame) AS rank#9]
+- LocalRelation [depName#5, empNo#6L, salary#7L]

== Analyzed Logical Plan ==
depName: string, empNo: bigint, salary: bigint, rank: int
Project [depName#5, empNo#6L, salary#7L, rank#9]
+- Project [depName#5, empNo#6L, salary#7L, rank#9, rank#9]
   +- Window [rank(salary#7L) windowspecdefinition(depname#5, salary#7L DESC, ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS rank#9], [depname#5], [salary#7L DESC]
      +- Project [depName#5, empNo#6L, salary#7L]
         +- LocalRelation [depName#5, empNo#6L, salary#7L]

== Optimized Logical Plan ==
Window [rank(salary#7L) windowspecdefinition(depname#5, salary#7L DESC, ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS rank#9], [depname#5], [salary#7L DESC]
+- LocalRelation [depName#5, empNo#6L, salary#7L]

== Physical Plan ==
Window [rank(salary#7L) windowspecdefinition(depname#5, salary#7L DESC, ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS rank#9], [depname#5], [salary#7L DESC]
+- *Sort [depname#5 ASC, salary#7L DESC], false, 0
   +- Exchange hashpartitioning(depname#5, 200)
      +- LocalTableScan [depName#5, empNo#6L, salary#7L]
```

=== [[lag]] `lag` Window Function

[source, scala]
----
lag(e: Column, offset: Int): Column
lag(columnName: String, offset: Int): Column
lag(columnName: String, offset: Int, defaultValue: Any): Column
lag(e: Column, offset: Int, defaultValue: Any): Column
----

`lag` returns the value in `e` / `columnName` column that is `offset` records before the current record. `lag` returns `null` value if the number of records in a window partition is less than `offset` or `defaultValue`.

[source, scala]
----
val buckets = spark.range(9).withColumn("bucket", 'id % 3)
// Make duplicates
val dataset = buckets.union(buckets)

import org.apache.spark.sql.expressions.Window
val windowSpec = Window.partitionBy('bucket).orderBy('id)
scala> dataset.withColumn("lag", lag('id, 1) over windowSpec).show
+---+------+----+
| id|bucket| lag|
+---+------+----+
|  0|     0|null|
|  3|     0|   0|
|  6|     0|   3|
|  1|     1|null|
|  4|     1|   1|
|  7|     1|   4|
|  2|     2|null|
|  5|     2|   2|
|  8|     2|   5|
+---+------+----+

scala> dataset.withColumn("lag", lag('id, 2, "<default_value>") over windowSpec).show
+---+------+----+
| id|bucket| lag|
+---+------+----+
|  0|     0|null|
|  3|     0|null|
|  6|     0|   0|
|  1|     1|null|
|  4|     1|null|
|  7|     1|   1|
|  2|     2|null|
|  5|     2|null|
|  8|     2|   2|
+---+------+----+
----

CAUTION: FIXME It looks like `lag` with a default value has a bug -- the default value's not used at all.

=== [[lead]] `lead` Window Function

[source, scala]
----
lead(columnName: String, offset: Int): Column
lead(e: Column, offset: Int): Column
lead(columnName: String, offset: Int, defaultValue: Any): Column
lead(e: Column, offset: Int, defaultValue: Any): Column
----

`lead` returns the value that is `offset` records after the current records, and `defaultValue` if there is less than `offset` records after the current record. `lag` returns `null` value if the number of records in a window partition is less than `offset` or `defaultValue`.

[source, scala]
----
val buckets = spark.range(9).withColumn("bucket", 'id % 3)
// Make duplicates
val dataset = buckets.union(buckets)

import org.apache.spark.sql.expressions.Window
val windowSpec = Window.partitionBy('bucket).orderBy('id)
scala> dataset.withColumn("lead", lead('id, 1) over windowSpec).show
+---+------+----+
| id|bucket|lead|
+---+------+----+
|  0|     0|   0|
|  0|     0|   3|
|  3|     0|   3|
|  3|     0|   6|
|  6|     0|   6|
|  6|     0|null|
|  1|     1|   1|
|  1|     1|   4|
|  4|     1|   4|
|  4|     1|   7|
|  7|     1|   7|
|  7|     1|null|
|  2|     2|   2|
|  2|     2|   5|
|  5|     2|   5|
|  5|     2|   8|
|  8|     2|   8|
|  8|     2|null|
+---+------+----+

scala> dataset.withColumn("lead", lead('id, 2, "<default_value>") over windowSpec).show
+---+------+----+
| id|bucket|lead|
+---+------+----+
|  0|     0|   3|
|  0|     0|   3|
|  3|     0|   6|
|  3|     0|   6|
|  6|     0|null|
|  6|     0|null|
|  1|     1|   4|
|  1|     1|   4|
|  4|     1|   7|
|  4|     1|   7|
|  7|     1|null|
|  7|     1|null|
|  2|     2|   5|
|  2|     2|   5|
|  5|     2|   8|
|  5|     2|   8|
|  8|     2|null|
|  8|     2|null|
+---+------+----+
----

CAUTION: FIXME It looks like `lead` with a default value has a bug -- the default value's not used at all.

=== [[cume_dist]] Cumulative Distribution of Records Across Window Partitions -- `cume_dist` Window Function

[source, scala]
----
cume_dist(): Column
----

`cume_dist` computes the cumulative distribution of the records in window partitions. This is equivalent to SQL's `CUME_DIST` function.

[source, scala]
----
val buckets = spark.range(9).withColumn("bucket", 'id % 3)
// Make duplicates
val dataset = buckets.union(buckets)

import org.apache.spark.sql.expressions.Window
val windowSpec = Window.partitionBy('bucket).orderBy('id)
scala> dataset.withColumn("cume_dist", cume_dist over windowSpec).show
+---+------+------------------+
| id|bucket|         cume_dist|
+---+------+------------------+
|  0|     0|0.3333333333333333|
|  3|     0|0.6666666666666666|
|  6|     0|               1.0|
|  1|     1|0.3333333333333333|
|  4|     1|0.6666666666666666|
|  7|     1|               1.0|
|  2|     2|0.3333333333333333|
|  5|     2|0.6666666666666666|
|  8|     2|               1.0|
+---+------+------------------+
----

=== [[row_number]] Sequential numbering per window partition -- `row_number` Window Function

[source, scala]
----
row_number(): Column
----

`row_number` returns a sequential number starting at `1` within a window partition.

[source, scala]
----
val buckets = spark.range(9).withColumn("bucket", 'id % 3)
// Make duplicates
val dataset = buckets.union(buckets)

import org.apache.spark.sql.expressions.Window
val windowSpec = Window.partitionBy('bucket).orderBy('id)
scala> dataset.withColumn("row_number", row_number() over windowSpec).show
+---+------+----------+
| id|bucket|row_number|
+---+------+----------+
|  0|     0|         1|
|  0|     0|         2|
|  3|     0|         3|
|  3|     0|         4|
|  6|     0|         5|
|  6|     0|         6|
|  1|     1|         1|
|  1|     1|         2|
|  4|     1|         3|
|  4|     1|         4|
|  7|     1|         5|
|  7|     1|         6|
|  2|     2|         1|
|  2|     2|         2|
|  5|     2|         3|
|  5|     2|         4|
|  8|     2|         5|
|  8|     2|         6|
+---+------+----------+
----

=== [[ntile]] `ntile` Window Function

[source, scala]
----
ntile(n: Int): Column
----

`ntile` computes the ntile group id (from `1` to `n` inclusive) in an ordered window partition.

[source, scala]
----
val dataset = spark.range(7).select('*, 'id % 3 as "bucket")

import org.apache.spark.sql.expressions.Window
val byBuckets = Window.partitionBy('bucket).orderBy('id)
scala> dataset.select('*, ntile(3) over byBuckets as "ntile").show
+---+------+-----+
| id|bucket|ntile|
+---+------+-----+
|  0|     0|    1|
|  3|     0|    2|
|  6|     0|    3|
|  1|     1|    1|
|  4|     1|    2|
|  2|     2|    1|
|  5|     2|    2|
+---+------+-----+
----

CAUTION: FIXME How is `ntile` different from `rank`? What about performance?

=== [[rank]][[dense_rank]][[percent_rank]] Ranking Records per Window Partition -- `rank` Window Function

[source, scala]
----
rank(): Column
dense_rank(): Column
percent_rank(): Column
----

`rank` functions assign the sequential rank of each distinct value per window partition. They are equivalent to `RANK`, `DENSE_RANK` and `PERCENT_RANK` functions in the good ol' SQL.

[source, scala]
----
val dataset = spark.range(9).withColumn("bucket", 'id % 3)

import org.apache.spark.sql.expressions.Window
val byBucket = Window.partitionBy('bucket).orderBy('id)

scala> dataset.withColumn("rank", rank over byBucket).show
+---+------+----+
| id|bucket|rank|
+---+------+----+
|  0|     0|   1|
|  3|     0|   2|
|  6|     0|   3|
|  1|     1|   1|
|  4|     1|   2|
|  7|     1|   3|
|  2|     2|   1|
|  5|     2|   2|
|  8|     2|   3|
+---+------+----+

scala> dataset.withColumn("percent_rank", percent_rank over byBucket).show
+---+------+------------+
| id|bucket|percent_rank|
+---+------+------------+
|  0|     0|         0.0|
|  3|     0|         0.5|
|  6|     0|         1.0|
|  1|     1|         0.0|
|  4|     1|         0.5|
|  7|     1|         1.0|
|  2|     2|         0.0|
|  5|     2|         0.5|
|  8|     2|         1.0|
+---+------+------------+
----

`rank` function assigns the same rank for duplicate rows with a gap in the sequence (similarly to Olympic medal places). `dense_rank` is like `rank` for duplicate rows but compacts the ranks and removes the gaps.

[source, scala]
----
// rank function with duplicates
// Note the missing/sparse ranks, i.e. 2 and 4
scala> dataset.union(dataset).withColumn("rank", rank over byBucket).show
+---+------+----+
| id|bucket|rank|
+---+------+----+
|  0|     0|   1|
|  0|     0|   1|
|  3|     0|   3|
|  3|     0|   3|
|  6|     0|   5|
|  6|     0|   5|
|  1|     1|   1|
|  1|     1|   1|
|  4|     1|   3|
|  4|     1|   3|
|  7|     1|   5|
|  7|     1|   5|
|  2|     2|   1|
|  2|     2|   1|
|  5|     2|   3|
|  5|     2|   3|
|  8|     2|   5|
|  8|     2|   5|
+---+------+----+

// dense_rank function with duplicates
// Note that the missing ranks are now filled in
scala> dataset.union(dataset).withColumn("dense_rank", dense_rank over byBucket).show
+---+------+----------+
| id|bucket|dense_rank|
+---+------+----------+
|  0|     0|         1|
|  0|     0|         1|
|  3|     0|         2|
|  3|     0|         2|
|  6|     0|         3|
|  6|     0|         3|
|  1|     1|         1|
|  1|     1|         1|
|  4|     1|         2|
|  4|     1|         2|
|  7|     1|         3|
|  7|     1|         3|
|  2|     2|         1|
|  2|     2|         1|
|  5|     2|         2|
|  5|     2|         2|
|  8|     2|         3|
|  8|     2|         3|
+---+------+----------+

// percent_rank function with duplicates
scala> dataset.union(dataset).withColumn("percent_rank", percent_rank over byBucket).show
+---+------+------------+
| id|bucket|percent_rank|
+---+------+------------+
|  0|     0|         0.0|
|  0|     0|         0.0|
|  3|     0|         0.4|
|  3|     0|         0.4|
|  6|     0|         0.8|
|  6|     0|         0.8|
|  1|     1|         0.0|
|  1|     1|         0.0|
|  4|     1|         0.4|
|  4|     1|         0.4|
|  7|     1|         0.8|
|  7|     1|         0.8|
|  2|     2|         0.0|
|  2|     2|         0.0|
|  5|     2|         0.4|
|  5|     2|         0.4|
|  8|     2|         0.8|
|  8|     2|         0.8|
+---+------+------------+
----

=== [[currentRow]] `currentRow` Window Function

[source, scala]
----
currentRow(): Column
----

`currentRow`...FIXME

=== [[unboundedFollowing]] `unboundedFollowing` Window Function

[source, scala]
----
unboundedFollowing(): Column
----

`unboundedFollowing`...FIXME

=== [[unboundedPreceding]] `unboundedPreceding` Window Function

[source, scala]
----
unboundedPreceding(): Column
----

`unboundedPreceding`...FIXME

=== [[i-want-more]] Further Reading and Watching

* https://databricks.com/blog/2015/07/15/introducing-window-functions-in-spark-sql.html[Introducing Window Functions in Spark SQL]
* http://www.postgresql.org/docs/current/static/tutorial-window.html[3.5. Window Functions] in the official documentation of PostgreSQL
* https://www.simple-talk.com/sql/t-sql-programming/window-functions-in-sql/[Window Functions in SQL]
* https://www.simple-talk.com/sql/learn-sql-server/working-with-window-functions-in-sql-server/[Working with Window Functions in SQL Server]
* https://msdn.microsoft.com/en-CA/library/ms189461.aspx[OVER Clause (Transact-SQL)]
* https://sqlsunday.com/2013/03/31/windowed-functions/[An introduction to windowed functions]
* https://blog.jooq.org/2013/11/03/probably-the-coolest-sql-feature-window-functions/[Probably the Coolest SQL Feature: Window Functions]
* https://sqlschool.modeanalytics.com/advanced/window-functions/[Window Functions]
