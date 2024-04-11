# Variables interaction

## Create a Variable

```csharp
var myVar = InformationModel.MakeVariable("MyVariable", OpcUa.DataTypes.Int32);
```

## Handle Struct variable

By connecting to an OPC-UA server from another vendor, it is possible for Optix to resolve structures as Struct! (or arrays of structs). 
This type of variable, at code level is readonly, to be able to modify the values within it, one must extract its content which is always expressed in an array of objects, modify this array and recreate a new Struct.

```csharp
[ExportMethod]
public void ModifyStructVariable (NodeId opcuaStructVariableNodeId, int indexToModify, string value)
{
    IUAVariable opcUaVariable = InformationModel.GetVariable(opcuaStructVariableNodeId);
    Struct structureVariable = opcUaVariable.RemoteRead();
    object[] newValues = structureVariable.Values as object[];
    newValues[indexToModify] = value;
    Struct newStructureToWrite = structureVariable.HasDataTypeId ? new Struct(structureVariable.DataTypeId, newValues) : new Struct(newValues);
    opcUaVariable.RemoteWrite(newStructureToWrite);
}
```

## Create a complex Variable

Some DataTypes are not exposed by OPC/UA, so you need to get to the DataType number (attaching the debugger or inspecting the decompiled sources) and then use that to build a NodeId

### TimeRange variable

TimeRange variables are structures with two datatimes inside it, its type as NodeID is contained in the FTOptix:Core name space and refers as primary ancestor to Struct
```csharp
[ExportMethod]
public void CreateTimeRangeVariable(string browseName)
{
    IUAVariable newTimeRangeVariable = InformationModel.MakeVariable(browseName, new NodeId(LogicObject.Context.GetNamespaceIndex("urn:FTOptix:Core"), 277u));
    TimeRange timeRangeValue = new TimeRange();
    timeRangeValue.StartTime = DateTime.Now.AddDays(-2);
    timeRangeValue.EndTime = DateTime.Now.AddDays(5);
    newTimeRangeVariable.Value = timeRangeValue;
    if (Project.Current.Get("Model").GetVariable(browseName) == null)
        Project.Current.Get("Model").Add(newTimeRangeVariable);
}

[ExportMethod]
public void ModifyTimeRangeVariable(NodeId timeRangeVariableNodeId, DateTime startTime, DateTime stopTime)
{
    IUAVariable timeRangeVariable = InformationModel.GetVariable(timeRangeVariableNodeId);
    if (timeRangeVariable == null)
        return;
    TimeRange timeRangeValue = timeRangeVariable.Value.Value as TimeRange;
    timeRangeValue.StartTime = startTime;
    timeRangeValue.EndTime = stopTime;
    timeRangeVariable.Value = timeRangeValue;
}
```

## Sync to variable change

For each variable that is created in FT Optix, the corresponding class is automatically generated, this is actually creating two properties/classes per each variable
- The first one is a variable with the same BrowseName
- The second one is the BrowseName of the variable concatenated by `Variable`, this is typically used to sync to change in value or to make DynamicLinks
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

## RemoteRead

```csharp
[ExportMethod]
public void ReadBitFromPlcInteger(out int value)
{
    var tag1 = Project.Current.Get<FTOptix.RAEtherNetIP.Tag>("CommDrivers/RAEtherNet_IPDriver1/RAEtherNet_IPStation1/Tags/Program:CustomUdtProgram/program_dint_1D");
    var remoteVariables = new List<RemoteVariable>()
    {
        new RemoteVariable(tag1),
    };
    var values = InformationModel.RemoteRead(remoteVariables).ToList();
    int[] plcArray = (int[])values[0].Value.Value;
    Log.Info($"Value of first element of the array is {plcArray[0]}");
    Log.Info($"Bit 5 of first element of the array is {GetBitValue(plcArray[0], 5)}");
    value = plcArray[0];
}

private bool GetBitValue(int inputValue, int bitPosition)
{
    var array = new BitArray(new int[] { inputValue });
    return array[bitPosition];
}
```

## Creating TagStructure

```csharp
var tagStructure = InformationModel.MakeVariable<FTOptix.CommunicationDriver.TagStructure>("FanArray", OpcUa.DataTypes.Structure, new[] { 11u });
```

## Creating Guid for NodeID to use NodeFactory.MakeDataType

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
