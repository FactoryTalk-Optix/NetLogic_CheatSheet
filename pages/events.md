# Events handling

> [!WARNING]
> This section is a work in progress and uses some low-level APIs.
> The documentation will be completed as soon as possible.

## Add MouseClick event to a Button

This code adds a GeneratePDF method to a button, and a SetVariableValue to a different button

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

        // Make sure to remove old handlers (if any)
        var oldEventHandlers = generateReportButton.Children.OfType<FTOptix.CoreBase.EventHandler>().ToList();
        oldEventHandlers.AddRange(changeColorButton.Children.OfType<FTOptix.CoreBase.EventHandler>());
        foreach (var oldEventHandler in oldEventHandlers)
            oldEventHandler.Delete();

        // Report object that we want to create with a button
        var report1 = Project.Current.GetObject("Reports/Report1");

        MakeEventHandler( // Add the GeneratePdf method to the button
            generateReportButton, // Object where to create the method
            FTOptix.UI.ObjectTypes.MouseClickEvent, // Method to be created
            report1, // Object pointer where to call the method
            "GeneratePdf", // Method name
            new List<Tuple<string, NodeId, object>> // Method arguments
            {
                // Each row is a tuple, each tuple contains a string (name of the argument), a NodeId (DataType) and an Object (value)
                new("OutputPath", FTOptix.Core.DataTypes.ResourceUri, (string)ResourceUri.FromApplicationRelativePath("MyReport.pdf")),
                new("LocaleId", OpcUa.DataTypes.String, "en-US")
            }
        );
        
        MakeEventHandler( // Add the SetVariableValue method to the button
            changeColorButton, // Object where to create the method
            FTOptix.UI.ObjectTypes.MouseClickEvent, // Event to be added
            InformationModel.GetObject(FTOptix.CoreBase.Objects.VariableCommands), // Object that exposes the method
            "Set", // Name of the method
            new List<Tuple<string, NodeId, object>> // Method arguments
            {
                // Each row is a tuple, each tuple contains a string (name of the argument), a NodeId (DataType) and an Object (value)
                new("VariableToModify", FTOptix.Core.DataTypes.VariablePointer, NodeId.Empty),
                new("Value", FTOptix.Core.DataTypes.Color, new Color(0xff3480ebu).ARGB),
                new("ArrayIndex", OpcUa.DataTypes.UInt32, 0u)
            }
        );

        // Set which variable to be modified by the Set method
        var changeColorButtonEventHandler = changeColorButton.Get("EventHandler");
        // This path can be different depending on the MethodContainer name assigned by the MakeEventHandler method
        var variableToModifyArgumentVariable = changeColorButtonEventHandler.GetVariable("MethodsToCall/MethodContainer1/InputArguments/VariableToModify");
        // Assign a DynamicLink to the VariableToModify
        variableToModifyArgumentVariable.SetDynamicLink(button1BackgroundColor);
        // Change the DynamicLink to make sure the NodeId attribute is used (IDE has a fallback to the @Value otherwise)
        var dynamicLink = variableToModifyArgumentVariable.Children.OfType<DynamicLink>().First();
        dynamicLink.Value = dynamicLink.Value + "@NodeId";
    }

    private FTOptix.CoreBase.EventHandler MakeEventHandler(
        IUANode parentNode, // The parent node to which the event handler is to be added
        NodeId listenEventTypeId, // The NodeID of the event to be listened
        IUAObject callingObject, // The object on which the method is to be called
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
        // This must be an unique name if the multiple methods are added to the same button. A random name can also be used
        var methodIndex = eventHandler.MethodsToCall.Any() ? eventHandler.MethodsToCall.Count + 1 : 1;
        var methodContainer = InformationModel.MakeObject($"MethodContainer{methodIndex}");
        eventHandler.MethodsToCall.Add(methodContainer);

        // Create the ObjectPointer variable and set its value to the object on which the method is to be called
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

## Add a method handler to an object

```csharp
#region Using directives

using System;
using System.Collections.Generic;
using System.Linq;
using FTOptix.Core;
using FTOptix.CoreBase;
using FTOptix.HMIProject;
using FTOptix.NetLogic;
using UAManagedCore;
using OpcUa = UAManagedCore.OpcUa;

#endregion

public class CreateEventHandler : BaseNetLogic
{
    private FTOptix.CoreBase.EventHandler MakeEventHandler(
       IUANode parentNode, // The parent node to which the event handler is to be added
       NodeId listenEventTypeId, // The NodeID of the event to be listened
       IUAObject callingObject, // The object on which the method is to be called
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

        // Create the ObjectPointer variable and set its value to the object on which the method is to be called
        var objectPointerVariable = InformationModel.MakeVariable<NodePointer>("ObjectPointer", OpcUa.DataTypes.NodeId);
        objectPointerVariable.Value = callingObject.NodeId;
        string resultPath = CreateRelativePath(parentNode, callingObject);
        var isCore = resultPath.Contains("Root");
        if (!isCore)
        {
            IUAVariable temp = null;
            objectPointerVariable.SetDynamicLink(temp, DynamicLinkMode.ReadWrite);
            objectPointerVariable.GetVariable("DynamicLink").Value = "../../../../" + resultPath;
        }

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

    private static string CreateRelativePath(IUANode source, IUANode target)
    {
        string result = "";
        var srcPath = GetCurrentProjectBrowsePath(source);
        var tarPath = GetCurrentProjectBrowsePath(target);
        var srcArr = srcPath.Split("/");
        var tarArr = tarPath.Split("/");
        var max = Math.Max(srcPath.Count(), tarPath.Count());
        int i = 0;
        for (; i < max; i++)
        {
            result = "..";
            if (srcArr[i] != tarArr[i])
            {
                break;
            }
        }
        for (int j = srcArr.Count() - 1; j > i; j--)
        {
            result = result + "/..";
        }
        for (int k = i; k < tarArr.Count(); k++)
        {
            result = result + "/" + tarArr[k];
        }
        return result;
    }

    private static string GetCurrentProjectBrowsePath(IUANode node)
    {
        if (node.Owner == Project.Current || node.BrowseName == "Root")
            return node.BrowseName;
        return GetCurrentProjectBrowsePath(node.Owner) + "/" + node.BrowseName;
    }
}
```

## Subscribe to OPCUA methods

This script can be put anywhere on the project and will listen to every OPCUA method being called (script or UI)

> [!NOTE]
> In FactoryTalk Optix, any method is mapped to an OPCUA method, so even when calling a NetLogic method, or a core command, this script will intercept the call.

```csharp
public class OpcMethodsAuditLogic : BaseNetLogic, IUAEventObserver
{
    public override void Start()
    {
        var serverObject = LogicObject.Context.GetObject(OpcUa.Objects.Server);
        //eventRegistration = serverObject.RegisterUAEventObserver(this, UAManagedCore.OpcUa.ObjectTypes.AuditUpdateMethodEventType);
    }

    public override void Stop()
    {
        // Insert code to be run when the user-defined logic is stopped
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

  //  private IEventRegistration eventRegistration;
}
```

## Add a VariableChanged event to a variable

> [!WARNING]
> This method can only work at DesignTime

```csharp
[ExportMethod]
public void AddVariableChangeEvent()
{
    IUAVariable targetVariable = null;
    try
    {
        targetVariable = InformationModel.GetVariable(LogicObject.GetVariable("VariableToAddChangeMethod").Value);
    }
    catch
    {
        Log.Error("Variable not found");
        return;
    }

    var variableOwner = targetVariable.Owner;
    var variableChangedEventDispatcher = InformationModel.Make<VariableChangedEventDispatcher>($"{targetVariable.BrowseName}Changed");
    variableChangedEventDispatcher.GetVariable("VariableNodePath").Value = $"../{targetVariable.BrowseName}";

    variableOwner.Add(variableChangedEventDispatcher);
}
```
