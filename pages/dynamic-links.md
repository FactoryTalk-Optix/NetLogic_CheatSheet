# Dynamic Links

## Simple Dynamic Link

By default a `DynamicLink` is made as _Read only_ unless you specify different mode

```csharp
// Get the element where the dynamic link has to be added
var myObj = Owner.Get<Motor>("Motor1");
// Reach the specific variable
var speedValue = Owner.GetObject("SpeedLabel").GetVariable("Text");
// Create the dynamic link and specify the mode (optional - read only if empty)
myObj.SpeedVariable.SetDynamicLink(speedValue, DynamicLinkMode.Read);
```

### Dynamic Link to a bit indexed word

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
    // Replace the dynamic link content to the real target value
    myBitAlarm.GetVariable("DynamicLink").Value = "../Int32Variable.1";
}
```

### Dynamic Link to a single element of array variable

```csharp
[ExportMethod]
public void StuffDynamicLinkToArrayElement(IUAVariable sourceVariable, IUAVariable targetDynamicLink, uint sourceVariableArrayIndex, int targetDynamicLinkArrayIndex, DynamicLinkMode dynamicLinkMode)
{
    // Prepare the BrowseName for the dynamic link variable
    string dynamicLinkVariableBrowseName = $"DynamicLink_{sourceVariableArrayIndex}";
    // Create the dynamic link object
    DynamicLink newDynamicLink = InformationModel.MakeVariable<DynamicLink>(dynamicLinkVariableBrowseName, FTOptix.Core.DataTypes.NodePath);
    // Set the dynamic link values
    newDynamicLink.Value = DynamicLinkPath.MakePath(sourceVariable, targetDynamicLink);
    newDynamicLink.Mode = dynamicLinkMode;
    newDynamicLink.ParentArrayIndexVariable.Value = sourceVariableArrayIndex;
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

### Formatted dynamic link

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

## Creating a String Formatter

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

## Creating an Expression Evaluator

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

## Creating a Key-Value Converter

```csharp
private void StuffCreateKeyPair(IUAVariable targetNode, IUAVariable sourceVariable)
{
    // Create the new key value converter
    ValueMapConverter newValueMapConverter = InformationModel.MakeObject<ValueMapConverter>("KeyValueConverter1", FTOptix.CoreBase.ObjectTypes.ValueMapConverter);
    // Create the pairs object
    IUAObject newPairs = InformationModel.MakeObject(new QualifiedName(FTOptix.CoreBase.ObjectTypes.ValueMapConverter.NamespaceIndex, "Pairs"));
    // For each pair, set properties
    for (int i = 0; i < 3; i++)
    {
        // First element is the default one
        string pairBrowseName = "Pair";
        if (i > 0)
            pairBrowseName += i.ToString();
        // Create the new pair
        IUAObject newPair = InformationModel.MakeObject(pairBrowseName, FTOptix.CoreBase.ObjectTypes.ValueMapPair);
        IUAVariable newKey = newPair.GetVariable("Key");
        IUAVariable newValue = newPair.GetVariable("Value");
        // Set the proper datatype
        newKey.DataType = OpcUa.DataTypes.UInt32;
        newValue.DataType = FTOptix.Core.DataTypes.Color;
        // Set the proper value
        newKey.Value = i;
        newValue.Value = System.Drawing.Color.Violet.ToArgb();
        // Add the pair to the map
        newPairs.Add(newPair);
    }
    // Add the map to the key value converter
    newValueMapConverter.Add(newPairs);
    newValueMapConverter.SourceVariable.SetDynamicLink(sourceVariable);
    // Add the key value converter to the variable
    targetNode.SetConverter(newValueMapConverter);
}
```

## Creating a Conditional Converter

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

## Resolve a DynamicLink

```csharp
// Get the element containing a DynamicLink
TrendPen originalPen = InformationModel.Get<TrendPen>(Owner.GetAlias("AliasPen").NodeId);
// Read the variable value
var myVariable = (String)originalPen.FindVariable("DynamicLink").Value;
// Resolve the dynamic link target
var result = LogicObject.Context.ResolvePath(myVariable);
var test = result.ResolvedNode;
// Set the target variable somewhere in the project
Owner.Find<ComboBox>("ComboBox1").SelectedItem = test.Owner.NodeId;
```

## Check for a DynamicLink

```csharp
// Check if the current node has a dynamic link
var dynamicLink = inputVariable.Refs.GetVariable(FTOptix.CoreBase.ReferenceTypes.HasDynamicLink);
// Retrieve the path of the dynamic link
var dynamicLinkPath = (string)dynamicLink.Value;
// Resolve the dynamic link and get to the target node
var targetVariable = LogicObject.Context.ResolvePath(inputVariable, dynamicLinkPath).ResolvedNode;
```

## Check for broken DynamicLink

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
