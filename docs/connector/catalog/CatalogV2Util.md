# CatalogV2Util Utility

## <span id="getTableProviderCatalog"> getTableProviderCatalog

```scala
getTableProviderCatalog(
  provider: SupportsCatalogOptions,
  catalogManager: CatalogManager,
  options: CaseInsensitiveStringMap): TableCatalog
```

`getTableProviderCatalog`...FIXME

`getTableProviderCatalog` is used when:

* `DataFrameReader` is requested to [load](../../DataFrameReader.md#load) (for a data source that is a [SupportsCatalogOptions])

* `DataFrameWriter` is requested to [save](../../DataFrameWriter.md#save) (for a data source that is a [SupportsCatalogOptions])

## <span id="loadTable"> Loading Table

```scala
loadTable(
  catalog: CatalogPlugin,
  ident: Identifier): Option[Table]
```

`loadTable` requests the given [CatalogPlugin](CatalogPlugin.md) (as a [TableCatalog](TableCatalog.md)) to [load the table](TableCatalog.md#loadTable) by the `Identifier`.

`loadTable` returns the [Table](../Table.md) if available or `None`.

`loadTable` is used when...FIXME

## <span id="createAlterTable"> Creating AlterTable Logical Command

```scala
createAlterTable(
  originalNameParts: Seq[String],
  catalog: CatalogPlugin,
  tableName: Seq[String],
  changes: Seq[TableChange]): AlterTable
```

`createAlterTable` converts the [CatalogPlugin](CatalogPlugin.md) to a [TableCatalog](CatalogHelper.md#asTableCatalog).

`createAlterTable` creates an [AlterTable](../../logical-operators/AlterTable.md) (with an `UnresolvedV2Relation`).

`createAlterTable` is used when:

* [ResolveCatalogs](../../logical-analysis-rules/ResolveCatalogs.md) and [ResolveSessionCatalog](../../logical-analysis-rules/ResolveSessionCatalog.md) logical resolution rules are executed (and resolve `AlterTableAddColumnsStatement`, `AlterTableReplaceColumnsStatement`, `AlterTableAlterColumnStatement`, `AlterTableRenameColumnStatement`, `AlterTableDropColumnsStatement`, `AlterTableSetPropertiesStatement`, `AlterTableUnsetPropertiesStatement`, `AlterTableSetLocationStatement` operators)
