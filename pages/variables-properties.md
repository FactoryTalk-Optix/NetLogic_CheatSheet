# Variables properties

## Adding engineering units to a variable

The engineering unit follows the OPC-UA standard and is identified by a unique ID that has been issued by the foundation, you can check [Here](https://reference.opcfoundation.org/Core/Part8/v104/docs/5.6.3).

> [!WARNING]
> It is mandatory to use IDs to generate an EUInformation

```csharp
private void GenerateEUInfo(NodeId targetVariable, uint eUID)
{
    // Check if the target variable is a valid IUAVariable
    if (InformationModel.GetVariable(targetVariable) is IUAVariable variableToAnalyze)
    {
        // Check if the variable already has an EngineeringUnits property
        // If it does, update the UnitId value
        if (variableToAnalyze.GetVariable("EngineeringUnits") is IUAVariable euInfo)
        {
            euInfo.GetVariable("UnitId").Value = eUID;
        }
        // If it doesn't, create a new EngineeringUnits property and set the UnitId value
        else
        {
            euInfo = InformationModel.MakeVariable<EUInformation>("EngineeringUnits", OpcUa.DataTypes.EUInformation);
            euInfo.QualifiedBrowseName = new QualifiedName(0,"EngineeringUnits");
            euInfo.GetVariable("UnitId").Value = eUID;
            variableToAnalyze.Refs.AddReference(OpcUa.ReferenceTypes.HasProperty, euInfo);
        }          
    }
}
```
