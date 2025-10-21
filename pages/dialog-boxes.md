# DialogBox

## Opening a DialogBox

```csharp
[ExportMethod]
public void OpenMyDialogBox()
{
    // Source type definition of the DialogBox
    var myDialogType = (DialogType)Project.Current.Get("UI/Templates/DialogBox1");
    // DialogBoxes needs a graphical container as parent in order to
    // understand to which session they have to spawn
    parentPanel = (FTOptix.UI.Panel)Owner;
    UICommands.OpenDialog(parentPanel, myDialogType);
}
```

### Open a DialogBox on all sessions (native and web)

```csharp
[ExportMethod]
public void OpenDialogBox(string DialogBoxName, NodeId[] AliasArray = null)
{
    DialogType dialogBox = (DialogType)Project.Current.Get($"UI/Screens/{DialogBoxName}");  // depends on path to Screens folder
 
    var npe = Project.Current.Get("UI/NativePresentationEngine");   // depends on UI and NativePresentationEngine paths
    var npeSessions = npe.Get("Sessions");      // npe has a single session
    var npeSession = npeSessions.Children[0];   // the single npe session is at [0]
    var npeWindow = npeSession.Get("UIRoot");   // "UIRoot" is the browsename of the main window
    UICommands.OpenDialog(npeWindow, dialogBox, AliasArray);
 
    var wpe = Project.Current.Get("UI/WebPresentationEngine");   // depends on UI and WebPresentationEngine paths
    var wpeSessions = wpe.Get("Sessions");      // wpe could have multiple sessions
    foreach (var wpeSession in wpeSessions.Children)
    {
        var wpeWindow = wpeSession.Get("UIRoot");   // "UIRoot" is the browsename of the main window
        if (wpeWindow != null)
        {
            UICommands.OpenDialog(wpeWindow, dialogBox, AliasArray);
        }
    }
}
```

## Close all DialogBoxes

> [!TIP]
> The following snippet must be placed in a session-based object (e.g. the `MainWindow` of a project).

```csharp
[ExportMethod]
public void CloseAllDialogs()
{
    // DialogBoxes are children of the current Session, so we can iterate in
    // children to get all of them
    foreach (Dialog item in Session.Get("UIRoot").Children.OfType<Dialog>().ToList()) {
        item.Close();
    }
}
```

## Close a DropDownButton

```csharp
/// <summary>
/// Close a DropDownButton UI widget by executing its Close method.
/// Input: assume `Owner` is the DropDownButton or that the code can access it via Owner.
/// </summary>
var drop = (DropDownButton)Owner;
// Execute the Close action exposed by the widget
drop.ExecuteMethod("Close");
```
