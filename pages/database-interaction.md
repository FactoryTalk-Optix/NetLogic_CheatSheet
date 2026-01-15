# Database Interaction

There are different ways to interact with database in FactoryTalk Optix, the native approach is to leverage the `Store` APIs to perform operations. The list of supported queries is documented [here](./queries.md).

## Insert

First you need to access the Database node, then get the Table by its BrowseName and then prepare the object which will contain the rows to add

```csharp
private void InsertData()
{
    // Get the Database object from the current project
    var myStore = Project.Current.Get<Store>("DataStores/EmbeddedDatabase");
    // Get a specific table by name
    var myTable = myStore.Tables.Get<Table>("TableName");
    // Prepare the header for the insert query (list of columns)
    string[] columns = { "Username", "Apples", "Bananas", "Timestamp" };
    // Create the new object, a bidimensional array where the first element
    // is the number of rows to be added, the second one is the number
    // of columns to be added (same size of the columns array)
    var values = new object[1, 4];
    // Set some values for each column
    values[0, 0] = (string)Owner.Owner.Get<Label>("LbUser").Text;
    values[0, 1] = (int)Owner.Owner.Get<SpinBox>("Apples").Value;
    values[0, 2] = (int)Owner.Owner.Get<SpinBox>("Bananas").Value;
    values[0, 3] = DateTime.Now;
    // Perform the insert query
    myTable.Insert(columns, values);
    // Some log for users
    Log.Info("InsertData", "Inserting data for user " + values[0, 0]);
}
```

## Update

```csharp
private void UpdateData()
{
    // Get the Database from the current project
    var myStore = Project.Current.Get<Store>("DataStores/EmbeddedDatabase");
    // Create the output to get the result (mandatory)
    Object[,] ResultSet;
    String[] Header;
    // Perform the query
    myStore.Query("UPDATE MyTable SET ColumnName=NewValue", out Header, out ResultSet);
}
```

## Delete

```csharp
private void DeleteData()
{
    // Get the Database from the current project
    var myStore = Project.Current.Get<Store>("DataStores/EmbeddedDatabase");
    // Create the output to get the result (mandatory)
    Object[,] ResultSet;
    String[] Header;
    // Perform the query
    myStore.Query("DELETE FROM MyTable WHERE ColumnName=NewValue", out Header, out ResultSet);
}
```

## Select

```csharp
private void SelectData()
{
    // Get the Database from the current project
    var myStore = Project.Current.Get<Store>("DataStores/EmbeddedDatabase");
    // Create the output to get the result (mandatory)
    Object[,] ResultSet;
    String[] Header;
    // Perform the query
    myStore.Query("SELECT * FROM TableName", out Header, out ResultSet);
    // Get the result
    Log.Info("SelectData", "Result: " + ResultSet[0, 0]);
}
```

```csharp
[ExportMethod]
public void SelectData(NodeId minDateTimePickerNodeId, NodeId maxDateTimePickerNodeId, out double averageValue)
{
    // Get the Database from the current project
    var myStore = Project.Current.Get<Store>("DataStores/EmbeddedDatabase1");
    var minDateTimePicker = InformationModel.Get<DateTimePicker>(minDateTimePickerNodeId);
    var minDate = minDateTimePicker.Value.ToString("yyyy-MM-ddTHH:mm:ss.fffffff");
    var maxDateTimePicker = InformationModel.Get<DateTimePicker>(maxDateTimePickerNodeId);
    var maxDate = maxDateTimePicker.Value.ToString("yyyy-MM-ddTHH:mm:ss.fffffff");
    // Create the output to get the result (mandatory)
    Object[,] ResultSet;
    String[] Header;
    var query = $"SELECT AVG(Speed) AS AverageSpeed FROM prod1_table WHERE Machine_state = \"RUN\" AND (Timestamp BETWEEN \"{minDate}\" AND \"{maxDate}\") ORDER BY AverageSpeed DESC";
    // Perform the query
    myStore.Query(query, out Header, out ResultSet);
    // Get the result
    averageValue = (double)ResultSet[0, 0];
}
```

### Get the list of columns name from a DataBase

#### From ODBC

```csharp
private void ExtractColumnNames()
{
    // Get the Database from the current project
    var myStore = Project.Current.Get<Store>("DataStores/EmbeddedDatabase");
    // Create the output to get the result (mandatory)
    Object[,] ResultSet;
    String[] Header;
    // Perform the query
    myStore.Query("SELECT COLUMN_NAME FROM INFORMATION_SCHEMA.COLUMNS WHERE TABLE_NAME = 'AlarmsEventLogger1'", out Header, out ResultSet);
}
```

#### From EmbeddedDatabase

The EmbeddedDatabase uses a SQLite technology which only supports a subset of features from a normal store, we will need some additional steps

```csharp
private void ExtractColumnNames()
{
    Regex regexTable = new Regex("(?<=\\bFROM\\s)(\\w+)");
    String tableName = regexTable.Match(Owner.GetVariable("Query").Value.Value.ToString(), 0, Owner.GetVariable("Query").Value.Value.ToString().Length).Value;

    // Check if we are using EmbeddedDatabase or SQL
    if (dataSource.GetType().FullName.ToString().Contains("SQLite")) 
    {
        // Extract list of tables
        var tablesList = myStore.Children.ToList()[0].Children.ToList();
        foreach (var table in tablesList) 
        {
        // Check if current table matches input query table
        if (((QPlatform.SQLiteStore.SQLiteStoreTable)table).BrowseName == tableName) 
        {
            var columnList = ((QPlatform.SQLiteStore.SQLiteStoreTable)table).Columns.ToList();
            // Extract columns name
            for (int i = 1; i < (columnList.Count - 2); i++) 
            {
                if (i == 1) 
                {
                    dataNames = columnList[i].BrowseName.ToString();
                } else 
                {
                    dataNames += ", " + columnList[i].BrowseName.ToString();
                }
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

## Dynamic creation of a DataGrid

```csharp
[ExportMethod]
public void QueryOnStore(string tableName)
{
    queryTask?.Dispose();
    var arguments = new object[] { tableName };
    queryTask = new LongRunningTask(QueryAndUpdate, arguments, LogicObject);
    queryTask.Start();
}

private void QueryAndUpdate(LongRunningTask myTask, object args)
{
    // Get the table name from the arguments
    var argumentsArray = (object[])args;
    var tableName = (string)argumentsArray[0];

    if (string.IsNullOrEmpty(tableName))
        return;

    // Get the DataGrid and Store
    var store = Project.Current.Get<Store>("DataStores/EmbeddedDatabase1");
    var dataGrid = Owner.Get<DataGrid>("DataGrid");

    if (store == null || dataGrid == null)
        return;

    // Reset the grid
    dataGrid.Query = "";

    // Prepare the query
    var query = $"SELECT * FROM {tableName}";

    // Run the query
    store.Query(query, out String[] header, out Object[,] resultSet);

    if (header == null || resultSet == null)
        return;

    // Clear existing rows
    dataGrid.Columns.Clear();

    DynamicLink lastDynamicLink = null;

    // Create columns based on the header
    foreach (var columnName in header)
    {
        var newDataGridColumn = InformationModel.MakeObject<DataGridColumn>(columnName);
        newDataGridColumn.Title = columnName;
        newDataGridColumn.DataItemTemplate = InformationModel.MakeObject<DataGridLabelItemTemplate>("DataItemTemplate");
        var dynamicLink = InformationModel.MakeVariable<DynamicLink>("DynamicLink", FTOptix.Core.DataTypes.NodePath);
        dynamicLink.Value = "{Item}/" + NodePath.EscapeNodePathBrowseName(columnName);
        newDataGridColumn.DataItemTemplate.GetVariable("Text").Refs.AddReference(FTOptix.CoreBase.ReferenceTypes.HasDynamicLink, dynamicLink);
        newDataGridColumn.OrderBy = dynamicLink.Value;
        dataGrid.Columns.Add(newDataGridColumn);
        lastDynamicLink = dynamicLink;
    }

    // Set last column as sort item - not really needed
    dataGrid.SortColumnVariable.Refs.AddReference(FTOptix.CoreBase.ReferenceTypes.HasDynamicLink, lastDynamicLink);

    // Add the new query to the grid
    dataGrid.Query = query + " ORDER BY Timestamp DESC";
}
```

## CSV interaction

A simple way to import or export data from a database is to use CSV files, here are some examples of how to read and write CSV files using C# in FT Optix.

### Read / Write CSV files

```csharp
/// <summary>
/// Read or write a CSV file from the file system.
/// Inputs: file path (string). Outputs: none.
/// Error modes: file not found, IO exceptions. Success: file is created/read.
/// </summary>
// READ CSV FILE
using System.IO;

// Path to the CSV file
string filePath = "path/to/file.csv";

// Read the contents of the CSV file
using (var reader = new StreamReader(filePath))
{
    while (!reader.EndOfStream)
    {
        // Read the current line
        var line = reader.ReadLine();

        // Split the line into an array of values (CSV separated by commas)
        var values = line.Split(',');

        // TODO: process values (e.g., map to variables, insert into DB, etc.)
    }
}

// WRITE CSV FILE - Solution 1: overwrite/create
using (var writer = new StreamWriter(filePath))
{
    // Write the header row
    writer.WriteLine("Column1,Column2,Column3");

    // Write some data rows
    writer.WriteLine("Value1,Value2,Value3");
    writer.WriteLine("Value4,Value5,Value6");
}

// WRITE CSV FILE - Solution 2: ensure file exists then write
try
{
    if (!File.Exists(filePath))
    {
        // Create a file to write to.
        using (StreamWriter sw = File.CreateText(filePath))
        {
            sw.WriteLine("Hello");
            sw.WriteLine("And");
            sw.WriteLine("Welcome");
        }
    }
    else
    {
        using (var writer = new StreamWriter(filePath))
        {
            writer.WriteLine("New row!");
        }
    }
    // Example: set a UI variable or log to notify success
    // result.Value = "File correctly created: " + filePath;
}
catch (Exception ex)
{
    // result.Value = "Impossible to create the file: " + ex.Message;
    Log.Error("CSV", "Unable to create or write CSV: " + ex.Message);
}

```

### Create an ODBC/Embedded DB table from a CSV schema file

```csharp
/// <summary>
/// Reads a CSV file that describes a table schema and creates a corresponding table in the specified ODBC/Embedded store.
/// CSV format expected: each line contains: ColumnName;ColumnType
/// Supported ColumnType values: String, Bool, Real, Int
/// </summary>
[ExportMethod]
public void TableCreation()
{
    var csvPath = GetCSVFilePath();
    if (string.IsNullOrEmpty(csvPath) || !System.IO.File.Exists(csvPath))
    {
        Log.Error("TableCreation", "CSV not found. Please configure the CSV to parse in the NetLogic configuration");
        return;
    }

    var storePointer = LogicObject.GetVariable("Store")?.Value;
    var storeDB = InformationModel.Get<ODBCStore>(storePointer);
    if (storeDB == null)
    {
        Log.Error("TableCreation", "Store is not correct or not configured");
        return;
    }

    Table myTable = null;
    using (var reader = new StreamReader(csvPath))
    {
        int lineIndex = 0;
        while (!reader.EndOfStream)
        {
            var line = reader.ReadLine();
            if (string.IsNullOrWhiteSpace(line))
                continue;

            // First non-empty line is expected to be the table name
            if (lineIndex == 0)
            {
                var tableName = line.Trim();
                if (storeDB.Find<Table>(tableName) != null)
                {
                    Log.Error("TableCreation", "The table already exists: " + tableName);
                    return;
                }
                myTable = InformationModel.Make<Table>(tableName);
                storeDB.Tables.Add(myTable);
                lineIndex++;
                continue;
            }

            // Parse column definition: name;type
            var values = line.Split(';');
            if (values.Length < 2)
            {
                Log.Warning("TableCreation", "Skipping invalid line: " + line);
                continue;
            }

            var colName = values[0].Trim();
            var colType = values[1].Trim();
            var myColumn = InformationModel.Make<StoreColumn>(colName);

            switch (colType)
            {
                case "String":
                    myColumn.DataType = OpcUa.DataTypes.String;
                    break;
                case "Bool":
                    myColumn.DataType = OpcUa.DataTypes.Boolean;
                    break;
                case "Real":
                    myColumn.DataType = OpcUa.DataTypes.Float;
                    break;
                case "Int":
                    myColumn.DataType = OpcUa.DataTypes.Int16;
                    break;
                default:
                    Log.Error("TableCreation", "Unsupported column type: " + colType + ". Table creation not completed.");
                    return;
            }

            myTable.Columns.Add(myColumn);
        }
    }

    Log.Info("TableCreation", "The table " + (myTable?.BrowseName ?? "(unknown)") + " has been correctly created!");
}
```
