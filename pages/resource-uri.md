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
