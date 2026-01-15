# Serial port

Communication using the serial port is only allowed using the SerialPort driver provided by FactoryTalk Optix.

On Windows, the name of the COM port must be specified as `COMx`, where `x` is the port number (e.g., `COM1`, `COM2`, etc.). On embedded devices, the name of the serial port is automatically mapped to the serial port index of the device (e.g., `COM0` is mapped to the serial port at index 0 - the embedded serial port of OptixPanels - which is internally mapped to `/dev/serial0`).

> [!TIP]
> For the OptixPanel standard, the embedded serial port is available as `COM1` (internally mapped to `/dev/serial1`).

So, if the project configures `COM0` as serial port name, on Windows it will try to open `COM0`, while on embedded devices it will try to open `/dev/serial0` with no need to change anything on the project.

> [!NOTE]
> All the serial port write operations (like the `Write` and `WriteBytes` methods) are **non-blocking**, meaning that the NetLogic code will continue immediately even if the data is not yet written to the serial port.

## Polling mode

```csharp
public class RuntimeNetlogic1 : BaseNetLogic
{
    private SerialPort serialPort;
    private PeriodicTask periodicTask;
    
    public override void Start()
    {
        serialPort = (SerialPort)Owner;
        periodicTask = new PeriodicTask(Read, 300, Owner);
        periodicTask.Start();
    }

    private void Read()
    {
        try
        {
            ReadImpl();
        }
        catch (Exception ex)
        {
            Log.Error("Failed to read from Modbus: " + ex);
        }
    }

    private void ReadImpl()
    {
        serialPort.WriteBytes(Serialize());
        var result = serialPort.ReadBytes(3);
        if ((result[1] & 0x80) == 0)
        {
            result = serialPort.ReadBytes((uint)(result[2] + 2));
            Log.Info(Deserialize(result));
        }
        else
        {
            Log.Error("Failed to read from Modbus");
        }
    }

    private byte[] Serialize()
    {
        var buffer = new byte[]
        {
            0x01, // UnitId
            0x03, // Function code
            0x00, // Starting address
            0x00,
            0x00, // Quantity Of Registers
            0x01,
            0x84, // CRC
            0x0a
        };
        return buffer;
    }

    private ushort Deserialize(byte[] buffer)
    {
        var first = (ushort)buffer[1];
        var second = (ushort)(buffer[0] << 8);
        return (ushort)(first | second);
    }
}
```

## Event mode

```csharp
public class RuntimeNetlogic1 : BaseNetLogic
{
    private SerialPort serialPort;
    private LongRunningTask task;

    public override void Start()
    {
        serialPort = (SerialPort)Owner;
        serialPort.Timeout = TimeSpan.FromMilliseconds(0.0);
        task = new LongRunningTask(Run, Owner);
        task.Start();
    }

    [ExportMethod]
    public void OnClick()
    {
        // Cancel current read
        serialPort.CancelRead();
        task.Cancel();
        // Start new read
        task = new LongRunningTask(Run, Owner);
        task.Start();
    }

    private void Run()
    {
        while(!task.IsCancellationRequested)
        {
            try
            {
                // Block until 3 bytes arrive on serial
                var result = serialPort.ReadBytes(3);
                foreach (var b in result)
                    Log.Info(String.Format("0x{0:x2}", b));
            }
            catch (ReadCanceledException ex)
            {
                // In case of read canceled, exit from the loop
                Log.Info(ex.Message);
                return;
            }
            catch (Exception e)
            {
                Log.Error(e.Message);
            }
        }
    }

    public override void Stop()
    {
        // Explicit call to Close to cancel pending read (if any)
        serialPort.Close();
        task.Cancel();
    }
}
```

## Custom protocol implementation example

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
using FTOptix.SerialPort;
using FTOptix.CommunicationDriver;
using System.Collections.Generic;
#endregion

public class SerialPortLogic : BaseNetLogic
{
    public override void Start()
    {
        // Get the input and output variables
        IncomingString = LogicObject.GetVariable("IncomingString");
        OutgoingString = LogicObject.GetVariable("OutgoingString");
        // Prepare the event to be triggered when the OutgoingString variable changes
        OutgoingString.VariableChange += OutgoingString_VariableChange;
        // Starts the COM port reading task as soon as the user-defined logic is started
        comPort = (SerialPort)Owner; // Cast the Owner property to the SerialPort type
        comPort.Timeout = TimeSpan.FromMilliseconds(100); // Adjust as needed
        serialPortTask = new LongRunningTask(ReadSerialPort, LogicObject); // Create a new task to read the COM port
        serialPortTask.Start(); // Start the task
    }

    private void OutgoingString_VariableChange(object sender, VariableChangeEventArgs e)
    {
        dataToSend = e.NewValue.Value.ToString();
    }

    public override void Stop()
    {
        OutgoingString.VariableChange -= OutgoingString_VariableChange;
        serialPortTask?.Dispose();
    }

    private void ReadSerialPort()
    {
        Log.Info("ReadSerialPort", "Starting to read the COM port");
        byte incomingChar;
        List<byte> buffer = [];

        // This loop keeps reading the COM port in case of an exception
        while (!serialPortTask.IsCancellationRequested)
        {
            Log.Info("ReadSerialPort", "Entering main loop");

            try
            {
                Log.Info("ReadSerialPort", "Opening the COM port");
                // Open the COM port
                comPort.Open();
                incomingChar = 0x00;
                Log.Info("ReadSerialPort", "COM port opened");

                // This loop reads the COM port until the termination character is received
                while (!serialPortTask.IsCancellationRequested)
                {
                    try
                    {
                        // Read the incoming character
                        incomingChar = comPort.ReadByte();

                        // If the incoming character is not the termination character, add it to the buffer
                        if (incomingChar != terminationChar)
                        {
                            buffer.Add(incomingChar);
                        }
                        else
                        {
                            // If the incoming character is the termination character,
                            // convert the buffer to a string and set the IncomingString variable
                            string message = System.Text.Encoding.ASCII.GetString(buffer.ToArray());
                            IncomingString.Value = message;
                            Log.Info("ReadSerialPort", $"Received message: {message}");
                            buffer.Clear();
                        }
                    }
                    catch (Exception ex)
                    {
                        if (!ex.Message.Contains("read timeout"))
                        {
                            // If another exception occurs, log the error and break the loop
                            Log.Error("ReadSerialPort", ex.Message);
                            throw;
                        }
                    }

                    // Check if data needs to be sent
                    if (dataToSend != null)
                    {
                        // Send the data and the termination character
                        Log.Info("ReadSerialPort", "Sending data: " + dataToSend);
                        comPort.WriteBytes(System.Text.Encoding.ASCII.GetBytes(dataToSend));
                        comPort.WriteBytes(new byte[] { terminationChar });
                        dataToSend = null;
                    }
                }
            }
            catch (Exception ex)
            {
                // Insert code to handle exceptions
                Log.Error("ReadSerialPort", ex.Message);
                comPort.Close();
            }
            finally
            {
                Log.Info("ReadSerialPort", "Closing the COM port");
                comPort.Close();
            }
        }

        Log.Info("ReadSerialPort", "Exiting main loop");
    }

    private const byte terminationChar = 0x0A;
    private IUAVariable IncomingString;
    private IUAVariable OutgoingString;
    private SerialPort comPort;
    private LongRunningTask serialPortTask;
    private string dataToSend;
}
```

## Using the serial port to control an external device

This example shows how to keep the TX line HIGH on a serial port to power an external device that requires it (like to power an external relay module).

> [!WARNING]
> The TX line of a serial port is not designed to provide power to external devices. Ensure that the external device's power requirements do not exceed the capabilities of the serial port to avoid damage. An intermediate driver circuit (current or voltage buffer) is strictly required.

> [!NOTE]
> Depending on the hardware interface, the TX line might be inverted. Please check the hardware documentation for more details.

```csharp
using System;
using System.Linq;
using FTOptix.CommunicationDriver;
using FTOptix.NetLogic;
using FTOptix.SerialPort;
using UAManagedCore;

public class MinimalSerialPortLogic : BaseNetLogic
{
    public override void Start()
    {
        // Setup Serial Port
        serialPort = (SerialPort)Owner;
        serialPort.Baudrate = 9600;
        serialPort.FlowControl = FlowControl.None;
        serialPort.DataSize = 8;
        serialPort.Parity = Parity.None;
        serialPort.StopBits = StopBits.One;
        serialPort.Timeout = TimeSpan.FromMilliseconds(100);

        // Calculate buffer size for 2 seconds transmission
        // Baudrate: 9600 bits/sec
        // Bits per byte: 10 (1 start + 8 data + 1 stop)
        // Bytes per second: 9600 / 10 = 960 bytes/sec
        // For 2000ms: 960 * 2 = 1920 bytes
        var bytesPerSecond = serialPort.Baudrate / (serialPort.DataSize + 2);
        var bufferSize = (bytesPerSecond * TX_DURATION_MS) / 1000;
        txBuffer = Enumerable.Repeat((byte)0x00, (int)bufferSize).ToArray();

        Log.Info("MinimalSerialPortLogic", $"Initialized with buffer size: {txBuffer.Length} bytes for {TX_DURATION_MS}ms transmission");
    }

    public override void Stop()
    {
        if (serialPort != null)
        {
            serialPort.Close();
            serialPort = null;
        }
    }

    /// <summary>
    /// Activates the TX pin for the configured duration by writing the pre-calculated buffer.
    /// WriteBytes is asynchronous - it queues the data and returns immediately.
    /// The actual transmission will take approximately TX_DURATION_MS.
    /// </summary>
    [ExportMethod]
    public void SetOutput()
    {
        try
        {
            var stopwatch = System.Diagnostics.Stopwatch.StartNew();
            serialPort.WriteBytes(txBuffer);
            stopwatch.Stop();

            Log.Info("SetOutput", $"TX activated. Buffer: {txBuffer.Length} bytes, Queue time: {stopwatch.ElapsedMilliseconds}ms, Expected TX duration: {TX_DURATION_MS}ms");
        }
        catch (Exception ex)
        {
            Log.Error("SetOutput", $"Failed to write to serial port: {ex.Message}");
        }
    }

    private SerialPort serialPort;
    private byte[] txBuffer;
    private const int TX_DURATION_MS = 2000;
}
```
