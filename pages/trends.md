# Trends

## Add a pen to a trend

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

## Add a Y-axis to a pen

```csharp
[ExportMethod]
public void AddYAxisToPen(NodeId penId)
{
    // Get the pen and the Y-axis
    var pen = InformationModel.Get<TrendPen>(penId);
    // Create a new Y-axis
    var yAxis = InformationModel.Make<ValueAxis>("YAxis");
    // Set the Y-axis properties
    yAxis.MinValueVariable.Value = -10;
    yAxis.MaxValueVariable.Value = 42;
    yAxis.PositionVariable.Value = 0; // 0 = left, 1 = right
    // Add the Y-axis to the pen
    pen.YAxes.Add(yAxis);
}
```
