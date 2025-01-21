# Audit Signature (CFR21 workflow)

## Add signature to a variable

```csharp
[ExportMethod]
public void AddSignatureToVariable()
{
    var sourceVarPointer = LogicObject.GetVariable("SourceVariable") ?? throw new ArgumentException("Cannot find source variable");
    var sourceVar = InformationModel.GetVariable(sourceVarPointer.Value) ?? throw new ArgumentException("Cannot find source variable");
    // Create the audit variable
    var sourceVar = InformationModel.Make<AuditInfo>("AuditSigning Signature");
    sourceVar.Add(auditVar);
}
```

## Add signature on all bits of a word

```csharp
[ExportMethod]
public void CreateAuditOnVariableSingleBits()
{
    // Check if InputVariable is populated
    var sourceVarPointer = LogicObject.GetVariable("SourceVariable") ?? throw new ArgumentException("Cannot find source variable");
    var sourceVar = InformationModel.GetVariable(sourceVarPointer.Value) ?? throw new ArgumentException("Cannot find source variable");

    // Check if TargetFolder is populated
    var targetFolderPointer = LogicObject.GetVariable("TargetFolder") ?? throw new ArgumentException("Cannot find target folder");
    var targetFolder = InformationModel.Get<Folder>(targetFolderPointer.Value) ?? throw new ArgumentException("Cannot find target folder");

    // Prepare object for the method
    var methodArguments = new object[] { sourceVar, targetFolder };

    // Execute the method in a separate thread
    var myTask = new LongRunningTask(CreateSingleBits, methodArguments, LogicObject);
    myTask.Start();
}

private void CreateSingleBits(LongRunningTask task, object methodArgumentsObj)
{
    // Extract the method arguments
    var methodArguments = (object[]) methodArgumentsObj;
    var sourceVar = (IUAVariable) methodArguments[0];
    var targetFolder = (Folder) methodArguments[1];

    // Variable to store the number of bits
    var numberOfBits = 0;

    // Get the type of the source variable. Add new datatypes if needed
    if (sourceVar.DataType == OpcUa.DataTypes.Int32 || sourceVar.DataType == OpcUa.DataTypes.UInt32)
        numberOfBits = 32;
    else if (sourceVar.DataType == OpcUa.DataTypes.Int16 || sourceVar.DataType == OpcUa.DataTypes.UInt16)
        numberOfBits = 16;
    else if (sourceVar.DataType == OpcUa.DataTypes.Float)
        numberOfBits = 32;
    else if (sourceVar.DataType == OpcUa.DataTypes.Double)
        numberOfBits = 64;
    else
        throw new ArgumentException("Unsupported data type");

    // Create the variable for each bit and add the auditing
    for (int i = 0; i < numberOfBits; i++)
    {
        // Create the new variable
        var newVariable = InformationModel.MakeVariable($"{sourceVar.BrowseName}_Bit{i}", OpcUa.DataTypes.Boolean);
        newVariable.SetDynamicLink(sourceVar, DynamicLinkMode.ReadWrite);

        // Modify the dynamic link to read only the bit
        var dynamicLinkVariable = newVariable.GetVariable("DynamicLink");
        dynamicLinkVariable.Value = dynamicLinkVariable.Value.Value + $".{i}";

        // Create the audit variable
        var auditVar = InformationModel.Make<AuditInfo>("AuditSigning Signature");
        newVariable.Add(auditVar);

        // Check if the variable already exists and delete it
        if (targetFolder.Children.OfType<IUAVariable>().Any(v => v.BrowseName == newVariable.BrowseName))
            targetFolder.Get(newVariable.BrowseName).Delete();

        // Add the variable to the target folder
        targetFolder.Add(newVariable);
    }
}
```