# Triggering events to the FactoryTalk Optix core

## Introduction to OPC/UA Events

OPC Unified Architecture (OPC UA) is a machine-to-machine communication protocol for industrial automation developed by the OPC Foundation. One of the key features of OPC UA is its ability to handle events. Events in OPC UA are notifications about changes or occurrences within the system that are of interest to clients.

### What are OPC/UA Events?

OPC UA events provide a mechanism for servers to notify clients about significant changes or occurrences. These events can be anything from simple state changes to complex alarms and conditions. Events are used to inform clients about changes in the system without the need for constant polling, making the communication more efficient.

### Types of Events

OPC UA defines several types of events, including:

- **Simple Events**: Basic notifications about changes or occurrences.
- **Condition Events**: Notifications about conditions that have a state and can change over time.
- **Alarm Events**: Special types of condition events that indicate abnormal or critical conditions.

### Event Model

The OPC UA event model is based on the concept of event types and event instances. Event types define the structure and semantics of events, while event instances are actual occurrences of these events. The event model allows for the creation of custom event types to suit specific application needs.

### References to Specifications

For more detailed information about OPC UA events, refer to the following OPC Foundation specifications:

- [OPC Unified Architecture, Part 3: Address Space Model](https://opcfoundation.org/developer-tools/specifications-unified-architecture/part-3-address-space-model/)
- [OPC Unified Architecture, Part 5: Information Model](https://opcfoundation.org/developer-tools/specifications-unified-architecture/part-5-information-model/)
- [OPC Unified Architecture, Part 9: Alarms and Conditions](https://opcfoundation.org/developer-tools/specifications-unified-architecture/part-9-alarms-conditions/)

These documents provide comprehensive details on the structure, types, and handling of events in OPC UA.

## Dispatching an OPC/UA event

> [!WARNING]
> Events are not directly supported in the FactoryTalk Optix IDE, and the code provided here is an example of how to manually trigger them using custom logic. This code is for reference only and should be adapted to your specific use case.

### 1. Creating the dispatch method

```csharp
[ExportMethod]
public void TriggerUserSessionEvent(object[] inputArgs)
{
    // Convert input arguments to the expected types
    var sourceName = (string) inputArgs[0];
    var sourceNode = (NodeId) inputArgs[1];
    var clientUserId = (string) inputArgs[2];
    var status = (bool) inputArgs[3];
    var message = new LocalizedText((string) inputArgs[4], "en-US");

    // Get the UserSessionEvent object type
    var userSessionEvent = (IUAObjectType) InformationModel.Get(FTOptix.Core.ObjectTypes.UserSessionEvent);

    // Create a list to store the event arguments
    List<object> argumentList = [];
    if (userSessionEvent.EventArguments != null)
    {
        argumentList.AddRange(new object[userSessionEvent.EventArguments.GetFields().Count]);
        foreach (var field in userSessionEvent.EventArguments.GetFields())
        {
            // Set the field value based on the field name
            object fieldValue = field switch
            {
                "EventId" => GenerateEventIdFromNodeId((Guid) Session.NodeId.Id),
                "EventType" => FTOptix.Core.ObjectTypes.UserSessionEvent,
                "SourceName" => sourceName,
                "SourceNode" => sourceNode,
                "ClientUserId" => clientUserId,
                "Status" => status,
                "Message" => message,
                "Time" => DateTime.Now,
                "Severity" => new Random().Next(1, 500),
                _ => null
            };
            if (fieldValue != null)
            {
                userSessionEvent.EventArguments.SetFieldValue(argumentList, field, fieldValue);
            }
        }
        // Dispatch the event to the Server object
        LogicObject.Context.GetObject(OpcUa.Objects.Server).DispatchUAEvent(FTOptix.Core.ObjectTypes.UserSessionEvent, argumentList.AsReadOnly());
    }
}

public static ByteString GenerateEventIdFromNodeId(Guid nodeId)
{
    string baseString = nodeId.ToString() + DateTime.UtcNow.Ticks.ToString();
    using SHA256 sha256 = SHA256.Create();
    byte[] hashBytes = sha256.ComputeHash(Encoding.UTF8.GetBytes(baseString));
    return new ByteString(hashBytes);
}
```

### 2. Triggering the event

```csharp

[ExportMethod]
public void TriggerEvent()
{
    DispatchEventToCore(Session.User.BrowseName, false, "Custom message triggered from a button");
}

private void DispatchEventToCore(string username, bool successful, string message)
{
    // Get the EventDispatcher NetLogic
    // This is the NetLogic object that dispatches events to the FactoryTalk Optix core
    // as created in the previous section
    var eventsDispatcherLogic = Project.Current.GetObject("NetLogic/EventsDispatcher");
    if (eventsDispatcherLogic == null)
    {
        Log.Error("LoginButtonLogic", "EventsDispatcher NetLogic not found");
        return;
    }
 
    // Create the input arguments for the event
    // The arguments are passed as an object array, to find the list of arguments
    // and the expected values, you can use an EventLogger object and generate
    // an event to see the expected arguments
    var inputArgs = new object[]
    {
        "LoginForm", // SourceName -> The friendly name of the source of the event
        Owner.NodeId, // SourceNode -> The NodeId of the source of the event
        username, // ClientUserId -> The user associated with the event
        successful, // Status -> The status of the event (boolean)
        message // Message -> The message associated with the event
    };

    // Trigger the event
    eventsDispatcherLogic.ExecuteMethod("TriggerUserSessionEvent", [inputArgs]);
}
```
