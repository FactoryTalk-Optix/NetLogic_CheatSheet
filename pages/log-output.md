# Log messages

Log messages can be displayed both at RunTime and DesignTime, at DesignTime logs will appear in `Optix Studio Output` while at RunTime they will appear in the `Emulator output` or the RunTime log

## General syntax

`Log . [level] ( [optional category] , [ message content] );`

## Examples

Different log levels can be triggered and then filtered in `Optix Studio Output` and `Emulator output`. Specifying an optional custom `Category` helps you understanding where this line was generated (for example the NetLogic name and/or the method name)

- `Log.Error(category, message);`
- `Log.Warning(category, message);`
- `Log.Info(category, message);`
- `Log.Debug(category, message);`
- `Log.Verbose1(category, message);`
- `Log.Verbose2(category, message);`

## Get the node path for any project node

- `Log.Node(IUANode);` -> Returns a string containing the node path for a specific element

## Getting total count of project nodes

- `LogicObject.Context.NodeCount` -> returns the total number of nodes (whole project)

## Reading debug logs

When debug logs are enabled, the output of some rendering methods is logged to the output, here is an example:

```txt
2024-10-22 10:55:05     NativeUI    StartNetLogics,10,6,600.000
```

The syntax is:

```txt
[Timestamp]    [Module Name]    [Method Name];[Total time in ms],[Number of processed items],[Items per second]
```

So, in the example above we get the NativeUI:

- Executing the `Start` method of all NetLogics in the page
- Starting a total of 6 NetLogics
- Running at a theoretical 600 NetLogics per second can be loaded (this is something like benchmarking)
