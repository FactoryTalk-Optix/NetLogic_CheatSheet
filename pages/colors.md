# Colors

FactoryTalk Optix allows you to set colors on UI objects (such as `Led`, `Rectangle`, `Panel`, etc.) using the `Color` struct. You can assign colors using pre-defined named colors or create custom colors with specific RGB/ARGB values.

## The Color Struct

The `Color` struct has the following properties:
- `A` (byte) - Alpha channel (transparency): 0 = fully transparent, 255 = fully opaque
- `R` (byte) - Red channel: 0-255
- `G` (byte) - Green channel: 0-255
- `B` (byte) - Blue channel: 0-255
- `ARGB` (uint) - Combined ARGB value as a single 32-bit unsigned integer

## Creating Colors

### Using Pre-defined Named Colors

FT Optix provides hundreds of named colors through the `Colors` static class, such as:
- `Colors.Red`
- `Colors.Blue`
- `Colors.Green`
- `Colors.Goldenrod`
- `Colors.Cyan`
- `Colors.Black`
- And many more...

```csharp
// Create a LED widget
var ledWidget = InformationModel.Make<Led>("statusIndicator");

// Set the LED color using a pre-defined named color
ledWidget.Color = Colors.Red;

// Add the LED to your UI
...
```

### Using ARGB Constructor (Explicit Bytes)

Create a custom color by specifying the Alpha, Red, Green, and Blue channels as individual bytes:

```csharp
// Create a LED widget
var ledWidget = InformationModel.Make<Led>("statusIndicator");

// Create a custom color: semi-transparent orange
// Parameters: (Alpha, Red, Green, Blue)
// Alpha = 0xff (fully opaque), Red = 0xff, Green = 0xaa, Blue = 0xbb
var customOrange = new Color(0xff, 0xff, 0xaa, 0xbb);

// Assign the custom color to the LED
ledWidget.Color = customOrange;

// Add the LED to your UI
...
```

### Using ARGB Constructor (Hex Value)

Create a custom color by providing a 32-bit ARGB value as a hex number:

```csharp
// Create a LED widget
var ledWidget = InformationModel.Make<Led>("statusIndicator");

// Create a semi-transparent blue color using a 32-bit ARGB hex value
// Format: 0xAARRGGBB (AA=Alpha, RR=Red, GG=Green, BB=Blue)
// 0x80 = 128 (50% transparent), 0x00=Red, 0x00=Green, 0xff=Blue
var semiTransparentBlue = new Color(0x800000ff);

// Assign the custom color to the LED
ledWidget.Color = semiTransparentBlue;

// Add the LED to your UI
...
```

### Using System.Drawing.Color

You can convert .NET's standard `System.Drawing.Color` to FT Optix `Color` using the `ToArgb()` method:

```csharp
// Create a LED widget
var ledWidget = InformationModel.Make<Led>("statusIndicator");

// Convert a System.Drawing.Color to FT Optix Color
// Goldenrod is a pre-defined System.Drawing color
var systemGoldenrod = System.Drawing.Color.Goldenrod;
var goldenrodColor = new Color(systemGoldenrod.ToArgb());

// Assign the color to the LED
ledWidget.Color = goldenrodColor;

// Add the LED to your UI
...
```

## Complete Example

```csharp
[ExportMethod]
public void SetupStatusLights()
{
    // Create three status indicator LEDs
    var operationalLed = InformationModel.Make<Led>("operationalIndicator");
    var warningLed = InformationModel.Make<Led>("warningIndicator");
    var errorLed = InformationModel.Make<Led>("errorIndicator");
    
    // Use pre-defined colors for standard status states
    operationalLed.Color = Colors.Green;
    warningLed.Color = Colors.Yellow;
    errorLed.Color = Colors.Red;
    
    // Create a custom semi-transparent overlay color
    var overlayColor = new Color(0x80, 0xff, 0xff, 0xff); // 50% transparent white
    
    // Add the LEDs to your UI panel
    Owner.Add(operationalLed);
    Owner.Add(warningLed);
    Owner.Add(errorLed);
}
```
