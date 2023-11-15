# NetLogic overview

- NetLogic are C# scripts that help you implementing custom and/or advanced logic in FT OPtix with custom code.
- There are no limits on the amount of C# snippet you can add to a project
- There are some limitations on the packages you can add to a NetLogic
    - Example:
        - `WPF` (WindowsForms) -> These are not supported as they are not CrossPlatform in the way FT Optix works
        - `MSSQL` (Microsoft SQL) -> This is not supported as FT Optix APIs offers a common syntax to access both EmbeddedDatabase and ODBC with same syntax by translating the output query for you. This is needed to make SQL independents from the type of database you are accessing
- NetLogic can be Runtime or Designtime
    - `Runtime NetLogic` -> This is the most common type of NetLogic, you can use it to trigger custom logic when the Runtime is running
    - `Designtime NetLogic` -> These king of NetLogic are typically used to make the designer's life easier by automating some repetitive operations (such as creating hundreds of alarms, variables, assigning DynamicLink and more)   
- Distinction between the two kind of NetLogic only exists in the YAML definition, the same APIs can be used both at Designtime and/or at Runtime
    - Designtime NetLogic can only access elements that exists at DesignTime, you cannot access a `CommDriver` or a `Database` if the application is not running

## Things to remember

- NetLogic class name is the same of the NetLogic object name
    - Don't change the NetLogic name from any external editor, rename it from Studio instead!
- NetLogic objects can be browsed from the NetSolution
    - Don't delete a NetLogic from the solution, remove it from Studio instead!
    - Don't move a NetLogic from any external editor, move it from Studio (better to copy paste the NetLogic in the new position and then delete it from the unused place)