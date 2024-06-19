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
        ThrowErrorToUI("DataGrid is null");
        return;
    }

    // Get the UI Selected Item variable
    var selectedRow = InformationModel.GetVariable(dataGridItem.UISelectedItem);
    if (selectedRow == null)
    {
        ThrowErrorToUI("Selected row is null");
        return;
    }

    // Get the value at the column called "ClientID" (string)
    var selectedRowClientId = selectedRow.Children.OfType<IUAVariable>().FirstOrDefault(x => x.BrowseName == "ClientID");
    if (selectedRowClientId == null)
    {
        ThrowErrorToUI("ClientID is null");
        return;
    }

    // Get the value at the column called "Used" (Bool)
    var selectedRowUsedCnt = selectedRow.Children.OfType<IUAVariable>().FirstOrDefault(x => x.BrowseName == "Used");
    if (selectedRowUsedCnt == null)
    {
        ThrowErrorToUI("Used is null");
        return;
    }

    // Get the value at the column called "LastUsedTime" (DateTime)
    var selectedRowLastUsedTime = selectedRow.Children.OfType<IUAVariable>().FirstOrDefault(x => x.BrowseName == "LastUsedTime");
    if (selectedRowLastUsedTime == null)
    {
        ThrowErrorToUI("LastUsedTime is null");
        return;
    }

    // Set the variables in the editor object
    sessionClientId.Value = selectedRowClientId.Value.Value.ToString();
    sessionUsedCnt.Value = Convert.ToInt32(selectedRowUsedCnt.Value.Value.ToString());
    sessionLastUsedTime.Value = Convert.ToDateTime(selectedRowLastUsedTime.Value.Value.ToString());
}
```
