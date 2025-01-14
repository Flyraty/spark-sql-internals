# In

`In` is a [predicate expression](Expression.md#Predicate).

`In` is <<creating-instance, created>> when:

* [Column.isin](../spark-sql-column-operators.md#isin) operator is used

* `AstBuilder` is requested to [parse SQL's IN predicate with a subquery](../sql/AstBuilder.md#withPredicate)

[TIP]
====
Use Catalyst DSL's [in](../catalyst-dsl/index.md#in) operator to create an `In` expression.

[source, scala]
----
in(list: Expression*): Expression
----
====

[source, scala]
----
// Using Catalyst DSL to create an In expression
import org.apache.spark.sql.catalyst.dsl.expressions._

// HACK: Disable symbolToColumn implicit conversion
// It is imported automatically in spark-shell (and makes demos impossible)
// implicit def symbolToColumn(s: Symbol): org.apache.spark.sql.ColumnName
trait ThatWasABadIdea
implicit def symbolToColumn(ack: ThatWasABadIdea) = ack

val value = 'a.long
import org.apache.spark.sql.catalyst.expressions.{Expression, Literal}
import org.apache.spark.sql.types.StringType
val list: Seq[Expression] = Seq(1, Literal.create(null, StringType), true)

val e = value in (list: _*)

scala> :type e
org.apache.spark.sql.catalyst.expressions.Expression

scala> println(e.dataType)
BooleanType

scala> println(e.sql)
(`a` IN (1, CAST(NULL AS STRING), true))
----

`In` expression can be <<eval, evaluated>> to a boolean value (i.e. `true` or `false`) or the special value `null`.

[source, scala]
----
import org.apache.spark.sql.functions.lit
val value = lit(null)
val list = Seq(lit(1))
val in = (value isin (list: _*)).expr

scala> println(in.sql)
(NULL IN (1))

import org.apache.spark.sql.catalyst.InternalRow
val input = InternalRow(1, "hello")

// Case 1: value.eval(input) was null => null
val evaluatedValue = in.eval(input)
assert(evaluatedValue == null)

// Case 2: v = e.eval(input) && ordering.equiv(v, evaluatedValue) => true
val value = lit(1)
val list = Seq(lit(1))
val in = (value isin (list: _*)).expr
val evaluatedValue = in.eval(input)
assert(evaluatedValue.asInstanceOf[Boolean])

// Case 3: e.eval(input) = null and no ordering.equiv(v, evaluatedValue) => null
val value = lit(1)
val list = Seq(lit(null), lit(2))
val in = (value isin (list: _*)).expr
scala> println(in.sql)
(1 IN (NULL, 2))

val evaluatedValue = in.eval(input)
assert(evaluatedValue == null)

// Case 4: false
val value = lit(1)
val list = Seq(0, 2, 3).map(lit)
val in = (value isin (list: _*)).expr
scala> println(in.sql)
(1 IN (0, 2, 3))

val evaluatedValue = in.eval(input)
assert(evaluatedValue.asInstanceOf[Boolean] == false)
----

[[creating-instance]]
`In` takes the following when created:

* [[value]] Value Expression.md[expression]
* [[list]] Expression.md[Expression] list

NOTE: <<list, Expression list>> must not be `null` (but can have expressions that can be evaluated to `null`).

[[toString]]
`In` uses the following *text representation* (i.e. `toString`):

```
[value] IN [list]
```

[source, scala]
----
import org.apache.spark.sql.catalyst.expressions.{In, Literal}
import org.apache.spark.sql.{functions => f}
val in = In(value = Literal(1), list = Seq(f.array("1", "2", "3").expr))
scala> println(in)
1 IN (array('1, '2, '3))
----

[[sql]]
`In` has the following Expression.md#sql[SQL representation]:

```
([valueSQL] IN ([listSQL]))
```

[source, scala]
----
import org.apache.spark.sql.catalyst.expressions.{In, Literal}
import org.apache.spark.sql.{functions => f}
val in = In(value = Literal(1), list = Seq(f.array("1", "2", "3").expr))
scala> println(in.sql)
(1 IN (array(`1`, `2`, `3`)))
----

[[inSetConvertible]]
`In` expression is *inSetConvertible* when the <<list, list>> contains spark-sql-Expression-Literal.md[Literal] expressions only.

[source, scala]
----
// FIXME Example 1: inSetConvertible true

// FIXME Example 2: inSetConvertible false
----

`In` expressions are analyzed using the following rules:

* [ResolveSubquery](../logical-analysis-rules/ResolveSubquery.md) resolution rule

* `InConversion` type coercion rule

[[InMemoryTableScanExec]]
`In` expression has a InMemoryTableScanExec.md#buildFilter-expressions[custom support] in InMemoryTableScanExec.md[InMemoryTableScanExec] physical operator.

[source, scala]
----
// FIXME
// Demo: InMemoryTableScanExec and In expression
// 1. Create an In(a: AttributeReference, list: Seq[Literal]) with the list.nonEmpty
// 2. Use InMemoryTableScanExec.buildFilter partial function to produce the expression
----

[[internal-registries]]
.In's Internal Properties (e.g. Registries, Counters and Flags)
[cols="1,2",options="header",width="100%"]
|===
| Name
| Description

| [[ordering]] `ordering`
| Scala's https://www.scala-lang.org/api/2.11.12/index.html#scala.math.Ordering[Ordering] instance that represents a strategy for sorting instances of a type.

Lazily-instantiated using `TypeUtils.getInterpretedOrdering` for the Expression.md#dataType[data type] of the <<value, value>> expression.

Used exclusively when `In` is requested to <<eval, evaluate a value>> for a given input row.
|===

=== [[checkInputDataTypes]] Checking Input Data Types -- `checkInputDataTypes` Method

[source, scala]
----
checkInputDataTypes(): TypeCheckResult
----

NOTE: `checkInputDataTypes` is part of the <<Expression.md#checkInputDataTypes, Expression Contract>> to checks the input data types.

`checkInputDataTypes`...FIXME

=== [[eval]] Evaluating Expression -- `eval` Method

[source, scala]
----
eval(input: InternalRow): Any
----

`eval` is part of the [Expression](Expression.md#eval) abstraction.

`eval` requests <<value, value>> expression to Expression.md#eval[evaluate a value] for the given [InternalRow](../InternalRow.md).

If the evaluated value is `null`, `eval` gives `null` too.

`eval` takes every Expression.md[expression] in <<list, list>> expressions and requests them to evaluate a value for the `input` internal row. If any of the evaluated value is not `null` and equivalent in the <<ordering, ordering>>, `eval` returns `true`.

`eval` records whether any of the expressions in <<list, list>> expressions gave `null` value. If no <<list, list>> expression led to `true` (per <<ordering, ordering>>), `eval` returns `null` if any <<list, list>> expression evaluated to `null` or `false`.

=== [[doGenCode]] Generating Java Source Code (ExprCode) For Code-Generated Expression Evaluation -- `doGenCode` Method

[source, scala]
----
doGenCode(ctx: CodegenContext, ev: ExprCode): ExprCode
----

NOTE: `doGenCode` is part of <<Expression.md#doGenCode, Expression Contract>> to generate a Java source code (ExprCode) for code-generated expression evaluation.

`doGenCode`...FIXME

```text
val in = $"id" isin (1, 2, 3)
val q = spark.range(4).filter(in)
val plan = q.queryExecution.executedPlan

import org.apache.spark.sql.execution.FilterExec
val filterExec = plan.collectFirst { case f: FilterExec => f }.get

import org.apache.spark.sql.catalyst.expressions.In
val inExpr = filterExec.expressions.head.asInstanceOf[In]

import org.apache.spark.sql.execution.WholeStageCodegenExec
val wsce = plan.asInstanceOf[WholeStageCodegenExec]
val (ctx, code) = wsce.doCodeGen

import org.apache.spark.sql.catalyst.expressions.codegen.CodeFormatter
scala> println(CodeFormatter.format(code))
...code omitted

// FIXME Make it work
// I thought I'd reuse ctx to have expression: id#14L evaluated
inExpr.genCode(ctx)
```
