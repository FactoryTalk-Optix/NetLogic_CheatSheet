# Data grids

## Getting column values from the UI Selected Item

Add a Runtime NetLogic as children of the DataGrid, add the following NetLogic and invoke the `DataGridSelectionChanged` method in the `User selection changed` event of the DataGrid itself, remember to pass the NodeId of the DataGrid as argument of the function (to make it session-independent).
The UI Selected Item is actually an object which has a children variable per each column of the DataGrid.

```csharp
[ExportMethod]
public void DataGridSelectionChanged(NodeId dataGrid)
{
    // Session variables where to store the output of the SelectedItem
    var sessionClientId = Session.GetVariable("selectedClientId");
    var sessionUsedCnt = Session.GetVariable("selectedUsedCnt");
    var sessionLastUsedTime = Session.GetVariable("selectedLastUsedTime");

    // Retrieve the DataGrid object
    var dataGridItem = InformationModel.Get<DataGrid>(dataGrid);
    if (dataGridItem == null)
    {
        Log.Error("DataGrid is null");
        return;
    }

    // Get the UI Selected Item variable (an object)
    var selectedRow = InformationModel.Get(dataGridItem.UISelectedItem);
    if (selectedRow == null)
    {
        Log.Error("Selected row is null");
        return;
    }

    // Get the value at the column called "ClientID" (string)
    var selectedRowClientId = selectedRow.Children.OfType<IUAVariable>().FirstOrDefault(x => x.BrowseName == "ClientID");
    if (selectedRowClientId == null)
    {
        Log.Error("ClientID is null");
        return;
    }

    // Get the value at the column called "Used" (Bool)
    var selectedRowUsedCnt = selectedRow.Children.OfType<IUAVariable>().FirstOrDefault(x => x.BrowseName == "Used");
    if (selectedRowUsedCnt == null)
    {
        Log.Error("Used is null");
        return;
    }

    // Get the value at the column called "LastUsedTime" (DateTime)
    var selectedRowLastUsedTime = selectedRow.Children.OfType<IUAVariable>().FirstOrDefault(x => x.BrowseName == "LastUsedTime");
    if (selectedRowLastUsedTime == null)
    {
        Log.Error("LastUsedTime is null");
        return;
    }

    // Set the variables in the editor object
    sessionClientId.Value = selectedRowClientId.Value.Value.ToString();
    sessionUsedCnt.Value = Convert.ToInt32(selectedRowUsedCnt.Value.Value.ToString());
    sessionLastUsedTime.Value = Convert.ToDateTime(selectedRowLastUsedTime.Value.Value.ToString());
}
```

## Interacting with the content of a DataGrid

```csharp
[ExportMethod]
public void UpdateDataGridValues()
{
    // Retrieve the DataGrid object
    DataGrid dg = (DataGrid)Owner;
    // Get the "Models" object which is the parent of all the rows of the DataGrid
    IUAObject t = dg.Children.GetObject("Models");

    // Get all rows as a list
    List<UAVariable> rows = new List<UAVariable>();
    foreach (var child in t.Children)
    {
        if (child is UAVariable variable)
            rows.Add(variable);
    }

    double previousEl = 0;
    double previousGas = 0;
    double previousWater = 0;
    double previousSteam = 0;

    // Row 0: header of the grid
    // Row 1: selected item
    // Row 2 and following: actual content of the grid, where each row is an object with a variable per each column of the grid
    for (int i = 2; i < rows.Count; i++)
    {
        var row = rows[i];
        // Get all columns as a list
        List<UAVariable> cols = new List<UAVariable>();
        foreach (var child in row.Children)
        {
            if (child is UAVariable variable)
                cols.Add(variable);
        }
        DateTime timestamp = (DateTime)cols[0].Value;
        if (timestamp.Minute == 0 && i > 0)
        {
            // Edit the value of a column by name
            previousEl = UpdateDGField(ref cols, 2, previousEl);
            previousGas = UpdateDGField(ref cols, 3, previousGas);
            previousWater = UpdateDGField(ref cols, 4, previousWater);
            previousSteam = UpdateDGField(ref cols, 5, previousSteam);
        }
    }
}

private double UpdateDGField(ref List<UAVariable> cols, int i, double prev)
{
    double current = Convert.ToDouble(cols[i].Value);
    if (current >= prev)
        cols[i+4].Value = current - prev;
    else
        cols[i+4].Value = (current + 32767) - prev;
    prev = current;
    return prev;
}
```