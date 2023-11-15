# DialogBox

## Opening a DialogBox

```csharp
var myDialogType = (DialogType)Project.Current.Get("UI/DialogBox1");
UICommands.OpenDialog(LogicObject.Get("NomeContainer"), myDialogType);
```

## Close all DialogBoxes

```csharp
public override void Stop()
{
    // Insert code to be executed when the user-defined logic is stopped
    foreach (Dialog item in Session.Get("UIRoot").Children.OfType<Dialog>().ToList()) {
        item.Close();
    }
}
```