# DialogBox

## Opening a DialogBox

The `UICommands.OpenDialog()` method opens a dialog (modal window) and associates it with a specific UI session. It has two overloads:
- `OpenDialog(IUANode parentItem, DialogType dialogType, NodeId aliasNode = null)` - opens a dialog and binds a single alias node
- `OpenDialog(IUANode parentItem, DialogType dialogType, NodeId[] aliasNodes)` - opens a dialog and binds multiple alias nodes

The `parentItem` must be a UI `Item` or `Window` to determine which session receives the dialog.

### Basic Example

```csharp
[ExportMethod]
public void OpenMyDialogBox()
{
    // Get the DialogBox type definition
    var myDialogType = (DialogType)Project.Current.Get("UI/Templates/DialogBox1");
    
    // DialogBoxes need a graphical container as parent to bind to the correct session
    var parentPanel = (FTOptix.UI.Panel)Owner;
    
    // Open the dialog without alias bindings
    UICommands.OpenDialog(parentPanel, myDialogType);
}
```

### Pass a Single Alias Node

Alias nodes are OPC UA references that the dialog can use to access objects. This is useful for passing data or objects to the dialog.

```csharp
[ExportMethod]
public void OpenDialogWithAlias()
{
    var dialogType = (DialogType)Project.Current.Get("UI/Templates/ConfigDialog");
    var configObject = Project.Current.Get("Objects/CurrentConfig");
    
    // Bind a single alias node - the dialog can access it through the alias
    UICommands.OpenDialog((FTOptix.UI.Panel)Owner, dialogType, configObject.NodeId);
}
```

### Pass Multiple Alias Nodes

You can pass an array of alias nodes to provide the dialog with access to multiple objects.

```csharp
[ExportMethod]
public void OpenDialogWithMultipleAliases()
{
    var dialogType = (DialogType)Project.Current.Get("UI/Templates/DetailDialog");
    var primaryData = Project.Current.Get("Objects/PrimaryData");
    var secondaryData = Project.Current.Get("Objects/SecondaryData");
    var settings = Project.Current.Get("Objects/Settings");
    
    // Create an array of alias nodes
    NodeId[] aliasNodes = new NodeId[]
    {
        primaryData.NodeId,
        secondaryData.NodeId,
        settings.NodeId
    };
    
    // Open the dialog with multiple alias bindings
    UICommands.OpenDialog((FTOptix.UI.Panel)Owner, dialogType, aliasNodes);
}
```

### Open a DialogBox on all sessions (native and web)

```csharp
[ExportMethod]
public void OpenDialogBox(string DialogBoxName, NodeId[] AliasArray = null)
{
    DialogType dialogBox = (DialogType)Project.Current.Get($"UI/Screens/{DialogBoxName}");
    
    // Open dialog on Native Presentation Engine
    var nativePresentationEngine = Project.Current.Get("UI/NativePresentationEngine");
    var nativePresentationEngineSessions = nativePresentationEngine.Get("Sessions");
    var nativePresentationEngineSession = nativePresentationEngineSessions.Children[0];   // Native PE only has one session, so get the first one
    var nativePresentationEngineWindow = nativePresentationEngineSession.Get("UIRoot");
    UICommands.OpenDialog(nativePresentationEngineWindow, dialogBox, AliasArray);
    
    // Open dialog on Web Presentation Engine (may have multiple sessions)
    var webPresentationEngine = Project.Current.Get("UI/WebPresentationEngine");
    var webPresentationEngineSessions = webPresentationEngine.Get("Sessions");
    foreach (var webPresentationEngineSession in webPresentationEngineSessions.Children)
    {
        var webPresentationEngineWindow = webPresentationEngineSession.Get("UIRoot");
        if (webPresentationEngineWindow != null)
        {
            UICommands.OpenDialog(webPresentationEngineWindow, dialogBox, AliasArray);
        }
    }
}
```

## Closing a DialogBox

### Close all DialogBoxes in the current session

> [!TIP]
> The following snippet must be placed in a session-based object (e.g. the `MainWindow` of a project).

```csharp
[ExportMethod]
public void CloseAllDialogs()
{
    // DialogBoxes are children of the current Session, so iterate through
    // the children of UIRoot to find and close all open dialogs
    var uiRoot = Session.Get("UIRoot");
    foreach (Dialog dialog in uiRoot.Children.OfType<Dialog>().ToList())
    {
        dialog.Close();
    }
}
```

### Close a specific DialogBox

```csharp
[ExportMethod]
public void CloseMyDialog()
{
    // Get reference to a specific dialog and close it
    var myDialog = (Dialog)Session.Get("UIRoot/MyDialogBox");
    if (myDialog != null)
    {
        myDialog.Close();
    }
}
```

### Close a DropDownButton

```csharp
/// <summary>
/// Close a DropDownButton UI widget by calling its Close method.
/// Input: assume `Owner` is the DropDownButton or that the code can access it via Owner.
/// </summary>
var dropDownButton = (DropDownButton)Owner;
// Call the Close method exposed by the widget
dropDownButton.ExecuteMethod("Close");
```
