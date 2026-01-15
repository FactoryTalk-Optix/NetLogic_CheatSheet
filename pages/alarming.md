# Alarms

## Create alarms at Runtime

```csharp
#region Using directives
using FTOptix.Alarm;
using FTOptix.Core;
using FTOptix.HMIProject;
using FTOptix.NetLogic;
using FTOptix.UI;
using UAManagedCore;
using OpcUa = UAManagedCore.OpcUa;
#endregion

public class RuntimeAlarmsLogic : BaseNetLogic
{
    public override void Start() 
    {
        // Insert code to be run when the user-defined logic is started
        DetectAlarmType();
    }

    public override void Stop()
    {
        // Insert code to be run when the user-defined logic is stopped
    }

    [ExportMethod]
    public void DetectAlarmType()
    {
        // Check which alarm type we need to create and display the related info on a Label
        var targetLabel = Owner.Get<Label>("AlarmType");
        var alarmTag = InformationModel.GetVariable(Owner.Get<ComboBox>("SourceVar").SelectedItem);
        if (alarmTag.DataType == OpcUa.DataTypes.Boolean)
        {
            targetLabel.LocalizedText = new LocalizedText(targetLabel.NodeId.NamespaceIndex, "DigitalAlarm");
        }
        else
        {
            targetLabel.LocalizedText = new LocalizedText(targetLabel.NodeId.NamespaceIndex, "AnalogAlarm");
        }
    }

    [ExportMethod]
    public void CreateAlarm()
    {
        // Get the alarms container
        var alarmsFolder = Project.Current.Get<Folder>("Alarms/RuntimeAlarms");
        var alarmTag = InformationModel.GetVariable(Owner.Get<ComboBox>("SourceVar").SelectedItem);
        var existingAlarm = alarmsFolder.Get(alarmTag.BrowseName) ?? null;
        // Check if alarm already exist (skip) or create new based on DataType
        if (existingAlarm == null)
        {
            if (alarmTag.DataType == OpcUa.DataTypes.Boolean)
            {
                var newAlarm = InformationModel.MakeObject<DigitalAlarm>(alarmTag.BrowseName);
                newAlarm.InputValueVariable.SetDynamicLink(alarmTag);
                newAlarm.Message = Owner.Get<TextBox>("AlarmMessage").Text;
                newAlarm.AutoAcknowledge = true;
                newAlarm.AutoConfirm = true;
                alarmsFolder.Add(newAlarm);
            }
            else
            {
                var newAlarm = InformationModel.MakeObject<ExclusiveLevelAlarmController>(alarmTag.BrowseName);
                newAlarm.InputValueVariable.SetDynamicLink(alarmTag);
                newAlarm.Message = Owner.Get<TextBox>("AlarmMessage").Text;
                newAlarm.HighLimit = 50;
                newAlarm.AutoAcknowledge = true;
                newAlarm.AutoConfirm = true;
                alarmsFolder.Add(newAlarm);
            }
        }
    }
    
    [ExportMethod]
    public void DeleteAlarm()
    {
        // Delete alarm if exist
        var alarmsFolder = Project.Current.Get<Folder>("Alarms/RuntimeAlarms");
        var alarmTag = InformationModel.GetVariable(Owner.Get<ComboBox>("SourceVar").SelectedItem);
        var existingAlarm = alarmsFolder.Get(alarmTag.BrowseName) ?? null;
        existingAlarm?.Delete();
    }
}

```

## Subscribe to new alarms and get notified at every Acknowledge or Confirm event

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
using FTOptix.Alarm;
using System.Collections.Generic;
using LogsHandler;
using System.Transactions;
#endregion

public class AlarmsObserverLogic : BaseNetLogic
{
    public override void Start()
    {
        // Get the affinity ID of the current NetLogic to avoid cross thread issues
        affinityId = LogicObject.Context.AssignAffinityId();
        // Create the observer to check when alarms are created and/or deleted
        StartObserver();
    }

    public override void Stop()
    {
        // Destroy the observer when closing the NetLogic
        alarmsCreationObserver?.Dispose();
    }

    public void StartObserver()
    {
        // Get the alarms server object
        var retainedAlarmsObject = LogicObject.Context.GetNode(FTOptix.Alarm.Objects.RetainedAlarms);
        // Get the object containing the actual list of alarms
        var localizedAlarmsObject = retainedAlarmsObject.GetVariable("LocalizedAlarms");
        var localizedAlarmsNodeId = (NodeId) localizedAlarmsObject.Value;
        IUANode localizedAlarmsContainer = null;
        if (localizedAlarmsNodeId?.IsEmpty == false)
            localizedAlarmsContainer = LogicObject.Context.GetNode(localizedAlarmsNodeId);
        if (localizedAlarmsContainer == null)
        {
            Log.Error("AlarmsObserverLogic", "LocalizedAlarms node not found");
            return;
        }
        // Create a new custom alarms observer
        var logsObserver = new AlarmsCreationObserver();
        // Register the observer to the server node
        alarmsCreationObserver = localizedAlarmsContainer.RegisterEventObserver(
            logsObserver, // Which observer to use
            EventType.ForwardReferenceAdded | // Register when alarms are created
            EventType.ForwardReferenceRemoved, // Register when alarms are disposed
            affinityId); // Pass the affinity ID
    }

    private IEventRegistration alarmsCreationObserver;
    private uint affinityId; // This can also be moved to a local variable
}

namespace LogsHandler
{
    class AlarmsCreationObserver : IReferenceObserver
    {
        public AlarmsCreationObserver()
        {
            // Just create the class instance, nothing really to do
            Log.Info("LogsEventObserver", "Starting alarms observer");
        }

        public void OnReferenceAdded(IUANode sourceNode, IUANode targetNode, NodeId referenceTypeId, ulong senderId)
        {
            // When an alarm gets activated it will be added as reference to the RetainedAlarms object
            // Here a custom logic can be added to trigger some actions
            Log.Info("LogsEventObserver", $"{targetNode.BrowseName} alarm got enabled");

            // Subscribe to some variables (can be customized as needed)
            try
            {
                if (targetNode is not UAObject alarmNode)
                {
                    Log.Error("LogsEventObserver", "Alarm node is invalid");
                    return;
                }

                // Trigger the VariableChange to some variables, can be customized to call different methods if needed
                alarmNode.GetVariable("ActiveState/Id").VariableChange += AlarmsCreationObserver_VariableChange;
                alarmNode.GetVariable("AckedState/Id").VariableChange += AlarmsCreationObserver_VariableChange;
                alarmNode.GetVariable("ConfirmedState/Id").VariableChange += AlarmsCreationObserver_VariableChange;
            }
            catch
            {
                Log.Error("LogsEventObserver", "Error while trying to subscribe to alarm variables");
            }
        }

        // Method to be called whenever a variable changes in value
        private void AlarmsCreationObserver_VariableChange(object sender, VariableChangeEventArgs e)
        {
            // Do something when variables are changing
            var variable = sender as IUAVariable;
            Log.Info("LogsEventObserver", $"{variable.Owner.BrowseName} alarm status changed to {e.NewValue}");
        }

        public void OnReferenceRemoved(IUANode sourceNode, IUANode targetNode, NodeId referenceTypeId, ulong senderId)
        {
            // Do something when the alarms deactivates
            Log.Info("LogsEventObserver", $"{targetNode.BrowseName} alarm got removed");
        }
    }
}
```

## Get the originating alarm from the RetainedAlarms object

```csharp
/// <summary>
/// Retrieves the <see cref="AlarmController"/> associated with the given retained alarm ID.
/// </summary>
/// <param name="retainedAlarmId">The <see cref="NodeId"/> of the alarm in the RetainedAlarm object.</param>
/// <returns>The <see cref="AlarmController"/> associated with the retained alarm.</returns>
/// <exception cref="System.ArgumentException">Thrown when the alarm is not found.</exception>
private static AlarmController GetAlarmFromRetainedAlarm(NodeId retainedAlarmId)
{
    // Get the alarm controller from the retained alarm
    var retainedAlarm = InformationModel.Get(retainedAlarmId);
    // Get the alarm controller from the retained alarm
    return InformationModel.Get<AlarmController>(retainedAlarm.GetVariable("ConditionId").Value) ?? throw new System.ArgumentException("Alarm not found");
}
```

## Acknowledge and Confirm multiple alarms from a DataGrid

> [!WARNING]
> The multi selection feature can only be used in the AlarmGrid component. The standard DataGrid object does not support multi selection and forcing it may cause unexpected results.

```csharp
public class AlarmGridLogic : BaseNetLogic
{
    public override void Start()
    {
        alarmsDataGridModel = Owner.Get<DataGrid>("AlarmsDataGrid").GetVariable("Model");
    }

    /// <summary>
    /// Acknowledges the selected alarms with a specified message.
    /// </summary>
    /// <param name="ackMessage">The acknowledgment message.</param>
    [ExportMethod]
    public void AckAlarmsWithMessage(LocalizedText ackMessage)
    {
        ProcessAlarms(ackMessage, (alarm, message) => alarm.Acknowledge(message));
    }

    /// <summary>
    /// Confirms the selected alarms with a specified message.
    /// </summary>
    /// <param name="confirmMessage">The confirmation message.</param>
    [ExportMethod]
    public void ConfirmAlarmsWithMessage(LocalizedText confirmMessage)
    {
        ProcessAlarms(confirmMessage, (alarm, message) => alarm.Confirm(message));
    }

    #region Private methods

    /// <summary>
    /// Processes the selected alarms with a specified action.
    /// </summary>
    /// <param name="message">The message to be used for the action.</param>
    /// <param name="alarmAction">The action to be performed on the alarms.</param>
    private void ProcessAlarms(LocalizedText message, Action<AlarmController, LocalizedText> alarmAction)
    {
        var dataGrid = Owner.Get<DataGrid>("AlarmsDataGrid");

        if (dataGrid.GetVariable("AllowMultiSelection").Value)
        {
            // Multi selection
            var selectedItemsNodes = dataGrid.GetOptionalVariableValue("UISelectedItems") ?? throw new System.ArgumentException("UISelectedItems variable not found in AlarmsDataGrid");

            var selectedItemsArray = (NodeId[]) selectedItemsNodes.Value;
            if (selectedItemsArray == null || selectedItemsArray.Length == 0)
            {
                throw new System.ArgumentException("No alarms selected");
            }

            // Process each selected alarm
            foreach (var nodeId in selectedItemsArray)
            {
                var alarm = GetAlarmFromRetainedAlarm(nodeId) ?? throw new System.ArgumentException("Alarm not found");
                alarmAction(alarm, message);
            }
        }
        else
        {
            // Single selection
            var alarm = GetAlarmFromRetainedAlarm(dataGrid.UISelectedItem) ?? throw new System.ArgumentException("Alarm not found");
            alarmAction(alarm, message);
        }
    }

    /// <summary>
    /// Retrieves the <see cref="AlarmController"/> associated with the given retained alarm ID.
    /// </summary>
    /// <param name="retainedAlarmId">The <see cref="NodeId"/> of the retained alarm.</param>
    /// <returns>The <see cref="AlarmController"/> associated with the retained alarm.</returns>
    /// <exception cref="System.ArgumentException">Thrown when the alarm is not found.</exception>
    private static AlarmController GetAlarmFromRetainedAlarm(NodeId retainedAlarmId)
    {
        // Get the alarm controller from the retained alarm
        var retainedAlarm = InformationModel.Get(retainedAlarmId);
        // Get the alarm controller from the retained alarm
        return InformationModel.Get<AlarmController>(retainedAlarm.GetVariable("ConditionId").Value) ?? throw new System.ArgumentException("Alarm not found");
    }

    #endregion

    #endregion

    private IUAVariable alarmsDataGridModel;
}
```

### Acknowledge and Confirm All Alarms

```csharp
/// <summary>
/// Run the AlarmCommands object methods to acknowledge and confirm all active alarms.
/// This is a global action and requires proper permissions.
/// </summary>
var alarmCommands = InformationModel.GetObject(FTOptix.Alarm.Objects.AlarmCommands);
alarmCommands.ExecuteMethod("AcknowledgeAll");
alarmCommands.ExecuteMethod("ConfirmAll");
```

### Log list of active alarms

```csharp
/// <summary>
/// Write to the log the list of retained (active) localized alarms.
/// </summary>
[ExportMethod]
public void LogActiveAlarms()
{
    var retainedAlarmsObject = LogicObject.Context.GetNode(FTOptix.Alarm.Objects.RetainedAlarms);
    // Get the object containing the actual list of alarms
    var localizedAlarmsObject = retainedAlarmsObject.GetVariable("LocalizedAlarms");
    var localizedAlarmsNodeId = (NodeId)localizedAlarmsObject.Value;
    IUANode localizedAlarmsContainer = null;
    if (localizedAlarmsNodeId?.IsEmpty == false)
        localizedAlarmsContainer = LogicObject.Context.GetNode(localizedAlarmsNodeId);
    if (localizedAlarmsContainer == null)
    {
        Log.Error("AlarmsObserverLogic", "LocalizedAlarms node not found");
        return;
    }
    foreach (var item in localizedAlarmsContainer.Children)
    {
        Log.Info(item.BrowseName);
    }
}
```

### Generate alarms for every bit of an array element

> [!WARNING]
> This helper will create a large number of alarms (one per bit per array element). Use carefully and prefer to run at DesignTime when possible.

```csharp
/// <summary>
/// Create digital alarms for each bit of an array of integer/word values.
/// Input: a NodeId variable "VarArrayPLC" pointing to the PLC array variable.
/// Behavior: the method creates a folder under Alarms and an alarm for each bit found in the array.
/// </summary>
[ExportMethod]
public void CreateBitAlarms()
{
    var plc = InformationModel.GetVariable(LogicObject.GetVariable("VarArrayPLC").Value);
    var dataTypeBrowse = InformationModel.Get(plc.DataType).BrowseName;

    int numBits = dataTypeBrowse switch
    {
        "Int32" => 32,
        "UInt32" => 32,
        "Int16" => 16,
        "UInt16" => 16,
        _ => 0
    };

    if (numBits == 0)
    {
        Log.Error("CreateBitAlarms", "Unsupported data type: " + dataTypeBrowse);
        return;
    }

    var newFolder = InformationModel.Make<Folder>(plc.BrowseName);
    Project.Current.Get("Alarms").Add(newFolder);

    // Assume plc is a UAVariable array; iterate over elements and bits
    var arrayLength = ((UAManagedCore.UAVariable)plc).ArrayDimensions[0];
    for (int i = 1; i < arrayLength; i++)
    {
        for (int j = 0; j < numBits; j++)
        {
            var alarm = InformationModel.Make<DigitalAlarm>($"Alarm_{plc.BrowseName}_{i}_{j}");
            // Create a dynamic link to the element and then point to the bit index
            alarm.InputValueVariable.SetDynamicLink((IUAVariable)plc, (uint)i, DynamicLinkMode.ReadWrite);
            var dynamicLink = alarm.InputValueVariable.GetVariable("DynamicLink");
            alarm.InputValueVariable.GetVariable("DynamicLink").Value = dynamicLink.Value + "." + j;
            alarm.Message = $"Alarm_{plc.BrowseName}_{i}_{j}";
            Project.Current.Get($"Alarms/{plc.BrowseName}").Add(alarm);
        }
    }
}
```

### Adding localized message to an existing alarm

```csharp
/// <summary>
/// Build a LocalizedText entry for an alarm and add it to the InformationModel translations if missing.
/// </summary>
private void AddLocalizedMessageToAlarm(AlarmController alarmInstance)
{
    var alarm = (DigitalAlarm)item;
    var localizedMessage = new LocalizedText(alarmInstance.BrowseName + "_" + item.BrowseName, alarmInstance.BrowseName + "_" + item.BrowseName, "en-US");
    if (!InformationModel.LookupTranslation(localizedMessage).HasTranslation)
    {
        InformationModel.AddTranslation(localizedMessage);
    }
    alarm.LocalizedMessage = localizedMessage;
}
```
