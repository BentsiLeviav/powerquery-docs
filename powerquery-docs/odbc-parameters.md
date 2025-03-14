---
title: Parameters for the Odbc.DataSource function
description: Describes the parameters that can be used with the Odbc.Datasource function in Power Query
author: ptyx507x
ms.topic: conceptual
ms.date: 3/1/2022
ms.author: miescobar
ms.subservice: custom-connectors
---

# Parameters for Odbc.DataSource

The [Odbc.DataSource](/powerquery-m/odbc-datasource) function takes two parameters&mdash;a `connectionString` for your driver, and an `options` record that lets you override various driver behaviors. Through the options record you can override capabilities and other information reported by the driver, control the navigator behavior, and affect the SQL queries generated by the M engine.

The supported options records fields fall into two categories&mdash;those that are public and always available, and those that are only available in an extensibility context.

The following table describes the public fields in the options record.

| Field | Description |
| --- | --- |
| `CommandTimeout` | A duration value that controls how long the server-side query is allowed to run before it's canceled.<br/><br/>Default: 10 minutes |
| `ConnectionTimeout` | A duration value that controls how long to wait before abandoning an attempt to make a connection to the server.<br/><br/>Default: 15 seconds |
| `CreateNavigationProperties` | A logical value that sets whether to generate navigation properties on the returned tables. Navigation properties are based on foreign key relationships reported by the driver. These properties show up as “virtual” columns that can be expanded in the query editor, creating the appropriate join.<br/><br/>If calculating foreign key dependencies is an expensive operation for your driver, you may want to set this value to false.<br/><br/>Default: true |
| `HierarchicalNavigation` | A logical value that sets whether to view the tables grouped by their schema names. When set to false, tables are displayed in a flat list under each database.<br/><br/>Default: false |
| `SqlCompatibleWindowsAuth` | A logical value that determines whether to produce a SQL Server compatible connection string when using Windows Authentication&mdash;`Trusted_Connection=Yes`.<br/><br/>If your driver supports Windows Authentication, but requires extra or alternative settings in your connection string, you should set this value to false and use the `CredentialConnectionString` options record field described in the next table.<br/><br/>Default: true |
| | |

The following table describes the options record fields that are only available through extensibility. Fields that aren't simple literal values are described in later sections.

| Field | Description |
| --- | --- |
| `AstVisitor` | A record containing one or more overrides to control SQL query generation. The most common usage of this field is to provide logic to generate a LIMIT/OFFSET clause for drivers that don't support TOP.<br/><br/>Fields include `Constant` and `LimitClause`.<br/><br/>More information: [Overriding AstVisitor](#overriding-astvisitor) |
| `CancelQueryExplicitly` | A logical value that instructs the M engine to explicitly cancel any running calls through the ODBC driver before terminating the connection to the ODBC server.<br/><br/>This field is useful in situations where query execution is managed independently of the network connections to the server, for example in some Spark deployments. In most cases, this value doesn't need to be set because the query in the server is canceled when the network connection to the server is terminated.<br/><br/>Default: false |
| `ClientConnectionPooling` | A logical value that enables client-side connection pooling for the ODBC driver. Most drivers will want to set this value to true.<br/><br/>Default: false |
| `CredentialConnectionString` | A text or record value used to specify credential-related connection string properties.<!--<br/><br/>Go to the Credential section for more information.--> |
| `HideNativeQuery` | A logical value that controls whether or not the connector shows generated SQL statements in the Power Query user experience. This should only be set to true if the back end data source natively supports SQL-92.<br/><br/>Default: false |
| `ImplicitTypeConversions` | A table value containing implicit type conversions supported by your driver or backend server. Values in this table are additive to the conversions reported by the driver itself.<br/><br/>This field is typically used with the `SQLGetTypeInfo` field when overriding data type information reported by the driver.<!--<br/><br/>Go to the ImplicitTypeConversions section for more information.--> |
| `OnError` | An error handling function that receives an `errorRecord` parameter of type `record`.<br/><br/>Common uses of this function include handling SSL connection failures, providing a download link if your driver isn't found on the system, and reporting authentication errors.<!--<br/><br/>Go to the OnError section for more information.--> |
| `SoftNumbers` | Allows the M engine to select a compatible data type when conversion between two specific numeric types isn't declared as supported in the SQL_CONVERT_* capabilities.<br/><br/>Default: false |
| `SqlCapabilities` | A record providing various overrides of driver capabilities, and a way to specify capabilities that aren't expressed through ODBC 3.8.<br/><br/>More information: [Overriding SqlCapabilities](#overriding-sqlcapabilities) |
| `SQLColumns` | A function that allows you to modify column metadata returned by the `SQLColumns` function.<br/><br/>More information: [Overriding SQLColumns](#overriding-sqlcolumns) |
| `SQLGetFunctions` | A record that allows you to override values returned by calls to `SQLGetFunctions`.<br/><br/>A common use of this field is to disable the use of parameter binding, or to specify that generated queries should use CAST rather than CONVERT.<br/><br>More information: [Overriding SQLGetFunctions](#overriding-sqlgetfunctions) |
| `SQLGetInfo` | A record that allows you to override values returned by calls to `SQLGetInfo`.<br/><br/>More information: [Overriding SQLGetInfo](#overriding-sqlgetinfo) |
| `SQLGetTypeInfo` | A table or function that returns a table that overrides the type information returned by `SQLGetTypeInfo`.<br/><br/>When the value is set to a table, the value completely replaces the type information reported by the driver. `SQLGetTypeInfo` won't be called.<br/><br/>When the value is set to a function, your function will receive the result of the original call to `SQLGetTypeInfo`, allowing you to modify the table.<br/><br/>This field is typically used when there's a mismatch between data types reported by `SQLGetTypeInfo` and `SQLColumns`.<br/><br/>More information: [Overriding SQLGetTypeInfo](#overriding-sqlgettypeinfo) |
| `SQLTables` | A function that allows you to modify the table metadata returned by a call to `SQLTables`.<!-- <br/><br/>Go to the SQLTables section for more information. --> |
| <a id="tolerate"></a>`TolerateConcatOverflow` | Allows concatenation of text values to occur even if the result might be truncated to fit within the range of an available type.<br/><br/>For example, when concatenating a VARCHAR(4000) field with a VARCHAR(4000) field on a system that supports a maximize VARCHAR size of 4000 and no CLOB type, the concatenation is folded even though the result might get truncated.<br/><br/>Default: false |
| `UseEmbeddedDriver` | **(internal use):** A logical value that controls whether the ODBC driver should be loaded from a local directory (using new functionality defined in the ODBC 4.0 specification). This value is generally only set by connectors created by Microsoft that ship with Power Query.<br/><br/>When set to false, the system ODBC driver manager is used to locate and load the driver.<br/><br/>Most connectors shouldn't need to set this field.<br/><br/>Default: false |
| | |

## Overriding AstVisitor

The `AstVisitor` field is set through the [Odbc.DataSource](/powerquery-m/odbc-datasource) options record. It's used to modify SQL statements generated for specific query scenarios.

>[!Note]
> Drivers that support LIMIT and OFFSET clauses (rather than TOP) will want to provide a `LimitClause` override for `AstVisitor`.

### Constant

Providing an override for this value has been deprecated and may be removed from future implementations.

### LimitClause

This field is a function that receives two `Int64.Type` arguments (`skip`, `take`), and returns a record with two text fields (`Text`, `Location`).

```
LimitClause = (skip as nullable number, take as number) as record => ...
```

The `skip` parameter is the number of rows to skip (that is, the argument to OFFSET). If an offset isn't specified, the skip value will be null. If your driver supports LIMIT, but doesn't support OFFSET, the `LimitClause` function should return an unimplemented error (...) when skip is greater than 0.

The `take` parameter is the number of rows to take (that is, the argument to LIMIT).

The `Text` field of the result contains the SQL text to add to the generated query.

The `Location` field specifies where to insert the clause. The following table describes supported values.

| Value | Description | Example |
| --- | --- | --- |
| `AfterQuerySpecification` | LIMIT clause is put at the end of the generated SQL.<br/><br/>This is the most commonly supported LIMIT syntax. | SELECT a, b, c<br/><br/>FROM table<br/><br/>WHERE a &gt; 10<br/><br/>**LIMIT 5** |
| `BeforeQuerySpecification` | LIMIT clause is put before the generated SQL statement. | **LIMIT 5 ROWS**<br/><br/>SELECT a, b, c<br/><br/>FROM table<br/><br/>WHERE a &gt; 10 |
| `AfterSelect` | LIMIT goes after the SELECT statement, and after any modifiers (such as DISTINCT). | SELECT DISTINCT **LIMIT 5** a, b, c<br/><br/>FROM table<br/><br/>WHERE a &gt; 10 |
| `AfterSelectBeforeModifiers` | LIMIT goes after the SELECT statement, but before any modifiers (such as DISTINCT). | SELECT **LIMIT 5** DISTINCT a, b, c<br/><br/>FROM table<br/><br/>WHERE a &gt; 10 |
| | | |

The following code snippet provides a LimitClause implementation for a driver that expects a LIMIT clause, with an optional OFFSET, in the following format: `[OFFSET <offset> ROWS] LIMIT <row_count>`

```
LimitClause = (skip, take) =>
    let
        offset = if (skip > 0) then Text.Format("OFFSET #{0} ROWS", {skip}) else "",
        limit = if (take <> null) then Text.Format("LIMIT #{0}", {take}) else ""
    in
        [
            Text = Text.Format("#{0} #{1}", {offset, limit}),
            Location = "AfterQuerySpecification"
        ]
```

The following code snippet provides a `LimitClause` implementation for a driver that supports LIMIT, but not OFFSET. Format: `LIMIT <row_count>`.

```
LimitClause = (skip, take) =>
    if (skip > 0) then error "Skip/Offset not supported"
    else
    [
        Text = Text.Format("LIMIT #{0}", {take}),
        Location = "AfterQuerySpecification"
    ]
```

<!--
## Overriding ImplicitTypeConversions

**TODO**

## Providing an OnError handler

**TODO**
-->

## Overriding SqlCapabilities

| Field | Details |
| --- | --- |
| `FractionalSecondsScale` | A number value ranging from 1 to 7 that indicates the number of decimal places supported for millisecond values. This value should be set by connectors that want to enable query folding over datetime values.<br/><br/>Default: null |
| `PrepareStatements` | A logical value that indicates that statements should be prepared using [SQLPrepare](/sql/odbc/reference/syntax/sqlprepare-function).<br/><br/>Default: false |
| `SupportsTop` | A logical value that indicates the driver supports the TOP clause to limit the number of returned rows.<br/><br/>Default: false |
| `StringLiteralEscapeCharacters` | A list of text values that specify the character(s) to use when escaping string literals and LIKE expressions.<br/><br/>Example: `{""}`<br/><br/>Default: null |
| `SupportsDerivedTable` | A logical value that indicates the driver supports derived tables (sub-selects).<br/><br/>This value is assumed to be true for drivers that set their conformance level to SQL_SC_SQL92_FULL (reported by the driver or overridden with the [Sql92Conformance setting](#conformance). For all other conformance levels, this value defaults to false.<br/><br/>If your driver doesn't report the SQL_SC_SQL92_FULL compliance level, but does support-derived tables, set this value to true.<br/><br/>Supporting derived tables is required for many DirectQuery scenarios. |
| `SupportsNumericLiterals` | A logical value that indicates whether the generated SQL should include numeric literals values. When set to false, numeric values are always specified using parameter binding.<br/><br>Default: false |
| `SupportsStringLiterals` | A logical value that indicates whether the generated SQL should include string literals values. When set to false, string values are always specified using parameter binding.<br/><br/>Default: false |
| `SupportsOdbcDateLiterals` | A logical value that indicates whether the generated SQL should include date literals values. When set to false, date values are always specified using parameter binding.<br/><br/>Default: false |
| `SupportsOdbcTimeLiterals` | A logical value that indicates whether the generated SQL should include time literals values. When set to false, time values are always specified using parameter binding.<br/><br/>Default: false |
| `SupportsOdbcTimestampLiterals` | A logical value that indicates whether the generated SQL should include timestamp literals values. When set to false, timestamp values are always specified using parameter binding.<br/><br/>Default: false |
| | |

## Overriding SQLColumns

`SQLColumns` is a function handler that receives the results of an ODBC call to [SQLColumns](/sql/odbc/reference/syntax/sqlcolumns-function). The source parameter contains a table with the data type information. This override is typically used to fix up data type mismatches between calls to `SQLGetTypeInfo` and `SQLColumns`.

For details of the format of the source table parameter, go to [SQLColumns Function](/sql/odbc/reference/syntax/sqlcolumns-function).

## Overriding SQLGetFunctions

This field is used to override `SQLFunctions` values returned by an ODBC driver. It contains a record whose field names are equal to the `FunctionId` constants defined for the ODBC
[SQLGetFunctions](/sql/odbc/reference/syntax/sqlgetfunctions-function) function. Numeric constants for each of these fields can be found in the [ODBC
specification](https://github.com/Microsoft/ODBC-Specification/blob/master/Windows/inc/sqlext.h).

| Field | Details |
| --- | --- |
| `SQL_CONVERT_FUNCTIONS` | Indicates which function(s) are supported when doing type conversions. By default, the M Engine attempts to use the CONVERT function. Drivers that prefer the use of CAST can override this value to report that only SQL_FN_CVT_CAST (numeric value of 0x2) is supported. |
| `SQL_API_SQLBINDCOL` | A logical (true/false) value that indicates whether the mashup engine should use the [SQLBindCol API](/sql/odbc/reference/syntax/sqlbindcol-function) when retrieving data. When set to false, [SQLGetData](/sql/odbc/reference/syntax/sqlgetdata-function) is used instead.<br/><br/>Default: false |
| | |

The following code snippet provides an example explicitly telling the M engine to use CAST rather than CONVERT.

```
SQLGetFunctions = [
    SQL_CONVERT_FUNCTIONS = 0x2 /* SQL_FN_CVT_CAST */
]
```

## Overriding SQLGetInfo

This field is used to override `SQLGetInfo` values returned by an ODBC driver. It contains a record whose fields are names equal to the `InfoType` constants defined for the ODBC [SQLGetInfo](/sql/odbc/reference/syntax/sqlgetinfo-function) function. Numeric constants for each of these fields can be found in the [ODBC specification](https://github.com/Microsoft/ODBC-Specification/blob/master/Windows/inc/sqlext.h). The full list of `InfoTypes` that are checked can be found in the mashup engine trace files.

The following table contains commonly overridden `SQLGetInfo` properties:

| Field | Details |
| --- | --- |
| <a id="conformance"></a>`SQL_SQL_CONFORMANCE` | An integer value that indicates the level of SQL-92 supported by the driver:<br/><br/>(1) SQL_SC_SQL92_ENTRY: Entry level SQL-92 compliant.<br/>(2) SQL_SC_FIPS127_2_TRANSITIONAL: FIPS 127-2 transitional level compliant.<br/>(4) SQL_SC_ SQL92_INTERMEDIATE" Intermediate level SQL-92 compliant.<br/>(8) SQL_SC_SQL92_FULL: Full level SQL-92 compliant.<br/><br/>In Power Query scenarios, the connector is used in a Read Only mode. Most drivers will want to report a SQL_SC_SQL92_FULL compliance level, and override specific SQL generation behavior using the `SQLGetInfo` and `SQLGetFunctions` properties. |
| `SQL_SQL92_PREDICATES` | A bitmask enumerating the predicates supported in a SELECT statement, as defined in SQL-92.<br/><br/>Go to [SQL_SP_* constants](https://github.com/Microsoft/ODBC-Specification/blob/master/Windows/inc/sqlext.h#L1765) in the ODBC specification. |
| `SQL_AGGREGATE_FUNCTIONS` | A bitmask enumerating support for aggregation functions.<br/><br/>SQL_AF_ALL<br/>SQL_AF_AVG<br/>SQL_AF_COUNT<br/>SQL_AF_DISTINCT<br/>SQL_AF_MAX<br/>SQL_AF_MIN<br/>SQL_AF_SUM<br/><br/>Go to [SQL_AF_* constants](https://github.com/Microsoft/ODBC-Specification/blob/master/Windows/inc/sqlext.h#L1523) in the ODBC specification. |
| `SQL_GROUP_BY` | An integer value that specifies the relationship between the columns in the GROUP BY clause and the non-aggregated columns in the select list:<br/><br/>SQL_GB_COLLATE: A COLLATE clause can be specified at the end of each grouping column.<br/><br/>SQL_GB_NOT_SUPPORTED: GROUP BY clauses aren't supported.<br/><br/>SQL_GB_GROUP_BY_EQUALS_SELECT: The GROUP BY clause must contain all non-aggregated columns in the select list. It can't contain any other columns. For example, SELECT DEPT, MAX(SALARY) FROM EMPLOYEE GROUP BY DEPT.<br/><br/>SQL_GB_GROUP_BY_CONTAINS_SELECT: The GROUP BY clause must contain all non-aggregated columns in the select list. It can contain columns that aren't in the select list. For example, SELECT DEPT, MAX(SALARY) FROM EMPLOYEE GROUP BY DEPT, AGE.<br/><br/>SQL_GB_NO_RELATION: The columns in the GROUP BY clause and the select list aren't related. The meaning of non-grouped, non-aggregated columns in the select list is data source–dependent. For example, SELECT DEPT, SALARY FROM EMPLOYEE GROUP BY DEPT, AGE.<br/><br/>Go to [SQL_GB_* constants](https://github.com/Microsoft/ODBC-Specification/blob/master/Windows/inc/sqlext.h#L1421) in the ODBC specification. |
| | |

The following helper function can be used to create bitmask values from a list of integer values:

```
Flags = (flags as list) =>
    let
        Loop = List.Generate(
                  ()=> [i = 0, Combined = 0],
                  each [i] < List.Count(flags),
                  each [i = [i]+1, Combined =*Number.BitwiseOr([Combined], flags{i})],
                  each [Combined]),
        Result = List.Last(Loop, 0)
    in
        Result;
```

## Overriding SQLGetTypeInfo

`SQLGetTypeInfo` can be specified in two ways:

- A fixed `table` value that contains the same type information as an ODBC call to `SQLGetTypeInfo`.
- A function that accepts a table argument, and returns a table. The argument contains the original results of the ODBC call to `SQLGetTypeInfo`. Your function implementation can modify or add to this table.

The first approach is used to completely override the values returned by the ODBC driver. The second approach is used if you want to add to or modify these values.

For details of the format of the types table parameter and expected return value, go to [SQLGetTypeInfo function reference](/sql/odbc/reference/syntax/sqlgettypeinfo-function).

### SQLGetTypeInfo using a static table

The following code snippet provides a static implementation for `SQLGetTypeInfo`.

```
SQLGetTypeInfo = #table(
    { "TYPE_NAME",      "DATA_TYPE", "COLUMN_SIZE", "LITERAL_PREF", "LITERAL_SUFFIX", "CREATE_PARAS",           "NULLABLE", "CASE_SENSITIVE", "SEARCHABLE", "UNSIGNED_ATTRIBUTE", "FIXED_PREC_SCALE", "AUTO_UNIQUE_VALUE", "LOCAL_TYPE_NAME", "MINIMUM_SCALE", "MAXIMUM_SCALE", "SQL_DATA_TYPE", "SQL_DATETIME_SUB", "NUM_PREC_RADIX", "INTERNAL_PRECISION", "USER_DATA_TYPE" }, {

    { "char",           1,          65535,          "'",            "'",              "max. length",            1,          1,                3,            null,                 0,                  null,                "char",            null,            null,            -8,              null,               null,             0,                    0                }, 
    { "int8",           -5,         19,             "'",            "'",              null,                     1,          0,                2,            0,                    10,                 0,                   "int8",            0,               0,               -5,              null,               2,                0,                    0                },
    { "bit",            -7,         1,              "'",            "'",              null,                     1,          1,                3,            null,                 0,                  null,                "bit",             null,            null,            -7,              null,               null,             0,                    0                },
    { "bool",           -7,         1,              "'",            "'",              null,                     1,          1,                3,            null,                 0,                  null,                "bit",             null,            null,            -7,              null,               null,             0,                    0                },
    { "date",           9,          10,             "'",            "'",              null,                     1,          0,                2,            null,                 0,                  null,                "date",            null,            null,            9,               1,                  null,             0,                    0                }, 
    { "numeric",        3,          28,             null,           null,             null,                     1,          0,                2,            0,                    0,                   0,                  "numeric",         0,               0,               2,               null,               10,               0,                    0                },
    { "float8",         8,          15,             null,           null,             null,                     1,          0,                2,            0,                    0,                   0,                  "float8",          null,            null,            6,               null,               2,                0,                    0                },
    { "float8",         6,          17,             null,           null,             null,                     1,          0,                2,            0,                    0,                   0,                  "float8",          null,            null,            6,               null,               2,                0,                    0                },
    { "uuid",           -11,        37,             null,           null,             null,                     1,          0,                2,            null,                 0,                  null,                "uuid",            null,            null,            -11,             null,               null,             0,                    0                },
    { "int4",           4,          10,             null,           null,             null,                     1,          0,                2,            0,                    0,                   0,                  "int4",            0,               0,               4,               null,               2,                0,                    0                },
    { "text",           -1,         65535,          "'",            "'",              null,                     1,          1,                3,            null,                 0,                  null,                "text",            null,            null,            -10,             null,               null,             0,                    0                },
    { "lo",             -4,         255,            "'",            "'",              null,                     1,          0,                2,            null,                 0,                  null,                "lo",              null,            null,            -4,              null,               null,             0,                    0                }, 
    { "numeric",        2,          28,             null,           null,             "precision, scale",       1,          0,                2,            0,                    10,                 0,                   "numeric",         0,               6,               2,               null,               10,               0,                    0                },
    { "float4",         7,          9,              null,           null,             null,                     1,          0,                2,            0,                    10,                 0,                   "float4",          null,            null,            7,               null,               2,                0,                    0                }, 
    { "int2",           5,          19,             null,           null,             null,                     1,          0,                2,            0,                    10,                 0,                   "int2",            0,               0,               5,               null,               2,                0,                    0                }, 
    { "int2",           -6,         5,              null,           null,             null,                     1,          0,                2,            0,                    10,                 0,                   "int2",            0,               0,               5,               null,               2,                0,                    0                }, 
    { "timestamp",      11,         26,             "'",            "'",              null,                     1,          0,                2,            null,                 0,                  null,                "timestamp",       0,               38,              9,               3,                  null,             0,                    0                }, 
    { "date",           91,         10,             "'",            "'",              null,                     1,          0,                2,            null,                 0,                  null,                "date",            null,            null,            9,               1,                  null,             0,                    0                }, 
    { "timestamp",      93,         26,             "'",            "'",              null,                     1,          0,                2,            null,                 0,                  null,                "timestamp",       0,               38,              9,               3,                  null,             0,                    0                }, 
    { "bytea",          -3,         255,            "'",            "'",              null,                     1,          0,                2,            null,                 0,                  null,                "bytea",           null,            null,            -3,              null,               null,             0,                    0                }, 
    { "varchar",        12,         65535,          "'",            "'",              "max. length",            1,          0,                2,            null,                 0,                  null,                "varchar",         null,            null,           -9,               null,               null,             0,                    0                }, 
    { "char",           -8,         65535,          "'",            "'",              "max. length",            1,          1,                3,            null,                 0,                  null,                "char",            null,            null,           -8,               null,               null,             0,                    0                }, 
    { "text",           -10,        65535,          "'",            "'",              "max. length",            1,          1,                3,            null,                 0,                  null,                "text",            null,            null,           -10,              null,               null,             0,                    0                }, 
    { "varchar",        -9,         65535,          "'",            "'",              "max. length",            1,          1,                3,            null,                 0,                  null,                "varchar",         null,            null,           -9,               null,               null,             0,                    0                },
    { "bpchar",         -8,         65535,           "'",            "'",              "max. length",            1,          1,                3,            null,                 0,                  null,                "bpchar",          null,            null,            -9,               null,               null,            0,                    0                } }
);
```

### SQLGetTypeInfo using a function

The following code snippets append the `bpchar` type to the existing types returned by the driver.

```
SQLGetTypeInfo = (types as table) as table =>
   let
       newTypes = #table(
           {
               "TYPE_NAME",
               "DATA_TYPE",
               "COLUMN_SIZE",
               "LITERAL_PREF",
               "LITERAL_SUFFIX",
               "CREATE_PARAS",
               "NULLABLE",
               "CASE_SENSITIVE",
               "SEARCHABLE",
               "UNSIGNED_ATTRIBUTE",
               "FIXED_PREC_SCALE",
               "AUTO_UNIQUE_VALUE",
               "LOCAL_TYPE_NAME",
               "MINIMUM_SCALE",
               "MAXIMUM_SCALE",
               "SQL_DATA_TYPE",
               "SQL_DATETIME_SUB",
               "NUM_PREC_RADIX",
               "INTERNAL_PRECISION",
               "USER_DATA_TYPE"
            },
            // we add a new entry for each type we want to add
            {
                {
                    "bpchar",
                    -8,
                    65535,
                    "'",
                    "'",
                    "max. length",
                    1,
                    1,
                    3,
                    null,
                    0,
                    null,
                    "bpchar",
                    null,
                    null,
                    -9,
                    null,
                    null,
                    0,
                    0
                }
            }),
        append = Table.Combine({types, newTypes})
    in
        append;
```

<!--
## Overriding SQLTables

**TODO**

## Creating Your Connector

**Checklist: TODO**
-->

## Setting the connection string

The connection string for your ODBC driver is set using the first argument to the [Odbc.DataSource](/powerquery-m/odbc-datasource) and [Odbc.Query](/powerquery-m/odbc-query) functions. The value can be text, or an M record. When using the record, each field in the record will become a property in the connection string. All connection strings require a `Driver` field (or `DSN` field if you require users to pre-configure a system level DSN). Credential-related properties are set separately. Other properties are driver specific.

The code snippet below shows the definition of a new data source function, creation of the `ConnectionString` record, and invocation of the [Odbc.DataSource](/powerquery-m/odbc-datasource) function.

```
[DataSource.Kind="SqlODBC", Publish="SqlODBC.Publish"]
shared SqlODBC.Contents = (server as text) =>
    let
        ConnectionString = [
            Driver = "SQL Server Native Client 11.0",
            Server = server,
            MultiSubnetFailover = "Yes",
            ApplicationIntent = "ReadOnly",
            APP = "PowerBICustomConnector"
        ],
        OdbcDatasource = Odbc.DataSource(ConnectionString)
    in
        OdbcDatasource;
```

<!--
## Setting credentials

**TODO**

## Disable Parameter Binding (if required)

**TODO**
-->

## Next steps

- [Test and troubleshoot an ODBC-based connector](odbc-troubleshooting.md)
