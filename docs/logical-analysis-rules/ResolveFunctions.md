# ResolveFunctions Logical Resolution Rule

`ResolveFunctions` is a logical resolution rule that the [logical query plan analyzer](../Analyzer.md#ResolveFunctions) uses to [resolve the following logical operators and expressions](#apply) in a logical query plan:

Logical Operator | Expression
-----------------|-----------
 `UnresolvedFunctionName` | [UnresolvedFunction](../expressions/UnresolvedFunction.md)
 [UnresolvedTableValuedFunction](#UnresolvedTableValuedFunction) | [UnresolvedGenerator](../expressions/UnresolvedGenerator.md)
 `UnresolvedTVFAliases`

`ResolveReferences` is a [Catalyst rule](../catalyst/Rule.md) for transforming [logical plans](../logical-operators/LogicalPlan.md) (`Rule[LogicalPlan]`).

`ResolveFunctions` is part of [Resolution](../Analyzer.md#Resolution) fixed-point batch of rules.

## <span id="apply"> Executing Rule

??? note "Signature"

    ```scala
    apply(
      plan: LogicalPlan): LogicalPlan
    ```

    `apply` is part of the [Rule](../catalyst/Rule.md#apply) abstraction.

---

`apply` resolves logical operators up (post-order, bottom-up) in trees with the following [TreePattern](../catalyst/TreePattern.md)s:

* `GENERATOR`
* `UNRESOLVED_FUNC`
* `UNRESOLVED_FUNCTION`
* `UNRESOLVED_TABLE_VALUED_FUNCTION`
* `UNRESOLVED_TVF_ALIASES`

### <span id="apply-UnresolvedFunctionName"> UnresolvedFunctionName

For an `UnresolvedFunctionName`, `apply`...FIXME

### <span id="apply-UnresolvedTableValuedFunction"> UnresolvedTableValuedFunction

For an [UnresolvedTableValuedFunction](../logical-operators/UnresolvedTableValuedFunction.md), `apply` first attempts to [resolveBuiltinOrTempTableFunction](#resolveBuiltinOrTempTableFunction), if available.

Otherwise, `apply` attempts to resolve a [CatalogPlugin](../connector/catalog/CatalogPlugin.md) and, for the [spark_catalog](../connector/catalog/CatalogV2Util.md#isSessionCatalog) only, requests the [SessionCatalog](../Analyzer.md#v1SessionCatalog) to [resolvePersistentTableFunction](../SessionCatalog.md#resolvePersistentTableFunction).

### <span id="apply-others"> Others

`apply`...FIXME

## <span id="resolveBuiltinOrTempTableFunction"> resolveBuiltinOrTempTableFunction

```scala
resolveBuiltinOrTempTableFunction(
  name: Seq[String],
  arguments: Seq[Expression]): Option[LogicalPlan]
```

Only for a one-element `name`, `resolveBuiltinOrTempTableFunction` requests the [SessionCatalog](../Analyzer.md#v1SessionCatalog) to [resolveBuiltinOrTempTableFunction](../SessionCatalog.md#resolveBuiltinOrTempTableFunction).

Otherwise, `resolveBuiltinOrTempTableFunction` returns `None` (_an undefined value_).

<!---
## Review Me

[[example]]
[source, scala]
----
import spark.sessionState.analyzer.ResolveFunctions

// Example: UnresolvedAttribute with VirtualColumn.hiveGroupingIdName (grouping__id) => Alias
import org.apache.spark.sql.catalyst.expressions.VirtualColumn
import org.apache.spark.sql.catalyst.analysis.UnresolvedAttribute
val groupingIdAttr = UnresolvedAttribute(VirtualColumn.hiveGroupingIdName)
scala> println(groupingIdAttr.sql)
`grouping__id`

// Using Catalyst DSL to create a logical plan with grouping__id
import org.apache.spark.sql.catalyst.dsl.plans._
val t1 = table("t1")
val plan = t1.select(groupingIdAttr)
scala> println(plan.numberedTreeString)
00 'Project ['grouping__id]
01 +- 'UnresolvedRelation `t1`

val resolvedPlan = ResolveFunctions(plan)
scala> println(resolvedPlan.numberedTreeString)
00 'Project [grouping_id() AS grouping__id#0]
01 +- 'UnresolvedRelation `t1`

import org.apache.spark.sql.catalyst.expressions.Alias
val alias = resolvedPlan.expressions.head.asInstanceOf[Alias]
scala> println(alias.sql)
grouping_id() AS `grouping__id`

// Example: UnresolvedGenerator => a) Generator or b) analysis failure
// Register a function so a function resolution works
import org.apache.spark.sql.catalyst.FunctionIdentifier
import org.apache.spark.sql.catalyst.catalog.CatalogFunction
val f1 = CatalogFunction(FunctionIdentifier(funcName = "f1"), "java.lang.String", resources = Nil)
import org.apache.spark.sql.catalyst.expressions.{Expression, Stack}
// FIXME What happens when looking up a function with the functionBuilder None in registerFunction?
// Using Stack as ResolveFunctions requires that the function to be resolved is a Generator
// You could roll your own, but that's a demo, isn't it? (don't get too carried away)
spark.sessionState.catalog.registerFunction(
  funcDefinition = f1,
  overrideIfExists = true,
  functionBuilder = Some((children: Seq[Expression]) => Stack(children = Nil)))

import org.apache.spark.sql.catalyst.analysis.UnresolvedGenerator
import org.apache.spark.sql.catalyst.FunctionIdentifier
val ungen = UnresolvedGenerator(name = FunctionIdentifier("f1"), children = Seq.empty)
val plan = t1.select(ungen)
scala> println(plan.numberedTreeString)
00 'Project [unresolvedalias('f1(), None)]
01 +- 'UnresolvedRelation `t1`

val resolvedPlan = ResolveFunctions(plan)
scala> println(resolvedPlan.numberedTreeString)
00 'Project [unresolvedalias(stack(), None)]
01 +- 'UnresolvedRelation `t1`

CAUTION: FIXME

// Example: UnresolvedFunction => a) AggregateWindowFunction with(out) isDistinct, b) AggregateFunction, c) other with(out) isDistinct
val plan = ???
val resolvedPlan = ResolveFunctions(plan)
----

=== [[apply]] Resolving grouping__id UnresolvedAttribute, UnresolvedGenerator and UnresolvedFunction Expressions In Entire Query Plan (Applying ResolveFunctions to Logical Plan) -- `apply` Method

[source, scala]
----
apply(plan: LogicalPlan): LogicalPlan
----

NOTE: `apply` is part of catalyst/Rule.md#apply[Rule Contract] to apply a rule to a spark-sql-LogicalPlan.md[logical plan].

`apply` takes a spark-sql-LogicalPlan.md[logical plan] and transforms each expression (for every logical operator found in the query plan) as follows:

* For spark-sql-Expression-UnresolvedAttribute.md[UnresolvedAttributes] with spark-sql-Expression-UnresolvedAttribute.md#name[names] as `grouping__id`, `apply` creates a spark-sql-Expression-Alias.md#creating-instance[Alias] (with a `GroupingID` child expression and `grouping__id` name).
+
That case seems mostly for compatibility with Hive as `grouping__id` attribute name is used by Hive.

* For spark-sql-Expression-UnresolvedGenerator.md[UnresolvedGenerators], `apply` requests the [SessionCatalog](../Analyzer.md#catalog) to [find a Generator function by name](../SessionCatalog.md#lookupFunction).
+
If some other non-generator function is found for the name, `apply` fails the analysis phase by reporting an `AnalysisException`:
+
```
[name] is expected to be a generator. However, its class is [className], which is not a generator.
```

* For spark-sql-Expression-UnresolvedFunction.md[UnresolvedFunctions], `apply` requests the [SessionCatalog](../Analyzer.md#catalog) to [find a function by name](../SessionCatalog.md#lookupFunction).

* expressions/AggregateWindowFunction.md[AggregateWindowFunctions] are returned directly or `apply` fails the analysis phase by reporting an `AnalysisException` when the `UnresolvedFunction` has spark-sql-Expression-UnresolvedFunction.md#isDistinct[isDistinct] flag enabled.
+
```
[name] does not support the modifier DISTINCT
```

* spark-sql-Expression-AggregateFunction.md[AggregateFunctions] are wrapped in a [AggregateExpression](../expressions/AggregateExpression.md) (with `Complete` aggregate mode)

* All other functions are returned directly or `apply` fails the analysis phase by reporting an `AnalysisException` when the `UnresolvedFunction` has spark-sql-Expression-UnresolvedFunction.md#isDistinct[isDistinct] flag enabled.
+
```
[name] does not support the modifier DISTINCT
```

`apply` skips expressions/Expression.md#childrenResolved[unresolved expressions].
-->
