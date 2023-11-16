# Execute commands

## OPC/UA Methods

See [here](./creating-objects.md)

## Execute external commands (CMD or external softwares)

```csharp
static void ExecuteCommand(string command)
{
    int exitCode;
    ProcessStartInfo processInfo;
    Process process;

    processInfo = new ProcessStartInfo("cmd.exe", "/c " + command);
    processInfo.CreateNoWindow = true;
    processInfo.UseShellExecute = false;
    // *** Redirect the output ***
    processInfo.RedirectStandardError = true;
    processInfo.RedirectStandardOutput = true;

    process = Process.Start(processInfo);
    process.WaitForExit();

    // *** Read the streams ***
    // Warning: This approach can lead to deadlocks, see Edit #2
    string output = process.StandardOutput.ReadToEnd();
    string error = process.StandardError.ReadToEnd();

    exitCode = process.ExitCode;

    Log.Info("output>>" + (String.IsNullOrEmpty(output) ? "(none)" : output));
    Log.Info("error>>" + (String.IsNullOrEmpty(error) ? "(none)" : error));
    Log.Info("ExitCode: " + exitCode.ToString(), "ExecuteCommand");
    process.Close();
}
```

## Execute FT Optix Core commands

```csharp
var coreCommandsObject = InformationModel.GetObject(FTOptix.CoreBase.Objects.CoreCommands);
coreCommandsObject.ExecuteMethod("Close");
```
