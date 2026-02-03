# Variables properties

## Read engineering units of a variable

The `EngineeringUnits` node can be retrieved from an `AnalogVariable` (or any variable where the EUInfo was added) using the `Get<T>` method, where `T` is the type of the property to retrieve, in this case `EUInformation`. Once you have the `EUInformation`, you can access its properties, such as `DisplayName`, which contains a localized text representing the engineering unit.

```csharp

/// <summary>
/// Gets the display name of the EU (Engineering Units) information.
/// </summary>
public void GetEUInfoDisplayName()
{
    // Retrieve the analog variable from the project
    var analogVariable = Project.Current.GetVariable("Model/Object1/AnalogVariable1");
    // Get the EUInformation property
    var euInfo = analogVariable.Get<EUInformation>("EngineeringUnits");
    // Lookup the translation for the display name
    var translation = InformationModel.LookupTranslation(euInfo.DisplayName);
    // Log the translation text
    Log.Info("RuntimeNetLogic1", $"Translation: {translation.Text}");
}
```

## Adding engineering units to a variable

The engineering unit follows the OPC-UA standard and is identified by a unique ID that has been issued by the foundation, you can check the [OPC-UA Engineering Units specification](https://reference.opcfoundation.org/Core/Part8/v104/docs/5.6.3).

> [!WARNING]
> It is mandatory to use IDs to generate an EUInformation

```csharp
[ExportMethod]
public void GenerateEUInfo(NodeID TargetVariable, uint eUID, bool)
{
    var targetVariable = Project.Current.GetVariable("Model/Variable1").NodeId;
    var eUID = 999;
    var isCustomEngineeringUnit = true;
    // Check if the target variable is a valid IUAVariable
    if (InformationModel.GetVariable(targetVariable) is IUAVariable variableToAnalyze)
    {
        // Check if the variable already has an EngineeringUnits property
        // If it does, update the UnitId value
        if (variableToAnalyze.GetVariable("EngineeringUnits") is IUAVariable euInfo)
        {
            euInfo.GetVariable("UnitId").Value = eUID;
            // Ensure the ModellingRule is set to Mandatory
            if (euInfo.ModellingRule != NamingRuleType.Mandatory)
            {
                euInfo.ModellingRule = NamingRuleType.Mandatory;
                euInfo.SetModellingRuleRecursive();
            }
        }
        // If it doesn't, create a new EngineeringUnits property and set the UnitId value
        else
        {
            euInfo = InformationModel.MakeVariable<EUInformation>("EngineeringUnits", OpcUa.DataTypes.EUInformation);
            euInfo.QualifiedBrowseName = new QualifiedName(0, "EngineeringUnits");
            euInfo.GetVariable("UnitId").Value = eUID;
            euInfo.ModellingRule = NamingRuleType.Mandatory;
            euInfo.SetModellingRuleRecursive();
            variableToAnalyze.Refs.AddReference(OpcUa.ReferenceTypes.HasProperty, euInfo);
        }
        if (isCustomEngineeringUnit)
        {
            euInfo.GetOrCreateVariable("NamespaceIndex").Value = targetVariable.NamespaceIndex;
        }
    }
}
```
