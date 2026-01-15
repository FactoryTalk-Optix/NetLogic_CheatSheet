# Calling or running commands

## OPC/UA Methods

See [here](./creating-objects.md)

## Run external commands (CMD or external softwares)

```csharp
using System.Diagnostics;

static void RunCommand(string command)
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
    Log.Info("RunCommand", "output>>" + (String.IsNullOrEmpty(output) ? "(none)" : output));
    Log.Info("RunCommand", "error>>" + (String.IsNullOrEmpty(error) ? "(none)" : error));
    Log.Info("RunCommand", "ExitCode: " + exitCode.ToString());
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

## Invoike FT Optix Core commands

```csharp
// Get the core commands object (exposed by FT Optix APIs)
var coreCommandsObject = InformationModel.GetObject(FTOptix.CoreBase.Objects.CoreCommands);
// Call a specific command
coreCommandsObject.ExecuteMethod("Close");
```

## Invoke FT Optix Alarm commands

```csharp
// Get the alarms object
var alarmCommands = InformationModel.GetObject(FTOptix.Alarm.Objects.AlarmCommands);
// Acknowledge all alarms
alarmCommands.ExecuteMethod("AcknowledgeAll");
// Confirm all alarms
alarmCommands.ExecuteMethod("ConfirmAll");
```

## Perform system shutdown (Windows only)

```csharp
/// <summary>
/// Trigger a Windows shutdown using the system 'shutdown' command.
/// Use with caution; this will stop the entire machine.
/// </summary>
[ExportMethod]
public void SystemShutDown()
{
    var psi = new ProcessStartInfo("shutdown", "/s /t 0");
    psi.CreateNoWindow = true;
    psi.UseShellExecute = false;
    Process.Start(psi);
}
```

## Run a .BAT or external executable

```csharp
/// <summary>
/// Start a BAT file or external program. Adjust filename and parameters as needed.
/// On Windows, you can call cmd with /k or /c; /k keeps the window open.
/// </summary>
private void ExecuteExternalCommand()
{
    var Test = new Process();
    string filename = @"C:\Path\To\Script.bat";
    string parameters = $"/k \"{filename}\"";
    Process.Start("cmd", parameters);
}
```

```csharp
// Example: launch VLC with parameters
[ExportMethod]
public void OpenVLC()
{
   Process VLC = new Process();
   VLC.StartInfo.FileName = @"C:\Program Files (x86)\VideoLAN\VLC\vlc.exe";
   VLC.StartInfo.Arguments = " --fullscreen --play-and-exit C:\\path\\to\\video.mp4";
   VLC.Start();
}
```

## Invoke a method from a different NetLogic

Calling a NetLogic instance with `var myNetLogic = new MyNetLogic();` will create a new instance of the NetLogic, but it will not be the same as the one that is running in the project so it should be avoided. To call a method from a different NetLogic, you need to get the NetLogic object from the project and then call the method on that object.

```csharp
[ExportMethod]
public void CallMethod1()
{
    // Get the NetLogic containing the method to invoke
    var myScript = Project.Current.GetObject("NetLogic/RuntimeNetLogic1");
    // Launch the method
    myScript.ExecuteMethod("Method1");
}
```

### Invoke a method from a different NetLogic with parameters

#### NetLogic to be called

```csharp
public class AlarmGridLogic : BaseNetLogic
{
    [ExportMethod]
    public void AckAllAlarmsWithMessage(object[] arguments)
    {
        var ackMessage = (LocalizedText) arguments[0];
        var alarmsList = GetAllAlarms();
        if (alarmsList.Count == 0)
        {
            Log.Warning("No alarms to acknowledge");
            return;
        }
        ProcessAlarms(ackMessage, alarmsList, (alarm, message) => alarm.Acknowledge(message));
    }
}
```

#### Caller

```csharp
public class AckConfirmMessageLogic : BaseNetLogic
{
    [ExportMethod]
    public void PerformAction()
    {
        // Get the target NetLogic object
        var alarmsNetLogic = GetNetLogicObject();
        // Get the action name to be invoked
        string action = Owner.GetVariable("MethodName").Value;
        // Get the comment to be added to the alarm
        var comment = Owner.Get<TextBox>("Comment").LocalizedText;
        // Create the object arguments to be passed to the method
        object[] arguments = new object[] { comment };
        alarmsNetLogic.ExecuteMethod(action, [arguments]);
        (Owner as Dialog)?.Close();
    }

    private IUAObject GetNetLogicObject()
    {
        // Here the NetLogic is passed as a variable to the owner object,
        // but it could be retrieved in a different way (like Project.Current.GetObject)
        return InformationModel.GetObject(Owner.GetVariable("AlarmGridLogic").Value);
    }
}
```
