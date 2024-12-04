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
        // Insert code to be executed when the user-defined logic is started
    }

    public override void Stop()
    {
        // Insert code to be executed when the user-defined logic is stopped
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