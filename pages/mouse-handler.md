# Mouse interaction

## Add MouseClick event to a Button

```csharp
#region Using directives
using System;
using System.Collections.Generic;
using System.Linq;
using UAManagedCore;
using OpcUa = UAManagedCore.OpcUa;
using FTOptix.HMIProject;
using FTOptix.UI;
using FTOptix.Core;
using FTOptix.CoreBase;
using FTOptix.NetLogic;
#endregion

public class DesignTimeNetLogic1 : BaseNetLogic
{
    [ExportMethod]
    public void CreateEventHandlers()
    {
        var mainWindow = Project.Current.Get<WindowType>("UI/MainWindow");
        var generateReportButton = mainWindow.Get<Button>("GenerateReportButton");
        var changeColorButton = mainWindow.Get<Button>("ChangeColorButton");
        var button1 = mainWindow.Get<Button>("Button1");
        var button1BackgroundColor = button1.BackgroundColorVariable;

        var oldEventHandlers = generateReportButton.Children.OfType<FTOptix.CoreBase.EventHandler>().ToList();
        oldEventHandlers.AddRange(changeColorButton.Children.OfType<FTOptix.CoreBase.EventHandler>());
        foreach (var oldEventHandler in oldEventHandlers)
            oldEventHandler.Delete();

        var report1 = Project.Current.GetObject("Reports/Report1");
        MakeEventHandler(
            generateReportButton,
            FTOptix.UI.ObjectTypes.MouseClickEvent,
        report1,
        "GeneratePdf",
        new List<Tuple<string, NodeId, object>>
        {
                new("OutputPath", FTOptix.Core.DataTypes.ResourceUri, (string)ResourceUri.FromApplicationRelativePath("MyReport.pdf")),
                new("LocaleId", OpcUa.DataTypes.String, "en-US")
        });
        MakeEventHandler(
        changeColorButton,
            FTOptix.UI.ObjectTypes.MouseClickEvent,
            InformationModel.GetObject(FTOptix.CoreBase.Objects.VariableCommands),
        "Set",
        new List<Tuple<string, NodeId, object>>
            {
                new("VariableToModify", FTOptix.Core.DataTypes.VariablePointer, NodeId.Empty),
                new("Value", FTOptix.Core.DataTypes.Color, new Color(0xff3480ebu).ARGB),
                new("ArrayIndex", OpcUa.DataTypes.UInt32, 0u)
            });
        var changeColorButtonEventHandler = changeColorButton.Get("EventHandler");
        var variableToModifyArgumentVariable = changeColorButtonEventHandler.GetVariable("MethodsToCall/MethodContainer1/InputArguments/VariableToModify");
        variableToModifyArgumentVariable.SetDynamicLink(button1BackgroundColor);
        var dynamicLink = variableToModifyArgumentVariable.Children.OfType<DynamicLink>().First();
        dynamicLink.Value = dynamicLink.Value + "@NodeId";
    }

    private FTOptix.CoreBase.EventHandler MakeEventHandler(
        IUANode parentNode, // The parent node to which the event handler is to be added
        NodeId listenEventTypeId, // The NodeID of the event to be listened
        IUAObject callingObject, // The object on which the method is to be executed
        string methodName, // The name of the method to be called
        List<Tuple<string, NodeId, object>> arguments = null // List of input arguments (name, data type NodeID, value)
    )
    {
        // Create event handler object
        var eventHandler = InformationModel.MakeObject<FTOptix.CoreBase.EventHandler>("EventHandler");
        parentNode.Add(eventHandler);

        // Set the ListenEventType variable value to the Node ID of the event to be listened
        eventHandler.ListenEventType = listenEventTypeId;

        // Create method container
        var methodContainer = InformationModel.MakeObject("MethodContainer1");
        eventHandler.MethodsToCall.Add(methodContainer);

        // Create the ObjectPointer variable and set its value to the object on which the method is to be executed
        var objectPointerVariable = InformationModel.MakeVariable<NodePointer>("ObjectPointer", OpcUa.DataTypes.NodeId);
        objectPointerVariable.Value = callingObject.NodeId;
        methodContainer.Add(objectPointerVariable);

        // Create the Method variable and set its value to the name of the method to be called
        var methodNameVariable = InformationModel.MakeVariable("Method", OpcUa.DataTypes.String);
        methodNameVariable.Value = methodName;
        methodContainer.Add(methodNameVariable);

        if (arguments != null)
            CreateInputArguments(methodContainer, arguments);

        return eventHandler;
    }

    private void CreateInputArguments(
        IUANode methodContainer,
        List<Tuple<string, NodeId, object>> arguments)
    {
        IUAObject inputArguments = InformationModel.MakeObject("InputArguments");
        methodContainer.Add(inputArguments);

        foreach (var arg in arguments)
        {
            var argumentVariable = inputArguments.Context.NodeFactory.MakeVariable(
                NodeId.Random(inputArguments.NodeId.NamespaceIndex),
                arg.Item1,
                arg.Item2,
                OpcUa.VariableTypes.BaseDataVariableType,
                false,
                arg.Item3);

            inputArguments.Add(argumentVariable);
        }
    }
}

```