# Variables interaction

## Create a simple Variable

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

### Enumeration DataType

Enumerations are DataTypes that contain a variable called EnumValues that contains an array of structures that define the entries of the enumeration in this order:

- `Key` : integer value
- `DisplayValue` : LocalizedText of the displayed text for the enumeration
- `Description` : Localized text of the description of the key-DisplayValue pair

#### Create a Enumeration DataType

```csharp
[ExportMethod]
public void MakeEnumerationAndVariable()
{
    // Create the values for the enumeration
    List<Tuple<int, string, string, string, string>> enumerationDataCollection = new List<Tuple<int, string, string, string, string>>()
    {
        {new Tuple<int, string, string, string, string>(0,"NA","en-US","Not available","en-US") },
        {new Tuple<int, string, string, string, string>(1,"Automatic","en-US","Automatic mode and running","en-US") },
        {new Tuple<int, string, string, string, string>(2,"Manual","en-US","Manual mode","en-US") },
        {new Tuple<int, string, string, string, string>(3,"Stop","en-US","","") },
        {new Tuple<int, string, string, string, string>(5,"Emergency restore","en-US","Automatic mode and operation phase after emergency stop","en-US") }
    };
    StuffMakeNewEnumeration(Project.Current.Get("Model"), "MachineOperationStatus", enumerationDataCollection);
    // Create a variable with DataType the new Enumeration if not exist
    if (Project.Current.Get("Model").Get("FeedingMachineStatus") == null)
    {
        IUAVariable newEnumerationVariable = InformationModel.MakeVariable("FeedingMachineStatus", Project.Current.Get("Model/MachineOperationStatus").NodeId);
        Project.Current.Get("Model").Add(newEnumerationVariable);
    }
}

/// <summary>
/// Generate a new OPC-UA enumeration with values filled by a Tuple including all necessary information
/// </summary>
/// <param name="newEnumerationOwner">Node where to add the new enumeration</param>
/// <param name="newEnumerationName">BrowseName of the new enumeration</param>
/// <param name="enumerationDataCollection">Tuple that is structured as: key, text value, LocaleId of text value, description, LocaleId of description</param>
private void StuffMakeNewEnumeration(IUANode newEnumerationOwner, string newEnumerationName, List<Tuple<int, string, string, string, string>> enumerationDataCollection)
{
    if (newEnumerationOwner.Get<IUADataType>(newEnumerationName) != null)
    {
        // If the enumeration already exists do not perform any operation
        Log.Warning("StuffMakeNewEnumeration", $"Enumeration with BrowseName {newEnumerationName} alread exist!");
        return;
    }
    // Generate a NodeId for the new Enumeration
    NodeId newEnumerationNodeId = NodeId.Random(newEnumerationOwner.NodeId.NamespaceIndex);
    // For enumeration I need to create the reference structure and the values to be assigned
    List<EnumField> newEnumerationFields = new List<EnumField>();
    List<Struct> newEnumerationValues = new List<Struct>();
    // With one foreach loop it populates both Lists
    foreach (Tuple<int, string, string, string, string> enumerationData in enumerationDataCollection)
    {
        // Generate the localizedText of display Value with Item 2 (value) and Item 3 (LocaleId)
        LocalizedText displayValue = new LocalizedText(enumerationData.Item2, enumerationData.Item3);
        // Generate the localizedText of description with Item 4 (value) and Item 5 (LocaleId)
        LocalizedText description = new LocalizedText(enumerationData.Item4, enumerationData.Item5);
        // Generate the Struct containing the enumeration values in the order Key, DisplayValue and Description
        List<object> newValues = new List<object>
        {
            enumerationData.Item1,
            displayValue,
            description
        };
        newEnumerationValues.Add(new Struct(OpcUa.DataTypes.EnumValueType, newValues.AsReadOnly()));
        // Generate the EnumField called Value<key> (ex Value0) containing the values of the enumeration in the order Key, DisplayValue and Description
        newEnumerationFields.Add(new EnumField($"Value{enumerationData.Item1}", enumerationData.Item1, displayValue, description));
    }
    // Generate a new EnumDefinition with the same nodeId and browseName of the new enumeration
    EnumDefinition newEnumerationDefinition = new EnumDefinition(newEnumerationName, newEnumerationNodeId, newEnumerationFields.AsReadOnly());
    // Generate the new Enumeration (is a DataType)
    IUADataType newEnumeration = Project.Current.Context.NodeFactory.MakeDataType(newEnumerationNodeId, newEnumerationName, OpcUa.DataTypes.Enumeration, enumDefinition: newEnumerationDefinition);
    // Generate the variable EnumValues, which contains all the values of the enumeration that you will see in the IDE
    IUAVariable newEnumValuesVariable = InformationModel.MakeVariable(new QualifiedName(0, "EnumValues"), OpcUa.DataTypes.EnumValueType, OpcUa.VariableTypes.BaseDataVariableType, new uint[1] { (uint) newEnumerationValues.Count });
    // Fill the variable EnumValues with the values passed in the method
    newEnumValuesVariable.Value = new UAValue(newEnumerationValues.ToArray());
    // Finalize by adding the variable to the enumeration and the latter to its owner
    newEnumeration.Add(newEnumValuesVariable);
    newEnumerationOwner.Add(newEnumeration);
}
```

### Read Enumeration DataType fields

```csharp
/// <summary>
/// Read the fields of an Enumeration DataType and log Value, DisplayName and Description.
/// </summary>
var enumerationDataType = Project.Current.Get<IUADataType>("Model/Enumeration1");
if (enumerationDataType != null)
{
    var fields = enumerationDataType.EnumDefinition.Fields;
    foreach (EnumField field in fields)
    {
        Log.Info("Value: " + field.Value + ", DisplayName: " + field.DisplayName.Text + ", Description: " + field.Description.Text);
    }
}
```

### Creating TagStructure

```csharp
// Create a new TagStructure variable
var tagStructure = InformationModel.MakeVariable<FTOptix.CommunicationDriver.TagStructure>("FanArray", OpcUa.DataTypes.Structure, new[] { 11u });
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

Registering for the `VariableChange` event in C#, registers a Core-side subscription to the `VariableChange` event. Simultaneously, a native C++ observer for the `VariableChange` event is created, which acts as a bridge by calling the C# method you subscribed to the `VariableChange` event with. Even if the `NetLogic` dies, without de-registration, the native observer remains alive, so the C# method is still triggered even after the `NetLogic` stops. This means it is **mandatory** to remove the observer before disposing the NetLogic.

Let me know if you need any further assistance!

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

public override void Stop()
{
    // Mandatory unsubscriber
    myVar.VariableChange -= MyVar_VariableChange;
}
```

## Single and multi dimensional arrays

### Create a mono-dimensional array variable

```csharp
// Use the MakeVariable method with the array dimensions as last parameter
var newVariable = InformationModel.MakeVariable("MyArray", OpcUa.DataTypes.Int32, new uint[] { 10 });
// Adds the MyArray variable (Int32[10]) to the Model folder
Project.Current.Get("Model").Add(newVariable);
```

### Create a multi-dimensional array variable

```csharp
// Use the MakeVariable method with the array dimensions as last parameter
var newMatrix = InformationModel.MakeVariable("MyMatrix", OpcUa.DataTypes.Float, new uint[] { 5, 5 });
// Adds the MyMatrix variable (Int32[5,5]) to the Model folder
Project.Current.Get("Model").Add(newMatrix);
```

### Access mono-dimensional arrays

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

### Access multi-dimensional arrays

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

### Convert an array of bytes to a string

Assuming the array of bytes comes from the PLC and we want to write the string to a Model variable

```csharp
// Get the array content from the controller. A RemoteRead can also be added if needed.
byte[] byteArray = (byte[]) Project.Current.GetVariable("CommDrivers/RAEtherNet_IPDriver1/RAEtherNet_IPStation1/Tags/Controller Tags/chsArray").Value.Value;
// Convert the array of bytes to the output string
string appStr = System.Text.Encoding.UTF8.GetString(byteArray, 0, byteArray.Length);
```

### Convert a string to an array of bytes

```csharp
// Convert the string back to a byte array
byte[] byteArray = System.Text.Encoding.UTF8.GetBytes(appStr);
// Optionally, you can add a RemoteWrite to force the new value in the controller
Project.Current.GetVariable("CommDrivers/RAEtherNet_IPDriver1/RAEtherNet_IPStation1/Tags/Controller Tags/chsArray").Value = byteArray;
```

### Read an array from a PLC (RemoteRead) and modify it

```csharp
// Read a PLC array variable
var arrayPLC = Project.Current.GetVariable("CommDrivers/CODESYSDriver1/CODESYSStation1/Tags/Application/PLC_PRG/arrayFloat");
// Perform a remote read to get the full array into local memory
float[] localArray = arrayPLC.RemoteRead();
for (int i = 0; i < localArray.Length; i++)
{
    // Modify values as needed
    localArray[i]++;
    Log.Info(localArray[i].ToString());
}
// Write the modified array back to the PLC variable
arrayPLC.Value = new UAValue(localArray);
```

### Work with two-dimensional arrays

```csharp
[ExportMethod]
public void ArrayBidimensionalExample()
{
    var test = Project.Current.GetVariable("Model/Variable2");
    int[,] array = new int[5,10];
    array[1,5] = 50;
    test.Value = new UAValue(array);
}
```

### Use ChildrenRemoteRead to inspect an aliased object structure

```csharp
var refLight = LogicObject.GetAlias("LightPointer");
if (refLight != null)
{
    var structure = refLight.ChildrenRemoteRead();
    Log.Info("IntermittentMode value in LightPointer structure: " + structure.SingleOrDefault(item => item.RelativePath == "IntermittentMode").Value);
}
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
// Required namespaces for the script
using System.Collections.Generic; // Provides collections such as List and Dictionary
using System.Linq; // Provides LINQ methods for querying collections
using FTOptix.NetLogic; // FTOptix-specific logic and framework utilities
using FTOptix.UI; // FTOptix UI elements
using UAManagedCore; // Core OPC UA components for FTOptix
#endregion

public class DesignTimeNetLogic1 : BaseNetLogic
{
    /// <summary>
    /// This method is used when the node is an instance of a type and no other super types are used.
    /// It checks and logs specific properties of the children nodes within a panel.
    /// </summary>
    [ExportMethod]
    public void SimpleInstance()
    {
        Log.Info("Method1 called");

        // Retrieve the panel object by name
        var myPanel = Owner.Get<Panel>("MyPanel1");
        
        // Get all child nodes of type IUAVariable within the panel
        var panelChildren = myPanel.GetNodesByType<IUAVariable>();
        Log.Info("Found " + panelChildren.Count().ToString() + " children in panel");

        // Iterate over each child node
        foreach (var item in panelChildren)
        {
            // Check if the variable's value differs from its corresponding object type's value
            if (myPanel.ObjectType.GetVariable(item.BrowseName).Value != myPanel.GetVariable(item.BrowseName).Value)
                Log.Warning(item.BrowseName + " has a different value from the object type");

            // Check if the variable has a dynamic link
            if (myPanel.GetVariable(item.BrowseName).Refs.GetNode(Optix.CoreBase.ReferenceTypes.HasDynamicLink) != null)
                Log.Warning(item.BrowseName + " has a dynamic link");

            // Check if the variable has a converter
            if (myPanel.GetVariable(item.BrowseName).Refs.GetNode(FTOptix.CoreBase.ReferenceTypes.HasConverter) != null)
                Log.Warning(item.BrowseName + " has a converter");
        }
    }

    /// <summary>
    /// This method is used for instances derived from a type with multiple levels of inheritance.
    /// It traverses up the type hierarchy to find and compare values for child nodes.
    /// </summary>
    [ExportMethod]
    public void RecursiveApproach()
    {
        // Retrieve the panel object by name
        var myPanel = Owner.Get<Panel>("MyPanel1");

        // Get all child nodes of type IUAVariable within the panel
        var panelChildren = myPanel.GetNodesByType<IUAVariable>();
        Log.Info("Found " + panelChildren.Count().ToString() + " children in panel");

        // Browse path used for locating nodes in the hierarchy
        var browsePath = new List<QualifiedName>();

        // Determine the node type or supertype for comparison
        IUANode newMatchingNode;
        if (myPanel.IsObjectTypeOrVariableType())
            newMatchingNode = FindNodeInTypeOrSuperType(myPanel.GetSuperType(), browsePath);
        else if (myPanel.IsObjectOrVariable())
            newMatchingNode = FindNodeInTypeOrSuperType(myPanel.GetTypeNode(), browsePath);
        else
            newMatchingNode = null;

        // Log an error if no matching node is found in the hierarchy
        if (newMatchingNode == null)
        {
            Log.Error("Can't get to the node supertype!");
            return;
        }

        // Compare instance values with type values for each child node
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

    /// <summary>
    /// Finds a node within a type or its supertypes by traversing the hierarchy.
    /// </summary>
    private IUANode FindNodeInTypeOrSuperType(IUANode typeNode, List<QualifiedName> browsePath)
    {
        while (typeNode != null)
        {
            // Attempt to locate the node at the current type level
            var childNode = FindNode(typeNode, browsePath);
            if (childNode != null)
                return childNode;

            // Move to the supertype and continue searching
            typeNode = typeNode.GetSuperType();
        }

        return null;
    }

    /// <summary>
    /// Locates a node by following a browse path from a parent node.
    /// </summary>
    private IUANode FindNode(IUANode parentNode, List<QualifiedName> browsePath)
    {
        var currentNode = parentNode;

        // Traverse the browse path in reverse
        for (int i = browsePath.Count - 1; i >= 0; --i)
        {
            var childNode = currentNode.Refs.GetNode(browsePath[i]);
            if (childNode == null || childNode.Owner != currentNode)
                return null;

            currentNode = childNode;
        }

        return currentNode;
    }
}

public static class PrototypeAnalyzerExtensions
{
    // Extensions for working with node prototypes and type hierarchies in FTOptix

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

## Remove the value from a NodeId variable

```csharp
private void ClearNodeIdValue()
{
    // Get the variable
    var myVar = Project.Current.GetVariable("Model/MyNodeIdVariable");
    // Clear the value
    myVar.Value = NodeId.Empty;
}
```
