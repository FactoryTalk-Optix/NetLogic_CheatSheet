# Creating elements

## UI Elements

### Add an element to the project

Native types are the one predefined in FTOptix, these are the most simple to create and add anywhere in the project. These types exists both at RunTime and DesignTime, so the syntax does not change depending on the execution

```csharp
// Create a new Panel called "MyNewPanelName"
var newPanel = InformationModel.Make<Panel>("MyNewPanelName");
// Set some properties (if needed)
newPanel.Width = 300.0;
newPanel.HorizontalAlignment = HorizontalAlignment.Stretch;
// Add it to "UI/Screens/Screen1"
Project.Current.Get("UI/Screens/Screen1").Add(newPanel);
```

#### Make it faster

When you need to create a lot of element in a short time (such in a `for` loop), adding each element to the page every time is not recommended as multiple async request are made to FTOptix APIs and some concurrency issues may appear, in this case we suggest to create the whole model in RAM and then add it to the page only when the whole structure is defined

```csharp
// Create a container for the new elements
var tempPanel = InformationModel.Make<Panel>("MotorsGroup");
// Set the container to fill the parent item
tempPanel.HorizontalAlignment = HorizontalAlignment.Stretch;
tempPanel.VerticalAlignment = VerticalAlignment.Stretch;
// Create the 100 motors
for (i = 0; i < 100; i++) {
    // Create the new Panel widget
    var newPanel = InformationModel.Make<Panel>("MyMotor" + i.ToString());
    // Set some properties
    newPanel.Width = 300.0;
    newPanel.HorizontalAlignment = HorizontalAlignment.Stretch;
    // Add the Panel to the new container
    tempPanel.Add(newPanel);
}
// Finally, add the new container to the Screen (with all the elements)
Project.Current.Get("UI/Screens/Screen1").Add(tempPanel);
```

### `Panel` Assigning the value of Move Target property

The Move target property is an optional extra, so it must be materialized before any value can be set. When creating a Panel from NetLogic, this step is mandatory.
This property may be *None* or *Self*. As this property is a **NodePointer**, you must create a dynamicLink with value **..@NodeId** if you want to set **Self**, otherwise you must simply remove the DynamicLink to set **None**.

```csharp
private void StuffMakePanelUIObject(IUANode panelOwner, string panelName, int width, int height, bool moveTargetSelf)
{
    // Get the owner of the Panel (in case we want to use the Panel to move the parent element)
    Panel panel = panelOwner.Get(panelName) as Panel;
    // Make sure the Panel exists
    if (panel == null)
    {
        panel = InformationModel.MakeObject<Panel>(panelName);
        panelOwner.Add(panel);
    }
    // Set some properties
    panel.Height = height;
    panel.Width = width;
    // Access the optional variable value
    IUAVariable moveTarget = panel.GetOrCreateVariable("MoveTarget"); 
    // Prepare the DynamicLink
    DynamicLink moveTargetLink = moveTarget?.GetVariable("DynamicLink") as DynamicLink;
    if (moveTarget != null && moveTargetSelf)
    {
        if (moveTargetLink == null)
        {
            // Add the destination for the NodeTarget (Owner)
            moveTargetLink = InformationModel.MakeVariable<DynamicLink>("DynamicLink", FTOptix.Core.DataTypes.NodePath);
            moveTargetLink.Value = "..@NodeId";
            moveTargetLink.Mode = DynamicLinkMode.Read;
            moveTarget.Refs.AddReference(FTOptix.CoreBase.ReferenceTypes.HasDynamicLink, moveTargetLink);
        }
    }
    else
    {
        // Remove any existing dynamic link from the MoveTarget
        if (moveTargetLink != null)
        {
            moveTarget?.Refs.RemoveReference(FTOptix.CoreBase.ReferenceTypes.HasDynamicLink, moveTargetLink.NodeId);
        }
    }
}
```

### Custom UI types instances (widget instances)

When creating custom types or templates, the equivalent C# class is automatically created by FTOptix, then you can instantiate such classes with the same syntax

**Please note**: The user-defined types does not exists when executing scripts at DesignTime, so they should be treated as normal objects (see the DesignTime snippet below)

#### RunTime creation of UI instances

```csharp
// This syntax is only valid at Runtime, while designing the app,
// the CustomMotorWidget class does not exist
var myCustomPanel = InformationModel.Make<CustomMotorWidget>("MotorWidget");
// Add it to the page
Owner.Add(myCustomPanel);
```

#### DesignTime creation of UI instances

```csharp
// This method works both at Runtime and Design-time
// Get the original type to be instantiated
var myCustomPanelType = Project.Current.Get("path/to/type")
// Create an object by specifying the custom type
var myCustomPanel = InformationModel.MakeObject("MotorWidget", myCustomPanelType.NodeId);
// Add the new instance to the screen
Project.Current.Get("UI/Screens/Screen1").Add(myCustomPanel);
```

## IUAObjects

Objects are generic containers that can hold variables or other OPC/UA objects and instances

### Create a custom Object type

```csharp
// Create a new Object Type
var newObject = InformationModel.MakeObjectType("NewMotorType");
// Add some properties to the Object type
newObject.Add(InformationModel.MakeVariable("Speed", OpcUa.DataTypes.Int32));
newObject.Add(InformationModel.MakeVariable("Power", OpcUa.DataTypes.Bool));
// Add the Object Type to the Model folder
Project.Current.Get("Model/Templates").Add(newObject);
```

### DesignTime creation of custom Object instances

After the custom type has been created, new instances can be created with a slightly different syntax

```csharp
// Get the custom type definition
var customType = Project.Current.Get("Model/Templates/CustomMotor");
// Create the new object as instance of the custom type
var customTypeInstance = InformationModel.MakeObject("MotorInstance1", customType.NodeId);
// Add the new instance to the Model folder
Project.Current.Get("Model/instances").Add(customTypeInstance);
```

### RunTime creation of custom Object instances

```csharp
// This syntax only works at Runtime, 
// get the MotorType class and create a new object
var myObj = InformationModel.MakeObject<MotorType>("NewMotorInstance");
// Add the object to the parent element
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
    // Node refers to the current object ("this")
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
// Cast myVar to IUAVariable to read the DataType,
// then get it from the InformationModel and read the BrowseName
String dataType = InformationModel.Get(((IUAVariable)myVar).DataType).BrowseName;
```

```csharp
// Get the Variable to read the type
var dataSource = InformationModel.Get(Owner.GetVariable("Model").Value);
// Get it as a string
String baseType = dataSource.GetType().FullName.ToString()
```

### More advanced scenarios

This method expects the `BrowseName` of a `Screen` and makes a recursive search starting from a root node

```csharp
private NodeId RecursiveSearch(IUANode inputObject, String screenType) {
    // Browse all children of the starting node
    foreach (IUANode childrenObject in inputObject.Children) {
        try {
            if (childrenObject is FTOptix.Core.Folder) {
                // Start looking into the folder
                Log.Verbose1("FindPages.Folder", "Found folder with name [" + childrenObject.BrowseName + "] and Type: [" + childrenObject.GetType().ToString() + "]");
                // Recursive call to keep browsing
                RecursiveSearch(childrenObject, screenType);
            } else if (((UAManagedCore.UAObjectType)childrenObject).SuperType.BrowseName == screenType) {
                // If the screen type was found, return it
                return childrenObject.NodeId
            } else {
                // Other type that does not need to be used
                Log.Verbose1("FindPages.Else", "Found unknown with name [" + childrenObject.BrowseName + "] and Type: [" + childrenObject.GetType().ToString() + "]");
            }
        } catch (Exception ex) {
            // Handle exceptions
            Log.Error("FindPages.Catch", "Exception thrown: " + ex.Message);
        }
    }
}

```
