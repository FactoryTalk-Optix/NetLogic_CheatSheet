# Engineering Units

Engineering units are exposed through the `EngineeringUnits` property (`EUInformation`) on analog variables (or any variable where this property was added).

When working with Engineering Units in FactoryTalk Optix, you can define custom units in an `EngineeringUnitDictionary` variable and reference them from the `EUInformation` of your variables. Alternatively, you can directly assign standard OPC UA engineering units by setting the appropriate `UnitId`.

AnalogVariables are not required to have engineering units, but when they do, it allows clients to understand the physical meaning of the variable's value and display it accordingly.

## EngineeringUnitDictionary

### Create an EngineeringUnitDictionary

The following example shows how to create (or overwrite) an `EngineeringUnitDictionary` variable in `Model` and add custom engineering units (including a `Percent` unit with `UnitId = 0`)

```csharp
using FTOptix.Core;
using FTOptix.NetLogic;
using UAManagedCore;

public class DesignTimeNetLogic1 : BaseNetLogic
{
    [ExportMethod]
    public void CreateCustomEngineeringUnitDictionary()
    {
        var model = Project.Current.Get("Model");
        if (model == null)
        {
            Log.Warning("DesignTimeNetLogic1", "Model folder not found");
            return;
        }

        const string dictionaryName = "EngineeringUnitDictionary1";

        // Recreate the dictionary to keep this example deterministic.
        var existingDictionary = model.GetVariable(dictionaryName);
        existingDictionary?.Delete();

        var dictionary = InformationModel.MakeVariable(
            dictionaryName,
            FTOptix.Core.DataTypes.EngineeringUnitDictionaryItem,
            FTOptix.Core.VariableTypes.EngineeringUnitDictionary,
            new uint[] { 0 });

        // UnitId values are project-defined for custom units.
        // Percent example: UnitId = 0.
        // Use LocalizedText(text, localeId) to populate Text/LocaleId.
        // LocalizedText("...") alone sets TextId, not Text.
        var units = new EngineeringUnitDictionaryItem[]
        {
            new EngineeringUnitDictionaryItem
            {
                UnitId = 0,
                DisplayName = new LocalizedText("%", "en-US"),
                Description = new LocalizedText("Percentage", "en-US"),
                PhysicalDimension = PhysicalDimension.None,
                Slope = 0.0,
                Intercept = 0.0
            },
            new EngineeringUnitDictionaryItem
            {
                UnitId = 10,
                DisplayName = new LocalizedText("bpm", "en-US"),
                Description = new LocalizedText("Beats per minute", "en-US"),
                PhysicalDimension = PhysicalDimension.Frequency,
                Slope = 60.0,
                Intercept = 0.0
            }
        };

        dictionary.Value = units;
        model.Add(dictionary);

        Log.Info("DesignTimeNetLogic1", $"Created {dictionaryName} with {units.Length} custom units");
    }
```

### Read an EngineeringUnitDictionary

```csharp
using FTOptix.Core;
using FTOptix.NetLogic;
using UAManagedCore;

public class DesignTimeNetLogic1 : BaseNetLogic
{
    [ExportMethod]
    public void PrintCustomEngineeringUnitDictionary()
    {
        var dictionary = Project.Current.GetVariable("Model/EngineeringUnitDictionary1");
        if (dictionary == null)
        {
            Log.Warning("DesignTimeNetLogic1", "EngineeringUnitDictionary1 not found");
            return;
        }

        if (dictionary.Value.Value is EngineeringUnitDictionaryItem[] typedUnits)
        {
            foreach (var unit in typedUnits)
            {
                Log.Info(
                    "DesignTimeNetLogic1",
                    $"UnitId={unit.UnitId}, DisplayName={unit.DisplayName}, Description={unit.Description}, " +
                    $"PhysicalDimension={unit.PhysicalDimension}, Slope={unit.Slope}, Intercept={unit.Intercept}");
            }

            return;
        }

        // Fallback in case runtime returns a raw Struct array.
        if (dictionary.Value.Value is Struct[] rawUnits)
        {
            foreach (var rawUnit in rawUnits)
            {
                var values = rawUnit.Values;
                if (values == null || values.Count < 6)
                    continue;

                Log.Info(
                    "DesignTimeNetLogic1",
                    $"UnitId={values[0]}, DisplayName={values[1]}, Description={values[2]}, " +
                    $"PhysicalDimension={values[3]}, Slope={values[4]}, Intercept={values[5]}");
            }

            return;
        }

        Log.Warning("DesignTimeNetLogic1", "Engineering unit dictionary has an unexpected value format");
    }
}
```

## EngineeringUnit

### Read engineering units of a variable

The `EngineeringUnits` node can be retrieved from an `AnalogItem` (or from a variable that contains this property) and inspected through `EUInformation` fields such as `UnitId`, `NamespaceIndex`, `DisplayName`, and `Description`.

```csharp
using FTOptix.Core;
using FTOptix.NetLogic;
using UAManagedCore;

public class RuntimeNetLogic1 : BaseNetLogic
{
    [ExportMethod]
    public void GetEUInfoDisplayName()
    {
        var analogVariable = Project.Current.GetVariable("Model/Object1/AnalogVariable1");
        if (analogVariable == null)
        {
            Log.Warning("RuntimeNetLogic1", "Variable not found");
            return;
        }

        var euInfo = analogVariable.Get<EUInformation>("EngineeringUnits");
        if (euInfo == null)
        {
            Log.Warning("RuntimeNetLogic1", "EngineeringUnits property not found");
            return;
        }

        var translation = InformationModel.LookupTranslation(euInfo.DisplayName);
        Log.Info("RuntimeNetLogic1", $"EU DisplayName: {translation.Text}");
    }
}
```

### Assign engineering units to a variable

The engineering unit follows the OPC UA model and is identified by a `UnitId`.

#### Standard engineering units

Reference: [OPC UA Engineering Units specification](https://reference.opcfoundation.org/Core/Part8/v104/docs/5.6.3)

> [!WARNING]
> - It is mandatory to set `UnitId` when generating or updating `EUInformation`.
> - Not all `UnitId` from the OPC UA specification are mapped into FactoryTalk Optix

For standard OPC UA units, set `UnitId` only.

Example below assigns **degree Celsius** (`UnitId = 4408652` from OPC UA Part 8):

```csharp
using FTOptix.Core;
using FTOptix.NetLogic;
using UAManagedCore;
using OpcUa = UAManagedCore.OpcUa;

public class DesignTimeNetLogic1 : BaseNetLogic
{
    [ExportMethod]
    public void AssignCelsiusUnit()
    {
        var targetVariable = Project.Current.GetVariable("Model/AnalogVariable1");
        if (targetVariable == null)
        {
            Log.Info("DesignTimeNetLogic1", "Target variable not found");
            return;
        }

        const int celsiusUnitId = 4408652;
        SetStandardEngineeringUnitById(targetVariable, celsiusUnitId);
        Log.Info("DesignTimeNetLogic1", $"Assigned Celsius (UnitId={celsiusUnitId}) to {targetVariable.BrowseName}");
    }

    private void SetStandardEngineeringUnitById(IUAVariable targetVariable, int euId)
    {
        var euInfo = targetVariable.GetVariable("EngineeringUnits");
        if (euInfo == null)
        {
            euInfo = InformationModel.MakeVariable<EUInformation>("EngineeringUnits", OpcUa.DataTypes.EUInformation);
            euInfo.QualifiedBrowseName = new QualifiedName(0, "EngineeringUnits");
            euInfo.ModellingRule = NamingRuleType.Mandatory;
            euInfo.SetModellingRuleRecursive();
            targetVariable.Refs.AddReference(OpcUa.ReferenceTypes.HasProperty, euInfo);
        }

        euInfo.GetVariable("UnitId").Value = euId;

        // NamespaceIndex is not required for standard OPC UA units.
        var namespaceIndexVariable = euInfo.GetVariable("NamespaceIndex");
        namespaceIndexVariable?.Delete();

        if (euInfo.ModellingRule != NamingRuleType.Mandatory)
        {
            euInfo.ModellingRule = NamingRuleType.Mandatory;
            euInfo.SetModellingRuleRecursive();
        }
    }
}
```

#### Custom engineering units from an EngineeringUnitDictionary

For custom units defined in an `EngineeringUnitDictionary`, set:

1. `UnitId` (the custom ID in your dictionary item)
2. `NamespaceIndex` (the namespace index of the dictionary variable)

If `NamespaceIndex` is missing, custom units may not resolve correctly.

Example below assigns a custom unit called **Percent** (`UnitId = 0`) from `EngineeringUnitDictionary1`:

```csharp
using FTOptix.Core;
using FTOptix.NetLogic;
using UAManagedCore;
using OpcUa = UAManagedCore.OpcUa;

public class DesignTimeNetLogic1 : BaseNetLogic
{
    [ExportMethod]
    public void AssignPercentUnit()
    {
        // This method assumes a custom unit, called "Percent" with UnitId=0 is defined in the EngineeringUnitDictionary. The dictionary should have the same NamespaceIndex of the project

        var targetVariable = Project.Current.GetVariable("Model/AnalogVariable1");
        if (targetVariable == null)
        {
            Log.Info("DesignTimeNetLogic1", "Target variable not found");
            return;
        }

        var euDictionary = Project.Current.GetVariable("Model/EngineeringUnitDictionary1");
        var customNamespaceIndex = euDictionary?.NodeId.NamespaceIndex ?? targetVariable.NodeId.NamespaceIndex;

        // Here is where the UnitId should be passed as an argument,
        // UnitId is displayed in the IDE after opening the custom EngineeringUnitDictionary
        SetEngineeringUnitById(targetVariable, 0u, true, customNamespaceIndex);
        Log.Info("DesignTimeNetLogic1", $"Assigned UnitId=0 to {targetVariable.BrowseName} (NamespaceIndex={customNamespaceIndex})");
    }

    private void SetEngineeringUnitById(IUAVariable targetVariable, uint euId, bool isCustomEngineeringUnit, int customNamespaceIndex)
    {
        var euInfo = targetVariable.GetVariable("EngineeringUnits");
        if (euInfo == null)
        {
            euInfo = InformationModel.MakeVariable<EUInformation>("EngineeringUnits", OpcUa.DataTypes.EUInformation);
            euInfo.QualifiedBrowseName = new QualifiedName(0, "EngineeringUnits");
            euInfo.ModellingRule = NamingRuleType.Mandatory;
            euInfo.SetModellingRuleRecursive();
            targetVariable.Refs.AddReference(OpcUa.ReferenceTypes.HasProperty, euInfo);
        }

        euInfo.GetVariable("UnitId").Value = (int)euId;

        if (isCustomEngineeringUnit)
            euInfo.GetOrCreateVariable("NamespaceIndex").Value = customNamespaceIndex;

        if (euInfo.ModellingRule != NamingRuleType.Mandatory)
        {
            euInfo.ModellingRule = NamingRuleType.Mandatory;
            euInfo.SetModellingRuleRecursive();
        }
    }
}
```
