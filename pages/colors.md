# Colors

FT Optix can interact with the color of the objects and also create custom colors

```csharp
// Create a Led
var myObject = InformationModel.Make<Led>("myLabel");
// Set Color to any valid RGB color
var myCustomColor = new Color(0xff, 0xaa, 0xbb, 0xcc);
// Set the Led color to the new color
myObject.Color = myCustomColor;
// Add the Led to the page
...
```

```csharp
// Create a Led
var myObject = InformationModel.Make<Led>("myLabel");
// Set the Led color to Red
myObject.Color = Colors.Red;
// Add the Led to the page
...
```