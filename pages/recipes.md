# Recipes

Recipes in FT Optix provide a structured way to store and manage sets of parameters that can be loaded into your application. They are particularly useful for manufacturing processes where different product variants require different machine settings.

> [!WARNING]
> This section is a work in progress and might be updated with more examples and explanations in the future.
> Please report any issues or suggestions to the FactoryTalk Optix support or community forums.

> [!NOTE]
> The recipes engine in FactoryTalk Optix allows a maximum of 2048 ingredients per recipe. This limit is imposed by the underlying database system and cannot be changed.

> [!NOTE]
> The content of this page can only be applied to the legacy recipe module of FactoryTalk Optix. A new Recipe module is coming with release 1.7.x, which will provide enhanced functionality and flexibility.

## Recipe System Overview

The recipe system in FT Optix consists of three main components:

1. **Recipe Schema**: Defines what variables are included in a recipe
2. **Recipe Store**: The database where recipe data is stored
3. **Recipe Controller**: Manages loading, saving, and applying recipes

> [!NOTE]
> Recipes work by mapping values between a database table and variables in your project. The schema defines this mapping.

## Working with Recipe Schemas

A recipe schema links variables in your project model to columns in a database table. When recipes are loaded or saved, values are transferred between these locations.

### Creating a Recipe Schema manually

1. Add a Schema to the Recipes folder
2. Set the target node (typically the Model folder)
3. Set the store where recipes will be saved
4. Use the UI to select variables to include in the schema

### Modifying Recipe Schemas programmatically

Recipe schemas can be modified at runtime using NetLogic. This is useful when you need to dynamically change which variables are included in recipes.

```csharp
[ExportMethod]
public void UpdateRecipeSchema()
{
    // Get the recipe schema from the project
    var recipeSchema = Project.Current.Get<RecipeSchema>("Recipes/MyRecipeSchema");
    
    // Get the EditModel and Root objects
    var editModel = recipeSchema.GetObject("EditModel");
    var root = recipeSchema.GetObject("Root");
    
    // Clear existing schema
    if (editModel != null)
        editModel.Children.ToList().ForEach(child => child.Delete());
    if (root != null)
        root.Children.ToList().ForEach(child => child.Delete());
    
    // Add variables to the schema
    var targetNode = InformationModel.Get(recipeSchema.TargetNode);
    var speedVar = targetNode.GetVariable("MotorSpeed");
    
    if (speedVar != null)
    {
        // Add to EditModel and Root
        var editVar = InformationModel.MakeVariable("MotorSpeed", speedVar.DataType);
        var rootVar = InformationModel.MakeVariable("MotorSpeed", speedVar.DataType);
        editModel.Add(editVar);
        root.Add(rootVar);
        
        // Update database columns
        var store = InformationModel.Get<Store>(recipeSchema.Store);
        var table = store.Tables.Get(recipeSchema.BrowseName);
        if (table != null)
        {
            var column = InformationModel.Make<StoreColumn>("MotorSpeed");
            column.DataType = speedVar.DataType;
            table.Columns.Add(column);
        }
    }
}
```

### Adding hierarchical variables to a Recipe Schema

Variables in nested folders or objects require special handling. The code below demonstrates how to add variables from various locations in your project model:

```csharp
[ExportMethod]
public void AddVariablesToRecipeSchema()
{
    // Get the recipe schema
    var recipeSchema = Project.Current.Get<RecipeSchema>("Recipes/MyRecipeSchema");
    if (recipeSchema == null)
    {
        Log.Error("Recipe schema not found");
        return;
    }
    
    // Get or create EditModel and Root
    var editModel = recipeSchema.GetObject("EditModel") ?? InformationModel.MakeObject("EditModel");
    var root = recipeSchema.GetObject("Root") ?? InformationModel.MakeObject("Root");
    
    if (recipeSchema.GetObject("EditModel") == null)
        recipeSchema.Add(editModel);
    if (recipeSchema.GetObject("Root") == null)
        recipeSchema.Add(root);
    
    // Get the database table
    var store = InformationModel.Get<Store>(recipeSchema.Store);
    var table = store.Tables.Get(recipeSchema.BrowseName);
    
    // Variables to add
    var variablePaths = new List<string>
    {
        "Settings/Temperature",
        "Settings/Speed",
        "Settings/BatchSize"
    };
    
    // Add each variable to the schema
    var targetNode = InformationModel.Get(recipeSchema.TargetNode);
    foreach (var path in variablePaths)
    {
        AddVariableToSchema(targetNode, path, editModel, root, table);
    }
}

private void AddVariableToSchema(IUANode targetNode, string path, IUAObject editModel, 
                                IUAObject root, Table table)
{
    // Get the variable
    var variable = targetNode.GetVariable(path);
    if (variable == null)
    {
        Log.Error($"Variable {path} not found");
        return;
    }
    
    // Create path segments
    var segments = path.Split('/');
    var varName = segments.Last();
    var folders = segments.Take(segments.Length - 1).ToArray();
    
    // Create folders in EditModel and Root if needed
    var currentEditFolder = editModel;
    var currentRootFolder = root;
    
    foreach (var folder in folders)
    {
        var editChildFolder = currentEditFolder.GetObject(folder);
        if (editChildFolder == null)
        {
            editChildFolder = InformationModel.MakeObject(folder);
            currentEditFolder.Add(editChildFolder);
        }
        currentEditFolder = editChildFolder;
        
        var rootChildFolder = currentRootFolder.GetObject(folder);
        if (rootChildFolder == null)
        {
            rootChildFolder = InformationModel.MakeObject(folder);
            currentRootFolder.Add(rootChildFolder);
        }
        currentRootFolder = rootChildFolder;
    }
    
    // Add variables to the folders
    var editVar = InformationModel.MakeVariable(varName, variable.DataType);
    var rootVar = InformationModel.MakeVariable(varName, variable.DataType);
    currentEditFolder.Add(editVar);
    currentRootFolder.Add(rootVar);
    
    // Add column to the table
    if (table != null)
    {
        var columnName = string.Join("/", segments);
        var column = InformationModel.Make<StoreColumn>(columnName);
        column.DataType = variable.DataType;
        table.Columns.Add(column);
    }
}
```

## Populating a Recipe Schema

The following example demonstrates how to populate a recipe schema with variables from different locations within your project model:

```csharp
[ExportMethod]
public void PopulateRecipeSchema()
{
    // List of tags with relative path from the root node to be added to the RecipeSchema
    var tagsToAdd = new List<string>
    {
        "Folder1/Variable3", // Variable in a folder
        "Variable2", // Variable in the Model
        "Object3/Object1/Variable2", // Variable in a nested object
        "Object2/Variable1", // Variable in an object
        "Object1" // Entire object (may contain other objects or variables)
    };

    //get the RecipeSchema object
    RecipeSchema recipeSchema = Project.Current.Get<RecipeSchema>("Recipes/RecipeSchema1");

    // Check if the RecipeSchema exists and if the Store and TargetNode are set
    if (recipeSchema == null || 
        recipeSchema.Store == NodeId.Empty || 
        recipeSchema.TargetNode == NodeId.Empty)
    {
        Log.Error("Invalid recipe schema configuration");
        return;
    }

    // Get the root node of the Information Model
    var rootNode = InformationModel.Get(recipeSchema.TargetNode);
    
    // Process each tag
    foreach (string path in tagsToAdd)
    {
        var node = rootNode.Get(path);
        if (node == null)
        {
            Log.Error($"Node '{path}' not found");
            continue;
        }
        
        if (node is IUAObject obj)
        {
            // Process all variables in the object
            foreach (var childVar in obj.FindNodesByType<IUAVariable>())
                AddNodeToSchema(childVar, recipeSchema);
        }
        else
        {
            // Process single variable
            AddNodeToSchema(node, recipeSchema);
        }
    }
}
```

### Modify an existing Recipe Schema programmatically

```csharp
/// <summary>
/// Add variables and corresponding store columns to an existing RecipeSchema.
/// Inputs: two model variables present in the project (Model/Variable2, Model/Variable3).
/// This is a DesignTime helper typically executed from a DesignTime NetLogic or exported method.
/// </summary>
[ExportMethod]
public void ModifyRecipeSchema()
{
    // Get variables from the model
    IUAVariable var2 = Project.Current.GetVariable("Model/Variable2");
    IUAVariable var3 = Project.Current.GetVariable("Model/Variable3");

    // Get the recipe schema to modify
    RecipeSchema recipeSchema = Project.Current.Get<RecipeSchema>("Recipes/RecipeSchema1");

    // Add variables to the EditModel (UI edit form)
    IUAObject editModelObj = recipeSchema.GetObject("EditModel");
    IUAVariable var2ToAddEditModel = InformationModel.MakeVariable(var2.BrowseName, var2.DataType);
    IUAVariable var3ToAddEditModel = InformationModel.MakeVariable(var3.BrowseName, var3.DataType);
    editModelObj.Add(var2ToAddEditModel);
    editModelObj.Add(var3ToAddEditModel);

    // Add variables to the Root (actual storage mapping)
    IUAObject rootObj = recipeSchema.GetObject("Root");
    IUAVariable var2ToAddRoot = InformationModel.MakeVariable(var2.BrowseName, var2.DataType);
    IUAVariable var3ToAddRoot = InformationModel.MakeVariable(var3.BrowseName, var3.DataType);
    rootObj.Add(var2ToAddRoot);
    rootObj.Add(var3ToAddRoot);

    // Create corresponding columns in the store table
    var DB_Table_Columns = Project.Current.Get("DataStores/EmbeddedDatabase1/Tables/RecipeSchema1/Columns");
    var TableColumn_Var2 = InformationModel.Make<StoreColumn>("/" + var2.BrowseName);
    TableColumn_Var2.DataType = var2.DataType;
    var TableColumn_Var3 = InformationModel.Make<StoreColumn>("/" + var3.BrowseName);
    TableColumn_Var3.DataType = var3.DataType;
    DB_Table_Columns.Add(TableColumn_Var2);
    DB_Table_Columns.Add(TableColumn_Var3);
}
```

## Clear EditModel content recursively

```csharp
/// <summary>
/// Helper methods to clear the EditModel of a recipe schema by resetting variables to default values.
/// Use with caution: this will set variables to empty/zero values depending on type.
/// </summary>
private void ClearEditModel(IUANode obj)
{
    foreach (var item in obj.Children)
    {
        switch (item.NodeClass)
        {
            case NodeClass.Object:
                // Recurse into nested objects
                ClearEditModel(item);
                break;
            case NodeClass.Variable:
                // Reset the variable value based on its current type
                ResetVariableValue(item as IUAVariable);
                break;
            default:
                break;
        }
    }
}

private void ResetVariableValue(IUAVariable iUAVariable)
{
    var val = iUAVariable.Value.Value;
    switch (val)
    {
        case string:
            iUAVariable.Value = string.Empty;
            break;
        case int:
        case short:
        case long:
        case float:
            iUAVariable.Value = 0;
            break;
        case bool:
            iUAVariable.Value = false;
            break;
        default:
            iUAVariable.Value = 0;
            break;
    }
}
```

> [!WARNING]
> When adding entire objects to a recipe schema, all variables within that object (including those in nested objects) will be included. Be careful with large object hierarchies as they might significantly increase the size of your recipe database.

## Working with Recipe Data at Runtime

The following examples demonstrate common recipe operations you might need to perform in your application.

> [!NOTE]
> Please see the `RecipesController.cs` file in the `Scripts` library of the FactoryTalk Optix IDE to see how to load, apply, create or update recipes at runtime.

### Importing/Exporting recipes to CSV

> [!NOTE]
> Please use the code in the `RecipesController.cs` file in the `Scripts` library of the FactoryTalk Optix IDE to import and export recipes to CSV format.

### Checking if a recipe exists

```csharp
[ExportMethod]
public bool RecipeExists(string recipeName)
{
    var recipeSchema = Project.Current.Get<RecipeSchema>("Recipes/RecipeSchema1");
    if (recipeSchema == null || string.IsNullOrEmpty(recipeName))
        return false;
        
    var store = InformationModel.Get<Store>(recipeSchema.Store);
    if (store == null)
        return false;
        
    var tableName = string.IsNullOrEmpty(recipeSchema.TableName) ? recipeSchema.BrowseName : recipeSchema.TableName;
    
    // Query the database to check if the recipe exists
    object[,] resultSet;
    string[] header;
    string query = $"SELECT * FROM \"{tableName}\" WHERE Name LIKE '{recipeName.Replace("'", "''")}'";
    store.Query(query, out header, out resultSet);
    
    return resultSet != null && resultSet.GetLength(0) > 0;
}
```
