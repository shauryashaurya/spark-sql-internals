# Metadata Columns

Spark 3.1.1 ([SPARK-31255](https://issues.apache.org/jira/browse/SPARK-31255)) introduced support for [MetadataColumn](../connector/catalog/MetadataColumn.md)s for additional metadata of a row.

`MetadataColumn`s can be defined for [Table](../connector/Table.md)s with [SupportsMetadataColumns](../connector/SupportsMetadataColumns.md).

Use [DESCRIBE TABLE EXTENDED](../sql/AstBuilder.md#visitDescribeRelation) SQL command to display the metadata columns of a table.

## <span id="METADATA_COL_ATTR_KEY"> __metadata_col { #__metadata_col }

`__metadata_col` is used when:

* `MetadataAttribute` is [created](MetadataAttribute.md#apply) and [destructured](MetadataAttribute.md#unapply)
* `FileSourceMetadataAttribute` is [created](FileSourceMetadataAttribute.md#apply) and requested to [removeInternalMetadata](FileSourceMetadataAttribute.md#removeInternalMetadata)
* `FileSourceConstantMetadataAttribute` is [created](FileSourceConstantMetadataAttribute.md#apply)
* `FileSourceGeneratedMetadataAttribute` is [created](FileSourceGeneratedMetadataAttribute.md#apply)
* `MetadataColumnHelper` is requested to [isMetadataCol](MetadataColumnHelper.md#isMetadataCol) and [markAsQualifiedAccessOnly](MetadataColumnHelper.md#markAsQualifiedAccessOnly)
* `MetadataColumnsHelper` is requested to [asStruct](MetadataColumnsHelper.md#asStruct)

## Logical Operators

Logical operators propagate metadata columns using [metadataOutput](../logical-operators/LogicalPlan.md#metadataOutput).

[ExposesMetadataColumns](../logical-operators/ExposesMetadataColumns.md) logical operators can generate metadata columns.

## <span id="DataSourceV2Relation"> DataSourceV2Relation

`MetadataColumn`s are disregarded (_filtered out_) from the [metadataOutput](../logical-operators/LogicalPlan.md#metadataOutput) in [DataSourceV2Relation](../logical-operators/DataSourceV2Relation.md) leaf logical operator when in name-conflict with output columns.
