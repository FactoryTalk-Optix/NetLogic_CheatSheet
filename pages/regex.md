# Regex

## Extract alphanumeric characters from a string

```csharp
Regex regexFileName = new Regex("[^a-zA-Z0-9 -]");
fileName = regexFileName.Replace(fileName, "");
```

## Extract the table name from a query

```csharp
Regex regexTable = new Regex("(?<=\\bFROM\\s)(\\w+)");
Var tableName = regexTable.Match(Query, 0, Query.Length)
```