# Dynamic Links

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
    var myVariable = (String)originalPen.FindVariable("DynamicLink").Value;
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
    var dynamicLinkNode = textVariable.Children.Get("DynamicLink");
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

### Advanced dynamic links

> [!WARNING]
> Advanced dynamic links are a very advanced topic and should be used with caution. It is recommended to use the default dynamic link APIs unless you have a specific use case.

#### Creating a String Formatter

```csharp
// Create a new variable
var variable4 = InformationModel.MakeVariable("Variable4", OpcUa.DataTypes.String); 
// Add the variable to the Model folder
Project.Current.Get("Model").Add(variable4); 
// Create a string formatter
var stringFormatter = InformationModel.Make<StringFormatter>("StringFormatter1"); 
// Set the base text (with placeholders)
stringFormatter.Format = "{0} and {1}"; 
// Configure placeholders
var source0 = InformationModel.MakeVariable("Source0", OpcUa.DataTypes.BaseDataType); 
stringFormatter.Refs.AddReference(FTOptix.CoreBase.ReferenceTypes.HasSource, source0); 
var source1 = InformationModel.MakeVariable("Source1", OpcUa.DataTypes.BaseDataType); 
stringFormatter.Refs.AddReference(FTOptix.CoreBase.ReferenceTypes.HasSource, source1); 
// Add the new converter to the variable
variable4.SetConverter(stringFormatter);
// Set the targets for the dynamic links 
var variable1 = Project.Current.GetVariable("Model/Variable1"); 
var variable2 = Project.Current.GetVariable("Model/Variable2"); 
source0.SetDynamicLink(variable1); 
source1.SetDynamicLink(variable2);
```

#### Creating an Expression Evaluator

```csharp
[ExportMethod]
public void Method1()
{
    // Get the source variables
    var var1 = Project.Current.GetVariable("Model/Variable1");
    var var2 = Project.Current.GetVariable("Model/Variable2");
    var var3 = Project.Current.GetVariable("Model/Variable3");
    // Create the expression evaluator object
    var expressionEvaluator = InformationModel.MakeObject<ExpressionEvaluator>("ExpressionEvaluator");
    // Set the format
    expressionEvaluator.Expression = "{0} + {1}";
    // Set the dynamic links to the source variables
    var source0 = InformationModel.MakeVariable("Source0", OpcUa.DataTypes.BaseDataType);
    source0.SetDynamicLink(var2);
    var source1 = InformationModel.MakeVariable("Source1", OpcUa.DataTypes.BaseDataType);
    source1.SetDynamicLink(var3);
    // Add the references (OPCUA specs)
    expressionEvaluator.Refs.AddReference(FTOptix.CoreBase.ReferenceTypes.HasSource, source0);
    expressionEvaluator.Refs.AddReference(FTOptix.CoreBase.ReferenceTypes.HasSource, source1);
    // Add the converter to the variable
    var1.SetConverter(expressionEvaluator);
}
```

#### Creating a Key-Value Converter

```csharp
/// <summary>
/// Creates a ValueMapConverter with multiple key-value pairs and associates it with a Label control.
/// The converter maps integer values to localized text strings, either through dynamic links or hard-coded values.
/// </summary>
[ExportMethod]
public void CreateKeyValueConverter()
{
    // Define constants for object paths and names
    const string parentBrowsePath = "Display/Popup1/Content";
    const string labelBrowseName = "Label99";
    const string aliasName = "{PopupAlias}";

    // Get the parent container where the label will be created
    var parent = Project.Current.Get(parentBrowsePath);
    if (parent == null)
    {
        throw new ArgumentException($"No parent {parentBrowsePath} found");
    }

    // Remove existing label if it exists to ensure clean recreation
    var label = parent.Get<Label>(labelBrowseName);
    label?.Delete();

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
    AddPairToKeyValueConverter(pairs, 0, null, new LocalizedText("Default value", "en-US"));
    // Pair 1: Dynamic link to first localized text variable
    AddPairToKeyValueConverter(pairs, 1, $"{aliasName}/localizedText1", null);
    // Pair 2: Dynamic link to second localized text variable
    AddPairToKeyValueConverter(pairs, 2, $"{aliasName}/localizedText2", null);
    // Pair 3: Hard-coded localized text for value 3
    AddPairToKeyValueConverter(pairs, 3, null, new LocalizedText("text associated to value 3 (hard-coded)", "en-US"));

    // Add the configured pairs collection to the converter
    converter.Add(pairs);

    // Configure the converter's source: link it to an integer variable that determines which value to display
    converter.SourceVariable.SetDynamicLink(null, DynamicLinkMode.Read);
    var srcDl = converter.SourceVariable.Refs.GetVariable(FTOptix.CoreBase.ReferenceTypes.HasDynamicLink) as IUAVariable;
    if (srcDl != null)
        srcDl.Value = $"{aliasName}/integer1";

    // Create and configure the Label control that will display the converted text
    label = InformationModel.MakeObject<Label>(labelBrowseName);
    label.Width = 255;
    label.LeftMargin = 165;
    label.TopMargin = 240;
    label.TextColor = Colors.Blue;
    
    // Associate the label's text variable with the converter
    // The label will automatically display text based on the integer value
    label.TextVariable.SetConverter(converter);
    
    // Add the configured label to the parent container
    parent.Add(label);
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

#### Creating a Conditional Converter

```csharp
private void StuffCreateConditionalConverter(IUAVariable targetNode, IUAVariable sourceVariable)
{
    // Create the conditional converter
    ConditionalConverter newConditionalConverter = InformationModel.MakeObject<ConditionalConverter>("ConditionalConverter1", FTOptix.CoreBase.ObjectTypes.ConditionalConverter);
    // Set the "false" condition
    newConditionalConverter.FalseValueVariable.DataType = FTOptix.Core.DataTypes.Color;
    newConditionalConverter.FalseValueVariable.Value = System.Drawing.Color.Red.ToArgb();
    // Set the "true" condition
    newConditionalConverter.TrueValueVariable.DataType = FTOptix.Core.DataTypes.Color;
    newConditionalConverter.TrueValueVariable.Value = System.Drawing.Color.Green.ToArgb();
    // Add the source variable
    newConditionalConverter.ConditionVariable.SetDynamicLink(sourceVariable);
    // Set the dynamic link to the object
    newConditionalConverter.Mode = DynamicLinkMode.Read;
    targetNode.SetConverter(newConditionalConverter);
}
```
