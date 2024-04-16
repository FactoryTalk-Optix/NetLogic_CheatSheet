# Execute commands

## OPC/UA Methods

See [here](./creating-objects.md)

## Execute external commands (CMD or external softwares)

```csharp
using System.Diagnostics;

static void ExecuteCommand(string command)
{
    int exitCode;
    ProcessStartInfo processInfo;
    Process process;
    // Create the new command and launch it using the command prompt
    processInfo = new ProcessStartInfo("cmd.exe", "/c " + command);
    processInfo.CreateNoWindow = true;
    processInfo.UseShellExecute = false;
    //Redirect the output
    processInfo.RedirectStandardError = true;
    processInfo.RedirectStandardOutput = true;
    // Start the process
    process = Process.Start(processInfo);
    process.WaitForExit();
    // Read the streams
    // Warning: This approach can lead to deadlocks
    string output = process.StandardOutput.ReadToEnd();
    string error = process.StandardError.ReadToEnd();
    // Read the exit code
    exitCode = process.ExitCode;
    // Print some output
    Log.Info("output>>" + (String.IsNullOrEmpty(output) ? "(none)" : output));
    Log.Info("error>>" + (String.IsNullOrEmpty(error) ? "(none)" : error));
    Log.Info("ExitCode: " + exitCode.ToString(), "ExecuteCommand");
    // Close the process handler
    process.Close();
}
```

## Open a WebBrowser to a specific page

```csharp
[ExportMethod]
public void OpenBrowser(string urlPath) {
    //string destinationPath = new ResourceUri(urlPath).Uri;
    string destinationPath = urlPath;
    Process myProcess = new Process();

    try {
        // true is the default, but it is important not to set it to false
        myProcess.StartInfo.UseShellExecute = true;
        myProcess.StartInfo.FileName = destinationPath;
        // Start the process
        myProcess.Start();
    } catch (Exception e) {
        // Print any exception (if needed)
        Log.Error("OpenInBrowser", e.Message);
    }
}
```

## Execute FT Optix Core commands

```csharp
// Get the core commands object (exposed by FT Optix APIs)
var coreCommandsObject = InformationModel.GetObject(FTOptix.CoreBase.Objects.CoreCommands);
// Call a specific command
coreCommandsObject.ExecuteMethod("Close");
```
