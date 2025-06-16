# Field tags

Field tags are functionally identical to Model variables (global variables), but with some additions.

Field tags can be forced to read or write a value to the field device (a.k.a. PLC) or to force the value to be synched to the controller with a specific refresh time.

> [!NOTE]
> Tags are automatically synchronized with the controller when they are used in:
> - A page being displayed in any of the UI sessions
> - Used in a DataLogger or EventLogger
> - Used as InputVariable of an Alarm
> - A ValueChange event is subscribed to the tag
>
> If none of these conditions are met, the tag will not be synchronized with the controller and will not be updated, so the following techniques to read or write values to the controller are needed.

## Introduction

Field tags in FactoryTalk Optix provide several methods for efficiently communicating with PLCs, each with different performance characteristics and use cases. Understanding these methods helps optimize both application performance and network traffic.

### Tags access methods

1. **RemoteVariableSynchronizer**

   - Optimizes data exchange by bundling requests and minimizing network traffic
   - Efficient for continuous monitoring of multiple tags at specified intervals
   - Expensive during initial setup but provides optimized ongoing communication
   - After setup, exchanges tag values in the most optimized way possible
   - Best for scenarios requiring regular polling of many dispersed tags

2. **RemoteRead/RemoteWrite**

   - Provides synchronous, on-demand reading/writing of specific tags
   - Best for atomic operations when immediate values are needed
   - Less efficient for large numbers of tags
   - InformationModel.RemoteRead() can bundle multiple tag reads for better performance

### RA Ethernet/IP Controller Array Reading (Optix 1.6+)

Starting from FactoryTalk Optix 1.6.x, a major optimization was introduced in the RA Ethernet/IP driver, which allows reading large arrays directly from the controller, significantly improving performance for bulk data operations. The user does not need to manually specify the read mode, as the system automatically optimizes the read operation for large arrays if possible.

This new algorithm:
- Is highly efficient for reading large blocks of contiguous data
- Can read large arrays (e.g., 10K DINT array) in ~200ms
- Requires data to be organized in arrays within the controller

Example:

```csharp
var tag = Project.Current.Get<Tag>("CommDrivers/RAEtherNet_IPDriver1/RAEtherNet_IPStation1/Tags/Controller Tags/DINT_A10K");
var value = tag.RemoteRead();
var arrayInts = (int[])value.Value;
```

### Performance Considerations

- **Network Traffic**: The `RemoteVariableSynchronizer` drastically reduces network load. For example, reading 10K DINTs with the RA Ethernet/IP driver will require 10x less network traffic when using a `RemoteVariableSynchronizer` compared to a `RemoteRead`.
- **Cached Values**: Using `Project.Current.GetVariable("MyTag").Value` accesses the last cached value without triggering new controller communication
- **Hardware Impact**: Consider your network infrastructure, PLC capabilities and host machine capabilities when designing communication patterns
- **Synchronization Frequency**: Always specify appropriate refresh intervals in Remote Variable Synchronizer to prevent overwhelming the controller (`TimeSpan.FromSeconds()`)

Learning to use these methods effectively can greatly enhance the performance and responsiveness of your FactoryTalk Optix applications, especially in scenarios involving large datasets or frequent updates.

For example:

- A `RemoteVariableSynchronizer` can be created in a global NetLogic to continuously monitor and synchronize multiple tags, ensuring they are always up-to-date without excessive network traffic.
- All the NetLogic in the project can simply access the synchronized variables using the `Project.Current.GetVariable("MyTag").Value` (either getting or setting the value) method without needing to perform individual reads, which is particularly useful for applications with many tags that need regular updates.

## Remote Variable synchronizer

The `RemoteVariableSynchronizer` allows users to keep some variables synched even if not in use by any current page, it will keep all the children elements updated with the field. This method handles an automatic optimization of the Tags in order to minimize the number of read/write from the controller (while the `RemoteRead` performs a request per each tag and it is much slower)

> [!NOTE]
> When the `RemoteVariableSynchronizer` is created, it will collect all the children variables and create an optimized read request to the controller, so it is important to create it only once and not every time a page is opened or a variable is needed.

> [!WARNING]
> The `RemoteVariableSynchronizer` might have significant impact on the communication driver, especially if used with a large number of tags. It is recommended to use it wisely and only for a small number of tags.

### Providing a TimeSpan argument

The `TimeSpan` argument of the `RemoteVariableSynchronizer` should be used to avoid overloading the field device. If the `RemoteVariableSynchronizer` is created without any arguments, it defaults to full-speed synchronization, which can be too fast for the field device to handle. This can lead to performance issues because the iPC (industrial PC) is typically faster than the controller. By specifying a `TimeSpan` argument, you define a refresh interval between subsequent reads, which helps in managing the load on the field device and ensures smoother operation.

Avoid this constructor: `variableSynchronizer = new RemoteVariableSynchronizer();`
Use this instead: `variableSynchronizer = new RemoteVariableSynchronizer(new TimeSpan.FromSeconds(5));`

The argument can be configure as needed, depending on the speed of the synchronization which is required.

### RemoteVariableSynchronizer example

```csharp
public override void Start() 
{
    // Get the field variable
    motorSpeed = LogicObject.Owner.GetVariable("Speed");
    // Create the VariableSynchronizer object and request the update every 5 seconds
    var updateRate = TimeSpan.FromSeconds(5);
    variableSynchronizer = new RemoteVariableSynchronizer(updateRate);
    // Add the variables to the synchronizer
    variableSynchronizer.Add(motorSpeed);
    // Add the event to listen to value changes
    motorSpeed.VariableChange += MotorSpeed_VariableChange;
}
public override void Stop() 
{
    // Destroy the synchronized on stop
    variableSynchronizer?.Dispose();
}
private void MotorSpeed_VariableChange(object sender, VariableChangeEventArgs e) 
{
    // Event listener when value change is detected
    if (motorSpeed.Value > 200) {
        Log.Warning("Speed limit reached!");
    }
}
private IUAVariable motorSpeed;
private RemoteVariableSynchronizer variableSynchronizer;
```

## RemoteRead

Remote read is a inline read of the field value, code is stopped to read the value, once the variable is updated, the code keeps running

> [!NOTE]
> The `ChildrenRemoteRead` is just as "slow" as the `RemoteRead`, it is just a recursive approach to a normal `RemoteRead`

### Read simple tags

```csharp
[ExportMethod]
public void ReadBitFromPlcInteger(out int value)
{
    // Get the plc tag to read
    var tag1 = Project.Current.GetVariable("CommDrivers/RAEtherNet_IPDriver1/RAEtherNet_IPStation1/Tags/Program:CustomUdtProgram/program_dint_1D");
    // Read the new values
    var variableValue = tag1.RemoteRead();
    // Do something with the values
    Log.Info($"Value of the plc tag is: {variableValue}");
}
```

### Read arrays

```csharp
[ExportMethod]
public void ReadBitFromPlcInteger(out int value)
{
    // Get the plc tag to read
    var tag1 = Project.Current.Get<FTOptix.RAEtherNetIP.Tag>("CommDrivers/RAEtherNet_IPDriver1/RAEtherNet_IPStation1/Tags/Program:CustomUdtProgram/program_dint_1D");
    // Create the remote read object
    var remoteVariables = new List<RemoteVariable>()
    {
        // Add the variables to read to the list
        new RemoteVariable(tag1),
    };
    // Read the new values
    var values = InformationModel.RemoteRead(remoteVariables).ToList();
    // Get the field values to a C# variable
    var plcArray = (int[])values[0].Value.Value;
    // Do something with the values
    Log.Info($"Value of first element of the array is {plcArray[0]}");
    Log.Info($"Bit 5 of first element of the array is {GetBitValue(plcArray[0], 5)}");
}

private bool GetBitValue(int inputValue, int bitPosition)
{
    // Read a specific bit from a word
    var array = new BitArray(new int[] { inputValue });
    return array[bitPosition];
}
```

### Adding a timeout

```csharp
// Get the variable to update
var currentSpeed = Project.Current.GetVariable("Model/Motor/Speed");
// Try to get the most updated value with a timeout of 100ms
var newValue = currentSpeed.RemoteRead(100.0);
```

## RemoteWrite

Similar to the `RemoteRead`, when a value should be sent immediately to the field, a `RemoteWrite` method can be used

### Write a simple value

```csharp
// Which tag to update
var currentSpeed = Project.Current.GetVariable("Model/Motor/Speed");
// Write the new value (no write timeout)
currentSpeed.RemoteWrite(42);
```

### Write multiple values

```csharp
// Which tags to update
var tag1 = Project.Current.Get<Tag>("CommDrivers/CodesysDriver1/CodesysStation1/Tags/Application/PLC_PRG/VAR1");
var tag3 = Project.Current.Get<Tag>("CommDrivers/CodesysDriver1/CodesysStation1/Tags/Application/PLC_PRG/VAR3");
// Create the RemoteWrite object
var remoteVariableValues = new List<RemoteVariableValue>()
{
    new RemoteVariableValue(tag3, 0),
    new RemoteVariableValue(tag1, 123)
};
// Perform the write operation (no write timeout)
InformationModel.RemoteWrite(remoteVariableValues);
```

### Adding a timeout

```csharp
// Get the variable to update
var currentSpeed = Project.Current.GetVariable("Model/Motor/Speed");
// Try to write the value 150 with a timeout of 100ms
currentSpeed.RemoteWrite(150, 100.0);
```
