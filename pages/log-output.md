# Log messages

Log messages can be displayed both at RunTime and DesignTime, at DesignTime logs will appear in `Optix Studio Output` while at RunTime they will appear in the `Emulator output` or the RunTime log

## General syntax

`Log . [level] ( [optional category] , [ message content] );`

## Examples

Different log levels can be triggered and then filtered in `Optix Studio Output` and `Emulator output`. Specifying an optional custom `Category` helps you understanding where this line was generated (for example the NetLogic name and/or the method name)

- `Log.Error(category, message);`
- `Log.Warning(category, message);`
- `Log.Info(category, message);`
- `Log.Debug(category, message);`
- `Log.Verbose1(category, message);`
- `Log.Verbose2(category, message);`

## Get the node path for any project node

- `Log.Node(IUANode);` -> Returns a string containing the node path for a specific element

## Getting total count of project nodes

- `LogicObject.Context.NodeCount` -> returns the total number of nodes (whole project)

## Intercepting FactoryTalk Optix Runtime messages

The messages that are printed to the "Runtime Output" (or "Emulator Output") are generated as per OPC/UA specs by the Server object, which can be subscribed to trigger custom logics

```csharp
#region Using directives
using System;
using UAManagedCore;
using OpcUa = UAManagedCore.OpcUa;
using FTOptix.NetLogic;
using System.Collections.Generic;
using LogsHandler;
#endregion

public class RuntimeLoggerMonitor : BaseNetLogic
{
    public override void Start()
    {
        // Build the observer for the output messages
        StartObserver();
    }

    public override void Stop()
    {
        // Destroy the observer when closing the NetLogic
        logsRegistration?.Dispose();
    }

    public void StartObserver()
    {
        // Get the server object
        var serverObject = LogicObject.Context.GetObject(OpcUa.Objects.Server);
        // Create a new observer
        var logsObserver = new LogsEventObserver();
        // Register the observer to the server node
        logsRegistration = serverObject.RegisterUAEventObserver(logsObserver, FTOptix.Core.ObjectTypes.LogEvent);
    }

    private IEventRegistration logsRegistration;
}

namespace LogsHandler
{
    class LogsEventObserver : IUAEventObserver
    {
        public LogsEventObserver()
        {
            Log.Info("LogsEventObserver", "Starting observer");
        }

        public void OnEvent(IUAObject eventNotifier, IUAObjectType eventType, IReadOnlyList<object> args, ulong senderId)
        {
            Console.WriteLine(((LocalizedText) args[7]).Text); // Content of the message
            Console.WriteLine(args[15]); // Module name
            Console.WriteLine(args[13]); // Message level
            // Do not use `Log.` events here or it will loop!
        }
    }
}

```

## Reading debug logs

When debug logs are enabled, the output of some rendering methods is logged to the output, here is an example:

```txt
2024-10-22 10:55:05     NativeUI    StartNetLogics,10,6,600.000
```

The syntax is:

```txt
[Timestamp]    [Module Name]    [Method Name];[Total time in ms],[Number of processed items],[Items per second]
```

So, in the example above we get the NativeUI:

- Executing the `Start` method of all NetLogics in the page
- Starting a total of 6 NetLogics
- Running at a theoretical 600 NetLogics per second can be loaded (this is something like benchmarking)
