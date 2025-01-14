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
        // Insert code to be executed when the user-defined logic is started
        DetectAlarmType();
    }

    public override void Stop()
    {
        // Insert code to be executed when the user-defined logic is stopped
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
