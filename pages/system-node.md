# The system object

The "System" object is a special node which provides some methods and events to interact with Optix devices (like OptixPanel, OptixEdge, etc)

## Load a new application

Please note:
- Application must be exported by FactoryTalk Optix Studio with the same version that was previously used to deploy the original application
- Exported application must be protected by a password
- The import procedure might take some time, please wait for the confirmation message

```csharp
#region Using directives
using System;
using UAManagedCore;
using OpcUa = UAManagedCore.OpcUa;
using FTOptix.UI;
using FTOptix.NativeUI;
using FTOptix.HMIProject;
using FTOptix.Retentivity;
using FTOptix.CoreBase;
using FTOptix.Core;
using FTOptix.NetLogic;
using FTOptix.System;
using System.IO;
using System.Runtime.Versioning;
#endregion

public class RuntimeNetLogic1 : BaseNetLogic
{
    public override void Start()
    {
        // Nothing to do
    }

    public override void Stop()
    {
        systemNode.OnLoadApplicationStatusEvent -= SystemNode_OnLoadApplicationStatusEvent;
    }

    [ExportMethod]
    public void LoadNewApplication(string password, bool deleteApplicationFiles)
    {
        // Get to the system node
        systemNode = Project.Current.Get<FTOptix.System.System>("System/System");
        // Get the new application path
        var newApplicationPath = ResourceUri.FromProjectRelativePath("NewApplication.z");

        // Subscribe to the application load event
        systemNode.OnLoadApplicationStatusEvent += SystemNode_OnLoadApplicationStatusEvent;

        // Load the new application
        if (systemNode.LoadApplication(newApplicationPath.Uri, password, deleteApplicationFiles))
        {
            // Call to the SystemManager API to load the new application was successful
            Log.Info("LoadApplication", "Load application call was successful, please wait for the process to complete");
        }
        else
        {
            // Call to the SystemManager API to load the new application failed
            Log.Error("LoadApplication", "Failed to load application, call to the SystemManager APIs failed");
        }
    }

    private void SystemNode_OnLoadApplicationStatusEvent(object sender, LoadApplicationStatusEvent e)
    {
        if (e.Status == LoadApplicationStatus.Success)
        {
            // Application loaded successfully
            Log.Info("LoadApplicationStatus", "Application loaded successfully");
        }
        else
        {
            // Application failed to load
            Log.Error("LoadApplicationStatus", "Failed to load application: " + e.Status);
        }
    }

    FTOptix.System.System systemNode;
}
```

## Example: system and session variables (user, time, USB, IP)

```csharp
/// <summary>
/// Periodically populate NetLogic variables with system and session information
/// (active user, locale, current time components, random values and network/IP).
/// Intended for Runtime NetLogic.
/// </summary>
private Random rnd = new Random();

private void OneSecondSystemVariables()
{
    // User and session info
    LogicObject.GetVariable("ActiveUser").Value = Session.User.BrowseName;
    LogicObject.GetVariable("ActiveUserLocaleId").Value = Session.User.LocaleIds[0];
    try { LogicObject.GetVariable("ActiveUserLanguage").Value = Session.User.Languages[0]; }
    catch { LogicObject.GetVariable("ActiveUserLanguage").Value = "Language not selected"; }

    // Time information
    LogicObject.GetVariable("ActTimeSec").Value = DateTime.Now.Second;
    LogicObject.GetVariable("ActTimeMin").Value = DateTime.Now.Minute;
    LogicObject.GetVariable("ActTimeHour").Value = DateTime.Now.Hour;
    LogicObject.GetVariable("ActTimeDay").Value = DateTime.Now.Day;
    LogicObject.GetVariable("ActTimeMonth").Value = DateTime.Now.Month;
    LogicObject.GetVariable("ActTimeYear").Value = DateTime.Now.Year;
    LogicObject.GetVariable("ActTimeDayOfWeek").Value = (int)DateTime.Now.DayOfWeek;
    LogicObject.GetVariable("ActTimeString").Value = DateTime.Now.ToShortTimeString();
    LogicObject.GetVariable("ActDateString").Value = DateTime.Now.ToShortDateString();

    // Random numbers
    LogicObject.GetVariable("SimRandInt").Value = rnd.Next(32768);
    LogicObject.GetVariable("SimSinDouble").Value = Math.Sin(Math.PI * rnd.Next(361) / 180.0);
    LogicObject.GetVariable("SimCosDouble").Value = Math.Cos(Math.PI * rnd.Next(361) / 180.0);

    // Active alarms count (safe navigation)
    IContext context = LogicObject.Context;
    var retainedAlarms = context.GetNode(QPlatform.Alarm.Objects.RetainedAlarms);
    var localizedAlarmsVariable = retainedAlarms?.Children.Get<IUAVariable>("LocalizedAlarms");
    var localizedAlarmsNodeId = localizedAlarmsVariable != null ? (NodeId)localizedAlarmsVariable.Value : NodeId.Empty;
    IUANode localizedAlarmsContainer = null;
    if (localizedAlarmsNodeId != null && !localizedAlarmsNodeId.IsEmpty)
        localizedAlarmsContainer = context.GetNode(localizedAlarmsNodeId);
    LogicObject.Children.Get<IUAVariable>("NumActiveAlarms").Value = localizedAlarmsContainer?.Children.Count ?? 0;
    LogicObject.Children.Get<IUAVariable>("LastAlarmText").Value = localizedAlarmsContainer?.Children.LastOrDefault()?.BrowseName ?? "";

    // USB present detection using ResourceUri catch
    var pathUsbResourceUri = new ResourceUri("%USB1%");
    try { var uri = pathUsbResourceUri.Uri; LogicObject.GetVariable("UsbPresent").Value = true; }
    catch { LogicObject.GetVariable("UsbPresent").Value = false; }

    // Network parameters (Windows only)
    if (LogicObject.GetVariable("Windows").Value)
    {
        try
        {
            string hostName = Dns.GetHostName();
            var host = Dns.GetHostEntry(hostName);
            var ipv4s = host.AddressList.Where(a => a.AddressFamily == System.Net.Sockets.AddressFamily.InterNetwork).ToArray();
            if (ipv4s.Length > 0) LogicObject.GetVariable("IP1").Value = ipv4s[0].ToString();
            if (ipv4s.Length > 1) LogicObject.GetVariable("IP2").Value = ipv4s[1].ToString();
        }
        catch (Exception ex)
        {
            Log.Warning("OneSecondSystemVariables", "Unable to read IP addresses: " + ex.Message);
        }
    }
}
```
