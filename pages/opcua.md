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

### FactoryTalk Optix 1.6.x or older

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
        OpcAttributes.Value,                    // Attribute ID to read (13 = Value), see enumeration above
        new uint[0]                             // Indexes, for simple scalar tags this is an empty array
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

### FactoryTalk Optix 1.7.x or newer

```csharp
struct ReadResult
{
    public object Value;
    public uint StatusCode;

    public ReadResult(UAManagedCore.Struct s)
    {
        Value = s.Values[0];
        StatusCode = (uint)s.Values[1];
    }

    public bool IsGood => (StatusCode & 0xC0000000) == 0;

    public string StatusName
    {
        get
        {
            if (Enum.IsDefined(typeof(OpcUaStatusCode), StatusCode))
                return ((OpcUaStatusCode)StatusCode).ToString();
            return $"0x{StatusCode:X8}";
        }
    }
}

enum OpcUaStatusCode : uint
{
    Good = 0x00000000,
    BadUnexpectedError = 0x80010000,
    BadInternalError = 0x80020000,
    BadNodeIdInvalid = 0x80330000,
    BadAttributeIdInvalid = 0x80350000,
    BadNodeIdUnknown = 0x80340000,
    BadNotReadable = 0x803A0000,
    BadNotWritable = 0x803B0000,
    BadTypeMismatch = 0x80740000,
    BadInvalidArgument = 0x80AB0000,
    BadOutOfRange = 0x803C0000,
    BadNoCommunication = 0x80310000,
    BadTimeout = 0x800A0000,
    BadServerNotConnected = 0x800D0000,
    BadSessionIdInvalid = 0x80250000,
    BadUserAccessDenied = 0x801F0000,
    BadSecurityModeInsufficient = 0x80E60000,
    BadServiceUnsupported = 0x800B0000,
}

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
public void ReadArbitraryValueFromRemoteOpcUaServer(string namespaceUri, string identifier)
{
    // Change this to read a different OPC UA attribute
    var attribute = OpcAttributes.Value;

    // Get the OPC/UA Client from the project
    var opc = Project.Current.Get<OPCUAClient>("OPC-UA/OPCUAClient1");

    // Find the local namespace index by searching all registered namespaces.
    // FTOptix builds aggregated URIs for dynamic namespaces as "{clientGuid}#{remoteNamespaceUri}",
    // so we match a namespace URI that equals or ends with the provided remote namespace URI.
    var context = LogicObject.Context;
    var localNsIndex = context.GetNamespaceIndexes()
        .Cast<int>()
        .FirstOrDefault(i => {
            var uri = context.GetNamespaceUri(i);
            return uri == namespaceUri || uri.EndsWith("#" + namespaceUri);
        }, -1);

    if (localNsIndex < 0)
    {
        Log.Error($"Namespace URI '{namespaceUri}' not found. Have you imported some tags in the Optix project?");
        return;
    }

    FTOptix.Core.ElementAccess ea = new FTOptix.Core.ElementAccess();
    UAValue eaValue = ea; // implicit conversion to UAValue
    var emptyElementAccess = (UAManagedCore.Struct)eaValue.Value;

    var nodeIdStr = $"ns={localNsIndex};s={identifier}";

    object[] opcVariable = new object[3]
    {
        new NodeId(localNsIndex, identifier),
        (uint)attribute,
        emptyElementAccess               // ElementAccess struct, NOT uint[]
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
    opc.ExecuteMethod("ReadValues", inputArgs, out outputArgs);

    // The outputArgs array contains the output parameters of the ReadValues method
    // ReadValueResult struct has 2 fields: [0] = Value, [1] = StatusCode (UInt32)
    var output = (UAManagedCore.Struct[])outputArgs[0];
    var result = new ReadResult(output[0]);

    if (!result.IsGood)
    {
        Log.Error($"Reading attribute `{attribute}` for `{nodeIdStr}` failed: {result.StatusName} (0x{result.StatusCode:X8})");
        return;
    }

    Log.Info($"Reading attribute `{attribute}` for `{nodeIdStr}` returned: `{result.Value}`");
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

## Manually create an OPC/UA Client Tag

This script presumes that the OPC/UA Client is already configured in the project and at least a tag was imported (needed to get the NameSpaceIndex). It creates a new tag in the project that will be used to read a variable from the OPC/UA Server.

```csharp
[ExportMethod]
public void CreateArbitraryTag()
{
    // Get the NameSpaceIndex of the OPC/UA Client, the best solution would be to read it from an imported tag.
    // This must be in the `.optix` file of the project, so it must be a known index.
    var nxIndex = LogicObject.Context.GetNamespaceIndex("0ccdea9893ed5102b4fae754ef1732f7#http://www.unifiedautomation.com/DemoServer/");
    // Create a new variable in the project with the NodeId of the OPC/UA variable to read in the remote server
    var variable = LogicObject.Context.NodeFactory.MakeVariable(new NodeId(nxIndex, "Demo.Static.Scalar.Double"), "Double", OpcUa.DataTypes.Double, OpcUa.VariableTypes.BaseDataVariableType);
    // Add the variable to the Objects folder of the OPC/UA client project
    LogicObject.Owner.GetObject("Objects").Add(variable);
}
