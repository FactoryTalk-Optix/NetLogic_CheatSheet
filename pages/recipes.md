# Recipes

Recipes in FT Optix provide a structured way to store and manage sets of parameters that can be loaded into your application. They are particularly useful for manufacturing processes where different product variants require different machine settings.

> [!WARNING]
> This section is a work in progress and might be updated with more examples and explanations in the future.
> Please report any issues or suggestions to the FactoryTalk Optix support or community forums.

> [!NOTE]
> The recipes engine in FactoryTalk Optix allows a maximum of 2048 ingredients per recipe. This limit is imposed by the underlying database system and cannot be changed.

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

> [!WARNING]
> When adding entire objects to a recipe schema, all variables within that object (including those in nested objects) will be included. Be careful with large object hierarchies as they might significantly increase the size of your recipe database.

## Working with Recipe Data at Runtime

The following examples demonstrate common recipe operations you might need to perform in your application.

> [!NOTE]
> The following snippets came from the `RecipesController.cs` file in the `Scripts` library of the FactoryTalk Optix project. Some adjustments may be needed to fit your specific project structure and requirements.

### Creating a new recipe

```csharp
[ExportMethod]
public void CreateNewRecipe(string recipeName)
{
    var recipeSchema = Project.Current.Get<RecipeSchema>("Recipes/RecipeSchema1");
    if (recipeSchema == null)
        return;
        
    try 
    {
        var store = InformationModel.Get<Store>(recipeSchema.Store);
        var editModel = recipeSchema.GetObject("EditModel");

        if (store != null && editModel != null)
        {
            // Create recipe in the store
            recipeSchema.CreateStoreRecipe(recipeName);
            
            // Save current values from EditModel to the recipe
            using (var transaction = new StoreTransaction())
            {
                recipeSchema.CopyToStoreRecipe(editModel.NodeId, recipeName, CopyErrorPolicy.BestEffortCopy);
                transaction.Commit();
            }
            
            // Provide feedback to the user
            Log.Info($"Recipe {recipeName} created successfully");
        }
    }
    catch (Exception e)
    {
        Log.Error($"Error creating recipe: {e.Message}");
    }
}
```

### Loading a recipe

```csharp
[ExportMethod]
public void LoadRecipe(string recipeName)
{
    var recipeSchema = Project.Current.Get<RecipeSchema>("Recipes/RecipeSchema1");
    if (recipeSchema == null)
        return;
    
    try
    {
        // Get the EditModel to load the recipe into
        var editModelNode = recipeSchema.GetObject("EditModel");
        if (editModelNode == null)
            return;
            
        // Load recipe from store to EditModel
        using (var transaction = new StoreTransaction())
        {
            // Different error policies can be used: BestEffortCopy, CancelOnError, or OverwriteIncompatible
            recipeSchema.CopyFromStoreRecipe(recipeName, editModelNode.NodeId, CopyErrorPolicy.BestEffortCopy);
            transaction.Commit();
        }
    }
    catch (Exception e)
    {
        Log.Error($"Error loading recipe: {e.Message}");
    }
}
```

### Applying a recipe

After loading a recipe into the EditModel, you can apply it directly to the target node:

```csharp
[ExportMethod]
public void ApplyRecipe()
{
    var recipeSchema = Project.Current.Get<RecipeSchema>("Recipes/RecipeSchema1");
    if (recipeSchema == null)
        return;
    
    try
    {
        // Get the EditModel to use as source for applying values
        var editModel = recipeSchema.GetObject("EditModel");
        if (editModel != null)
        {
            // Apply values from EditModel to target node (typically your Model folder)
            recipeSchema.CopyFromEditModel(editModel.NodeId, recipeSchema.TargetNode, CopyErrorPolicy.BestEffortCopy);
            Log.Info("Recipe applied successfully");
        }
    }
    catch (Exception e)
    {
        Log.Error($"Error applying recipe: {e.Message}");
    }
}
```

### Applying a recipe directly from the database

```csharp
[ExportMethod]
public void ApplyRecipeFromDB(string recipeName)
{
    var recipeSchema = Project.Current.Get<RecipeSchema>("Recipes/RecipeSchema1");
    if (recipeSchema == null)
        return;
        
    try
    {
        // Apply recipe directly from store to target node
        recipeSchema.CopyFromStoreRecipe(recipeName, recipeSchema.TargetNode, CopyErrorPolicy.BestEffortCopy);
        Log.Info($"Recipe {recipeName} applied from database");
        
        // Optionally store the last applied recipe name
        var lastAppliedVar = Project.Current.GetVariable("Model/LastAppliedRecipe");
        if (lastAppliedVar != null)
            lastAppliedVar.Value = recipeName;
    }
    catch (Exception e)
    {
        Log.Error($"Error applying recipe from database: {e.Message}");
    }
}
```

### Loading values from PLC to recipe

```csharp
[ExportMethod]
public void LoadFromPLC()
{
    var recipeSchema = Project.Current.Get<RecipeSchema>("Recipes/RecipeSchema1");
    if (recipeSchema == null)
        return;
        
    try
    {
        var editModel = recipeSchema.GetObject("EditModel");
        if (editModel != null)
        {
            // Copy values from PLC (via TargetNode) to the EditModel
            recipeSchema.Copy(recipeSchema.TargetNode, editModel.NodeId, CopyErrorPolicy.BestEffortCopy);
            Log.Info("Values loaded from PLC to recipe edit model");
        }
    }
    catch (Exception e)
    {
        Log.Error($"Error loading from PLC: {e.Message}");
    }
}
```

### Deleting a recipe

```csharp
[ExportMethod]
public void DeleteRecipe(string recipeName)
{
    var recipeSchema = Project.Current.Get<RecipeSchema>("Recipes/RecipeSchema1");
    if (recipeSchema == null || string.IsNullOrEmpty(recipeName))
        return;
        
    try
    {
        // Delete the recipe from the store
        recipeSchema.DeleteStoreRecipe(recipeName);
        Log.Info($"Recipe {recipeName} deleted successfully");
    }
    catch (Exception e)
    {
        Log.Error($"Error deleting recipe: {e.Message}");
    }
}
```

### Importing/Exporting recipes to CSV

> [!NOTE]
> Please use the code in the `RecipesController.cs` file in the `Scripts` library of the FactoryTalk Optix to import and export recipes to CSV format.

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
