# Dynamic Links

## Simple Dynamic Link

By default a `DynamicLink` is made as _Read only_ unless you specify different mode

```csharp
var myObj = Owner.Get<Motor>("Motor1");
var speedValue = Owner.GetObject("SpeedLabel").GetVariable("Text");
myObj.SpeedVariable.SetDynamicLink(speedValue, DynamicLinkMode.Read);
```

### Dynamic Link to a bit indexed word

```csharp
[ExportMethod]
public void AddDynamicLinkToBitOfIntegerVariable()
{
    var myBitAlarm = Project.Current.GetVariable("Model/BitVariable");
    IUAVariable tempVariable = null;
    myBitAlarm.SetDynamicLink(tempVariable, DynamicLinkMode.ReadWrite);
    myBitAlarm.GetVariable("DynamicLink").Value = "../Int32Variable.1";
}
```

## Creating a String Formatter

```csharp
var variable4 = InformationModel.MakeVariable("Variable4", OpcUa.DataTypes.String); 
Project.Current.Get("Model").Add(variable4); 
var stringFormatter = InformationModel.Make<StringFormatter>("StringFormatter1"); 
stringFormatter.Format = "{0} and {1}"; 
var source0 = InformationModel.MakeVariable("Source0", OpcUa.DataTypes.BaseDataType); 
stringFormatter.Refs.AddReference(FTOptix.CoreBase.ReferenceTypes.HasSource, source0); 
var source1 = InformationModel.MakeVariable("Source1", OpcUa.DataTypes.BaseDataType); 
stringFormatter.Refs.AddReference(FTOptix.CoreBase.ReferenceTypes.HasSource, source1); 
variable4.SetConverter(stringFormatter); var variable1 = Project.Current.GetVariable("Model/Variable1"); 
var variable2 = Project.Current.GetVariable("Model/Variable2"); 
source0.SetDynamicLink(variable1); 
source1.SetDynamicLink(variable2);
```
