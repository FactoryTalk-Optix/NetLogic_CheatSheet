# Trends

## Pens

### Add a pen to a trend

```csharp
[ExportMethod]
public void AddPen(NodeId penId)
{
    // Get the variable that will be used as a pen
    var tag = InformationModel.GetVariable(penId);
    // Create the pen object
    var pen = InformationModel.Make<TrendPen>(tag.BrowseName + "_pen");
    // Link the pen to the variable
    pen.SetDynamicLink(tag, DynamicLinkMode.ReadWrite);
    // Set the pen properties
    pen.Color = Colors.Coral;
    pen.Thickness = 3;
    // Add the pen to the trend
    trend.Pens.Add(pen);
    // Refresh the trend to show the new pen
    trend.Refresh();
}
```

### Remove a pen from a trend

```csharp
/// <summary>
/// Remove Trend pens at runtime using ComboBoxes to select DataLogger and Variable.
/// </summary>
[ExportMethod]
public void RemovePen()
{
    // Remove a pen from a trend by name
    var nameCombo = Owner.Owner.Get<ComboBox>("ComboBox1");
    var myTrend = Owner.Owner.Owner.Owner.Get<Trend>("Trend1");
    myTrend.Pens.Remove(((UAManagedCore.LocalizedText)nameCombo.SelectedValue).Text);
}
```

## Y-axis

### Add a Y-axis to a pen

#### FactoryTalk Optix version 1.7.1.x and later

```csharp
[ExportMethod]
public void AddYAxisToPen()
{
    // Get the pen to interact with
    var trend = Owner.Get<Trend>("Trend1");

    // Get the pen and add a Y-axis to it
    var pen = trend.Pens.Get<TrendPen>("Variable1_pen");
    var newAxis = pen.GetOrAddYAxis();

    // Set the Y-axis properties
    newAxis.MinValue = 0;
    newAxis.MaxValue = 100;

    // Refresh the trend to show the new Y-axis
    trend.Refresh();
}
```

#### FactoryTalk Optix before version 1.7.1.x

```csharp
[ExportMethod]
public void AddYAxisToPen()
{
    // Get the pen and the Y-axis
    var trend = Owner.Get("Trend1") as Trend;
    var pen = trend.Pens.Get<TrendPen>("TrendPen1");

    // Create a new Y-axis
    var uiNamespaceIndex = NamespaceMapProvider.GetNamespaceIndex("urn:FTOptix:UI");
    var yAxis = InformationModel.Make<FTOptix.UI.ValueAxis>(new QualifiedName(uiNamespaceIndex, "YAxis"));

    // Set the Y-axis properties
    yAxis.MinValueVariable.Value = -10;
    yAxis.MaxValueVariable.Value = 42;
    yAxis.PositionVariable.Value = 0; // 0 = left, 1 = right

    // Assign the Y-axis to the pen
    pen.Add(yAxis);
}
```



### Remove a Y-axis from a pen

#### FactoryTalk Optix version 1.7.1.x and later

```csharp
[ExportMethod]
public void RemoveYAxisFromPen()
{
    // Get the pen to interact with
    var trend = Owner.Get<Trend>("Trend1");

    // Get the pen and remove its Y-axis
    var pen = trend.Pens.Get<TrendPen>("Variable1_pen");
    pen.RemoveYAxis();

    // Refresh the trend to show the change
    trend.Refresh();
}
```

#### FactoryTalk Optix before version 1.7.1.x

```csharp
[ExportMethod]
public void RemoveYAxisFromPen()
{
    // Get the pen and its Y-axis
    var trend = Owner.Get("Trend1") as Trend;
    var pen = trend.Pens.Get<TrendPen>("TrendPen1");
    var yAxis = pen.Children.OfType<FTOptix.UI.ValueAxis>().FirstOrDefault();

    // Remove the Y-axis from the pen
    if (yAxis != null)
        pen.Remove(yAxis);
}
```
