# Resource URIs

## Important note

Although multiple solutions are shown here to set the target path for a ResourceUri variable, please remember that the WebPresentationEngine only supports accessing elements that are in the `%PROJECTDIR%` and a relative path is used (example: `%PROJECTDIR%/imgs/logo.svg`), using an absolute path for an element (example: `C:/OptixProjects/NewHMIProject1/Project Files/imgs/logo.svg`), will never work in the WebPresentationEngine for safety reasons.

## Create a ResourceUri from scratch

```csharp
// Get the working folder from the current context
var baseUri = new Uri(System.IO.Directory.GetCurrentDirectory());
// Get the ProjectDir folder
var chartUri = new Uri(baseUri, Project.Current.ProjectDirectory);
// Browse to the NetSolution folder
var nextStep = new Uri(chartUri, "NetSolution");
// Browse to NetSolution/Charts.xlst
var finalPath = new Uri(nextStep, "Charts.xslt");
// Get the full path
string filePath = finalPath.AbsolutePath;
Log.Info("Final path: " + filePath);
Log.Info(MethodBase.GetCurrentMethod().Name, $"File name is: {fileName}");
```

## Reading a ResourceUri

This assumes the variable is set to ResourceUri

```csharp
// Get a variable with datatype set to ResourceUri
var csvPathVariable = LogicObject.Get<IUAVariable>("CSVPath");
// Get the full path
string myString = new ResourceUri(csvPathVariable.Value).Uri;
```

```csharp
/// <summary>
/// Examples to log common ResourceUri locations: project, application and USB.
/// This helps when debugging paths used by NetLogic or UI resources.
/// </summary>
// Project files
Log.Info(ResourceUri.FromProjectRelativePath("").Uri);
// Application files
Log.Info(ResourceUri.FromApplicationRelativePath("").Uri);
// USB device 1 (may throw if not present)
try
{
    Log.Info(ResourceUri.FromUSBRelativePath(1, "").Uri);
}
catch (Exception ex)
{
    Log.Warning("ResourceUri", "USB1 not available: " + ex.Message);
}
```

```csharp
pdfPathStr = new ResourceUri(((IUAVariable)myPathNode).Value).Uri;
```

## Set the path of an Image to the ProjectFiles

### Using absolute path

> [!TIP]
> This method is not recommended as it will not work if the project is moved to another folder or computer, and might have issues with the WebPresentationEngine. Use the relative path below.

```csharp
// Note: the Path.DirectorySeparatorChar is used to make the application cross-platform capable
singlePiece.Get<Image>("PuzzleImage").Path = ResourceUri.FromProjectRelativePath("").Uri + Path.DirectorySeparatorChar + "imgs" + Path.DirectorySeparatorChar + "Puzzle" + Path.DirectorySeparatorChar + "Piece" + (i + 1).ToString() + ".png";
```

### Using relative path

```csharp
[ExportMethod]
public void Method2()
{
    // Insert code to be executed by the method
    var image = Owner.Get<AdvancedSVGImage>("AdvancedSVGImage1");
    // Set the path to %PROJECTDIR%/logo.svg
    image.PathVariable.Value = ResourceUri.FromProjectRelativePath("logo.svg");
}
```

## Get a file from ProjectFiles

```csharp
string localPath = Path.DirectorySeparatorChar + "imgs" + Path.DirectorySeparatorChar + "Puzzle" + Path.DirectorySeparatorChar + "Piece" + (i + 1).ToString() + ".png";
singlePiece.Get<Image>("PuzzleImage").Path = "ns=" + singlePiece.NodeId.NamespaceIndex.ToString() + ";%PROJECTDIR%" + localPath;
```

```csharp
string localPath = "imgs" + Path.DirectorySeparatorChar + "Puzzle" + Path.DirectorySeparatorChar + "Piece" + (i + 1).ToString() + ".png";
singlePiece.Get<Image>("PuzzleImage").Path = Path.Combine(ResourceUri.FromProjectRelativePath(""), localPath);
```

```csharp
// Start from the ProjectDir and get the Template-line-smooth.html file
var templatePath = ResourceUri.FromProjectRelativePath("Template-line-smooth.html");
// Remove the Template- prefix
templatePath = ResourceUri.FromProjectRelativePath(templatePath.Uri.Replace("Template-", ""));
// Write the new content to the file
File.WriteAllText(templatePath.Uri, text);
```

## USB Check (Windows)

```csharp
/// <summary>
/// Periodically checks for removable USB drives (Windows) and logs their presence.
/// Inputs: none. This is intended for a Runtime NetLogic.
/// Notes: Uses System.IO.DriveInfo and will only run on platforms that expose drive letters (Windows).
/// </summary>
private PeriodicTask task;

public override void Start()
{
    task = new PeriodicTask(CheckUSB, 1000, LogicObject);
    task.Start();
}

private void CheckUSB()
{
    var usbDevices = System.IO.DriveInfo.GetDrives();
    foreach (var device in usbDevices)
    {
        if (device.DriveType == System.IO.DriveType.Removable && device.IsReady)
        {
            Log.Info("RuntimeNetLogic1", $"USB device found: {device.Name} - {device.VolumeLabel}");
        }
    }
}

public override void Stop()
{
    task.Dispose();
}
```

## Ensure a directory exists and write a file (Windows)

```csharp
/// <summary>
/// Create a folder if missing and write a small test file. Useful when exporting logs or CSV files.
/// Inputs: NetLogic variables `FolderPath` (string) and `FileName` (string).
/// </summary>
[ExportMethod]
public void CreateFileTest()
{
    var folder = LogicObject.GetVariable("FolderPath");
    var fileName = LogicObject.GetVariable("FileName");

    bool exists = System.IO.Directory.Exists(folder.Value.ToString());

    if (!exists)
        System.IO.Directory.CreateDirectory(folder.Value.ToString());

    using (var writer = new StreamWriter(System.IO.Path.Combine(folder.Value.ToString(), fileName.Value.ToString())))
    {
        // Write some data rows
        writer.WriteLine("Test file created from NetLogic");
    }

}
```

## Project / Application / USB resource URIs

```csharp
/// <summary>
/// Examples showing how to log or inspect the different ResourceUri roots available from C#.
/// Use these to resolve files inside the project, application or on attached USB devices.
/// </summary>
// Project files root
Log.Info(ResourceUri.FromProjectRelativePath("").Uri);

// Application files root
Log.Info(ResourceUri.FromApplicationRelativePath("").Uri);

// Access USB slot 1 (if present)
// [!WARNING]
// Accessing USB paths at runtime can throw if the device is not present — always guard with try/catch.
try
{
    Log.Info(ResourceUri.FromUSBRelativePath(1, "").Uri);
}
catch (Exception ex)
{
    Log.Warning("ResourceUri", "USB not available: " + ex.Message);
}
```
