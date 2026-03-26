# Variables structures

Struct variables represent complex data types composed of multiple fields, allowing you to work with structured data from OPC-UA servers or other data sources. This snippet collections covers how to handle, read, and modify struct variables in FactoryTalk Optix NetLogic.

## Handle Struct Variables

When connecting to an OPC-UA server from another vendor, Optix can resolve structures as Struct variables (or arrays of structs). At the code level, this type of variable is read-only. To modify the values within it, you must extract its content (which is always expressed as an array of objects), modify the array, and then recreate a new Struct.

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

## Reading a Structure Member

Since a struct in the .NET environment is a simple array of objects, you can retrieve the index of your structure's field by accessing the StructDefinition of the datatype.

```csharp
    /// <summary>
    /// Retrieves the value of a specific field from a UA Struct variable by field name.
    /// </summary>
    /// <param name="structVariable">The structure variable</param>
    /// <param name="fieldName">The name of the field to retrieve</param>
    /// <returns>The field value, or null if not found or on error</returns>
    private object GetStructFieldValue(IUAVariable structVariable, string fieldName)
    {
        // Validate that the variable's value is a Struct
        if (structVariable.Value.Value is not Struct structValue)
        {
            Log.Warning($"Variable {structVariable.BrowseName} is not a Structure.");
            return null;
        }

        // Verify that the struct has a DataTypeId to look up its definition
        if (!structValue.HasDataTypeId)
        {
            Log.Warning($"Structure variable {structVariable.BrowseName} does not have a DataTypeId.");
            return null;
        }

        // Retrieve the DataType definition to access the field names and their order
        if (InformationModel.Get<IUADataType>(structValue.DataTypeId) is not IUADataType dataType)
        {
            Log.Warning($"DataTypeId {structValue.DataTypeId} of variable {structVariable.BrowseName} is not a valid DataType.");
            return null;
        }

        // Find the index of the field by name in the StructDefinition.Fields list
        // The index corresponds to the position in the Values array
        var index = dataType.StructDefinition.Fields.ToList().FindIndex(f => f.Name == fieldName);
        if (index == -1)
        {
            Log.Warning($"Field {fieldName} not found in Structure variable {structVariable.BrowseName}.");
            return null;
        }

        // Return the field value from the Values array using the found index
        return structValue.Values[index];
    }
```

## Writing a Structure Member

Like reading, writing requires an additional step: you must extract the entire Struct and regenerate it with the new values (see ModifyStructVariable).

```csharp
    /// <summary>
    /// Sets the value of a specific field in a UA Struct variable by field name.
    /// </summary>
    /// <param name="structIUAVariable">The UA variable containing the struct</param>
    /// <param name="fieldName">The name of the field to set</param>
    /// <param name="value">The new value for the field</param>
    /// <returns>True if the field was successfully set, false if not found or on error</returns>
    private bool SetStructFieldValue(IUAVariable structIUAVariable, string fieldName, object value)
    {
        // Validate that the variable's value is a Struct
        if (structIUAVariable.Value.Value is not Struct structureVariable)
        {
            Log.Warning(LogicObject.BrowseName, $"Variable {structIUAVariable.BrowseName} is not a Structure.");
            return false;
        }

        // Verify that the struct has a DataTypeId to look up its definition
        if (!structureVariable.HasDataTypeId)
        {
            Log.Warning(LogicObject.BrowseName, $"Structure variable {structIUAVariable.BrowseName} does not have a DataTypeId.");
            return false;
        }

        // Retrieve the DataType definition to access the field names and their order
        if (InformationModel.Get<IUADataType>(structureVariable.DataTypeId) is not IUADataType dataType)
        {
            Log.Warning(LogicObject.BrowseName, $"DataTypeId {structureVariable.DataTypeId} of variable {structIUAVariable.BrowseName} is not a valid DataType.");
            return false;
        }

        // Find the index of the field by name in the StructDefinition.Fields list
        // The index corresponds to the position in the Values array
        var index = dataType.StructDefinition.Fields.ToList().FindIndex(f => f.Name == fieldName);
        if (index == -1)
        {
            Log.Warning(LogicObject.BrowseName, $"Field {fieldName} not found in Structure variable {structIUAVariable.BrowseName}.");
            return false;
        }

        // Update the field value in the Values array using the found index and write it back to the variable
        var values = structureVariable.Values.ToArray();
        values[index] = value;
        structIUAVariable.Value = new Struct(structureVariable.DataTypeId, values);

        return true;
    }
```

### Creating TagStructure

```csharp
// Create a new TagStructure variable
var tagStructure = InformationModel.MakeVariable<FTOptix.CommunicationDriver.TagStructure>("FanArray", OpcUa.DataTypes.Structure, new[] { 11u });
```

### Generic method to retrive the index of a field

```csharp
    /// <summary>
    /// Finds the index of a specific field in a UA Struct DataType definition by field name.
    /// </summary>
    /// <param name="dataType">The UA DataType containing the struct definition</param>
    /// <param name="fieldName">The name of the field to find</param>
    /// <returns>The index of the field, or -1 if not found</returns>
    private int GetFieldIndex(IUADataType dataType, string fieldName)
    {
        // Find the index of the field by name in the StructDefinition.Fields list
        // The index corresponds to the position in the Values array
        var index = dataType.StructDefinition.Fields.ToList().FindIndex(f => f.Name == fieldName);
        if (index == -1)
        {
            Log.Warning(LogicObject.BrowseName, $"Field {fieldName} not found in DataType {dataType.BrowseName}.");
        }
        return index;
    }
```
