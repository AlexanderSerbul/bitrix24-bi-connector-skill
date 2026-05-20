# BI-connector usage rules

If the bi-token isn't known yet, ask the client for it. They can find it inside their Bitrix24 portal at the menu path: "CRM → Analytics → Operational analytics → BI-analytics → BI-analytics settings → Key management".
Or instruct the client to open the key-management page of their portal directly, e.g. https://serbul.bitrix24.ru/biconnector/key_list.php

## Listing BI entities
HTTP POST https://serbul.bitrix24.ru/bitrix/tools/biconnector/gds.php?show_tables

POST request body:
```
{
    "key": "#BI_TOKEN#" // bi-token requested above
}
```

This bi-token must be passed in the body of every POST request to the BI-connector.

Returns:
```
[
    [
        "task_efficiency", // table name
        "Task work efficiency" // table description
    ],
    [
        "crm_dynamic_items_31",
        "Smart process: Smart Invoice"
    ],
...
]
```

## Getting columns of a BI entity
HTTP POST https://serbul.bitrix24.ru/bitrix/tools/biconnector/gds.php?desc&table=#TABLE_NAME#

Returns:
```
[
    {
        "CONCEPT_TYPE": "DIMENSION", // BI field type, METRIC or DIMENSION; METRICs are typically grouped by DIMENSIONs
        "ID": "ID", // the identifier to pass in field requests
        "NAME": "Unique key", // human-readable field name
        "DESCRIPTION": "",
        "TYPE": "NUMBER",
        "IS_PRIMARY": "Y",
        "AGGREGATION_TYPE": null,
        "GROUP_KEY": null,
        "GROUP_CONCAT": null,
        "GROUP_COUNT": null,
        "IS_VALUE_SPLITABLE": null
    },
	...
]
```

## Getting data from a BI entity
HTTP POST https://serbul.bitrix.ru/bitrix/tools/biconnector/pbi.php?table=#TABLE_NAME#

POST request body:
```
{
    "key": "#BI_TOKEN#",
    "fields":[{"name":"DATE_CREATE"},{"name":"STATUS_SEMANTIC_ID"}, ...] // the fields you need from the table description
}
```

Returns valid JSON with `null` or `""` for empty values:
```
[
    [
        "DATE_CREATE",
        "STATUS_SEMANTIC_ID"
    ],
    [
        "2018-07-19 17:48:58",
        "P"
    ],
	...
]
```

### Filtering by date range when retrieving data

Example request body:
```
{
    "key": "#BI_TOKEN#",
    "dateRange": {
        "startDate": "2018-07-19",
        "endDate": "2018-08-02"
    },
    "configParams": {
        "timeFilterColumn": "DATE_CREATE"
    },        
    "fields":[{"name":"DATE_CREATE"},{"name":"STATUS_SEMANTIC_ID"}]
}
```

When filtering, `startDate` is treated as starting at 00:00:00 of that day, and `endDate` is treated as ending at 23:59:59.999. It's also advisable to specify the date or date/time column to filter on via the `timeFilterColumn` key. If the column isn't specified, the first date or date/time column in the table (in column order) is used automatically.

It's very important to always supply a date range for filtering, to avoid pulling enormous amounts of data (thousands or millions of rows can be returned), even if the user doesn't explicitly ask for one. A range of, say, the last 3 months is a sensible default. To request data for a single day, set `startDate = endDate`.

### Predicate filtering when retrieving data

Add a `dimensionsFilters` key to the data request body. Its value is a list of lists of conditions. At the top level the lists are combined with AND, and within each list the conditions (represented as objects) are combined with OR.

The operator logic can be inverted:
```
type is a full-fledged modifier that inverts any operator. So for negation you can use:

EXCLUDE + IS_NULL = "not null" (our missing IS_NOT_NULL)
EXCLUDE + EQUALS = "not equal"
EXCLUDE + CONTAINS = "does not contain"
EXCLUDE + IN_LIST = "not in list"
and so on.
```

When predicate filtering is used in a data request, a date-range filter isn't required — predicates can dramatically reduce the number of returned rows and the data volume.

Example of one condition:
```
    "dimensionsFilters": [
        [
            {
                "fieldName": "ID",
                "values": ["1"],
                "type": "INCLUDE",
                "operator": "EQUALS"
            }
        ]
    ]
```

Example of two conditions combined with OR. Any number of conditions is allowed:
```
    "dimensionsFilters": [
        [
            {
                "fieldName": "ID",
                "values": ["2"],
                "type": "INCLUDE",
                "operator": "EQUALS"
            },
            {
                "fieldName": "ID",
                "values": ["12706"],
                "type": "INCLUDE",
                "operator": "EQUALS"
            }
        ]
    ]
```

Example using AND logic. Two top-level lists (the number of lists is unlimited) are combined with AND, while conditions within each list are combined with OR:
```
    "dimensionsFilters": [
        [
            {
                "fieldName": "ID",
                "values": ["2"],
                "type": "INCLUDE",
                "operator": "EQUALS"
            },
            {
                "fieldName": "ID",
                "values": ["12706"],
                "type": "INCLUDE",
                "operator": "EQUALS"
            }
        ],

        [
            {
                "fieldName": "CREATED_BY_ID",
                "values": ["1"],
                "type": "INCLUDE",
                "operator": "EQUALS"
            }
        ]        
    ]
```



#### EQUALS — equality check
Checks that the BI entity's ID column equals the string "1". `values` always contains strings.
```
   "dimensionsFilters": [
        [
            {
                "fieldName": "ID",
                "values": ["1"],
                "type": "INCLUDE",
                "operator": "EQUALS"
            }
        ]
    ]
```

##### CONTAINS — substring check

```
    "dimensionsFilters": [
        [
            {
                "fieldName": "TITLE",
                "values": ["CRM: Test task case "],
                "type": "INCLUDE",
                "operator": "CONTAINS"
            }
        ]
    ]
```

##### IN_LIST — match against any value in the list

```
    "dimensionsFilters": [
        [
            {
                "fieldName": "ID",
                "values": ["1", "2"],
                "type": "INCLUDE",
                "operator": "IN_LIST"
            }
        ]
    ]
```

### IS_NULL — null check

Unfortunately, the BI-connector protocol doesn't explicitly mark columns as nullable in the table metadata, but if users know for certain that a null check will work for a given field, such queries are useful.

Examples:
```
    "dimensionsFilters": [
        [
            {
                "fieldName": "DEADLINE",
                "values": [],
                "type": "INCLUDE",
                "operator": "IS_NULL"
            }
        ]
    ]
```

### BETWEEN — range membership check

It isn't strictly specified whether this works for non-numeric values in the table.
Both the left and right boundaries are inclusive.

```
    "dimensionsFilters": [
        [
            {
                "fieldName": "ID",
                "values": ["1", "12706"],
                "type": "INCLUDE",
                "operator": "BETWEEN"
            }
        ]
    ]
```

### NUMERIC_GREATER_THAN

It isn't strictly specified whether this works for non-numeric values in the table.

```
    "dimensionsFilters": [
        [
            {
                "fieldName": "ID",
                "values": ["2"],
                "type": "INCLUDE",
                "operator": "NUMERIC_GREATER_THAN"
            }
        ]
    ]
```

### NUMERIC_GREATER_THAN_OR_EQUAL

It isn't strictly specified whether this works for non-numeric values in the table.

```
    "dimensionsFilters": [
        [
            {
                "fieldName": "ID",
                "values": ["2"],
                "type": "INCLUDE",
                "operator": "NUMERIC_GREATER_THAN_OR_EQUAL"
            }
        ]
    ]
```

### NUMERIC_LESS_THAN

It isn't strictly specified whether this works for non-numeric values in the table.

```
    "dimensionsFilters": [
        [
            {
                "fieldName": "ID",
                "values": ["2"],
                "type": "INCLUDE",
                "operator": "NUMERIC_LESS_THAN"
            }
        ]
    ]
```

### NUMERIC_LESS_THAN_OR_EQUAL

It isn't strictly specified whether this works for non-numeric values in the table.

```
    "dimensionsFilters": [
        [
            {
                "fieldName": "ID",
                "values": ["2"],
                "type": "INCLUDE",
                "operator": "NUMERIC_LESS_THAN_OR_EQUAL"
            }
        ]
    ]
```
