# Resource URIs

## Create a ResourceUri from scratches

```csharp
var baseUri = new Uri(System.IO.Directory.GetCurrentDirectory());
var chartUri = new Uri(baseUri, Project.Current.ProjectDirectory);
var nextStep = new Uri(chartUri, "NetSolution");
var finalPath = new Uri(nextStep, "Charts.xslt");
string filePath = finalPath.AbsolutePath;
Log.Info("Final path: " + filePath);
Log.Info(MethodBase.GetCurrentMethod().Name, $"File name is: {fileName}");
```

## Reading a ResourceUri children of the NetLogic

```csharp
var csvPathVariable = LogicObject.Get<IUAVariable>("CSVPath");
string myString = new ResourceUri(csvPathVariable.Value).Uri;
```

## Set the path of an Image to the ProjectFiles

```csharp
singlePiece.Get<Image>("PuzzleImage").Path = ResourceUri.FromProjectRelativePath("").Uri + Path.DirectorySeparatorChar + "imgs" + Path.DirectorySeparatorChar + "Puzzle" + Path.DirectorySeparatorChar + "Piece" + (i + 1).ToString() + ".png";
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
var templatePath = ResourceUri.FromProjectRelativePath("Template-line-smooth.html");
templatePath = ResourceUri.FromProjectRelativePath(templatePath.Uri.Replace("Template-", ""));
File.WriteAllText(templatePath.Uri, text);
```

## Read the full path from a ResourceUri Variable

```
pdfPathStr = new ResourceUri(((IUAVariable)myPathNode).Value).Uri;
```
