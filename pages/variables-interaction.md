# Variables interaction

## Create a Variable

```csharp
using OpcUa = UAManagedCore.OpcUa;

[ExportMethod]
public void CreateVariable()
{
    var myVar = InformationModel.MakeVariable("MyVariable", OpcUa.DataTypes.Int32);
}
```

## Handle Struct variable

By connecting to an OPC-UA server from another vendor, it is possible for Optix to resolve structures as Struct! (or arrays of structs). 
This type of variable, at code level is readonly, to be able to modify the values within it, one must extract its content which is always expressed in an array of objects, modify this array and recreate a new Struct.

```csharp
[ExportMethod]
public void ModifyStructVariable (NodeId opcuaStructVariableNodeId, int indexToModify, string value)
{
    // Get the variable to modify
    IUAVariable opcUaVariable = InformationModel.GetVariable(opcuaStructVariableNodeId);
    // Read the value from the field (assuming we use a PLC/OPCUA client)
    Struct structureVariable = opcUaVariable.RemoteRead();
    // Prepare the structure to be modified
    object[] newValues = structureVariable.Values as object[];
    newValues[indexToModify] = value;
    // Set the new content for the structure
    Struct newStructureToWrite = structureVariable.HasDataTypeId ? new Struct(structureVariable.DataTypeId, newValues) : new Struct(newValues);
    // Write the value to the field
    opcUaVariable.RemoteWrite(newStructureToWrite);
}
```

## Create a complex Variable

Some DataTypes are not exposed by OPC/UA, so you need to get to the DataType number (by attaching the debugger or inspecting the decompiled sources) and then use that info to build a valid NodeId

### TimeRange variable

TimeRange variables are structures with two DateTime inside, its type as NodeID is contained in the FTOptix:Core name space and refers as primary ancestor to Struct

#### Create a TimeRange variable

```csharp
[ExportMethod]
public void CreateTimeRangeVariable(string browseName)
{
    // Create the TimeRange object
    IUAVariable newTimeRangeVariable = InformationModel.MakeVariable(browseName, new NodeId(LogicObject.Context.GetNamespaceIndex("urn:FTOptix:Core"), 277u));
    // Prepare the TimeRange variable
    TimeRange timeRangeValue = new TimeRange();
    // Set the TimeRange value
    timeRangeValue.StartTime = DateTime.Now.AddDays(-2);
    timeRangeValue.EndTime = DateTime.Now.AddDays(5);
    newTimeRangeVariable.Value = timeRangeValue;
    // Add the value to the variable
    if (Project.Current.Get("Model").GetVariable(browseName) == null)
        Project.Current.Get("Model").Add(newTimeRangeVariable);
}
```

#### Edit a TimeRange variable

```csharp
[ExportMethod]
public void ModifyTimeRangeVariable(NodeId timeRangeVariableNodeId, DateTime startTime, DateTime stopTime)
{
    // Get the TimeRange variable
    IUAVariable timeRangeVariable = InformationModel.GetVariable(timeRangeVariableNodeId);
    if (timeRangeVariable == null)
        return;
    // Read the current value
    TimeRange timeRangeValue = timeRangeVariable.Value.Value as TimeRange;
    // Set the new values
    timeRangeValue.StartTime = startTime;
    timeRangeValue.EndTime = stopTime;
    // Update the variable value
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
public override void Start() 
{
    // Get the base variable
    IUAVariable myVar = Project.Current.GetVariable("Model/MotorType/Speed");
    // Create the new event and call the method when a change is detected
    myVar.VariableChange += MyVar_VariableChange;
}

private void MyVar_VariableChange(object sender, VariableChangeEventArgs e) 
{
    // Event listener to print old value and new value
    Log.Info("Old value: " + e.OldValue.ToString());
    Log.Info("New value: " + e.NewValue.ToString());
}
```

## Access mono-dimensional arrays

```csharp
// Get the model variable
var MyVar = Project.Current.GetVariable("Model/Variable"); 
// Create a C# variable to manipulate values
var tempVar = (float[])MyVar.Value.Value;
// Set the elements of the C# variable
tempVar[0] = (float)Project.Current.GetVariable("CommDrivers/EthernetIPDriver1/EthernetIPStation1/TagName").Value; 
// Write the new value(s) to the model variable
MyVar.SetValue(tempVar);
```

## Access multi-dimensional arrays

```csharp
// Read the model matrix
var MatrixMover = Project.Current.GetVariable("Model/PosMoverA");
// Create a C# variable
var tempVar = (float[,])MatrixMover.Value.Value;
// Set some values in the C# variable
tempVar[0, 0] = (float)Project.Current.GetVariable("CommDrivers/EthernetIPDriver1/EthernetIPStation1/Tags/Program:MainProgram/AOI_CalcPosMovA/x_Pos").Value;
tempVar[0, 1] = (float)Project.Current.GetVariable("CommDrivers/EthernetIPDriver1/EthernetIPStation1/Tags/Program:MainProgram/AOI_CalcPosMovA/y_Pos").Value;
tempVar[1, 0] = (float)Project.Current.GetVariable("CommDrivers/EthernetIPDriver1/EthernetIPStation1/Tags/Program:MainProgram/AOI_CalcPosMovA/x_Pos").Value;
tempVar[1, 1] = (float)Project.Current.GetVariable("CommDrivers/EthernetIPDriver1/EthernetIPStation1/Tags/Program:MainProgram/AOI_CalcPosMovA/y_Pos").Value;
// Update the model variable with the new value
MatrixMover.SetValue(tempVar);
```

## Variable synchronizer

The `VariableSynchronizer` allows users to keep some variables synched even if not in use by any current page, it will keep all the children elements updated with the field. This method handles an automatic optimization of the Tags in order to minimize the number of read/write from the controller (while the `RemoteRead` performs a request per each tag and it is much slower)

### Please note

- The `ChildrenRemoteRead` is just as "slow" as the `RemoteRead`, it is just a recursive approach to a normal `RemoteRead`
- The `VariableSynchronizer` is made to sync a small number of tags and may have significant impact on the communication driver, make sure to use it wisely

```csharp
public override void Start() 
{
    // Get the field variable
    motorSpeed = LogicObject.Owner.GetVariable("Speed");
    // Create the VariableSynchronizer object
    variableSynchronizer = new RemoteVariableSynchronizer();
    // Add the variables to the synchronizer
    variableSynchronizer.Add(motorSpeed);
    // Add the event to listen to value changes
    motorSpeed.VariableChange += MotorSpeed_VariableChange;
}
public override void Stop() 
{
    // Destroy the synchronized on stop
    variableSynchronizer?.Dispose();
}
private void MotorSpeed_VariableChange(object sender, VariableChangeEventArgs e) 
{
    // Event listener when value change is detected
    if (motorSpeed.Value > 200) {
        Log.Warning("Speed limit reached!");
    }
}
private IUAVariable motorSpeed;
private RemoteVariableSynchronizer variableSynchronizer;
```

## RemoteRead

Remote read is a inline read of the field value, code is stopped to read the value, once the variable is updated, the code keeps running

```csharp
[ExportMethod]
public void ReadBitFromPlcInteger(out int value)
{
    // Get the plc tag to read
    var tag1 = Project.Current.Get<FTOptix.RAEtherNetIP.Tag>("CommDrivers/RAEtherNet_IPDriver1/RAEtherNet_IPStation1/Tags/Program:CustomUdtProgram/program_dint_1D");
    // Create the remote read object
    var remoteVariables = new List<RemoteVariable>()
    {
        // Add the variables to read to the list
        new RemoteVariable(tag1),
    };
    // Read the new values
    var values = InformationModel.RemoteRead(remoteVariables).ToList();
    // Get the field values to a C# variable
    var plcArray = (int[])values[0].Value.Value;
    // Do something with the values
    Log.Info($"Value of first element of the array is {plcArray[0]}");
    Log.Info($"Bit 5 of first element of the array is {GetBitValue(plcArray[0], 5)}");
}

private bool GetBitValue(int inputValue, int bitPosition)
{
    // Read a specific bit from a word
    var array = new BitArray(new int[] { inputValue });
    return array[bitPosition];
}
```

## Creating TagStructure

```csharp
// Create a new TagStructure variable
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

## Check if a variable on an instance has a dynamic link, converter or the default value was overwritten

```csharp
#region Using directives
using System.Collections.Generic;
using System.Linq;
using FTOptix.NetLogic;
using FTOptix.UI;
using UAManagedCore;
#endregion

public class DesignTimeNetLogic1 : BaseNetLogic
{
    /// <summary>
    /// This method is used when the node is an instance of a type and no other super types are used
    /// </summary>
    [ExportMethod]
    public void SimpleInstance()
    {
        Log.Info("Method1 called");
        // Insert code to be executed by the method
        var myPanel = Owner.Get<Panel>("MyPanel1");
        var panelChildren = myPanel.GetNodesByType<IUAVariable>();
        Log.Info("Found " + panelChildren.Count().ToString() + " children in panel");
        foreach (var item in panelChildren)
        {
            // Do something with the controls
            if (myPanel.ObjectType.GetVariable(item.BrowseName).Value != myPanel.GetVariable(item.BrowseName).Value)
                Log.Warning(item.BrowseName + " has a different value from the object type");

            if (myPanel.GetVariable(item.BrowseName).Refs.GetVariable(FTOptix.CoreBase.ReferenceTypes.HasDynamicLink) != null)
                Log.Warning(item.BrowseName + " has a dynamic link");

            if (myPanel.GetVariable(item.BrowseName).Refs.GetVariable(FTOptix.CoreBase.ReferenceTypes.HasConverter) != null)
                Log.Warning(item.BrowseName + " has a converter");
        }
    }


    /// <summary>
    /// This approach is used when the instance comes from a type derived from another type (and so on)
    /// This will go up to all supertypes to extract the values from anywhere
    /// </summary>
    [ExportMethod]
    public void RecursiveApproach()
    {
        var myPanel = Owner.Get<Panel>("MyPanel1");
        var panelChildren = myPanel.GetNodesByType<IUAVariable>();
        Log.Info("Found " + panelChildren.Count().ToString() + " children in panel");
        var browsePath = new List<QualifiedName>();

        IUANode newMatchingNode;
        if (myPanel.IsObjectTypeOrVariableType())
            newMatchingNode = FindNodeInTypeOrSuperType(myPanel.GetSuperType(), browsePath);
        else if (myPanel.IsObjectOrVariable())
            newMatchingNode = FindNodeInTypeOrSuperType(myPanel.GetTypeNode(), browsePath);
        else
            newMatchingNode = null;

        if (newMatchingNode == null)
        {
            Log.Error("Can't get to the node supertype!");
            return;
        }

        foreach (var node in panelChildren)
        {
            Log.Info(node.BrowseName);
            var instanceValue = myPanel.GetVariable(node.BrowseName).Value;
            var typeValue = newMatchingNode.GetVariable(node.BrowseName).Value;

            if (instanceValue != typeValue)
                Log.Warning("Value of node " + node.BrowseName + " has a different value");

            if (myPanel.GetVariable(node.BrowseName).Refs.GetVariable(FTOptix.CoreBase.ReferenceTypes.HasDynamicLink) != null)
                Log.Warning(node.BrowseName + " has a dynamic link");

            if (myPanel.GetVariable(node.BrowseName).Refs.GetVariable(FTOptix.CoreBase.ReferenceTypes.HasConverter) != null)
                Log.Warning(node.BrowseName + " has a converter");
        }
    }

    private IUANode FindNodeInTypeOrSuperType(IUANode typeNode, List<QualifiedName> browsePath)
    {
        while (typeNode != null)
        {
            var childNode = FindNode(typeNode, browsePath);
            if (childNode != null)
                return childNode;

            typeNode = typeNode.GetSuperType();
        }

        return null;
    }

    private IUANode FindNode(IUANode parentNode, List<QualifiedName> browsePath)
    {
        var currentNode = parentNode;

        for (int i = browsePath.Count - 1; i >= 0; --i)
        {
            var childNode = currentNode.Refs.GetNode(browsePath[i]);
            if (childNode == null)
                return null;

            if (childNode.Owner != currentNode)
                return null;

            currentNode = childNode;
        }

        return currentNode;
    }
}

public static class PrototypeAnalyzerExtensions
{
    public static IUANode GetTypeNode(this IUANode node)
    {
        if (node is IUAObject obj)
            return obj.ObjectType;

        if (node is IUAVariable var)
            return var.VariableType;

        return null;
    }

    public static IUANode GetSuperType(this IUANode node)
    {
        if (node is IUAObjectType objType)
            return objType.SuperType;

        if (node is IUAVariableType varType)
            return varType.SuperType;

        return null;
    }

    public static bool IsObjectTypeOrVariableType(this IUANode node) => node.NodeClass == NodeClass.ObjectType || node.NodeClass == NodeClass.VariableType;

    public static bool IsObjectOrVariable(this IUANode node) => node.NodeClass == NodeClass.Object || node.NodeClass == NodeClass.Variable;

    public static IUANode GetPrototype(this IUANode node)
    {
        if (node is IUAObject obj)
            return obj.Prototype;

        if (node is IUAVariable var)
            return var.Prototype;

        return null;
    }

    public static void SetPrototype(this IUANode node, IUANode prototype)
    {
        if (node is IUAObject obj)
            obj.Prototype = (IUAObject)prototype;
        else if (node is IUAVariable var)
            var.Prototype = (IUAVariable)prototype;
    }

    public static IReadOnlyList<IUANode> GetInstances(this IUANode node) => node.InverseRefs.GetNodes(UAManagedCore.OpcUa.ReferenceTypes.HasTypeDefinition, false);
}
```
## Loading environment variables

Some elements can be loaded from Environment Variables, for example it is good practice to load secrets of Databases or external sources from env. This can be done using the [System.Environment.GetEnvironmentVariable](https://learn.microsoft.com/en-us/dotnet/fundamentals/runtime-libraries/system-environment-getenvironmentvariable), please note about some limitations, for example the `User` and `Machine` target are only supported on Windows platforms, when working with Linux (like Docker Containers), they should be loaded from `Process`.

```csharp
public override void Start()
{
    var myStore = (ODBCStore)Owner;

    // Load DB password
    string env_password = Environment.GetEnvironmentVariable("DB_PASSWORD", EnvironmentVariableTarget.User); // Windows only
    if (string.IsNullOrEmpty(env_password))
        env_password = Environment.GetEnvironmentVariable("DB_PASSWORD", EnvironmentVariableTarget.Machine); // Windows only
    if (string.IsNullOrEmpty(env_password))
        env_password = Environment.GetEnvironmentVariable("DB_PASSWORD", EnvironmentVariableTarget.Process); // Required on Linux

    // Load variables to the project
    if (!string.IsNullOrEmpty(env_password))
    {
        Log.Info("Loading database password from environment variable DB_PASSWORD");
        myStore.Password = env_password;
    }
}
```