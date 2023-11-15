# Creating elements

## Add an element to the project

Native types are the one predefined in FTOptix, these are the most simple to create and add anywhere in the project

```csharp
var newPanel = InformationModel.Make<Panel>("MyNewPanelName");
newPanel.Width = 300.0;
newPanel.HorizontalAlignment = HorizontalAlignment.Stretch;
Project.Current.Get("UI/Screens/Screen1").Add(newPanel);
```

### Make it faster

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

### Custom UI types instances

When creating custom types or templates, the equivalent C# class is automatically created by FTOptix, then you can instantiate such classes with the same syntax

```csharp
var myCustomPanel = InformationModel.Make<CustomMotorWidget>("MotorWidget");
Project.Current.Get("UI/Screens/Screen1").Add(myCustomPanel);
```

## Create a custom OPC/UA Object type

```csharp
var newObject = InformationModel.MakeObjectType("NewMotorType");
newObject.Add(InformationModel.MakeVariable("Speed", OpcUa.DataTypes.Int32));
newObject.Add(InformationModel.MakeVariable("Power", OpcUa.DataTypes.Bool));
Project.Current.Get("Model/Templates").Add(newObject);
```

## DesignTime creation of custom OPC/UA Object instances

After the custom type has been created, new instances can be created with a slightly different syntax

```csharp
var customType = Project.Current.Get("Model/Templates/CustomMotor");
var customTypeInstance = InformationModel.MakeObject("MotorInstance1", customType.NodeId);
Project.Current.Get("Model/instances").Add(customTypeInstance);
```

## Runtime creation of custom OPC/UA Object instances

```csharp
var myObj = InformationModel.MakeObject<MotorType>("NewMotorInstance");
Owner.Add(myObj);
```
