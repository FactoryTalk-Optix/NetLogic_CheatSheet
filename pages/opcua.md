# OPC/UA

## Node's references

In OPC/UA, some nodes may have additional references are explicit relationship (a named pointer) from one Node to another

```csharp
[ExportMethod]
public void ReadReference(NodeId element)
{
    // Get to the element where we want to investigate the OPC/UA references
    IUANode targetNode = InformationModel.Get(element);
    // Assuming the node exists
    if (targetNode != null)
    {
        // Loop per each reference and list it in the console output
        foreach (var item in targetNode.Refs.GetReferences())
        {
            Log.Info("References", $"ReferenceTypeId: {item.ReferenceTypeId}, TargetNodeId: {item.TargetNodeId}, TargetNode: {item.TargetNode.BrowseName}");
        }
    }
    else
    {
        Log.Info("References", "Cannot find such node in the current project");
    }
}
```

## Read arbitrary tag from an OPC/UA Server

> [!WARNING]
> This method involves some low-level OPC/UA operations and may not be recommended for production usage. It is recommended to use the OPC/UA Client TagImporter instead.

This method allows to read any OPC/UA variables from a remote server knowing its `NameSpaceIndex` and `ID`. The following examples reads the `Server > ServerStatus > State` (0/2259) variable from the UaAnsiCServer from the OPC Foundation 

```csharp
// This list comes from the OPC/UA specification
enum OpcAttributes
{
    NodeId = 1,
    NodeClass = 2,
    BrowseName = 3,
    DisplayName = 4,
    Description = 5,
    WriteMask = 6,
    UserWriteMask = 7,
    IsAbstract = 8,
    Symmetric = 9,
    InverseName = 10,
    ContainsNoLoops = 11,
    EventNotifier = 12,
    Value = 13,
    DataType = 14,
    ValueRank = 15,
    ArrayDimensions = 16,
    AccessLevel = 17,
    UserAccessLevel = 18,
    MinimumSamplingInterval = 19,
    Historizing = 20,
    Executable = 21,
    UserExecutable = 22,
    DataTypeDefinition = 23,
    RolePermissions = 24,
    UserRolePermissions = 25,
    AccessRestrictions = 26,
    AccessLevelEx = 27,
    BrowsePath = 64,
    Quality = 65,
    StatusCode = 66,
    SourceTimestamp = 67,
    ServerTimestamp = 68,
    ActualDataType = 69,
    ActualArrayDimensions = 70
};

[ExportMethod]
public void ReadArbitraryValueFromRemoteOpcUaServer(int nameSpaceIndex, uint identifier)
{
    // Get the OPC/UA Client from the project
    var opc = Project.Current.Get<OPCUAClient>("OPC-UA/OPCUAClient1");

    // OPC/UA Variable to read from the remote server
    object[] opcVariable = new object[3]
    {
        new NodeId(nameSpaceIndex, identifier), // NodeId of the variable to read (UaExpert can be used to get these values)
        OpcAttributes.Value, // Attribute ID to read (13 = Value), see enumeration above
        new uint[0] // Indexes, for simple scalar tags this is an empty array
    };

    // Create a Struct object that holds the information of the remote variable to be passed to the OPC/UA Client
    // Syntax: new Struct(NodeId dataTypeId, object[] values)
    // Where:
    // - DataTypeId is the NodeId of the DataType expected as input argument of the method (50 is the ReadValues DataType)
    // - Values is an array containing the information of the variable to read
    UAManagedCore.Struct asd = new Struct(new NodeId(FTOptix.OPCUAClient.ObjectTypes.OPCUAClient.NamespaceIndex, 50), opcVariable);

    // The ExecuteMethod method of the OPC/UA Client is used to call the ReadValues method of the OPC/UA Server
    // This expects an object[] as input parameters, so we will create a mono-dimensional array with the Struct object
    object[] inputArgs = new object[1];
    // The first element of the array is the Struct object that we created
    inputArgs[0] = new UAManagedCore.Struct[1];
    // The first element of the Struct array is the variable information to read
    ((UAManagedCore.Struct[])inputArgs[0])[0] = asd;

    // The ExecuteMethod method returns an object[] with the output parameters of the method
    object[] outputArgs = new object[1];

    // Call the ReadValues method of the OPC/UA Server
    opc.ExecuteMethod("ReadValues", inputArgs, out outputArgs); // OPCUA\OPCUAClient\OPCUAClient\Module.xml.in

    // The outputArgs array contains the output parameters of the ReadValues method
    var output = (UAManagedCore.Struct[])outputArgs[0];
    Log.Info(output[0].Values[0].ToString());
}
```

## Change the Status property of a Tag (changing the DataValue structure)

This may be needed in some OPC/UA companion specifications where the Status of a tag is used to detect other status that are not only good or bad

```csharp
[ExportMethod]
public void UpdateStatusCode()
{
    var myVariable = Project.Current.GetVariable("Model/MyFolder/Test");
    // The StatusCode shortcut is read-only
    Log.Info($"Original status code: {myVariable.StatusCode}");
    // new DataValue(UAValue value, uint statusCode, DateTime sourceTimestamp)
    // new DataValue(UAValue value, uint statusCode, DateTime sourceTimestamp, DateTime serverTimestamp)
    myVariable.DataValue = new DataValue(myVariable.Value, 100, DateTime.UtcNow);
    // The StatusCode shortcut is read-only
Log.Info($"New status code: {myVariable.StatusCode}");
}
```