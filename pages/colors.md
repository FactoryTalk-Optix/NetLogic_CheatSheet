# Colors

FT Optix can interact with the color of the objects and also create custom colors

```csharp
var myObj = InformationModel.Make<Led>("myLabel");
var myCustomColor = new Color(0xff, 0xaa, 0xbb, 0xcc);
myObj.Color = myCustomColor;
```

```csharp
var myObj = InformationModel.Make<Led>("myLabel");
var myCustomColor = Colors.Red;
myObj.Color = myCustomColor;
```