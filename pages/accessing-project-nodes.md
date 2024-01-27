# Accessing project nodes

In order to manipulate a project element, you first need to access it, this can be either done by getting its NodeId or its node object

- Examples:
    - `var myNodeId = Project.Current.Get( [path/to/node] ).NodeId;`
    - `var myObject = Project.Current.Get( [path/to/node] );`

    - Where `path/to/node` can be obtained by right clicking any element of the IDE and then clicking `Copy path to node`, here you need to remember to remove the project name from the pasted element
        - Example:
            - `NewHMIProject25/Model/MyCustomMotor` --> `Model/MyCustomMotor`

The `Get` method also accepts an input type that will be passed to the variable type
- Example:
    - `var myButton = … .Get<Button>("path/to/button");` -> Will create a _myButton_ variable of Type `FTOptix.UI.Button` pointing to the desired element
- After the object has been casted to the proper path, you can access its properties using IntelliSense
    - Example:
        - `myButton.Text = Hello;`

You can also search for a specific object by knowing its `BrowseName` by using the `Find` method, this can be useful but it is not recommended as it is very slow to perform (it will recursively browse the whole project until it gets to the first element with such node)
- Example:
    - `var myObj = Project.Current.Find("ObjectName");`

## Casting vs Type passing

- Type parameter can be used with "generic methods" that allows using different types. A generic method can be called either without an argument (anything that applies to any node such as `BrowseName` or `NodeId`) or by specifying the type of argument within angle brackets
    - Example:
        - `var myfolder = Project.Current.Get<Folder>("Model\MyFolder");`

- A cast is an explicit conversion, to perform a cast, specify the type that you are casting to in parentheses in front of the value or variable to be converted.
    - Example:
        - `var myfolder = (Folder)Project.Current.Get("Model\MyFolder");`

Result is very similar, as best practice we suggest to use the type passing approach unless casting is a specific requirement to achieve the result

## Project context

Accessing the path of the node will return the project-level node, this is good for global elements (Model Variables, PLC Tags, etc) but it cannot be used to access session-based elements, as each user session will have a unique instance of windows and pages with uniques NodeIds

To access data from the current session is better to place the NetLogic in a session-based object (such as the page you want to access or manipulate) and then use the relative addressing to retrieve the right element
    - Example:
        - `var ButtonText = Owner.Get("MotorButton/Text");`

## Node's references

In OPC/UA, some nodes may have additional references are explicit relationship (a named pointer) from one Node to another

```csharp
[ExportMethod]
public void ReadReference(NodeId element)
{
    IUANode targetNode = InformationModel.Get(element);
    if (targetNode != null)
    {
        foreach (var item in targetNode.Refs.GetReferences())
        {
            Log.Info("References", $"ReferenceTypeId: {item.ReferenceTypeId}, TargetNodeId: {item.TargetNodeId}, TargetNode: {item.TargetNode.BrowseName}");
        }
    }
    else
    {
        Log.Info("References", "Cannot find such node in the current project");
    }
}
```
