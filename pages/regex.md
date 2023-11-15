# Regex

## Extract alphanumeric characters from a string

```csharp
Regex rgxFileName = new Regex("[^a-zA-Z0-9 -]");
fileName = rgxFileName.Replace(fileName, "");
```

## Extract the table name from a query

```csharp
Regex rgxTable = new Regex("(?<=\\bFROM\\s)(\\w+)");
Var tableName = rgxTable.Match(Query, 0, Query.Length)
```