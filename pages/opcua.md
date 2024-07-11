# OPC/UA

## Read arbitrary tag from an OPC/UA Server

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
