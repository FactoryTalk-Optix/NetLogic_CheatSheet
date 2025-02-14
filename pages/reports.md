# Reports

## Generate a report and trigger some action based on the result code

```csharp
#region Using directives
using System;
using System.IO;
using System.Linq;
using System.Reflection.Emit;
using FTOptix.Core;
using FTOptix.HMIProject;
using FTOptix.NetLogic;
using FTOptix.Report;
using FTOptix.UI;
using UAManagedCore;
#endregion

public partial class PdfReportLogic : BaseNetLogic
{
    public override void Start()
    {
        // Try to assign a value to the button
        try
        {
            generatePdfButton = Owner.Get<Button>("TrackedValues/Generate");
            viewPdfButton = Owner.Get<Button>("TrackedValues/View");
            viewPdfButton.Enabled = false;
        }
        catch
        {
            // Button does not exist
            Log.Warning("PdfReportLogic", "Can't find PDF buttons, maybe they were renamed?");
        }
    }

    public override void Stop()
    {
        myReport.OnGeneratePdfCompleted -= MyReport_OnGeneratePdfCompleted;
    }

    [ExportMethod]
    public void GenerateReport(string fileName, string pdfLocale)
    {
        // Hide button
        generatePdfButton.Enabled = false;

        // Check if the path is empty
        if (string.IsNullOrEmpty(fileName))
        {
            Log.Error("PdfReportLogic", "Empty PDF name");
            SetOutputMessage("ReportEmptyPath");
            throw new ArgumentNullException();
        }

        // Check if the pdf has the extension
        if (!fileName.EndsWith(".pdf", StringComparison.InvariantCultureIgnoreCase))
        {
            Log.Warning("PdfReportLogic", "PDF extension not found, adding it");
            fileName += ".pdf";
        }

        // Make sure to use only the file name
        fileName = fileName.Split('\\', '/').Last();

        // Check if the path is an Optix path variable
        var pdfResourceUri = ResourceUri.FromProjectRelativePath(fileName);

        // Check if the locale is valid
        if (string.IsNullOrEmpty(pdfLocale))
        {
            Log.Error("PdfReportLogic", "Empty locale");
            SetOutputMessage("ReportEmptyLocale");
            throw new ArgumentNullException();
        }

        // Check if the locale is valid using regex
        if (!LocaleIdRegex().IsMatch(pdfLocale))
        {
            Log.Error("PdfReportLogic", "Invalid locale");
            SetOutputMessage("ReportInvalidLocale");
            throw new ArgumentException();
        }

        // Generate the report
        myReport = Project.Current.Get<Report>("Reports/LoggerReport");
        Log.Debug("PdfReportLogic", $"Generating PDF report with locale: {pdfLocale}");
        Log.Debug("PdfReportLogic", $"Saving PDF report to: {pdfResourceUri.Uri}");
        myReport.GeneratePdf(pdfResourceUri, pdfLocale, out Guid operationId);

        // Subscribe to the report generation event
        myReport.OnGeneratePdfCompleted += MyReport_OnGeneratePdfCompleted;

        Log.Debug("PdfReportLogic", $"Report generation started with operation ID: {operationId}");
    }

    private void MyReport_OnGeneratePdfCompleted(object sender, GeneratePdfCompletedEvent e)
    {
        Log.Info("PdfReportLogic", $"Report generation completed with result: {e.Result}");
        if (e.Result == GeneratePdfCompletedResult.PdfSuccessfullyGenerated)
        {
            SetOutputMessage("ReportSuccess");
            viewPdfButton.Enabled = true;
        }
        else
        {
            SetOutputMessage("ReportFailed");
            viewPdfButton.Enabled = false;
            Log.Error("PdfReportLogic", $"PDF generation failed, error: {e.Result}");
        }

        generatePdfButton.Enabled = true;
    }

    // Set output message
    private void SetOutputMessage(string translationKey)
    {
        var outputLabel = Owner.Get<FTOptix.UI.Label>("TrackedValues/OutputMessage");
        try
        {
            outputLabel.LocalizedText = new LocalizedText(outputLabel.NodeId.NamespaceIndex, translationKey);
        }
        catch
        {
            Log.Warning("PdfReportLogic", $"Translation key not found: {translationKey}");
            outputLabel.Text = translationKey;
        }
    }

    // Regex for locale ID
    [System.Text.RegularExpressions.GeneratedRegex("^[a-z]{2}-[A-Z]{2}$")]
    private static partial System.Text.RegularExpressions.Regex LocaleIdRegex();
    // PDF generator button
    private FTOptix.UI.Button generatePdfButton;
    private FTOptix.UI.Button viewPdfButton;
    private FTOptix.Report.Report myReport;
}
```
