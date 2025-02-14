# JSON Index

JSON index can be applied to JSON string columns to accelerate the value lookup and filtering for the column.

## When to use JSON index

JSON string can be used to represent the array, map, nested field without forcing a fixed schema. It is very flexible, but the flexibility comes with a cost - filtering on JSON string column is very expensive.

Suppose we have some JSON records similar to the following sample record stored in the `person` column:

```javascript
{
  "name": "adam",
  "age": 30,
  "country": "us",
  "addresses":
  [
    {
      "number" : 112,
      "street" : "main st",
      "country" : "us"
    },
    {
      "number" : 2,
      "street" : "second st",
      "country" : "us"
    },
    {
      "number" : 3,
      "street" : "third st",
      "country" : "ca"
    }
  ]
}
```

Without an index, in order to look up a key and filter records based on the value, we need to scan and reconstruct the JSON object from the JSON string for every record, look up the key and then compare the value.

For example, in order to find all persons whose name is "adam", the query will look like:

```sql
SELECT ... FROM mytable WHERE JSON_EXTRACT_SCALAR(person, '$.name', 'STRING') = 'adam'
```

JSON index is designed to accelerate the filtering on JSON string columns without scanning and reconstructing all the JSON objects.

## Configure JSON index

To enable the JSON index, set the following config in the table config:

```javascript
{
  "tableIndexConfig": {        
    "jsonIndexColumns": [
      "person",
      ...
    ],
    ...
  }
}
```

Note that JSON index can only be applied to `STRING` columns whose values are JSON strings.

## How to use JSON index

JSON index can be used via the `JSON_MATCH` predicate: `JSON_MATCH(<column>, '<filterExpression>')`. For example, to find all persons whose name is "adam", the query will look like:

```sql
SELECT ... FROM mytable WHERE JSON_MATCH(person, 'name=''adam''')
```

Note that the quotes within the filter expression need to be escaped.

## Supported filter expressions

### Simple key lookup

Find all persons whose name is "adam".

```sql
SELECT ... FROM mytable WHERE JSON_MATCH(person, 'name=''adam''')
```

### Chained key lookup

Find all persons who have an address \(one of the addresses\) with number 112.

```sql
SELECT ... FROM mytable WHERE JSON_MATCH(person, 'addresses.number=112')
```

### Nested filter expression

Find all persons whose name is "adam" and also have an address \(one of the addresses\) with number 112.

```sql
SELECT ... FROM mytable WHERE JSON_MATCH(person, 'name=''adam'' AND addresses.number=112')
```

### Array access

Find all persons whose first address has number 112.

```sql
SELECT ... FROM mytable WHERE JSON_MATCH(person, '"addresses[0].number"=112')
```

Note that `addresses[0].number` needs to be escaped because it contains special character.

### Existence check

Find all persons who have phone field within the JSON.

```sql
SELECT ... FROM mytable WHERE JSON_MATCH(person, 'phone IS NOT NULL')
```

Find all persons whose first address does not contain floor field within the JSON.

```sql
SELECT ... FROM mytable WHERE JSON_MATCH(person, '"addresses[0].floor" IS NULL')
```

## Limitations

1. JSON index does not work on multi-dimensional array, e.g. `addresses[0][1]` is not supported.
2. The key \(left-hand side\) of the filter expression must be the leaf level of the JSON object, e.g. `addresses='main st'` won't work.

