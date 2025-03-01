# Column

[[creating-instance]]
[[expr]]
`Column` represents a column in a Dataset.md[Dataset] that holds a Catalyst expressions/Expression.md[Expression] that produces a value per row.

NOTE: A `Column` is a value generator for every row in a `Dataset`.

[[star]]
A special column `*` references all columns in a `Dataset`.

With the SparkSession.md#implicits[implicits] converstions imported, you can create "free" column references using Scala's symbols.

[source, scala]
----
val spark: SparkSession = ...
import spark.implicits._

import org.apache.spark.sql.Column
scala> val nameCol: Column = 'name
nameCol: org.apache.spark.sql.Column = name
----

NOTE: _"Free" column references_ are ``Column``s with no association to a `Dataset`.

You can also create free column references from ``$``-prefixed strings.

[source, scala]
----
// Note that $ alone creates a ColumnName
scala> val idCol = $"id"
idCol: org.apache.spark.sql.ColumnName = id

import org.apache.spark.sql.Column

// The target type triggers the implicit conversion to Column
scala> val idCol: Column = $"id"
idCol: org.apache.spark.sql.Column = id
----

Beside using the `implicits` conversions, you can create columns using spark-sql-functions.md#col[col] and spark-sql-functions.md#column[column] functions.

[source, scala]
----
import org.apache.spark.sql.functions._

scala> val nameCol = col("name")
nameCol: org.apache.spark.sql.Column = name

scala> val cityCol = column("city")
cityCol: org.apache.spark.sql.Column = city
----

Finally, you can create a bound `Column` using the `Dataset` the column is supposed to be part of using Dataset.md#apply[Dataset.apply] factory method or Dataset.md#col[Dataset.col] operator.

NOTE: You can use bound `Column` references only with the ``Dataset``s they have been created from.

[source, scala]
----
scala> val textCol = dataset.col("text")
textCol: org.apache.spark.sql.Column = text

scala> val idCol = dataset.apply("id")
idCol: org.apache.spark.sql.Column = id

scala> val idCol = dataset("id")
idCol: org.apache.spark.sql.Column = id
----

You can reference nested columns using `.` (dot).

[[operators]]
.Column Operators
[cols="1,3",options="header",width="100%"]
|===
| Operator
| Description

| <<as, as>>
| Specifying type hint about the expected return value of the column

| <<name, name>>
|
|===

[NOTE]
====
`Column` has a reference to Catalyst's expressions/Expression.md[Expression] it was created for using `expr` method.

[source, scala]
----
scala> window('time, "5 seconds").expr
res0: org.apache.spark.sql.catalyst.expressions.Expression = timewindow('time, 5000000, 5000000, 0) AS window#1
----
====

!!! TIP
    Read about typed column references in [TypedColumn](TypedColumn.md).

=== [[as]] Specifying Type Hint -- `as` Operator

```scala
as[U : Encoder]: TypedColumn[Any, U]
```

`as` creates a [TypedColumn](TypedColumn.md) (that gives a type hint about the expected return value of the column).

[source, scala]
----
scala> $"id".as[Int]
res1: org.apache.spark.sql.TypedColumn[Any,Int] = id
----

=== [[name]] `name` Operator

[source, scala]
----
name(alias: String): Column
----

`name`...FIXME

NOTE: `name` is used when...FIXME

=== [[withColumn]] Adding Column to Dataset -- `withColumn` Method

[source, scala]
----
withColumn(colName: String, col: Column): DataFrame
----

`withColumn` method returns a new `DataFrame` with the new column `col` with `colName` name added.

NOTE: `withColumn` can replace an existing `colName` column.

[source, scala]
----
scala> val df = Seq((1, "jeden"), (2, "dwa")).toDF("number", "polish")
df: org.apache.spark.sql.DataFrame = [number: int, polish: string]

scala> df.show
+------+------+
|number|polish|
+------+------+
|     1| jeden|
|     2|   dwa|
+------+------+

scala> df.withColumn("polish", lit(1)).show
+------+------+
|number|polish|
+------+------+
|     1|     1|
|     2|     1|
+------+------+
----

You can add new columns do a `Dataset` using Dataset.md#withColumn[withColumn] method.

[source, scala]
----
val spark: SparkSession = ...
val dataset = spark.range(5)

// Add a new column called "group"
scala> dataset.withColumn("group", 'id % 2).show
+---+-----+
| id|group|
+---+-----+
|  0|    0|
|  1|    1|
|  2|    0|
|  3|    1|
|  4|    0|
+---+-----+
----

=== [[apply]] Creating Column Instance For Catalyst Expression -- `apply` Factory Method

[source, scala]
----
val spark: SparkSession = ...
case class Word(id: Long, text: String)
val dataset = Seq(Word(0, "hello"), Word(1, "spark")).toDS

scala> val idCol = dataset.apply("id")
idCol: org.apache.spark.sql.Column = id

// or using Scala's magic a little bit
// the following is equivalent to the above explicit apply call
scala> val idCol = dataset("id")
idCol: org.apache.spark.sql.Column = id
----

=== [[like]] `like` Operator

CAUTION: FIXME

[source, scala]
----
scala> df("id") like "0"
res0: org.apache.spark.sql.Column = id LIKE 0

scala> df.filter('id like "0").show
+---+-----+
| id| text|
+---+-----+
|  0|hello|
+---+-----+
----

=== [[symbols-as-column-names]] Symbols As Column Names

[source, scala]
----
scala> val df = Seq((0, "hello"), (1, "world")).toDF("id", "text")
df: org.apache.spark.sql.DataFrame = [id: int, text: string]

scala> df.select('id)
res0: org.apache.spark.sql.DataFrame = [id: int]

scala> df.select('id).show
+---+
| id|
+---+
|  0|
|  1|
+---+
----

=== [[over]] Defining Windowing Column (Analytic Clause) -- `over` Operator

[source, scala]
----
over(): Column
over(window: WindowSpec): Column
----

`over` creates a _windowing column_ (_aka_ _analytic clause_) that allows to execute a spark-sql-functions.md[aggregate function] over a [window](window-functions/WindowSpec.md) (i.e. a group of records that are in _some_ relation to the current record).

TIP: Read up on windowed aggregation in Spark SQL in spark-sql-functions-windows.md[Window Aggregate Functions].

[source, scala]
----
scala> val overUnspecifiedFrame = $"someColumn".over()
overUnspecifiedFrame: org.apache.spark.sql.Column = someColumn OVER (UnspecifiedFrame)

import org.apache.spark.sql.expressions.Window
import org.apache.spark.sql.expressions.WindowSpec
val spec: WindowSpec = Window.rangeBetween(Window.unboundedPreceding, Window.currentRow)
scala> val overRange = $"someColumn" over spec
overRange: org.apache.spark.sql.Column = someColumn OVER (RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
----

=== [[cast]] `cast` Operator

`cast` method casts a column to a data type. It makes for type-safe maps with [Row](Row.md) objects of the proper type (not `Any`).

[source,scala]
----
cast(to: String): Column
cast(to: DataType): Column
----

`cast` uses spark-sql-CatalystSqlParser.md[CatalystSqlParser] to parse the data type from its canonical string representation.

==== [[cast-example]] cast Example

[source, scala]
----
scala> val df = Seq((0f, "hello")).toDF("label", "text")
df: org.apache.spark.sql.DataFrame = [label: float, text: string]

scala> df.printSchema
root
 |-- label: float (nullable = false)
 |-- text: string (nullable = true)

// without cast
import org.apache.spark.sql.Row
scala> df.select("label").map { case Row(label) => label.getClass.getName }.show(false)
+---------------+
|value          |
+---------------+
|java.lang.Float|
+---------------+

// with cast
import org.apache.spark.sql.types.DoubleType
scala> df.select(col("label").cast(DoubleType)).map { case Row(label) => label.getClass.getName }.show(false)
+----------------+
|value           |
+----------------+
|java.lang.Double|
+----------------+
----

=== [[generateAlias]] `generateAlias` Method

[source, scala]
----
generateAlias(e: Expression): String
----

`generateAlias`...FIXME

`generateAlias` is used when:

* `Column` is requested to <<named, named>>
* `RelationalGroupedDataset` is requested to [alias](basic-aggregation/RelationalGroupedDataset.md#alias)

=== [[named]] `named` Method

[source, scala]
----
named: NamedExpression
----

`named`...FIXME

`named` is used when the following operators are used:

* [Dataset.select](spark-sql-dataset-operators.md#select)
* [KeyValueGroupedDataset.agg](basic-aggregation/KeyValueGroupedDataset.md#agg)
