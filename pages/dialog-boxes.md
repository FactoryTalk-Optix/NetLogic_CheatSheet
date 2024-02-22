# DialogBox

## Opening a DialogBox

```csharp
// Source type definition of the DialogBox
var myDialogType = (DialogType)Project.Current.Get("UI/DialogBox1");
// DialogBoxes needs a graphical container as parent in order to
// understand to which session they have to spawn
parentPanel = (FTOptix.UI.Panel)Owner;
UICommands.OpenDialog(parentPanel, myDialogType);
```

## Close all DialogBoxes

```csharp
public override void Stop()
{
    // DialogBoxes are children of the current Session, so we can iterate in
    // children to get all of them
    foreach (Dialog item in Session.Get("UIRoot").Children.OfType<Dialog>().ToList()) {
        item.Close();
    }
}
```