---
id: 2134
title: 'Syncing bulk data into SQL Server using DataTable, SQLBulkCopy &#038; SQL MERGE'
date: 2019-07-27T18:45:53+10:00
author: eddiewould
layout: post
guid: https://eddiewould.com/?p=2134
permalink: /2019/07/27/syncing-bulk-data-into-sql-server-using-datatable-sqlbulkcopy-sql-merge/
ocean_gallery_link_images:
  - 'off'
ocean_sidebar:
  - "0"
ocean_second_sidebar:
  - "0"
ocean_disable_margins:
  - enable
ocean_display_top_bar:
  - default
ocean_display_header:
  - default
ocean_center_header_left_menu:
  - "0"
ocean_custom_header_template:
  - "0"
ocean_header_custom_menu:
  - "0"
ocean_menu_typo_font_family:
  - "0"
ocean_disable_title:
  - default
ocean_disable_heading:
  - default
ocean_disable_breadcrumbs:
  - default
ocean_display_footer_widgets:
  - default
ocean_display_footer_bottom:
  - default
ocean_custom_footer_template:
  - "0"
ocean_link_format_target:
  - self
ocean_quote_format_link:
  - post
categories:
  - Uncategorized
tags:
  - ETL sync data SQL DataTable Merge
---

A recipe for efficient bulk updates in SQL Server using DataTable, staging tables & ANSI merge.

These days (for the most part) I'm an advocate of using a (micro) ORM for most *CRUD* operations - delving into SQL usually isn't justified. That said, with clients I've had over the years I've seen a recurrent need for a "bulk data sync" process - essentially take a feed of data from an external system and "merge" it into a table in our database. Any records that are matched should be updated, any records that don't exist should be created and (optionally) any records that exist but no-longer exist in the external source should be deleted. The requirements are usually that it handles large volumes of data efficently (1M+ rows is not out of the question) and runs atomically (either everything is updated or nothing is).

This post gives a recipe for taking a DataTable, loading it into a (temporary) staging table then generating & executing an SQL merge statement to "make it so" (update/insert into the destination table from the staging table).

## Components

<a href="https://en.wikipedia.org/wiki/Merge_(SQL)">SQL Merge</a> (part of the ANSI SQL standard) is basically an "upsert" - i.e. match two sets of records (source & target) on some condition(s) and - when matched - update the corresponding target row, otherwise (when unmatched i.e. exists in source but not target) insert into the target table. SQL Server extends the syntax to allow specifying an action to take for rows that exist in the target but don't exist in the source (e.g. delete).

<a href="https://docs.microsoft.com/en-us/dotnet/api/system.data.sqlclient.sqlbulkcopy?view=netframework-4.8">SQLBulkCopy</a> (System.Data.SqlClient) is a mechanism for efficently loading data into a table in SQL Server from .NET - either from a DataTable, an array of DataRows or an IDataReader.

<a href="https://docs.microsoft.com/en-us/dotnet/api/system.data.datatable?view=netframework-4.8">DataTable</a> represents an in-memory table of data - i.e. columns (name + type) and a set of rows (values). As an aside, this class is extremely useful for writing unit testable code - many libraries/frameworks allow reading from/writing to a DataTable so you can have the business logic operate on the data table, unit test that (inspect rows/columns) and then be relatively confident that the data will be read/written correctly.

## Approach

The interface (for calling code) is quite simple - you pass in a connection string, a table of data to upsert and the name of the field on which to merge; you get back a result indicating the number of records that were updated and the number that were inserted.

```csharp
public class MergeResult
{
	public int UpdatedCount { get; set; }
	public int InsertedCount { get; set; }
}

public interface IRepository
{
	Task<MergeResult> MergeRecordsAsync(string connectionString, DataTable dataTable, string mergeOnField);
}
```

We drive the column names from the DataTable - in particular, this code will only affect columns that exist in the supplied DataTable (the data in any other columns will not be affected).

```csharp
var columnNames = new HashSet<string>(dataTable.Columns.Cast<DataColumn>().Select(c => c.ColumnName));
```

The SQL MERGE needs to read data from a (source) table in SQL Server - we make use of a temporary table to avoid having to deal with cleaning it up & other such concerns. Temporary tables must be named beginning with the '#' character and live until the connection is closed. We execute the statement below to create our temporary table:

```sql
SELECT TOP (0) {columnNamesList} INTO {stagingTableName} FROM {tableName}
```

The bulk copy is relatively straight-forward - set the destination table name, tell it to copy all columns and then write the data table.

```csharp
using (var bulkCopy = new SqlBulkCopy(connection))
{
	bulkCopy.DestinationTableName = stagingTableName;
	foreach (var columnName in columnNames)
	{
		bulkCopy.ColumnMappings.Add(columnName, columnName);
	}

	await bulkCopy.WriteToServerAsync(dataTable);
}
```

The generation of the MERGE statement is probably the interesting bit. It starts with a template string:

```sql
DECLARE @SummaryOfChanges TABLE(Change VARCHAR(20));

MERGE INTO {TARGET_TABLE_NAME} WITH (HOLDLOCK) AS target
USING (SELECT {FIELD_LIST} FROM {SOURCE_TABLE_NAME}) as source
ON (target.[{MERGE_FIELD_NAME}] = source.[{MERGE_FIELD_NAME}])
WHEN MATCHED AND ({UPDATE_REQUIRED_EXPRESSION}) THEN
	UPDATE SET {UPDATES_LIST}
WHEN NOT MATCHED THEN
	INSERT ({FIELD_LIST}) VALUES ({SOURCE_FIELD_LIST})
OUTPUT $action INTO @SummaryOfChanges;

SELECT Change, COUNT(1) AS CountPerChange
FROM @SummaryOfChanges
GROUP BY Change;
```

The variables in the template surrounded by _{ }_ will be populated at execution time. Essentially though, we're telling SQL Server to 

* Match the rows in _TARGET_TABLE_NAME_ against those in _SOURCE_TABLE_NAME_ on the equality of a single field (_MERGE_FIELD_NAME_).
* When the rows are matched *and an update is actually required*, perform the required UPDATE (_set target.column1 = source.column1, target.column2 = source.column2_ etc)
* When a matching source row is _not_ found in the target, INSERT a row into _TARGET_TABLE_NAME_
* OUTPUT the $action ( 'INSERT' / 'UPDATE') into a table variable
* Count the number of times each action occurred (i.e. number of inserts/updates).

We populate the template using simple string replacement (extension function not shown here) as follows:

```csharp
var mergeSql = MergeSqlTemplate.FormatFromValues(new
{
	MERGE_FIELD_NAME = mergeOnField,
	FIELD_LIST = columnNamesList,
	SOURCE_TABLE_NAME = stagingTableName,
	TARGET_TABLE_NAME = tableName,
	UPDATES_LIST = string.Join(", ", columnNames.Select(name => $"target.[{name}] = source.[{name}]")),
	SOURCE_FIELD_LIST = string.Join(", ", columnNames.Select(name => $"source.[{name}]")),
	UPDATE_REQUIRED_EXPRESSION = string.Join(" OR ", columnNames.Select(name => $"IIF((target.[{name}] IS NULL AND source.[{name}] IS NULL) OR target.[{name}] = source.[{name}], 1, 0) = 0")) // Take care around null values
});
```

It's mostly pretty self-explanatory - a little bit of string joining etc. One part that's worth drawing attention to is how we populate _UPDATE_REQUIRED_EXPRESSION_ - this expression is used to test whether the row *actually needs updating*. The reason we do that is to 1) avoid unnecessary writes & 2) ensure the "updated" count is accurate - if we didn't, it would tell us that every row that was matched was updated (even if the data was already up-to-date).

An update is required *if at least one column needs updating*. A column needs updating if the target column value is not the same as the source column value. We might naively simply test _target.someColumn != source.someColumn_ however with ANSI null handling this would not handle null values correctly. I've therefore used IFF to return a value of 1 if the values are equal (taking null into account) and return a boolean value of TRUE (row needs updating) if the IFF statement returns 0 (values are not equal). Note that we could actually simplify this approach by using a conjunction (AND) rather than a disjunction (OR), avoiding the need for IFF.

After generating the merge statement, all that's left to do is to _execute_ the statement and read the results back:

```csharp
var mergeCommand = new SqlCommand(mergeSql, connection);
var reader = await mergeCommand.ExecuteReaderAsync();

var result = new MergeResult();

if (reader.HasRows)
{
	while (reader.Read())
	{
		var actionType = reader.GetString(0);
		var count = reader.GetInt32(1);

		switch (actionType)
		{
			case "UPDATE":
				result.UpdatedCount = count;
				break;
			case "INSERT":
				result.InsertedCount = count;
				break;
		}
	}
}

return result;
```

Note: This implementation never deletes from the target table (only ever inserts/updates). It could trivially be modified to do so by adding a _WHEN NOT MATCHED BY SOURCE_ clause to the _MERGE_ statement which deletes the row from the target table - and update the statistics accordingly. If you're doing that then you'd want to ensure the input DataTable always contains the full current data set.