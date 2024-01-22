# Creating elements

## UI Elements

### Add an element to the project

Native types are the one predefined in FTOptix, these are the most simple to create and add anywhere in the project. These types exists both at RunTime and DesignTime, so the syntax does not change depending on the execution

```csharp
var newPanel = InformationModel.Make<Panel>("MyNewPanelName");
newPanel.Width = 300.0;
newPanel.HorizontalAlignment = HorizontalAlignment.Stretch;
Project.Current.Get("UI/Screens/Screen1").Add(newPanel);
```

#### Make it faster

When you need to create a lot of element in a short time (such in a `for` loop), adding each element to the page every time is not recommended as multiple async request are made to FTOptix APIs and some concurrency issues may appear, in this case we suggest to create the whole model in RAM and then add it to the page only when the whole structure is defined

```csharp
var tempPanel = InformationModel.Make<Panel>("MotorsGroup");
tempPanel.HorizontalAlignment = HorizontalAlignment.Stretch;
tempPanel.VerticalAlignment = VerticalAlignment.Stretch;
for (i = 0; i < 100; i++) {
    var newPanel = InformationModel.Make<Panel>("MyMotor" + i.ToString());
    newPanel.Width = 300.0;
    newPanel.HorizontalAlignment = HorizontalAlignment.Stretch;
    tempPanel.Add(newPanel);
}
Project.Current.Get("UI/Screens/Screen1").Add(tempPanel);
```

### Custom UI types instances (widget instances)

When creating custom types or templates, the equivalent C# class is automatically created by FTOptix, then you can instantiate such classes with the same syntax

**Please note**: The user-defined types does not exists when executing scripts at DesignTime, so they should be treated as normal objects (see the DesignTime snippet below)

#### RunTime creation of UI instances

```csharp
var myCustomPanel = InformationModel.Make<CustomMotorWidget>("MotorWidget");
Project.Current.Get("UI/Screens/Screen1").Add(myCustomPanel);
```

#### DesignTime creation of UI instances

```csharp
var myCustomPanelType = Project.Current.Get("path/to/type")
var myCustomPanel = InformationModel.MakeObject("MotorWidget", myCustomPanelType.NodeId);
Project.Current.Get("UI/Screens/Screen1").Add(myCustomPanel);
```

## OPC/UA objects

OPC/UA objects are generic containers that can hold variables or other OPC/UA objects and instances

### Create a custom OPC/UA Object type

```csharp
var newObject = InformationModel.MakeObjectType("NewMotorType");
newObject.Add(InformationModel.MakeVariable("Speed", OpcUa.DataTypes.Int32));
newObject.Add(InformationModel.MakeVariable("Power", OpcUa.DataTypes.Bool));
Project.Current.Get("Model/Templates").Add(newObject);
```

### DesignTime creation of custom OPC/UA Object instances

After the custom type has been created, new instances can be created with a slightly different syntax

```csharp
var customType = Project.Current.Get("Model/Templates/CustomMotor");
var customTypeInstance = InformationModel.MakeObject("MotorInstance1", customType.NodeId);
Project.Current.Get("Model/instances").Add(customTypeInstance);
```

### RunTime creation of custom OPC/UA Object instances

```csharp
var myObj = InformationModel.MakeObject<MotorType>("NewMotorInstance");
Owner.Add(myObj);
```

## Custom behavior

OPC/UA objects can expose custom OPC/UA methods that can be called from any connected client.

How to add a custom behavior:

1. Create an Object
1. Add some children variables
1. Convert the Object into Type
1. Right click on the Object Type and click `Add Custom Behavior`
1. Double click the Object Type and Visual Studio will appear

Now that the new class is created you can add as many C# methods as you need. Children variables can be accessed by calling `Node.[VariableName]`

```csharp
[ExportMethod]
public void StartMotor() {
    Node.Running = true;
    Node.CurSpeed = Node.SetSpeed;
}

[ExportMethod]
public void StopMotor() {
    Node.Running = false;
    Node.CurSpeed = (Int32)0;
}

[ExportMethod]
Public void Banana(int a, int b, out int c) {
    c = a + b;
}

#region Auto-generated code, do not edit!
protected new MotorWithBehaviorType Node => (MotorWithBehaviorType)base.Node;
#endregion
```

## Detect the Object Type of an element

### Extract the BrowseName of the Type of any element

```csharp
String dataType = InformationModel.Get(((IUAVariable)myVar).DataType).BrowseName;
```

```csharp
var dataSource = InformationModel.Get(Owner.GetVariable("Model").Value);
String baseType = dataSource.GetType().FullName.ToString()
```

### More advanced scenarios

This method expects the `BrowseName` of a `Screen` and makes a recursive search starting from a root node

```csharp
private NodeId RecursiveSearch(IUANode inputObject, String screenType) {
    foreach (IUANode childrenObject in inputObject.Children) {
        try {
            if (childrenObject is FTOptix.Core.Folder) {
                Log.Verbose1("FindPages.Folder", "Found folder with name [" + childrenObject.BrowseName + "] and Type: [" + childrenObject.GetType().ToString() + "]");
                RecursiveSearch(childrenObject, screenType);
            } else if (((UAManagedCore.UAObjectType)childrenObject).SuperType.BrowseName == screenType) {
                return childrenObject.NodeId
            } else {
                Log.Verbose1("FindPages.Else", "Found unknown with name [" + childrenObject.BrowseName + "] and Type: [" + childrenObject.GetType().ToString() + "]");
            }
        } catch (Exception ex) {
            Log.Error("FindPages.Catch", "Exception thrown: " + ex.Message);
        }
    }
}

```
