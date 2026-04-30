# Dynamic Links

> [!IMPORTANT]
> Starting from FactoryTalk Optix 1.7.x, if the DynamicLink optimization is enabled at `Project Root > RuntimeOptimization > DynamicLinks`, all DynamicLinks in the project will be automatically converted to StaticLinks before the deployment and the `DynamicLink` node wil effectively be removed, therefore, any DynamicLink manipulation at runtime with any of the snippets below will not be possible. If a specific link needs to be manipulated at runtime, it should be manually excluded from the optimization by navigating to the DynamicLink popup in the IDE, then in the Advanced tab click the gear icon and set `Disabled` in the `Runtime Optimization` menu.

> [!NOTE]
> **API Version Guide:** This documentation covers multiple FactoryTalk Optix versions. When creating dynamic links to aliases, use `SetDynamicLinkToAlias()` for **version 1.8.x and later** (recommended), or the manual path construction approach for **version 1.7.x and earlier** (legacy). The modern API is simpler, more maintainable, and type-safe.

## Simple Dynamic Link

By default a `DynamicLink` is made as _Read only_ unless you specify different mode

```csharp
    ...
    // Get the element where the dynamic link has to be added
    var myObj = Owner.Get<Motor>("Motor1");
    // Reach the specific variable
    var speedValue = Owner.GetObject("SpeedLabel").GetVariable("Text");
    // Create the dynamic link and specify the mode (optional - read only if empty)
    myObj.SpeedVariable.SetDynamicLink(speedValue, DynamicLinkMode.Read);
    ...
```

### Dynamic link manipulation

> [!WARNING]
> Manipulating a dynamic link is a very advanced topic and should be used with caution. It is recommended to use the default dynamic link APIs unless you have a specific use case.

#### Resolve a DynamicLink

```csharp
    ...
    // Get the element containing a DynamicLink
    TrendPen originalPen = InformationModel.Get<TrendPen>(Owner.GetAlias("AliasPen").NodeId);
    // Read the variable value
    var myVariable = (String)originalPen.Refs.GetVariable(FTOptix.CoreBase.ReferenceTypes.HasDynamicLink)?.Value;
    // Resolve the dynamic link target
    var result = LogicObject.Context.ResolvePath(myVariable);
    var resolvedNode = result.ResolvedNode;
    // Set the target variable somewhere in the project
    Owner.Find<ComboBox>("ComboBox1").SelectedItem = resolvedNode.Owner.NodeId;
    ...
```

```csharp
    ...
    // Get the element containing a DynamicLink
    var textVariable = Owner.Children.Get<IUAVariable>("Text");
    // Get the DynamicLink node
    var dynamicLinkNode = textVariable.Refs.GetVariable(FTOptix.CoreBase.ReferenceTypes.HasDynamicLink);
    // Get the pointed node
    var pointedNode = dynamicLinkNode.Refs.GetNodes(FTOptix.Core.ReferenceTypes.Resolves).First();
    ...
```

#### Check for a DynamicLink

```csharp
// Check if the current node has a dynamic link
var dynamicLink = inputVariable.Refs.GetVariable(FTOptix.CoreBase.ReferenceTypes.HasDynamicLink);
// Retrieve the path of the dynamic link
var dynamicLinkPath = (string)dynamicLink.Value;
// Resolve the dynamic link and get to the target node
var targetVariable = LogicObject.Context.ResolvePath(inputVariable, dynamicLinkPath).ResolvedNode;
```

#### Check for broken DynamicLink

```csharp
[ExportMethod]
public void GetBrokenDynamicLinks()
{
    var nodesWithBrokenDynamicLink = new List<string>();
    var projectNodes = Project.Current.Parent.FindNodesByType<IUANode>();
 
    foreach (var n in projectNodes)
    {
        // Check if the current node has a dynamic link
        var dynamicLink = n.Refs.GetVariable(FTOptix.CoreBase.ReferenceTypes.HasDynamicLink);
        if (dynamicLink == null) continue;
        // Retrieve the path of the dynamic link
        var dynamicLinkPath = (string)dynamicLink.Value;
        // Resolve the dynamic link and get to the target node
        var targetVariable = LogicObject.Context.ResolvePath(n, dynamicLinkPath).ResolvedNode;
            if (targetVariable == null)
        {
            nodesWithBrokenDynamicLink.Add("Node BrowsePath: " + Log.Node(n));
        }
    }
 
    foreach (var info in nodesWithBrokenDynamicLink)
    {
        Log.Info(info);
    }
}
```

#### Change the EU Mode of a DynamicLink

```csharp
[ExportMethod]
public void SetEuModeOfText()
{
    // Get to the label that will display the value of the variable
    var myLabelText = Owner.GetVariable("Label1/Text");
    // Get the dynamic link at the text property
    var dynamicLink = myLabelText.Refs.GetVariable(FTOptix.CoreBase.ReferenceTypes.HasDynamicLink);
    // Set the EU mode of the dynamic link to "Localize"
    ((DynamicLink)dynamicLink).EUMode = DynamicLinkEUMode.SetParentLocalizedEngineeringUnit;
    // Set the EU mode of the dynamic link to "Set Parent as Source" (no conversion)
    //((DynamicLink)dynamicLink).EUMode = DynamicLinkEUMode.SetParentEngineeringUnitAsSource;
}
```

#### Dynamic Link to a bit indexed word

```csharp
[ExportMethod]
public void AddDynamicLinkToBitOfIntegerVariable()
{
    // Get the variable
    var myBitAlarm = Project.Current.GetVariable("Model/BitVariable");
    // Create a fake variable
    IUAVariable tempVariable = null;
    // Add a dynamic link to the fake variable (to trigger the link mechanism)
    myBitAlarm.SetDynamicLink(tempVariable, DynamicLinkMode.ReadWrite);
    // Replace the dynamic link content to the real target value using HasDynamicLink reference
    var myBitAlarmDl = myBitAlarm.Refs.GetVariable(FTOptix.CoreBase.ReferenceTypes.HasDynamicLink) as IUAVariable;
    if (myBitAlarmDl != null)
        myBitAlarmDl.Value = "../Int32Variable.1";
}
```

#### Dynamic Link to a single element of array variable

```csharp
private void StuffDynamicLinkToArrayElement(IUAVariable sourceVariable, IUAVariable targetDynamicLink, uint sourceVariableArrayIndex, int targetDynamicLinkArrayIndex, DynamicLinkMode dynamicLinkMode)
{
    // Prepare the BrowseName for the dynamic link variable
    string dynamicLinkVariableBrowseName = $"DynamicLink_{sourceVariableArrayIndex}";
    // Create the dynamic link object
    DynamicLink newDynamicLink = InformationModel.MakeVariable<DynamicLink>(dynamicLinkVariableBrowseName, FTOptix.Core.DataTypes.NodePath);
    // Set the dynamic link values
    newDynamicLink.Value = DynamicLinkPath.MakePath(sourceVariable, targetDynamicLink);
    newDynamicLink.Mode = dynamicLinkMode;
    // Get the parent element access variable as Struct for getting the values
    Struct parentElementAccess = newDynamicLink.ParentElementAccessVariable.Value;
    // Generate the new values for the parent element access variable (the first element is the index in case of an array, the second is the field index in case of struct)
    var newValues = parentElementAccess.Values.ToList();
    // Set the target index as array of uint (OPCUA specs)
    newValues[0] = new uint[]{sourceVariableArrayIndex};
    // Overwrite the struct with the new values
    newDynamicLink.ParentElementAccessVariable.Value = new Struct(parentElementAccess.DataTypeId,newValues);
    // Check if the target of dynamic link is an array
    if (targetDynamicLinkArrayIndex >= 0)
    {
        //If it is set the index by adding square brackets with inside the index value at the end of the path
        newDynamicLink.Value = $"{newDynamicLink.Value.Value}[{targetDynamicLinkArrayIndex}]";
    }
    // Add the dynamic link reference to the dynamic link (OPCUA specs)
    sourceVariable.Refs.AddReference(FTOptix.CoreBase.ReferenceTypes.HasDynamicLink, newDynamicLink);
    // Set the proper modelling rule (OPCUA specs)
    newDynamicLink.SetModellingRuleRecursive();
}
```

### Create a dynamic link to a specific element of an array variable

```csharp
/// <summary>
/// Creates a DynamicLink variable that points to a single element of a target array variable.
/// Useful when you want to map one element of an array to another array element.
/// </summary>
[ExportMethod]
public void StuffDynamicLinkToArrayElement(IUAVariable targetVariable, IUAVariable targetDynamicLink, uint arrayIndexToLink, int targetDynamicLinkArrayIndex, DynamicLinkMode dynamicLinkMode)
{
    // Prepare the BrowseName for the dynamic link variable
    string dynamicLinkVariableBrowseName = $"DynamicLink_{arrayIndexToLink}";
    // Create the dynamic link object
    DynamicLink newDynamicLink = InformationModel.MakeVariable<DynamicLink>(dynamicLinkVariableBrowseName, FTOptix.Core.DataTypes.NodePath);
    // Set the dynamic link base path and mode
    newDynamicLink.Value = DynamicLinkPath.MakePath(targetVariable, targetDynamicLink);
    newDynamicLink.Mode = dynamicLinkMode;
    newDynamicLink.ParentArrayIndexVariable.Value = arrayIndexToLink;
    // If the target is an array element, append the index
    if (targetDynamicLinkArrayIndex >= 0)
    {
        newDynamicLink.Value = $"{newDynamicLink.Value.Value}[{targetDynamicLinkArrayIndex}]";
    }
    // Add the dynamic link reference to the source variable
    targetVariable.Refs.AddReference(FTOptix.CoreBase.ReferenceTypes.HasDynamicLink, newDynamicLink);
    // Ensure modelling rules are set
    newDynamicLink.SetModellingRuleRecursive();
}
```

### Find broken dynamic links in the project

> [!TIP]
> This feature is now available as native functionality in FactoryTalk Optix starting from version 1.6.x.

```csharp
/// <summary>
/// Scans project nodes searching for DynamicLink variables that cannot be resolved.
/// Logs the browse path of nodes with broken dynamic links.
/// </summary>
[ExportMethod]
public void GetBrokenDynamicLinks()
{
    var nodesWithBrokenDynamicLink = new List<string>();
    var projectNodes = Project.Current.Parent.FindNodesByType<IUANode>();

    foreach (var n in projectNodes)
    {
        var dynamicLink = n.Refs.GetVariable(FTOptix.CoreBase.ReferenceTypes.HasDynamicLink);
        if (dynamicLink == null) continue;
        var dynamicLinkPath = (string)dynamicLink.Value;
        var targetVariable = LogicObject.Context.ResolvePath(n, dynamicLinkPath).ResolvedNode;
        if (targetVariable == null)
        {
            nodesWithBrokenDynamicLink.Add("Node BrowsePath: " + Log.Node(n));
        }
    }

    foreach (var info in nodesWithBrokenDynamicLink)
    {
        Log.Info(info);
    }
}
```

#### Formatted dynamic link

A formatted dynamic link is very similar to a string formatter and it is used to dynamically reference different objects in the project, such as using the same widget to control different motors using a SpinBox

```csharp
public void StuffCreateNewDynamicLinkFormatter()
{
    // Get to the text box where the dynamic link has to be created
    IUAVariable targetVariable = Owner.Get("TextBox1").GetVariable("Text");
    // Remove dynamic link (if any)
    targetVariable.ResetDynamicLink();
    // Create a new empty dynamic link object
    DynamicLink newDynamicLink = InformationModel.MakeVariable<DynamicLink>("DynamicLink", FTOptix.Core.DataTypes.NodePath);
    newDynamicLink.Value = "";
    // Create a string formatter to be added to the dynamic link
    StringFormatter newStringFormatter = InformationModel.MakeObject<StringFormatter>("DynamicLinkFormatter", FTOptix.CoreBase.ObjectTypes.StringFormatter);
    // Set the proper format for the string formatter, where curly brackets are used as placeholders
    newStringFormatter.Format = "/Objects/StuffTest/Model/Folder1/Variable5[{0},{1}]";
    // Set the sources for the placeholders
    IUAVariable source0 = InformationModel.MakeVariable("Source0", OpcUa.DataTypes.BaseDataType);
    IUAVariable source1 = InformationModel.MakeVariable("Source1", OpcUa.DataTypes.BaseDataType);
    source0.SetDynamicLink(Project.Current.GetVariable("Model/Folder1/Variable1"));
    source1.SetDynamicLink(Project.Current.GetVariable("Model/Folder1/Variable2"));
    // Add the references about the children variables to the string formatter
    newStringFormatter.Refs.AddReference(FTOptix.CoreBase.ReferenceTypes.HasSource, source0);
    newStringFormatter.Refs.AddReference(FTOptix.CoreBase.ReferenceTypes.HasSource, source1);
    // Set the dynamic link mode (if needed)
    newDynamicLink.Mode = DynamicLinkMode.ReadWrite;
    // Set the proper references and modelling rules (OPCUA specs)
    newDynamicLink.Refs.AddReference(FTOptix.CoreBase.ReferenceTypes.HasConverter, newStringFormatter);
    newStringFormatter.SetModellingRuleRecursive();
    targetVariable.Refs.AddReference(FTOptix.CoreBase.ReferenceTypes.HasDynamicLink, newDynamicLink);
    newDynamicLink.SetModellingRuleRecursive();       
}
```

### Dynamic link to an Alias

#### FactoryTalk Optix 1.8.x and later (Recommended)

> [!NOTE]
> The `SetDynamicLinkToAlias()` API is the recommended approach for FactoryTalk Optix 1.8.x and later. It provides a cleaner, more intuitive way to create dynamic links to alias nodes without manual path construction.

##### Link to a direct child of an alias

```csharp
[ExportMethod]
public void CreateDynamicLinkToAlias()
{
    // Get the Text variable of a TextBox
    var myVariable = Owner.GetVariable("TextBox1/Text");
    // Link to {MotorAlias}/Speed in ReadWrite mode
    myVariable.SetDynamicLinkToAlias("MotorAlias", new string[] { "Speed" }, DynamicLinkMode.ReadWrite);
}
```

##### Link to a nested path inside an alias

```csharp
[ExportMethod]
public void CreateDynamicLinkToNestedAlias()
{
    // Get the Text variable of a Label
    var myVariable = Owner.GetVariable("Label1/Text");
    // Link to {MotorAlias}/Configuration/MaxSpeed in Read mode (default)
    myVariable.SetDynamicLinkToAlias("MotorAlias", new string[] { "Configuration", "MaxSpeed" });
}
```

##### Link to a specific OPC UA attribute of an alias node

```csharp
[ExportMethod]
public void CreateDynamicLinkToAliasAttribute()
{
    // Get the Text variable of a Label
    var myVariable = Owner.GetVariable("Label1/Text");
    // Link to the BrowseName attribute of {MotorAlias} itself
    myVariable.SetDynamicLinkToAlias("MotorAlias", AttributeId.BrowseName, DynamicLinkMode.Read);
}
```

##### Link to a specific OPC UA attribute of a sub-item inside an alias

```csharp
[ExportMethod]
public void CreateDynamicLinkToAliasSubItemAttribute()
{
    // Get the Text variable of a Label
    var myVariable = Owner.GetVariable("Label1/Text");
    // Link to the BrowseName attribute of {MotorAlias}/Configuration/Speed
    myVariable.SetDynamicLinkToAlias("MotorAlias", new string[] { "Configuration", "Speed" }, AttributeId.BrowseName, DynamicLinkMode.Read);
}
```

#### FactoryTalk Optix up to 1.7.x (Legacy)

> [!WARNING]
> This approach is deprecated and only needed for FactoryTalk Optix versions prior to 1.8.x. Use the `SetDynamicLinkToAlias()` API above for newer versions.

```csharp
[ExportMethod]
public void CreateDynamicLinkToAlias()
{
    var myVariable = Owner.GetVariable("TextBox1/Text");
    myVariable.SetDynamicLink(null, DynamicLinkMode.ReadWrite);
    var myVariableDl = myVariable.Refs.GetVariable(FTOptix.CoreBase.ReferenceTypes.HasDynamicLink);
    if (myVariableDl != null)
        myVariableDl.Value = "{MotorAlias}/Speed";
}
```

## Advanced dynamic links

> [!WARNING]
> Advanced dynamic links are a very advanced topic and should be used with caution. It is recommended to use the default dynamic link APIs unless you have a specific use case.

### Understanding Converters

Converters transform values flowing through dynamic links. Before creating converters, understand these key concepts:

**SetConverter Behavior**: When you call `variable.SetConverter(converter)`:
- Any existing converter on the variable is automatically deleted
- Any existing dynamic link on the variable is automatically deleted
- The new converter becomes the transformation layer for that variable

**Converter Modes**:
- `DynamicLinkMode.Read` (or default) - One-way conversion: source → target only
- `DynamicLinkMode.ReadWrite` - Two-way conversion: allows source ↔ target updates
- `DynamicLinkMode.OneWayToSource` - One-way in opposite direction: target → source only

**Source Variable Naming**: For `StringFormatter` and `ExpressionEvaluator`, source variables **must** follow this naming pattern:
- `Source0`, `Source1`, `Source2`, etc.
- The number must match the placeholder index in the format/expression string

### Creating Converters

##### Creating a String Formatter

A `StringFormatter` converter combines multiple input variables into a formatted string using placeholders like `{0}`, `{1}`, etc.

```csharp
[ExportMethod]
public void CreateStringFormatterConverter()
{
    // Create a target variable that will display the formatted string
    var displayLabel = Owner.Get<Label>("Label1").TextVariable;
    
    // Create a StringFormatter converter with placeholders
    var stringFormatter = InformationModel.Make<StringFormatter>("StringFormatter1");
    stringFormatter.Format = "Temperature: {0}°C, Humidity: {1}%";
    
    // Create source variables that will feed the placeholders
    // Sources MUST be named "Source0", "Source1", etc. to match placeholder indices
    var source0 = InformationModel.MakeVariable("Source0", OpcUa.DataTypes.BaseDataType);
    var source1 = InformationModel.MakeVariable("Source1", OpcUa.DataTypes.BaseDataType);
    
    // Add the source variables to the converter
    stringFormatter.Refs.AddReference(FTOptix.CoreBase.ReferenceTypes.HasSource, source0);
    stringFormatter.Refs.AddReference(FTOptix.CoreBase.ReferenceTypes.HasSource, source1);
    
    // Connect the sources to actual variables via dynamic links
    var temperatureVariable = Project.Current.GetVariable("Model/Temperature");
    var humidityVariable = Project.Current.GetVariable("Model/Humidity");
    source0.SetDynamicLink(temperatureVariable);
    source1.SetDynamicLink(humidityVariable);
    
    // Apply the converter to the target variable
    // Note: SetConverter automatically removes any previous converter or dynamic link
    displayLabel.SetConverter(stringFormatter);
}
```

###### Important Notes
- Source variables **must** be named `Source0`, `Source1`, `Source2`, etc., matching the placeholder indices in the format string
- The `SetConverter()` method automatically removes any previous converter or dynamic link from the target variable

##### Creating an Expression Evaluator

An `ExpressionEvaluator` converter evaluates a mathematical expression using input variables as operands. This is useful for calculations like scaling, offset, or combining multiple values.

```csharp
[ExportMethod]
public void CreateExpressionEvaluatorConverter()
{
    // Get the target variable where the calculated result will be displayed
    var resultLabel = Owner.Get<Label>("Label1").TextVariable;
    
    // Get the source variables to use in the expression
    var voltageVariable = Project.Current.GetVariable("Model/Voltage");
    var currentVariable = Project.Current.GetVariable("Model/Current");
    
    // Create the expression evaluator converter
    var expressionEvaluator = InformationModel.MakeObject<ExpressionEvaluator>("PowerCalculator");
    
    // Set a mathematical expression combining placeholders
    // In this example: Power (Watts) = Voltage * Current
    expressionEvaluator.Expression = "{0} * {1}";
    
    // Create source variables that will feed the expression
    // Sources MUST be named "Source0", "Source1", etc., matching placeholder indices
    var source0 = InformationModel.MakeVariable("Source0", OpcUa.DataTypes.BaseDataType);
    var source1 = InformationModel.MakeVariable("Source1", OpcUa.DataTypes.BaseDataType);
    
    // Add source variables to the evaluator
    expressionEvaluator.Refs.AddReference(FTOptix.CoreBase.ReferenceTypes.HasSource, source0);
    expressionEvaluator.Refs.AddReference(FTOptix.CoreBase.ReferenceTypes.HasSource, source1);
    
    // Connect sources to actual variables via dynamic links
    source0.SetDynamicLink(voltageVariable);
    source1.SetDynamicLink(currentVariable);
    
    // Apply the converter to the target variable
    resultLabel.SetConverter(expressionEvaluator);
}
```

###### Supported Expression Syntax

The expression evaluator supports multiple operators and functions, please see the full list in the [official documentation](https://www.rockwellautomation.com/en-us/docs/factorytalk-optix/1-7-0/contents-ditamap/creating-projects/converters/expression-evaluator.html).

##### Creating a Key-Value Converter (Instance)

A `ValueMapConverter` instance maps discrete values (like integers) to other values (like localized text or colors). This is useful for displaying status text or changing colors based on numeric states.

This section creates and applies a converter instance directly to a target variable (the `Text` property of a `Label`).

###### Simple Example Using Pairs Object

```csharp
[ExportMethod]
public void CreateSimpleKeyValueConverter()
{
    // Get the target variable that will display the mapped value
    var statusLabel = Owner.Get<Label>("Label1").TextVariable;
    
    // Create the ValueMapConverter (must pass the ObjectType explicitly)
    var converter =
        InformationModel.MakeObject<ValueMapConverter>("StatusConverter", FTOptix.CoreBase.ObjectTypes.ValueMapConverter);
    
    // Create the Pairs container using a QualifiedName with the converter's namespace
    var pairs =
        InformationModel.MakeObject(new QualifiedName(FTOptix.CoreBase.ObjectTypes.ValueMapConverter.NamespaceIndex, "Pairs"));
    pairs.ModellingRule = NamingRuleType.Mandatory;
    converter.Add(pairs);
    
    // Add simple key-value pairs
    AddSimplePair(pairs, 0, "Idle");
    AddSimplePair(pairs, 1, "Running");
    AddSimplePair(pairs, 2, "Error");
    AddSimplePair(pairs, 3, "Maintenance");
    
    // Link the converter's source to the state variable
    var deviceStateVariable = Project.Current.GetVariable("Model/DeviceState");
    converter.SourceVariable.SetDynamicLink(deviceStateVariable);
    
    // Apply the converter to the target variable
    statusLabel.SetConverter(converter);
}

private void AddSimplePair(IUAObject pairsContainer, int key, string value)
{
    // Create a ValueMapPair object
    var pair = InformationModel.MakeObject($"Pair{key}", FTOptix.CoreBase.ObjectTypes.ValueMapPair);
    pair.ModellingRule = NamingRuleType.Mandatory;
    
    // Set the key value
    var keyVariable = pair.GetVariable("Key");
    keyVariable.ModellingRule = NamingRuleType.Mandatory;
    keyVariable.DataType = OpcUa.DataTypes.Int32;
    keyVariable.Value = key;
    
    // Set the value
    var valueVariable = pair.GetVariable("Value");
    valueVariable.ModellingRule = NamingRuleType.Mandatory;
    valueVariable.DataType = OpcUa.DataTypes.String;
    valueVariable.Value = value;
    
    // Add the pair to the Pairs container
    pairsContainer.Add(pair);
}
```

> [!NOTE]
> This simple approach creates hard-coded key-value pairs. Use the advanced approach below only if you need dynamic links for values that change at runtime based on other variables.

###### Advanced Example With Dynamic Links

```csharp
/// <summary>
/// Creates a ValueMapConverter with multiple key-value pairs, including dynamic links for runtime-resolved values.
/// </summary>
[ExportMethod]
public void CreateAdvancedKeyValueConverter()
{
    const string aliasName = "{PopupAlias}";

    // Get the target variable that will display the mapped value
    var statusLabel = Owner.Get<Label>("Label1").TextVariable;

    // Create the ValueMapConverter object that will map integer keys to localized text values
    var converter =
        InformationModel.MakeObject<ValueMapConverter>("KeyValueConverter1", FTOptix.CoreBase.ObjectTypes.ValueMapConverter);
    
    // Create the Pairs container to hold all key-value mappings
    var pairs =
        InformationModel.MakeObject(new QualifiedName(FTOptix.CoreBase.ObjectTypes.ValueMapConverter.NamespaceIndex, "Pairs"));

    // Set modeling rule to ensure pairs object follows converter structure requirements
    pairs.ModellingRule = NamingRuleType.Mandatory;

    // Add key-value pairs to the converter:
    // Pair 0: Default fallback value (hard-coded)
    AddAdvancedPair(pairs, 0, null, new LocalizedText("Default value", "en-US"));
    // Pair 1: Dynamic link to first localized text variable
    AddAdvancedPair(pairs, 1, $"{aliasName}/localizedText1", null);
    // Pair 2: Dynamic link to second localized text variable
    AddAdvancedPair(pairs, 2, $"{aliasName}/localizedText2", null);
    // Pair 3: Hard-coded localized text for value 3
    AddAdvancedPair(pairs, 3, null, new LocalizedText("text associated to value 3 (hard-coded)", "en-US"));

    // Add the configured pairs collection to the converter
    converter.Add(pairs);

    // Configure the converter's source: link it to an integer variable that determines which value to display
    converter.SourceVariable.SetDynamicLink(null, DynamicLinkMode.Read);
    var srcDl = converter.SourceVariable.Refs.GetVariable(FTOptix.CoreBase.ReferenceTypes.HasDynamicLink) as IUAVariable;
    if (srcDl != null)
        srcDl.Value = $"{aliasName}/integer1";

    // Apply the converter to the target variable
    statusLabel.SetConverter(converter);
}

/// <summary>
/// Adds a key-value pair to the ValueMapConverter's pairs collection.
/// Each pair maps an integer key to either a dynamically-linked or hard-coded localized text value.
/// </summary>
/// <param name="pairs">The pairs collection to add the new pair to</param>
/// <param name="key">The integer key that will trigger this value when matched</param>
/// <param name="dynamicLinkString">Optional dynamic link path to a variable (takes precedence if provided)</param>
/// <param name="hardCodedText">Optional hard-coded localized text value (used when no dynamic link is provided)</param>
private void AddAdvancedPair(IUAObject pairs, int key, string dynamicLinkString, LocalizedText hardCodedText)
{
    // Create unique browse name for this pair
    string pairBrowseName = $"Pair{key}";
    
    // Create the ValueMapPair object
    var newPair = InformationModel.MakeObject(pairBrowseName, FTOptix.CoreBase.ObjectTypes.ValueMapPair);
    newPair.ModellingRule = NamingRuleType.Mandatory;

    // Get the Key and Value variables from the pair
    var newKey = newPair.GetVariable("Key");
    var newValue = newPair.GetVariable("Value");
    
    // Set modeling rules to ensure proper structure
    newKey.ModellingRule = NamingRuleType.Mandatory;
    newValue.ModellingRule = NamingRuleType.Mandatory;

    // Configure the Key variable with the integer key value
    newKey.DataType = OpcUa.DataTypes.Int32;
    newKey.Value = key;

    // Configure the Value variable as LocalizedText type
    newValue.DataType = OpcUa.DataTypes.LocalizedText;

    // Set the value either through dynamic link or hard-coded value
    // Dynamic link allows runtime changes to the displayed text
    if (!string.IsNullOrEmpty(dynamicLinkString))
    {
        // Create dynamic link to external variable
        newValue.SetDynamicLink(null, DynamicLinkMode.ReadWrite);
        var pairDl = newValue.Refs.GetVariable(FTOptix.CoreBase.ReferenceTypes.HasDynamicLink) as IUAVariable;
        if (pairDl != null)
            pairDl.Value = dynamicLinkString;
    }
    else
    {
        // Use hard-coded localized text value
        newValue.SetValue(hardCodedText);
    }

    // Add the configured pair to the pairs collection
    pairs.Add(newPair);
}
```

##### Creating a Key-Value Converter (Type)

Use this approach when you want to create a reusable converter type in the project model (for example under the Converters folder), instead of applying a runtime instance directly to a target variable.

```csharp
/// <summary>
/// Creates a new ValueMapConverter type in the Converters folder with key-value pairs
/// that map integer keys to localized text values. ModellingRule properties are set after
/// nodes are added to the project to ensure correct serialization behavior.
/// </summary>
[ExportMethod]
public void CreateKeyValueConverterType()
{
    // Get the Converters folder
    var parent = Project.Current.Get("Converters");
    if (parent == null)
    {
        Log.Error("Converters folder not found in the project.");
        return;
    }

    // Create the converter type object
    var converter = InformationModel.MakeObjectType<ValueMapConverterType>("MyKeyValueConverter");

    // Create the Pairs container
    var pairs =
        InformationModel.MakeObject(new QualifiedName(FTOptix.CoreBase.ObjectTypes.ValueMapConverter.NamespaceIndex, "Pairs"));

    // Add key-value pairs
    AddPairToKeyValueConverter(pairs, 0, null, new LocalizedText("Default value", "en-US"));
    AddPairToKeyValueConverter(pairs, 1, null, new LocalizedText("text associated to value 1 (hard-coded)", "en-US"));
    AddPairToKeyValueConverter(pairs, 2, null, new LocalizedText("text associated to value 2 (hard-coded)", "en-US"));
    AddPairToKeyValueConverter(pairs, 3, null, new LocalizedText("text associated to value 3 (hard-coded)", "en-US"));

    // Add pairs to the converter and add converter to the project
    converter.Add(pairs);
    parent.Add(converter);

    // Set modelling rules after instantiation to control serialization
    pairs.ModellingRule = NamingRuleType.None;
    foreach (var pair in pairs.Children)
    {
        pair.ModellingRule = NamingRuleType.None;
        foreach (var child in pair.Children)
            child.ModellingRule = NamingRuleType.None;
    }
}

/// <summary>
/// Adds a key-value pair to the ValueMapConverter's pairs collection.
/// Each pair maps an integer key to either a dynamically-linked or hard-coded localized text value.
/// </summary>
/// <param name="pairs">The pairs collection to add the new pair to</param>
/// <param name="key">The integer key that will trigger this value when matched</param>
/// <param name="dynamicLinkString">Optional dynamic link path to a variable (takes precedence if provided)</param>
/// <param name="hardCodedText">Optional hard-coded localized text value (used when no dynamic link is provided)</param>
private void AddPairToKeyValueConverter(IUAObject pairs, int key, string dynamicLinkString, LocalizedText hardCodedText)
{
    // Create unique browse name for this pair
    string pairBrowseName = $"Pair{key}";

    // Create the ValueMapPair object
    var newPair = InformationModel.MakeObject(pairBrowseName, FTOptix.CoreBase.ObjectTypes.ValueMapPair);

    // Get the Key and Value variables from the pair
    var newKey = newPair.GetVariable("Key");
    var newValue = newPair.GetVariable("Value");

    // Configure the Key variable with the integer key value
    newKey.DataType = OpcUa.DataTypes.Int32;
    newKey.Value = key;

    // Configure the Value variable as LocalizedText type
    newValue.DataType = OpcUa.DataTypes.LocalizedText;

    // Set the value either through dynamic link or hard-coded value
    if (!string.IsNullOrEmpty(dynamicLinkString))
    {
        // Create dynamic link to external variable
        newValue.SetDynamicLink(null, DynamicLinkMode.ReadWrite);
        var pairDl = newValue.Refs.GetVariable(FTOptix.CoreBase.ReferenceTypes.HasDynamicLink) as IUAVariable;
        if (pairDl != null)
            pairDl.Value = dynamicLinkString;
    }
    else
    {
        // Use hard-coded localized text value
        newValue.SetValue(hardCodedText);
    }

    // Add the configured pair to the pairs collection
    pairs.Add(newPair);
}
```

##### Creating a Conditional Converter

A `ConditionalConverter` outputs one of two values based on a boolean condition. This is useful for switching between states, such as changing a color based on a device status.

```csharp
[ExportMethod]
public void CreateConditionalConverter()
{
    // Get the target variable where the converted value will be displayed
    var statusIndicatorColor = Owner.Get<Rectangle>("Rectangle1").FillColorVariable;
    
    // Create the conditional converter
    var conditionalConverter = InformationModel.MakeObject<ConditionalConverter>("StatusColorConverter");
    
    // Configure the true condition value (device is running - green)
    conditionalConverter.TrueValueVariable.DataType = FTOptix.Core.DataTypes.Color;
    conditionalConverter.TrueValueVariable.Value = System.Drawing.Color.Green.ToArgb();
    
    // Configure the false condition value (device is stopped - red)
    conditionalConverter.FalseValueVariable.DataType = FTOptix.Core.DataTypes.Color;
    conditionalConverter.FalseValueVariable.Value = System.Drawing.Color.Red.ToArgb();
    
    // Link the condition variable to a boolean source
    var deviceRunningVariable = Project.Current.GetVariable("Model/DeviceIsRunning");
    conditionalConverter.ConditionVariable.SetDynamicLink(deviceRunningVariable);
    
    // Set the converter mode (ReadWrite allows both directions, Read is one-way)
    conditionalConverter.Mode = DynamicLinkMode.Read;
    
    // Apply the converter to the target variable
    statusIndicatorColor.SetConverter(conditionalConverter);
}
```

###### Important Notes
- The `ConditionVariable` must be linked to a boolean variable
- `TrueValueVariable` and `FalseValueVariable` must have the same data type as the target variable
- Use `DynamicLinkMode.Read` for one-way conversion (source → target)
- Use `DynamicLinkMode.ReadWrite` to allow bidirectional updates
