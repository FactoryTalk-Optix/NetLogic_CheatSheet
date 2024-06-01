# Advanced SVG image

The AdvancedSVGImage container, supports direct manipulation of the image buffer, allowing for fast and direct access to the content, this allows runtime editing that can efficiently manipulate the path or properties of an SVG image without impacting the InformationModel

This feature was introduced in FT Optix 1.4.x

## Sample image

This is the sample image (containing a "star" shape) that we used in the examples below to be animated

```svg
<?xml version="1.0" encoding="iso-8859-1"?>
<svg version="1.1" id="Overall" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" xml:space="preserve" viewBox="0 0 300 300">
  <g id="MyPaths" opacity="1.0">
	  <path style="fill:#3C4660;" d="M 0 67.2262 L 72.1921 67.2262 L 94.5 0 L 116.8079 67.2262 L 189 67.2262 L 130.5954 108.7738 L 152.9045 176 L 94.5 134.4513 L 36.0954 176 L 58.4045 108.7738 L 0 67.2262 Z" transform="rotate(0, 150, 219)"/>
  </g>
  <g id="Box1">
    <polygon style="fill:#CB8252;" points="25,270 75,270 75,220 25,220" transform="rotate(0, 20, 20)"/>
  </g>
</svg>
```

## Making the image disappear

```csharp
[ExportMethod]
public void replaceSVG()
{
    // LogicObject.Owner is the button, so LogicObject.Owner is the MainWindow
    AdvancedSVGImage svgImage = LogicObject.Owner.Children.Get<AdvancedSVGImage>("AdvancedSVGImage1");

    string myXML = "<?xml version=\"1.0\" encoding=\"iso-8859-1\"?>\r\n<svg version=\"1.1\" id=\"Overall\" xmlns=\"http://www.w3.org/2000/svg\" xmlns:xlink=\"http://www.w3.org/1999/xlink\" xml:space=\"preserve\" viewBox=\"0 0 300 300\">\r\n  <g id=\"Box1\">\r\n    <polygon style=\"fill:#CB8252;\" points=\"25,270 75,270 75,220 25,220\" transform=\"rotate(0, 20, 20)\"/>\r\n  </g>\r\n</svg>\r\n";

    //Update the SVG
    svgImage.SetImageContent(myXML);
}
```

## Rotate the image

```csharp
#region Using directives
using FTOptix.NetLogic;
using FTOptix.UI;
using System.Xml.Linq;
using System.Linq;
using UAManagedCore;
using System.Threading;
#endregion

public class RuntimeNetLogic1 : BaseNetLogic
{
    public override void Start()
    {
        // LogicObject.Owner is the button, so LogicObject.Owner is the MainWindow
        svgImage = LogicObject.Owner.Children.Get<AdvancedSVGImage>("AdvancedSVGImage1");
        // Retrieve the Path to the SVG
        var imageAbsolutePath = svgImage.Path.Uri;
        // Load the SVG into an XDocument
        xDocument = XDocument.Load(imageAbsolutePath);
        // Find the first path element
        var pathNode = xDocument.Descendants().Where(x => x.Name.LocalName == "path").FirstOrDefault();
        // Find the first polygon element
        var polyNode = xDocument.Descendants().Where(x => x.Name.LocalName == "polygon").FirstOrDefault();
        // Get the polygon transform attribute
        polyTransformAttribute = polyNode.Attribute("transform");
        // Get the polygon transform attribute
        pathTransformAttribute = pathNode.Attribute("transform");
    }

    public override void Stop()
    {
        keepGoing = false;
        if (myLRT != null)
            myLRT.Dispose();
    }

    private void doRotate()
    {
        int degrees = 3;
        while (keepGoing)
        {
            Thread.Sleep(50);
            degrees += 3;
            if (degrees > 360)
                degrees = 0;
            string newTransformPoly = new string("rotate(" + degrees + ", 50, 250)");
            polyTransformAttribute.SetValue(newTransformPoly);
            string newTransformPath = new string("rotate(" + degrees + ", 96, 97)");
            pathTransformAttribute.SetValue(newTransformPath);
            //Update the SVG
            svgImage.SetImageContent(xDocument.ToString());
        }
    }

    [ExportMethod]
    public void rotateSVG()
    {
        keepGoing = false;
        if (myLRT != null)
            myLRT.Dispose();
        keepGoing = true;
        myLRT = new LongRunningTask(doRotate, LogicObject);
        myLRT.Start();
    }

    private XDocument xDocument;
    private AdvancedSVGImage svgImage;
    private XAttribute polyTransformAttribute;
    private XAttribute pathTransformAttribute;
    private LongRunningTask myLRT;
    private bool keepGoing = true;
}
```

