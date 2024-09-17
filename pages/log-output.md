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

## Extra

- `Log.Node(IUANode);` -> Returns a string containing the node path for a specific element

## Getting total count of project nodes

- `LogicObject.Context.NodeCount` -> returns the total number of nodes (whole project)
