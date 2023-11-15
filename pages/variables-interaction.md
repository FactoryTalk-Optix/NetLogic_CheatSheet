# Variables interaction

## Create a Variable

```csharp
var myVar = InformationModel.MakeVariable("MyVariable", OpcUa.DataTypes.Int32);
```

## Sync to variable change

For each variable that is created in FT Optix, the corresponding class is automatically generated, this is actually creating two properties/classes per each variable
- First one is a variable with the same BrowseName
- Second one is the BrowseName of the variable concatenated by `Variable`, this is typically used to sync to change in value or to make DynamicLinks
- Example:
    - User creates a `MotorType` which exposes a `Speed` property
    - Optix creates:
        - `MotorType.Speed` -> This is used to access the value of the variable
        - `MotorType.SpeedVariable` -> This is used to sync to change in value

### Subscribing to a value change

```csharp
public override Start() {
    IUAVariable myVar = Project.Current.GetVariable("Model/MotorType/Speed");
    myVar.VariableChange += MyVar_VariableChange;
}

private void MyVar_VariableChange(object sender, VariableChangeEventArgs e) {
    Log.Info("Old value: " + e.OldValue.ToString());
    Log.Info("New value: " + e.NewValue.ToString());
}
```