# Accessing project nodes

Before reading this page, please make sure to have the concept of [InformationModel and Session](./information-model.md).

## Navigating to a node

### The `Node.Get` syntax

The `Get` method is used to fetch nodes from the current project, it can retrieve both session and global objects (nodes), this method also accepts a type that will be passed to the variable type
- Example:
    - `var myButton = … .Get<Button>("path/to/button");` -> Will create a _myButton_ variable of Type `FTOptix.UI.Button` pointing to the desired element
- After the object has been casted to the proper path, you can access its properties using IntelliSense
    - Example:
        - `myButton.Text = Hello;`

### Accessing global objects

In order to manipulate any project element, you first need to access it, this can be either done by getting its NodeId or its node object

- Examples:
    - `var myNodeId = Project.Current.Get( [path/to/node] ).NodeId;`
    - `var myObject = Project.Current.Get( [path/to/node] );`

    - Where `path/to/node` can be obtained by right clicking any element of the IDE and then clicking `Copy path to node`, here you need to remember to remove the project name from the pasted element
        - Example:
            - `NewHMIProject25/Model/MyCustomMotor` --> `Model/MyCustomMotor`

### Accessing session-based objects

Now that we know how session works, it's easy to understand the benefits of placing a NetLogic in a session-based object (such as the page you want to access or manipulate) and then use the relative addressing to retrieve the right element
    - Example:
        - `var ButtonText = Owner.Get("MotorButton/Text");`

### The `Node.Find` syntax

You can also search for a specific object by knowing its `BrowseName` by using the `Find` method, this can be useful but it is not recommended as it is very slow to perform (it will recursively browse the whole project until it gets to the first element with such node)
- Example:
    - `var myObj = Project.Current.Find("ObjectName");`

## Casting vs Type passing

### Passing the object type

Type parameter can be used with "generic methods" that allows using different types. A generic method can be called either without an argument (anything that applies to any node such as `BrowseName` or `NodeId`) or by specifying the type of argument within angle brackets
- Example:
    - `var myfolder = Project.Current.Get<Folder>("Model\MyFolder");`

Using this type passing option, the returned node is `null` if:
- Node does not exist
- The node does not match the specified type (e.g: `MyFolder` is an `Object` and not a `Folder`)

### Casting from IUANode

A cast is an explicit conversion, to perform a cast, specify the type that you are casting to in parentheses in front of the value or variable to be converted.
    - Example:
        - `var myfolder = (Folder)Project.Current.Get("Model\MyFolder");`

Using this casting option, the node is returned as null only if completely missing, if the object type cannot be casted, an exception is thrown.

### Summary

Result is very similar, as best practice we suggest to use the type passing approach unless casting is a specific requirement to achieve the result
