# MQTT

The MQTT (Message Queuing Telemetry Transport) protocol is a lightweight messaging protocol designed for constrained devices and low-bandwidth, high-latency, or unreliable networks. It is widely used in IoT (Internet of Things) applications for its efficiency and simplicity.

Starting from FactoryTalk Optix 1.6.X, the MQTT module provides built-in support for MQTT, allowing you to easily integrate MQTT messaging into your applications.

## General Concepts about MQTT in FactoryTalk Optix

FactoryTalk Optix provides a robust MQTT module that allows you to:
- Publish and subscribe to topics using the `MQTTClient` and `MQTTPublisher` classes.
- Manage incoming and outgoing messages efficiently.
- Dynamically configure MQTT clients and publishers.

## Using MQTT in FactoryTalk Optix

### Managing MQTT Client IDs

The `MQTTClient` class allows you to dynamically set the `ClientId` for an MQTT client. This is useful for ensuring unique identifiers for clients in a network.

```csharp
public override void Start()
{
    // Retrieve the MQTT client instance from the project
    var mqttClient = Project.Current.Get<MQTTClient>("MQTT/MQTTClient");
    
    // Check if the default ClientId is set and update it to a unique value
    if (mqttClient.ClientId == "FTOptix-1")
    {
        mqttClient.ClientId = $"OptixUser-{Guid.NewGuid().ToString().Split("-")[0]}";
        
        // Restart the client to apply the new ClientId
        mqttClient.Stop();
        mqttClient.Start();
        
        // Log the updated ClientId
        Log.Info("ClientIdGeneratorLogic", $"ClientId was set to {mqttClient.ClientId}");
    }
}
```

### Handling Incoming MQTT Messages

As the `MQTTSubscriber`, when set to `PlainTextSubscription` allows to read messages from a topic and write the received data to a Model variable, you can use the `VariableChange` event to process incoming MQTT messages. This ensures that your application can react to new messages in real-time.

```csharp
public override void Start()
{
    // Get the variable that stores incoming MQTT messages
    mqttIncomingMessage = Project.Current.GetVariable("Model/Data/MQTT/IncomingMessage");
    
    // Attach an event handler to process changes in the variable
    mqttIncomingMessage.VariableChange += MqttIncomingMessagePile_VariableChanged;
}

private void MqttIncomingMessagePile_VariableChanged(object sender, VariableChangeEventArgs e)
{
    // Ensure the new value is not null
    if (e.NewValue == null)
        return;

    // Retrieve the incoming message as a string
    var incomingMessage = (string)e.NewValue;
    
    // Log the received message for debugging purposes
    Log.Debug("MqttChatLogic", $"Received MQTT message: {incomingMessage}");
}
```

### Displaying MQTT Messages in the UI

The `ScrollView` and `ColumnLayout` classes can be used to dynamically display incoming MQTT messages in the user interface. These classes are part of the FactoryTalk Optix UI framework and allow developers to create flexible and dynamic layouts for displaying data.

- **`ScrollView`**: This is a container that provides scrolling functionality for its child elements. It ensures that the user can view all messages even if the content exceeds the visible area.
- **`ColumnLayout`**: This layout organizes its child elements vertically, making it ideal for displaying a list of messages.
- **`MqttChatMessage`**: This is a custom UI element that represents a single MQTT message. It includes variables such as `Content` to store the message text.

The following code demonstrates how to use these classes:

```csharp
private void MqttIncomingMessagePile_VariableChanged(object sender, VariableChangeEventArgs e)
{
    // Create a new column layout to organize messages vertically
    var newColumnLayout = InformationModel.Make<ColumnLayout>("VerticalLayout");
    
    // Retrieve the pile of messages from the variable
    var messagesPile = (string[])mqttIncomingMessagePile.Value.Value;
    
    foreach (var message in messagesPile)
    {
        // Skip empty messages
        if (string.IsNullOrEmpty(message))
            continue;
        
        // Create a new UI element for the message
        var textBlock = InformationModel.Make<MqttChatMessage>(Guid.NewGuid().ToString());
        
        // Deserialize the message content
        var mqttMessage = MqttChatLogic.SampleChatMessage.Deserialize(message);
        
        // Set the content of the text block
        textBlock.GetVariable("Content").Value = mqttMessage.Content;
        
        // Add the text block to the column layout
        newColumnLayout.Add(textBlock);
    }
    
    // Clear the existing layout and add the new one
    scrollView.Get("VerticalLayout")?.Delete();
    scrollView.Add(newColumnLayout);
}
```

#### Serialization and Deserialization

The `SampleChatMessage` class is used to represent an MQTT message. It includes properties such as `Sender`, `Content`, and `Timestamp`. This class uses JSON serialization to convert the message into a string format for transmission and deserialization to reconstruct the object from the string.

- **Serialization**: Converts the object into a JSON string using `JsonSerializer.Serialize`.
- **Deserialization**: Converts the JSON string back into an object using `JsonSerializer.Deserialize`.

> [!NOTE]
> The MQTT publisher works with any kind of string message as payload, but using serialization allows to structure the data and include multiple fields in a single message and it is widely used in IoT applications. Creating a custom class and using the serialization/deserialization methods makes it easier to manage the message content.

```csharp
public class SampleChatMessage
{
    public string Sender { get; }
    public string Content { get; }
    public string Timestamp { get; }

    [JsonConstructor]
    public SampleChatMessage(string sender, string content, string timestamp)
    {
        this.Sender = sender;
        this.Content = content;
        this.Timestamp = timestamp ?? new DateTime(0).ToString("yyyy/MM/dd HH:mm:ss");
    }

    public string Serialize()
    {
        return JsonSerializer.Serialize(this);
    }

    public static SampleChatMessage Deserialize(string json)
    {
        try
        {
            return JsonSerializer.Deserialize<SampleChatMessage>(json);
        }
        catch (JsonException ex)
        {
            Log.Warning("MqttChatLogic", $"Failed to deserialize MQTT message: {ex.Message}, content: {json}");
            return null;
        }
    }
}
```

### Sending MQTT Messages

You can publish messages to an MQTT topic using the `MQTTPublisher` class. This allows you to send data to other clients or systems. The `MQTTPublisher` is configured with a topic, quality of service (QoS), and retain flag, which determine how the message is delivered.

The following code demonstrates how to send a message:

```csharp
[ExportMethod]
public void SendMessage(string content)
{
    // Retrieve the MQTT publisher and client instances
    var mqttPublisher = Project.Current.Get<MQTTPublisher>("MQTT/Mosquitto_MQTTClient/MQTTPublisher");
    var mqttClient = Project.Current.Get<MQTTClient>("MQTT/Mosquitto_MQTTClient");
    
    // Create a new MQTT message with the provided content
    var dateTime = DateTime.UtcNow.ToString("yyyy/MM/dd HH:mm:ss");
    var mqttMessage = new SampleChatMessage("ClientId", content, dateTime);
    
    // Serialize the MQTT configuration and message payload
    var mqttConfigJson = new OptixMqttConfig(mqttPublisher).Serialize();
    var mqttMessageJson = mqttMessage.Serialize();
    
    // Publish the message to the configured topic
    mqttClient.Publish(mqttConfigJson, mqttMessageJson);
}
```

#### Why Serialization is Needed

Serialization is essential for converting the `SampleChatMessage` object into a format that can be transmitted over the MQTT protocol. The JSON format is lightweight and widely supported, making it ideal for IoT applications. By serializing the message, you ensure that all its properties are preserved during transmission.

#### Understanding `OptixMqttConfig`

The `OptixMqttConfig` class is a helper class designed to encapsulate the configuration details of an MQTT publisher. It simplifies the process of serializing and transmitting the configuration data required for publishing messages.

```csharp
public class OptixMqttConfig
{
    public string Topic { get; }
    public int QoS { get; }
    public bool Retain { get; }

    public OptixMqttConfig(MQTTPublisher mqttPublisher)
    {
        Topic = mqttPublisher.Topic;
        QoS = (int) mqttPublisher.QoS;
        Retain = mqttPublisher.Retain;
    }

    public string Serialize()
    {
        return JsonSerializer.Serialize(this);
    }
}
```

##### Key Properties

- **`Topic`**: The MQTT topic to which the message will be published. This is a string that acts as the address for the message.
- **`QoS`**: The Quality of Service level for the message. It determines the guarantee of message delivery
- **`Retain`**: A boolean flag indicating whether the message should be retained by the broker. Retained messages are delivered to new subscribers immediately upon subscription.

##### Example Usage

The `OptixMqttConfig` class is used in the `SendMessage` method to serialize the MQTT publisher's configuration:

```csharp
var mqttPublisher = Project.Current.Get<MQTTPublisher>("MQTT/Mosquitto_MQTTClient/MQTTPublisher");
var mqttConfig = new MqttChatLogic.OptixMqttConfig(mqttPublisher);
var mqttConfigJson = mqttConfig.Serialize();
```

This serialized configuration is then passed as the first argument to the `Publish` method of the `MQTTClient` class.

## Best Practices for MQTT in FactoryTalk Optix

- **Dispose Resources:** Always dispose of MQTT clients and tasks when they are no longer needed to free up resources.
- **Handle Errors Gracefully:** Use try-catch blocks to handle exceptions during MQTT operations.
- **Use Unique Client IDs:** Ensure that each MQTT client has a unique `ClientId` to avoid conflicts.
- **Optimize Message Handling:** Use efficient data structures and algorithms to process incoming messages.
