# Field tags

Field tags are functionally identical to Model variables (global variables), but with some additions.

Field tags can be forced to read or write a value to the field device (a.k.a. PLC) or to force the value to be synched to the controller with a specific refresh time.

## Variable synchronizer

The `VariableSynchronizer` allows users to keep some variables synched even if not in use by any current page, it will keep all the children elements updated with the field. This method handles an automatic optimization of the Tags in order to minimize the number of read/write from the controller (while the `RemoteRead` performs a request per each tag and it is much slower)

### Providing a TimeSpan argument

The `TimeSpan` argument of the `RemoteVariableSynchronizer` should be used to avoid overloading the field device. If the `RemoteVariableSynchronizer` is created without any arguments, it defaults to full-speed synchronization, which can be too fast for the field device to handle. This can lead to performance issues because the iPC (industrial PC) is typically faster than the controller. By specifying a `TimeSpan` argument, you define a refresh interval between subsequent reads, which helps in managing the load on the field device and ensures smoother operation.

Avoid this constructor: `variableSynchronizer = new RemoteVariableSynchronizer();`
Use this instead: `variableSynchronizer = new RemoteVariableSynchronizer(new TimeSpan.FromSeconds(5));`

The argument can be configure as needed, depending on the speed of the synchronization which is required.

### Please note

- The `ChildrenRemoteRead` is just as "slow" as the `RemoteRead`, it is just a recursive approach to a normal `RemoteRead`
- The `VariableSynchronizer` is made to sync a small number of tags and may have significant impact on the communication driver, make sure to use it wisely

```csharp
public override void Start() 
{
    // Get the field variable
    motorSpeed = LogicObject.Owner.GetVariable("Speed");
    // Create the VariableSynchronizer object
    variableSynchronizer = new RemoteVariableSynchronizer(new TimeSpan.FromSeconds(5));
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
