# Template Library

## Import an object from the template library

This feature allows you to import objects from the template library into your project. The imported object will be placed in the specified folder. The object can be an instance or a type.

> [!TIP]
> This feature got introduced in FactoryTalk Optix 1.6.X

### Import item syntax

```csharp
public static class TemplateLibrary
{
    public static IUANode ImportLibraryItem(IUANode destinationNode, NodeClass nodeClass, string libraryName,
        string itemPath, List<TypeConflictResolutionChoice> conflictResolutionChoices = null,
        bool preserveTypeDependencyPaths = false);
}
```

### Import the LoginForm from the Widgets library example

```csharp
[ExportMethod]
public void ImportLoginFormFromLibrary()
{
    // The place where the object will be imported
    var targetFolder = Project.Current.Get("UI");
    // Import the object
    // Syntax: ImportLibraryItem(<item destination>, <node class (type or instance)>, <name of the library>, <name of the library item>)
    var myWidget = TemplateLibrary.ImportLibraryItem(targetFolder, NodeClass.ObjectType, "Widgets", "LoginForm");
    // Log the result
    Log.Info($"Imported LoginForm from library to: {Log.Node(myWidget)}");
}
```

## Specifying a conflict resolution

If the imported element already exists in the target folder, you can specify a conflict resolution strategy.

### Conflict resolution syntax

A list of `TypeConflictResolutionChoice` structures is used to specify the conflict resolution strategy. The structure contains the following fields:

```csharp
public struct TypeConflictResolutionChoice
{
    public TypeConflictResolutionChoice(string browseName, NodeClass nodeClass, ConflictResolution resolution)
    {
        BrowseName = browseName;
        NodeClass = nodeClass;
        Resolution = resolution;
    }

    public readonly string BrowseName;
    public readonly NodeClass NodeClass;
    public readonly ConflictResolution Resolution;
}
```

### Conflict resolution example

 The following example will import the LoginForm from the TeamplateLibrary by overwriting the existing element (if any):

```csharp
using UAManagedCore;
using FTOptix.HMIProject;
using FTOptix.NetLogic;
using System.Collections.Generic;
using static UAManagedCore.TemplateLibrary;

public class DesignTimeNetLogic1 : BaseNetLogic
{
    [ExportMethod]
    public void ImportLoginFormFromLibrary()
    {
        // Conflict resolution strategy
        var conflictResolutionChoices = new List<TypeConflictResolutionChoice>
        {
            // This is the list of elements which are inside the LoginForm object, we can specify how to handle each of them
            new TypeConflictResolutionChoice("LoginForm", NodeClass.ObjectType, ConflictResolution.Replace),
            new TypeConflictResolutionChoice("Login", NodeClass.ObjectType, ConflictResolution.Replace),
            new TypeConflictResolutionChoice("LoginPasswordExpiredDialog", NodeClass.ObjectType, ConflictResolution.Replace),
            new TypeConflictResolutionChoice("LoginChangePasswordForm", NodeClass.ObjectType, ConflictResolution.Replace),
            new TypeConflictResolutionChoice("Logout", NodeClass.ObjectType, ConflictResolution.Replace),
            new TypeConflictResolutionChoice("OAuth2Login", NodeClass.ObjectType, ConflictResolution.Replace)
        };
        // The place where the object will be imported
        var targetFolder = Project.Current.Get("UI");
        // Import the object
        // Syntax: ImportLibraryItem(<item destination>, <node class (type or instance)>, <name of the library>, <name of the library item>)
        var myWidget = TemplateLibrary.ImportLibraryItem(targetFolder, NodeClass.ObjectType, "Widgets", "LoginForm", conflictResolutionChoices);
        // Log the result
        Log.Info($"Imported LoginForm from library to: {Log.Node(myWidget)}");
    }
}
```

## Getting better performance

If you are importing a large object with many dependencies, you can improve the performance by calling the import routine using a LongRunningTask.

### Importing a large object using a LongRunningTask

```csharp
[ExportMethod]
public void ImportLargeObjectFromLibrary()
{
    // Import the object
    var importWidgetTask = new LongRunningTask(ImportLoginForm, LogicObject);
    importWidgetTask.Start();
}

private void ImportLoginForm()
{
    // The place where the object will be imported
    var targetFolder = Project.Current.Get("UI");
    // Import the object
    // Syntax: ImportLibraryItem(<item destination>, <node class (type or instance)>, <name of the library>, <name of the library item>)
    var myWidget = TemplateLibrary.ImportLibraryItem(targetFolder, NodeClass.ObjectType, "Widgets", "LoginForm");
    // Log the result
    Log.Info($"Imported LoginForm from library to: {Log.Node(myWidget)}");
}
```
