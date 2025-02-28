# OPC/UA Observers

> [!WARNING]
> Observers are low-level features and should be used with caution. 
> This feature is not officially supported and may be subject to changes in future releases.

The OPC/UA protocol implements some ways to monitor changes in the nodes tree, this allows the creation of observers that can monitor some project nodes and triggers some logics based on a set of events (which can be specified using a mask).

This technique allows multiple scenarios like:

- Detecting the nodes status (started, stopped, attached, detached, dispose)
- Detecting changes in values
- Detecting changes in attributes
- Detecting changes in references
- Detecting OPC/UA events

Subscribers must be added to all the elements that needs to be monitored, attaching an observer to the `Model` folder, won't enable the listener on all children nodes

## References to OPC/UA Specifications

- **OPC UA Part 4 - Services**: Describes the services provided by the OPC UA server, including event subscriptions.
- **OPC UA Part 9 - Alarms & Conditions**: Provides details on how events and conditions are managed in OPC UA.

For more detailed information, refer to the [OPC UA Specifications](https://opcfoundation.org/developer-tools/specifications-unified-architecture).

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

    // This node must stay "alive" as long as the subscriber is needed
    // Do not move it to a local variable or the observer won't work
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

## Subscribe to methods execution

This script can be put anywhere on the project and will listen to every OPCUA method being called (script or UI)

```csharp
public class RuntimeNetLogic1 : BaseNetLogic
{
    public override void Start()
    {
        var serverObject = LogicObject.Context.GetObject(OpcUa.Objects.Server);
        var eventHandler = new EventsHandler.EventsHandler();
        eventRegistration = serverObject.RegisterUAEventObserver(eventHandler, UAManagedCore.OpcUa.ObjectTypes.AuditUpdateMethodEventType);
    }

    public override void Stop()
    {
        // Insert code to be executed when the user-defined logic is stopped
    }

    private IEventRegistration eventRegistration;
}

namespace EventsHandler
{
    public class EventsHandler : IUAEventObserver
    {
        public EventsHandler()
        {
            Log.Info("EventsHandler", "EventsHandler instance created");
        }

        public void OnEvent(IUAObject eventNotifier, IUAObjectType eventType, IReadOnlyList<object> eventData, ulong senderId)
        {
            StringBuilder builder = new StringBuilder();
            builder.Append($"Event of type {eventType.BrowseName} triggered");
            var eventArguments = eventType.EventArguments;
            foreach (var eventField in eventArguments.GetFields())
            {
                var fieldValue = eventArguments.GetFieldValue(eventData, eventField);
                builder.Append($"\t{eventField} = {fieldValue?.ToString() ?? "null"}");
            }
            builder.Append("\n");
            Log.Info(builder.ToString());
        }
    }
}
```
