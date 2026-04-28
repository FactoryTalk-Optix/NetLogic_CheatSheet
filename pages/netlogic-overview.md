# NetLogic overview

- NetLogic are C# scripts that help you implementing custom and/or advanced logic in FT OPtix with custom code.
- There are no limits on the amount of C# snippet you can add to a project
- There are some limitations on the packages you can add to a NetLogic
    - Example:
        - `WPF` (WindowsForms) -> These are not supported as they are not cross platform in the way FT Optix works
        - `ActiveX` -> These are not supported as they are old and unsecure
        - `MSSQL` (Microsoft SQL) -> This is not supported as FT Optix APIs offers a common syntax to access both EmbeddedDatabase and ODBC with same syntax by translating the output query for you. This is needed to make SQL independents from the type of database you are accessing
            - This limitation was removed starting from FT Optix 1.6.0.44, assuming at least one ODBC Connector is added to the project (even if unused)
- NetLogic can be RunTime or DesignTime
    - `RunTime NetLogic` -> This is the most common type of NetLogic, you can use it to trigger custom logic when the RunTime is running
    - `DesignTime NetLogic` -> These king of NetLogic are typically used to make the designer's life easier by automating some repetitive operations (such as creating hundreds of alarms, variables, assigning DynamicLink and more)   
- Distinction between the two kind of NetLogic only exists in the YAML definition, the same APIs can be used both at DesignTime and/or at RunTime
    - DesignTime NetLogic can only access elements that exists at DesignTime, you cannot access a `CommDriver` or a `Database` if the application is not running

## Methods visible to FT Optix

Not all methods in a NetLogic class are exposed to the runtime in the same way. FT Optix distinguishes three categories:

### Start() and Stop()

These are the two lifecycle methods inherited from `BaseNetLogic`. They are always called by the runtime automatically - you never invoke them yourself.

- `Start()` is called when the node (page, object) is activated
- `Stop()` is called when the node is deactivated, in reverse order relative to `Start()`
- Both must return quickly; slow implementations block the startup of subsequent NetLogics (see [thread-model.md](thread-model.md))

```csharp
public override void Start()
{
    // Setup: register observers, create tasks, read initial values
}

public override void Stop()
{
    // Teardown: dispose tasks, unregister observers
}
```

### `[ExportMethod]` - callable from the UI or other NetLogics

Methods decorated with `[ExportMethod]` are **registered as OPC UA Methods** on the NetLogic node. This makes them:
- Callable from buttons, event handlers, and other UI controls in Studio (they appear in the method picker)
- Callable from other NetLogics via `ExecuteMethod`
- Callable by external OPC UA clients

Rules for `[ExportMethod]`:
- Must be `public`
- Can have input parameters (mapped to OPC UA input arguments)
- Can have `out` parameters to return a value (which gets mapped to OPC UA output arguments)
  - All methods must be `void` return type, use `out` parameters to return values instead
- The method name must be unique within the NetLogic class

```csharp
// Simple ExportMethod with no parameters
[ExportMethod]
public void ResetCounter()
{
    LogicObject.GetVariable("Counter").Value = 0;
}

// ExportMethod with input and output
[ExportMethod]
public void Multiply(double a, double b, out double result)
{
    result = a * b;
}
```

> [!WARNING]
> If called from a UI button, an `[ExportMethod]` runs on the **UI thread**. Blocking operations will freeze the UI. See [thread-model.md](./thread-model.md) for the correct pattern.

### Public methods - internal use only

`public` methods that are **not** decorated with `[ExportMethod]` are plain C# class members. They are:
- **Not visible** to Studio, event handlers, or OPC UA clients
- Callable only from C# code within the same NetLogic or from other NetLogics via direct object reference

Use them to factor out reusable logic within the class without exposing it to the outside world.

```csharp
// Visible to Studio and OPC UA clients
[ExportMethod]
public void TriggerAlarm(string message)
{
    SetAlarmState(true, message); // calls the private helper below
}

// NOT visible to Studio - internal helper only
public void SetAlarmState(bool active, string message)
{
    LogicObject.GetVariable("AlarmActive").Value = active;
    LogicObject.GetVariable("AlarmMessage").Value = message;
}
```

### Private and internal methods

`private` (and `internal`) methods are standard C# - not accessible from outside the class at all. Use them freely for implementation details.

### Summary table

| Method type | Visible in Studio? | Called by runtime? | Callable from other NetLogics? |
|-------------|-------------------|-------------------|-------------------------------|
| `Start()` / `Stop()` | No (automatic) | Yes, automatically | No (use ExportMethod instead) |
| `[ExportMethod]` public | **Yes** | No (on-demand) | Yes, via `ExecuteMethod` |
| `public` (no attribute) | No | No | Yes, via object reference |
| `private` / `internal` | No | No | No |

---

## Things to remember

- NetLogic class name is the same of the NetLogic object name
    - Don't change the NetLogic name from any external editor, rename it from Studio instead!
- NetLogic objects can be browsed from the NetSolution
    - Don't delete a NetLogic from the solution, remove it from Studio instead!
    - Don't move a NetLogic from any external editor, move it from Studio (better to copy paste the NetLogic in the new position and then delete it from the unused place)

## DesignTime helper: create a Runtime NetLogic (Blinker example)

```csharp
/// <summary>
/// DesignTime helper that scaffolds a runtime NetLogic source file and supporting model nodes (Blinkers example).
/// Inputs: none. This is executed at DesignTime (Exported method).
/// Output: new folder/variables under Model, a NetLogic object under NetLogic and a .cs file written into NetSolution.
/// Caution: This writes files into the project - use only in design phase.
/// </summary>
[ExportMethod]
public void AddBlinkers() {
    var varModelFolder = Project.Current.Get("Model");
    varModelFolder.Add(InformationModel.Make<Folder>("Blinkers"));

    var varBlinkersFolder = Project.Current.Get("Model/Blinkers");
    varBlinkersFolder.Add(InformationModel.MakeVariable("P1ms", OpcUa.DataTypes.Boolean));
    varBlinkersFolder.Add(InformationModel.MakeVariable("P10ms", OpcUa.DataTypes.Boolean));
    varBlinkersFolder.Add(InformationModel.MakeVariable("P100ms", OpcUa.DataTypes.Boolean));
    varBlinkersFolder.Add(InformationModel.MakeVariable("P1s", OpcUa.DataTypes.Boolean));
    varBlinkersFolder.Add(InformationModel.MakeVariable("P10s", OpcUa.DataTypes.Boolean));

    var varNetLogicFolder = Project.Current.Get("NetLogic");
    varNetLogicFolder.Add(InformationModel.Make<NetLogicObject>("BlinkerNetLogic"));

    var varNetLogicFileUri = ResourceUri.FromProjectRelativePath("NetSolution/BlinkerNetLogic.cs");
    // The `varBlinkersNetLogicContent` variable should contain the runtime NetLogic source text (omitted here for brevity)
    File.WriteAllText(varNetLogicFileUri.Uri, varBlinkersNetLogicContent);
}
```
