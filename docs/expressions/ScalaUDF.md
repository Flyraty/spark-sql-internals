# ScalaUDF

`ScalaUDF` is an [Expression](Expression.md) to manage the lifecycle of a [user-defined function](#function) (and hook it to Catalyst execution path).

`ScalaUDF` is a `NonSQLExpression` (i.e. has no representation in SQL).

`ScalaUDF` is a [UserDefinedExpression](UserDefinedExpression.md)

## Creating Instance

`ScalaUDF` takes the following to be created:

* <span id="function"> Function
* <span id="dataType"> [DataType](../types/DataType.md)
* <span id="children"> Child [Expression](Expression.md)s
* <span id="inputEncoders"> Input [ExpressionEncoder](../ExpressionEncoder.md)s
* <span id="outputEncoder"> Output [ExpressionEncoder](../ExpressionEncoder.md)
* <span id="udfName"> Name (optional)
* <span id="nullable"> `nullable` flag (default: `true`)
* <span id="udfDeterministic"> `udfDeterministic` flag (default: `true`)

`ScalaUDF` is created when:

* `UDFRegistration` is requested to [register a UDF](../UDFRegistration.md#register)
* `BaseDynamicPartitionDataWriter` is requested for `partitionPathExpression`
* `SparkUserDefinedFunction` is requested to `createScalaUDF`
* [HandleNullInputsForUDF](../logical-analysis-rules/HandleNullInputsForUDF.md) and `ResolveEncodersInUDF` logical evaluation rules are executed
* `ImplicitTypeCasts` utility is used for `transform` function

## <span id="deterministic"> deterministic

```scala
deterministic: Boolean
```

`ScalaUDF` is `deterministic` when all the following hold:

1. [udfDeterministic](#udfDeterministic) is enabled
1. All the [children](#children) are [deterministic](Expression.md#deterministic)

`deterministic` is part of the [Expression](Expression.md#deterministic) abstraction.

## <span id="toString"> Text Representation

```scala
toString: String
```

`toString` uses the [name](#name) and the [children] for the text representation:

```text
[name]([comma-separated children])
```

`toString` is part of the [TreeNode](../catalyst/TreeNode.md#toString) abstraction.

## <span id="name"> Name

```scala
name: String
```

`name` is the [udfName](#udfName) (if defined) or `UDF`.

`name` is part of the [UserDefinedExpression](UserDefinedExpression.md#name) abstraction.

## <span id="doGenCode"> Code-Generated Expression Evaluation

```scala
doGenCode(
  ctx: CodegenContext,
  ev: ExprCode): ExprCode
```

`doGenCode`...FIXME

`doGenCode` is part of the [Expression](Expression.md#doGenCode) abstraction.

## <span id="eval"> Interpreted Expression Evaluation

```scala
eval(
  input: InternalRow): Any
```

`eval`...FIXME

`eval` is part of the [Expression](Expression.md#eval) abstraction.

## <span id="nodePatterns"> Node Patterns

```scala
nodePatterns: Seq[TreePattern]
```

`nodePatterns` is [SCALA_UDF](../catalyst/TreePattern.md#SCALA_UDF).

`nodePatterns` is part of the [TreeNode](../catalyst/TreeNode.md#nodePatterns) abstraction.

## Analysis

[Logical Analyzer](../Analyzer.md) uses [HandleNullInputsForUDF](../logical-analysis-rules/HandleNullInputsForUDF.md) and `ResolveEncodersInUDF` logical evaluation rules to analyze queries with `ScalaUDF` expressions.

## Demo

Let's define a zero-argument UDF.

```scala
val myUDF = udf { () => "Hello World" }
```

```scala
// "Execute" the UDF
// Attach it to an "execution environment", i.e. a Dataset
// by specifying zero columns to execute on (since the UDF is no-arg)
import org.apache.spark.sql.catalyst.expressions.ScalaUDF
val scalaUDF = myUDF().expr.asInstanceOf[ScalaUDF]

assert(scalaUDF.resolved)
```

Let's execute the UDF (on every row in a `Dataset`).
We simulate it relying on the `EmptyRow` that is the default `InternalRow` of `eval`.

```text
scala> scalaUDF.eval()
res2: Any = Hello World
```

Let's define a UDF of one argument.

```scala
val lengthUDF = udf { s: String => s.length }.withName("lengthUDF")
val c = lengthUDF($"name")
```

```text
scala> println(c.expr.treeString)
UDF:lengthUDF('name)
+- 'name
```

```scala
import org.apache.spark.sql.catalyst.expressions.ScalaUDF
assert(c.expr.isInstanceOf[ScalaUDF])
```

Let's define another UDF of one argument.

```text
val hello = udf { s: String => s"Hello $s" }

// Binding the hello UDF to a column name
import org.apache.spark.sql.catalyst.expressions.ScalaUDF
val helloScalaUDF = hello($"name").expr.asInstanceOf[ScalaUDF]

assert(helloScalaUDF.resolved == false)
```

```text
// Resolve helloScalaUDF, i.e. the only `name` column reference

scala> helloScalaUDF.children
res4: Seq[org.apache.spark.sql.catalyst.expressions.Expression] = ArrayBuffer('name)
```

```scala
// The column is free (i.e. not bound to a Dataset)
// Define a Dataset that becomes the rows for the UDF
val names = Seq("Jacek", "Agata").toDF("name")
```

```text
scala> println(names.queryExecution.analyzed.numberedTreeString)
00 Project [value#1 AS name#3]
01 +- LocalRelation [value#1]
```

Resolve the references using the `Dataset`.

```scala
val plan = names.queryExecution.analyzed
val resolver = spark.sessionState.analyzer.resolver
import org.apache.spark.sql.catalyst.analysis.UnresolvedAttribute
val resolvedUDF = helloScalaUDF.transformUp { case a @ UnresolvedAttribute(names) =>
  // we're in controlled environment
  // so get is safe
  plan.resolve(names, resolver).get
}
assert(resolvedUDF.resolved)
```

```text
scala> println(resolvedUDF.numberedTreeString)
00 UDF(name#3)
01 +- name#3: string
```

```text
import org.apache.spark.sql.catalyst.expressions.BindReferences
val attrs = names.queryExecution.sparkPlan.output
val boundUDF = BindReferences.bindReference(resolvedUDF, attrs)

// Create an internal binary row, i.e. InternalRow
import org.apache.spark.sql.catalyst.encoders.ExpressionEncoder
val stringEncoder = ExpressionEncoder[String]
val row = stringEncoder.toRow("world")
```

Yay! It works!

```text
scala> boundUDF.eval(row)
res8: Any = Hello world
```

Just to show the regular execution path (i.e. how to execute an UDF in a context of a `Dataset`).

```scala
val q = names.select(hello($"name"))
```

```text
scala> q.show
+-----------+
|  UDF(name)|
+-----------+
|Hello Jacek|
|Hello Agata|
+-----------+
```
