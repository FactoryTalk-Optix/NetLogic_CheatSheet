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
public override void Start() {
    IUAVariable myVar = Project.Current.GetVariable("Model/MotorType/Speed");
    myVar.VariableChange += MyVar_VariableChange;
}

private void MyVar_VariableChange(object sender, VariableChangeEventArgs e) {
    Log.Info("Old value: " + e.OldValue.ToString());
    Log.Info("New value: " + e.NewValue.ToString());
}
```

## Access mono-dimensional arrays

```csharp
IUAVariable MyVar;
float[] tempVar;
MyVar = Project.Current.GetVariable("Model/Variable"); 
tempVar = (float[])MyVar.Value.Value;
tempVar[0] = (float)Project.Current.GetVariable("CommDrivers/EthernetIPDriver1/EthernetIPStation1/TagName").Value; 
MyVar.SetValue(tempVar);
```

## Access multi-dimensional arrays

```csharp
IUAVariable MatrixMover;
float[,] tempVar;

MatrixMover = Project.Current.GetVariable("Model/PosMoverA");
tempVar = (float[,])MatrixMover.Value.Value;
tempVar[0, 0] = (float)Project.Current.GetVariable("CommDrivers/EthernetIPDriver1/EthernetIPStation1/Tags/Program:MainProgram/AOI_CalcPosMovA/x_Pos").Value;
tempVar[0, 1] = (float)Project.Current.GetVariable("CommDrivers/EthernetIPDriver1/EthernetIPStation1/Tags/Program:MainProgram/AOI_CalcPosMovA/y_Pos").Value;
tempVar[1, 0] = (float)Project.Current.GetVariable("CommDrivers/EthernetIPDriver1/EthernetIPStation1/Tags/Program:MainProgram/AOI_CalcPosMovA/x_Pos").Value;
tempVar[1, 1] = (float)Project.Current.GetVariable("CommDrivers/EthernetIPDriver1/EthernetIPStation1/Tags/Program:MainProgram/AOI_CalcPosMovA/y_Pos").Value;

MatrixMover.SetValue(tempVar);
```

## Variable synchronizer

The `VariableSynchronizer` allows users to keep some variables synched even if not in use by any current page, it will keep all the children elements updated with the field. This method handles an automatic optimization of the Tags in order to minimize the number of read/write from the controller (while the `RemoteRead` performs a request per each tag and it is much slower)

Please note: the `ChildrenRemoteRead` is just as "slow" as the `RemoteRead`, it is just a recursive approach to a normal `RemoteRead`

```csharp
public override void Start() {
    motorSpeed = LogicObject.Owner.GetVariable("Speed");
    variableSynchronizer = new RemoteVariableSynchronizer();
    variableSynchronizer.Add(motorSpeed);
    motorSpeed.VariableChange += MotorSpeed_VariableChange;
}
public override void Stop() {
    variableSynchronizer?.Dispose();
}
private void MotorSpeed_VariableChange(object sender, VariableChangeEventArgs e) {
    if (motorSpeed.Value > 200) {
        Log.Warning("Speed limit reached!");
    }
}
private IUAVariable motorSpeed;
private RemoteVariableSynchronizer variableSynchronizer;
```

## Creating TagStructure

```csharp
var tagStructure = InformationModel.MakeVariable<FTOptix.CommunicationDriver.TagStructure>("FanArray", OpcUa.DataTypes.Structure, new[] { 11u });
```

## Creating Guid for NodeID for using NodeFactory.MakeDataType

```csharp
public static Guid CreateGuidFromModelElement(string parentGUID, string itemName)
    {
        return CreateGuidFromText($"{parentGUID}.{itemName}");
    }
public static Guid CreateGuidFromText(string text)
    {
        var hashBytes = Encoding.UTF8.GetBytes(text);
        var hasher = MD5.Create();
        var hashValue = hasher.ComputeHash(hashBytes);
        var guid = new Guid(hashValue);
        return guid;
    }
```
