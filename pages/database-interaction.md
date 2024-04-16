# Database Interaction

Database interaction is only possible via the FT Optix API

## Insert

First you need to access the Database node, then get the Table by its BrowseName and then prepare the object which will contain the rows to add

```csharp
// Get the Database object from the current project
var myStore = Project.Current.Get<Store>("DataStores/EmbeddedDatabase");
// Get a specific table by name
var myTable = myStore.Tables.Get<Table>("TicTacToe");
// Prepare the header for the insert query (list of columns)
string[] columns = { "Username", "Seed", "GameTime", "Timestamp" };
// Create the new object, a bidimensional array where the first element
// is the number of rows to be added, the second one is the number
// of columns to be added (same size of the columns array)
var values = new object[1, 4];
// Set some values for each column
values[0, 0] = (string)Owner.Owner.Get<Label>("LbUser").Text;
values[0, 1] = (string)Owner.Owner.Get<Label>("LbSeed").Text;
values[0, 2] = (DateTime.Now - startGameTime).TotalSeconds;
values[0, 3] = DateTime.Now;
// Perform the insert query
myTable.Insert(columns, values);
// Some log for users
Log.Info("TicTacToe.Score", "Inserting " + values[0, 1]);
```

## Update

```csharp
// Get the Database from the current project
var myStore = Project.Current.Get<Store>("DataStores/EmbeddedDatabase");
// Create the output to get the result (mandatory)
Object[,] ResultSet;
String[] Header;
// Perform the query
myStore.Query("UPDATE MyTable SET ColumnName=NewValue", out Header, out ResultSet);
```

## Delete

```csharp
// Get the Database from the current project
var myStore = Project.Current.Get<Store>("DataStores/EmbeddedDatabase");
// Create the output to get the result (mandatory)
Object[,] ResultSet;
String[] Header;
// Perform the query
myStore.Query("DELETE FROM MyTable WHERE ColumnName=NewValue", out Header, out ResultSet);
```

### Select

```csharp
// Get the Database from the current project
var myStore = Project.Current.Get<Store>("DataStores/EmbeddedDatabase");
// Create the output to get the result (mandatory)
Object[,] ResultSet;
String[] Header;
// Perform the query
myStore.Query("SELECT * FROM TableName", out Header, out ResultSet);
```

### Get the list of columns name from a DataBase

#### From ODBC

```csharp
// Get the Database from the current project
var myStore = Project.Current.Get<Store>("DataStores/EmbeddedDatabase");
// Create the output to get the result (mandatory)
Object[,] ResultSet;
String[] Header;
// Perform the query
myStore.Query("SELECT COLUMN_NAME FROM INFORMATION_SCHEMA.COLUMNS WHERE TABLE_NAME = 'AlarmsEventLogger1'", out Header, out ResultSet);
```

#### From EmbeddedDatabase

The EmbeddedDatabase uses a SQLite technology which only supports a subset of features from a normal store, we will need some additional steps

```csharp
Regex regexTable = new Regex("(?<=\\bFROM\\s)(\\w+)");
String tableName = regexTable.Match(Owner.GetVariable("Query").Value.Value.ToString(), 0, Owner.GetVariable("Query").Value.Value.ToString().Length).Value;
               
// Check if we are using EmbeddedDatabase or SQL
if (dataSource.GetType().FullName.ToString().Contains("SQLite")) {
    // Extract list of tables
    var tablesList = myStore.Children.ToList()[0].Children.ToList();
    foreach (var table in tablesList) {
    // Check if current table matches input query table
    if (((QPlatform.SQLiteStore.SQLiteStoreTable)table).BrowseName == tableName) {
        var columnList = ((QPlatform.SQLiteStore.SQLiteStoreTable)table).Columns.ToList();
        // Extract columns name
        for (int i = 1; i < (columnList.Count - 2); i++) {
            if (i == 1) {
                dataNames = columnList[i].BrowseName.ToString();
            } else {
                dataNames += ", " + columnList[i].BrowseName.ToString();
            }
        }
    }
}
```

## Creating random Database data

```csharp
private void PopulateTableWithRandomData(string tableName)
{
    // Read the column names from the current  project
    var tableColumnsNames = GetTableColumnsNames(tableName);
    var tableColumnsTypes = GetTableColumnsTypes(tableName);
    // Add some data (using a for loop to specify entries)
    for (int i = 0; i < 100; i++)
    {
        object[,] values = GenerateTableRowData(tableColumnsTypes, tableColumnsNames);
        // Perform the insert
        _store.Insert(tableName, tableColumnsNames.ToArray(), values);
    }
}

private object[,] GenerateTableRowData(List<Type> tableColumnsTypes, List<string> tableColumnsNames)
{
    Random random = new Random();
    object[,] values = new object[1, tableColumnsTypes.Count];
    // Generate a random value for each column
    var rowData = tableColumnsTypes.Select(rowType => GetRandomValue(rowType, random)).ToList();
    // Add the new data to the returned object
    for (int i = 0; i < tableColumnsTypes.Count; i++)
    {
        values[0, i] = rowData[i];
    }
    return values;
}

private static object GetRandomValue(Type dataType, Random random)
{
    if (dataType == typeof(int)) return random.Next(1, 100);
    if (dataType == typeof(float)) return random.NextDouble() * 100.0;
    if (dataType == typeof(string)) return Guid.NewGuid().ToString();
    if (dataType == typeof(bool)) return random.Next(2);
    return null;
}

```
