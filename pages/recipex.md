# RecipeX

> [!NOTE]
> This feature is available starting from FactoryTalk Optix 1.7.X and should not be confused with [legacy recipe](./recipes.md) features.

## Overview

The **RecipeX** module provides a new way to handle recipes in FactoryTalk Optix applications. Recipes are structured data templates used to store, retrieve, and transfer configuration parameters and data items. The RecipeX system supports:

- Recipe creation, duplication, renaming, and deletion
- Data item management and value manipulation
- Metadata handling with custom fields
- Recipe transfers between store and target locations
- Edit model management for recipe editing workflows

Before reading this chapter, please check the [Recipe Schemas Documentation](https://www.rockwellautomation.com/en-us/docs/factorytalk-optix/1-7-0/contents-ditamap/creating-projects/recipes/recipex-schemas.html) for an introduction to recipe schemas and their configuration in the FactoryTalk Optix HMI project.

> [!NOTE]
> If a recipe schema is edited, the Runtime will require a Refactor to update the database table structure accordingly. It is advised to backup your project before making changes to recipe schemas.

## Core Types

### FTOptix.RecipeX.RecipeId

Uniquely identifies a recipe using name and version information.

```csharp
[MapDataType(NamespaceUri = Module.Uri, Number = 10)]
public class RecipeId : TypedStruct
{
    public string Name { get; set; }        // Recipe name
    public string Version { get; set; }     // Recipe version
}
```

**Usage Example:**
```csharp
var recipeId = new FTOptix.RecipeX.RecipeId { Name = "Recipe1", Version = "1.0" };
```

### Recipe

Represents a complete recipe with metadata and timestamps.

```csharp
[MapDataType(NamespaceUri = Module.Uri, Number = 14)]
public classRecipe : TypedStruct
{
    public FTOptix.RecipeX.RecipeId FTOptix.RecipeX.RecipeId { get; set; }                     // Recipe identifier
    public DateTime RecipeSchemaTimestamp { get; set; }        // Schema timestamp
    public DateTime CreatedAt { get; set; }                    // Creation timestamp
    public DateTime ModifiedAt { get; set; }                   // Last modification timestamp
    public FTOptix.RecipeX.RecipeMetadata[] MetadataValues { get; set; }       // Custom metadata fields
}
```

### RecipeMetadata

Represents custom metadata associated with a recipe.

```csharp
[MapDataType(NamespaceUri = Module.Uri, Number = 12)]
public classRecipeMetadata : TypedStruct
{
    public string Name { get; set; }                 // Metadata field name
    public LocalizedText DisplayName { get; set; }   // Localized display name
    public NodeId DataTypeId { get; set; }           // OPC UA data type reference
    public object Value { get; set; }                // Metadata value
}
```

### RecipeDataItem

Describes a data item within a recipe, including path information for accessing nested properties.

```csharp
[MapDataType(NamespaceUri = Module.Uri, Number = 171)]
public classRecipeDataItem : TypedStruct
{
    public string[] ItemRelativeBrowsePath { get; set; }       // Path to the recipe item, relative to the schema's target node
    public string[] DataItemRelativeBrowsePath { get; set; }   // Path to data item, relative to the item node reached by ItemRelativeBrowsePath
    public FTOptix.Core.ElementAccessStruct ElementAccess { get; set; }     // Array element access (index/range)
    public NodeId DataTypeId { get; set; }                     // Data type identifier
}
```

### FTOptix.RecipeX.ErrorPolicy

Defines error handling behavior for recipe operations.

```csharp
public enum FTOptix.RecipeX.ErrorPolicy : int
{
    Strict = 0,         // Stop on first error
    BestEffort = 1      // Continue despite errors
}
```

### The Edit Model

An **Edit Model** is a temporary, session-exclusive working copy of a recipe used to support editing workflows. Key characteristics:

- **Session-exclusive**: only one session can hold an Edit Model at a time; concurrent access is not allowed. Each session can have a dedicated Edit Model instance if needed.
- **Non-destructive**: changes made inside the Edit Model are isolated from the persisted recipe until explicitly committed back to the store.
- **Queryable**: the Edit Model supports filtering and sorting, making it suitable as the data source for recipe-editing UIs (e.g. grids, lists).
- **Lifecycle**: the Edit Model is created by loading a recipe via `TransferFromStoreToEditModel` and is automatically destroyed when released or when `TransferFromEditModelToStore` completes. The `requesterNodeId` parameter identifies the session owner and is used to enforce the exclusivity constraint. An `EditModel` can also be created programmatically via `CreateEditModel` for use cases that require more direct control over its lifecycle and interactions or retrieved from an existing interactive session where a `ListView` is being used by the user to interact with a recipe.

#### FTOptix.RecipeX.EditModel Class

The `FTOptix.RecipeX.EditModel` class exposes the following members. Obtain an instance by resolving the `EditModelNodeId` returned by `CreateEditModel` and casting it:

```csharp
var editModel = (FTOptix.RecipeX.EditModel)InformationModel.Get(createResult.EditModelNodeId);
```

**Properties:**

| Member | Type | Description |
|---|---|---|
| `RecipeSchemaNodeId` | `NodeId` | NodeId of the `FTOptix.RecipeX.RecipeSchema` that owns this Edit Model |
| `RequesterNodeId` | `NodeId` | NodeId of the session owner; used to enforce exclusivity |
| `TargetNodeId` | `NodeId` | NodeId of the target node the Edit Model was created against |
| `FTOptix.RecipeX.RecipeId` | `FTOptix.RecipeX.RecipeId` | Identifies which recipe is loaded in the Edit Model |
| `PendingChanges` | `bool` | `true` if the Edit Model contains unsaved changes relative to the store; set to `false` after a successful commit |

**Methods:**

| Method | Description |
|---|---|
| `TransferFromStore(FTOptix.RecipeX.RecipeId)` | Loads a recipe from the store into this Edit Model |
| `TransferToStore(FTOptix.RecipeX.RecipeId)` | Commits Edit Model changes back to the store |
| `TransferToTarget(NodeId targetNodeId, FTOptix.RecipeX.ErrorPolicy)` | Applies Edit Model values to a target node |
| `TransferFromTarget(NodeId targetNodeId, FTOptix.RecipeX.ErrorPolicy)` | Overwrites Edit Model values with the current state of a target node |
| `GetDataItemValue(NodeId itemNodeId, string[] dataItemRelativeBrowsePath, FTOptix.Core.ElementAccessStruct elementAccess, NodeId dataTypeId)` | Reads the current in-memory value of a data item held by this Edit Model, without transferring anything to/from the store or target |

> [!TIP]
> `GetDataItemValue` was added into FactoryTalk Optix 1.8.x and is the recommended way to inspect values that a user is currently editing (e.g. in a `ListView` bound to a recipe) **before** calling `TransferFromEditModelToStore` or `TransferFromEditModelToTarget`. Unlike `RecipeSchema.GetRecipeDataItemValue`, which reads from the persisted recipe in the store, `EditModel.GetDataItemValue` reads the live, unsaved value from the Edit Model itself, so it reflects any pending edits. Note that this method takes the resolved `itemNodeId` (the actual node reached by navigating `ItemRelativeBrowsePath` from the Edit Model's `TargetNodeId`), not the browse path itself.

Edit Model instances can be programmatically created and/or interacted with as shown in the examples below.

## Understanding Data Items

### What is a Data Item?

A **data item** is the fundamental building block of a recipe. It represents a single value that can be stored, retrieved, and transferred as part of a recipe. Data items map to variables or properties in your application model and provide a structured way to capture and apply configurations.

### How Data Items Work

When you define a recipe schema in the FactoryTalk Optix designer, you select variables and properties to include in the recipe. Each selected variable becomes a **data item**. The recipe system then:

1. **Captures** current values from the target location when saving
2. **Stores** values in the database as part of the recipe
3. **Restores** values to the target location when applying

### Data Item Structure

Data items are hierarchically organized using **browse paths** - arrays of strings that represent the navigation path through your object model:

```txt
ItemRelativeBrowsePath     = ["Settings", "Motor"]
DataItemRelativeBrowsePath = ["Speed", "MaxRPM"]
```

**Please note:** Both paths are **relative**, never absolute.
- `ItemRelativeBrowsePath` is relative to the **target node** configured on the `FTOptix.RecipeX.RecipeSchema` (e.g. `TargetNode = Section1`).
- `DataItemRelativeBrowsePath` is relative to the **item node** that is reached after navigating `ItemRelativeBrowsePath` from the target node.

This structure allows you to:
- Navigate through nested objects and folders
- Access properties within objects
- Handle array subscripts using `ElementAccess`

**Visual Example:**
```txt
Target Object (Settings)
|- Motor (Item)
|  |- Speed (DataItem)     <- Gets/Sets here
|  |- MaxRPM (DataItem)    <- Gets/Sets here
|- Conveyor (Item)
   |- Enable (DataItem)
```

### Common Data Item Scenarios

#### Simple Variable (No Nesting)

```csharp
var itemPath = new string[] { "Temperature" };
var dataItemPath = new string[] { };  // Empty = direct variable
var elementAccess = new FTOptix.Core.ElementAccessStruct();

// Gets/sets "Temperature" variable directly
var result = schema.GetRecipeDataItemValue(
    recipeId, 
    itemPath, 
    dataItemPath, 
    elementAccess);
```

#### Nested Object Property

```csharp
var itemPath = new string[] { "MotorSettings" };
var dataItemPath = new string[] { "Speed" };
var elementAccess = new FTOptix.Core.ElementAccessStruct();

// Gets/sets MotorSettings.Speed
var result = schema.GetRecipeDataItemValue(
    recipeId, 
    itemPath, 
    dataItemPath, 
    elementAccess);
```

#### Deeply Nested Property

```csharp
var itemPath = new string[] { "Parameters", "Industrial", "Motor1" };
var dataItemPath = new string[] { "Configuration", "MaxSpeed" };
var elementAccess = new FTOptix.Core.ElementAccessStruct();

// Gets/sets Parameters -> Industrial -> Motor1 -> Configuration -> MaxSpeed
var result = schema.GetRecipeDataItemValue(
    recipeId, 
    itemPath, 
    dataItemPath, 
    elementAccess);
```

#### Array Element Access

```csharp
var itemPath = new string[] { "Sensors" };
var dataItemPath = new string[] { "Readings" };
var elementAccess = new FTOptix.Core.ElementAccessStruct 
{ 
    Index = 0  // Get first array element
};

// Gets/sets Sensors.Readings[0]
var result = schema.GetRecipeDataItemValue(
    recipeId, 
    itemPath, 
    dataItemPath, 
    elementAccess);
```

### Working with Data Items in Code

#### Retrieving All Data Items in a Recipe

```csharp
[ExportMethod]
public void InspectDataItems(string recipeName, string version)
{
    var schema = (FTOptix.RecipeX.RecipeSchema)LogicObject.Owner.GetObject("RecipeSchema1");
    var recipeId = new FTOptix.RecipeX.RecipeId { Name = recipeName, Version = version };
    
    // Get the structure of data items in this recipe
    var result = schema.GetDataItems(recipeId);
    
    if (result.ResultCode == GetDataItemsResultCode.Success)
    {
        Log.Info($"Recipe contains {result.DataItems.Length} data items:");
        
        foreach (var dataItem in result.DataItems)
        {
            // Build readable path
            var itemPath = string.Join("/", dataItem.ItemRelativeBrowsePath);
            var dataPath = string.Join("/", dataItem.DataItemRelativeBrowsePath);
            
            if (!string.IsNullOrEmpty(dataPath))
            {
                Log.Info($"  {itemPath} -> {dataPath}");
            }
            else
            {
                Log.Info($"  {itemPath}");
            }
        }
    }
}
```

#### Reading Individual Data Item Values

```csharp
[ExportMethod]
public object ReadDataItemValue(string recipeName, string[] itemPath, 
    string[] dataItemPath)
{
    var schema = (FTOptix.RecipeX.RecipeSchema)LogicObject.Owner.GetObject("RecipeSchema1");
    var recipeId = new FTOptix.RecipeX.RecipeId { Name = recipeName, Version = "1.0" };
    
    var result = schema.GetRecipeDataItemValue(
        recipeId,
        itemPath,
        dataItemPath,
        new FTOptix.Core.ElementAccessStruct());
    
    if (result.ResultCode == GetRecipeDataItemValueResultCode.Success)
    {
        return result.DataItemValue;
    }
    else
    {
        Log.Error($"Failed to read data item: {result.ResultCode}");
        return null;
    }
}
```

#### Writing Data Item Values

```csharp
[ExportMethod]
public void WriteDataItemValue(string recipeName, string[] itemPath, 
    string[] dataItemPath, object value)
{
    var schema = (FTOptix.RecipeX.RecipeSchema)LogicObject.Owner.GetObject("RecipeSchema1");
    var recipeId = new FTOptix.RecipeX.RecipeId { Name = recipeName, Version = "1.0" };
    
    var result = schema.SetRecipeDataItemValue(
        recipeId,
        itemPath,
        dataItemPath,
        new FTOptix.Core.ElementAccessStruct(),
        value);
    
    if (result == FTOptix.RecipeX.SetRecipeDataItemValueResultCode.Success)
    {
        Log.Info("Data item value updated");
    }
    else
    {
        Log.Error($"Failed to update data item: {result}");
    }
}
```

#### Comparing Current vs. Stored Values

```csharp
[ExportMethod]
public bool CompareDataItemWithTarget(string recipeName, string[] itemPath, 
    string[] dataItemPath, string targetVariablePath)
{
    var schema = (FTOptix.RecipeX.RecipeSchema)LogicObject.Owner.GetObject("RecipeSchema1");
    var recipeId = new FTOptix.RecipeX.RecipeId { Name = recipeName, Version = "1.0" };
    var target = LogicObject.Owner.GetVariable(targetVariablePath);
    
    // Get stored recipe value
    var recipeResult = schema.GetRecipeDataItemValue(
        recipeId,
        itemPath,
        dataItemPath,
        new FTOptix.Core.ElementAccessStruct());
    
    if (recipeResult.ResultCode != GetRecipeDataItemValueResultCode.Success)
        return false;
    
    // Get current target value
    var targetValue = target?.Value.Value;
    
    // Compare
    bool valuesMatch = Equals(recipeResult.DataItemValue, targetValue);
    
    Log.Info($"Recipe value: {recipeResult.DataItemValue}");
    Log.Info($"Target value: {targetValue}");
    Log.Info($"Match: {valuesMatch}");
    
    return valuesMatch;
}
```

### Data Item Tips and Best Practices

1. **Path Ordering Matters**: The order of path segments must exactly match your object hierarchy
2. **Empty Data Item Path**: If the item itself is the variable, use an empty array for `dataItemRelativeBrowsePath`
3. **Type Consistency**: Ensure the value type matches the data item's data type
4. **Array Indices**: Use `FTOptix.Core.ElementAccessStruct` only for array subscripts; regular object properties go in the data item path
5. **Performance**: Reading/writing individual data items has overhead; consider batch operations when possible

### Common Mistakes

**Wrong:** Using the whole path as item path
```csharp
// Incorrect
var itemPath = new string[] { "Settings", "Motor", "Speed" };
var dataItemPath = new string[] { };
```

**Right:** Splitting item and property paths
```csharp
// Correct
var itemPath = new string[] { "Settings", "Motor" };
var dataItemPath = new string[] { "Speed" };
```

---

## Result Types

All methods return result types containing both a result code and relevant data. These are used to handle success and error conditions consistently.

### GetRecipesResult

Result of retrieving all recipes from a recipe schema.

```csharp
[MapDataType(NamespaceUri = Module.Uri, Number = 182)]
public class GetRecipesResult : TypedStruct
{
    public Recipe[] Recipes { get; set; }                  // Array of recipes
    public GetRecipesResultCode ResultCode { get; set; }   // Result code
}
```

**Result Codes:**
- `Success = 0` - Recipes retrieved successfully
- `StoreError = 1` - Error accessing the store
- `GenericError = 2` - General error occurred

### GetDataItemsResult

Result of retrieving data items from a recipe.

```csharp
[MapDataType(NamespaceUri = Module.Uri, Number = 178)]
public class GetDataItemsResult : TypedStruct
{
    public RecipeDataItem[] DataItems { get; set; }            // Array of data items
    public GetDataItemsResultCode ResultCode { get; set; }     // Result code
}
```

**Result Codes:**
- `Success = 0` - Data items retrieved successfully
- `RecipeNotFound = 1` - Specified recipe not found
- `StoreError = 2` - Error accessing the store
- `GenericError = 3` - General error occurred

### GetRecipeDataItemValueResult

Result of retrieving a specific data item value from a recipe.

```csharp
[MapDataType(NamespaceUri = Module.Uri, Number = 127)]
public class GetRecipeDataItemValueResult : TypedStruct
{
    public object DataItemValue { get; set; }                          // Retrieved value (scalar or array)
    public GetRecipeDataItemValueResultCode ResultCode { get; set; }   // Result code
}
```

**Result Codes:**
- `Success = 0` - Value retrieved successfully
- `GenericError = 1` - General error occurred
- `StoreError = 2` - Error accessing the store
- `DataItemNotFound = 3` - Data item not found in recipe
- `UnexpectedEmptyRecipeName = 4` - Recipe name is empty/null
- `RecipeNotFound = 5` - Recipe not found in store

### GetDataItemValueResult

Result of `FTOptix.RecipeX.EditModel.GetDataItemValue`, used to read the current in-memory value of a data item held by an Edit Model (i.e. the value as currently edited, before it is transferred to the store or a target).

```csharp
public class GetDataItemValueResult : TypedStruct
{
    public object DataItemValue { get; set; }                  // Value currently held in the Edit Model (scalar or array)
    public GetDataItemValueResultCode ResultCode { get; set; } // Result code
}
```

**Result Codes:**
- `Success = 0` - Value retrieved successfully
- `GenericError = 1` - General error occurred
- `DataItemNotFound = 2` - Data item not found in the Edit Model

### FTOptix.RecipeX.GetRecipeMetadataValueResult

Result of retrieving a single metadata value.

```csharp
[MapDataType(NamespaceUri = Module.Uri, Number = 134)]
public classGetRecipeMetadataValueResult : TypedStruct
{
    public RecipeMetadata MetadataValue { get; set; }                      // Retrieved metadata
    public GetRecipeMetadataValueResultCode ResultCode { get; set; }       // Result code
}
```

**Result Codes:**
- `Success = 0` - Metadata retrieved successfully
- `GenericError = 1` - General error occurred
- `StoreError = 2` - Error accessing the store
- `UnexpectedEmptyRecipeName = 3` - Recipe name is empty/null
- `RecipeNotFound = 4` - Recipe not found in store
- `MetadataNotFound = 5` - Metadata field not found

### GetRecipeMetadataValuesResult

Result of retrieving all metadata for a recipe.

```csharp
[MapDataType(NamespaceUri = Module.Uri, Number = 138)]
public class GetRecipeMetadataValuesResult : TypedStruct
{
    public RecipeMetadata[] MetadataValues { get; set; }                  // Array of metadata
    public GetRecipeMetadataValuesResultCode ResultCode { get; set; }     // Result code
}
```

**Result Codes:**
- `Success = 0` - Metadata values retrieved successfully
- `GenericError = 1` - General error occurred
- `StoreError = 2` - Error accessing the store
- `UnexpectedEmptyRecipeName = 3` - Recipe name is empty/null
- `RecipeNotFound = 4` - Recipe not found in store

### RenameRecipeResult

Result of renaming a recipe.

```csharp
[MapDataType(NamespaceUri = Module.Uri, Number = 115)]
public class RenameRecipeResult : TypedStruct
{
    public FTOptix.RecipeX.RecipeId NewRecipeId { get; set; }              // New recipe identifier (after rename)
    public RenameRecipeResultCode ResultCode { get; set; } // Result code
}
```

**Result Codes:**
- `Success = 0` - Recipe renamed successfully
- `GenericError = 1` - General error occurred
- `StoreError = 2` - Error accessing the store
- `UnexpectedEmptyRecipeName = 3` - Recipe name is empty/null
- `SourceRecipeDoesNotExist = 4` - Recipe not found
- `TargetRecipeAlreadyExist = 5` - Target recipe name already in use

### FTOptix.RecipeX.CreateEditModelResult

Result of creating an edit model.

```csharp
[MapDataType(NamespaceUri = Module.Uri, Number = 142)]
public classCreateEditModelResult : TypedStruct
{
    public NodeId EditModelNodeId { get; set; }                   // NodeId of created edit model
    public CreateEditModelResultCode ResultCode { get; set; }     // Result code
}
```

**Result Codes:**
- `Success = 0` - Edit model created successfully
- `GenericError = 1` - General error occurred
- `StoreError = 2` - Error accessing the store
- `ParentNodeNotFound = 3` - Parent node for edit model not found
- `RequesterNodeNotFound = 4` - Requester node not found

## FTOptix.RecipeX.RecipeSchema Class

The main class for recipe operations in FTOptix. Inherit from `UAObject` and provides methods for all recipe management tasks.

```csharp
[MapType(NamespaceUri = Module.Uri, Number = 1)]
public classRecipeSchema : UAObject
{
    // Properties
    public NodeId TargetNode { get; set; }                  // Target node for recipe operations
    public NodeId Store { get; set; }                       // Store node reference
    public RecipeSchemaStatus Status { get; }              // Current status
    public UInt32 RemoteReadTimeout { get; set; }          // Remote read timeout in ms
    public PlaceholderChildNodeCollection<UAVariable> MetaData { get; } // Metadata variables
    
    // Methods...
}
```

## FTOptix.RecipeX.RecipeSchema Methods

### Recipe Management Methods

#### CreateRecipe

Creates a new empty recipe in the store.

```csharp
public CreateRecipeResultCode CreateRecipe(FTOptix.RecipeX.RecipeId recipeId)
```

**Parameters:**
- `recipeId` - Identifier with name and version

**Returns:** `CreateRecipeResultCode`
- `Success` - Recipe created successfully
- `GenericError` - General error occurred
- `StoreError` - Error accessing the store
- `UnexpectedEmptyRecipeName` - Recipe name is empty
- `RecipeAlreadyExist` - Recipe with same name/version exists
- `TargetNotFound` - Target node not found

**Example:**
```csharp
var schema = (FTOptix.RecipeX.RecipeSchema)hmiProject.GetVariable("RecipeSchema1").Value;
var recipeId = new FTOptix.RecipeX.RecipeId { Name = "MyRecipe", Version = "1.0" };
var result = schema.CreateRecipe(recipeId);
if (result == CreateRecipeResultCode.Success)
{
    Log.Info("Recipe created successfully");
}
```

#### DeleteRecipe

Deletes a recipe from the store.

```csharp
public DeleteRecipeResultCode DeleteRecipe(FTOptix.RecipeX.RecipeId recipeId)
```

**Parameters:**
- `recipeId` - Recipe to delete

**Returns:** `DeleteRecipeResultCode`
- `Success` - Recipe deleted successfully
- `GenericError` - General error occurred
- `StoreError` - Error accessing the store
- `UnexpectedEmptyRecipeName` - Recipe name is empty
- `RecipeDoesNotExist` - Recipe not found

**Example:**
```csharp
var result = schema.DeleteRecipe(recipeId);
```

#### DuplicateRecipe

Creates a copy of an existing recipe with a new identifier.

```csharp
public DuplicateRecipeResultCode DuplicateRecipe(FTOptix.RecipeX.RecipeId recipeId, FTOptix.RecipeX.RecipeId newRecipeId)
```

**Parameters:**
- `recipeId` - Source recipe to duplicate
- `newRecipeId` - Identifier for the new recipe

**Returns:** `DuplicateRecipeResultCode`
- `Success` - Recipe duplicated successfully
- `GenericError` - General error during duplication
- `StoreError` - Error accessing the store
- `UnexpectedSourceEmptyRecipeName` - Source recipe name is empty
- `UnexpectedTargetEmptyRecipeName` - Target recipe name is empty
- `SourceRecipeDoesNotExist` - Source recipe not found
- `TargetRecipeAlreadyExist` - Target recipe name already exists
- `TargetNotFound` - Target node not found

**Example:**
```csharp
var sourceId = new FTOptix.RecipeX.RecipeId { Name = "Recipe1", Version = "1.0" };
var targetId = new FTOptix.RecipeX.RecipeId { Name = "Recipe1_Copy", Version = "1.0" };
var result = schema.DuplicateRecipe(sourceId, targetId);
```

#### RenameRecipe

Renames an existing recipe.

```csharp
public RenameRecipeResult RenameRecipe(FTOptix.RecipeX.RecipeId recipeId, string newName)
```

**Parameters:**
- `recipeId` - Recipe to rename
- `newName` - New recipe name

**Returns:** `RenameRecipeResult`
- `NewRecipeId` - Updated recipe identifier
- `ResultCode` - Operation result

**Example:**
```csharp
var result = schema.RenameRecipe(recipeId, "NewRecipeName");
if (result.ResultCode == RenameRecipeResultCode.Success)
{
    Log.Info($"Recipe renamed to: {result.NewRecipeId.Name}");
}
```

### Data Retrieval Methods

#### GetRecipes

Retrieves all recipes from the schema.

```csharp
public GetRecipesResult GetRecipes()
```

**Returns:** `GetRecipesResult`
- `Recipes` - Array of all recipes
- `ResultCode` - Operation result

**Example:**
```csharp
var result = schema.GetRecipes();
if (result.ResultCode == GetRecipesResultCode.Success)
{
    foreach (var recipe in result.Recipes)
    {
        Log.Info($"Recipe: {recipe.FTOptix.RecipeX.RecipeId.Name} v{recipe.FTOptix.RecipeX.RecipeId.Version}");
    }
}
```

#### GetDataItems

Retrieves the data item structure for a recipe.

```csharp
public GetDataItemsResult GetDataItems(FTOptix.RecipeX.RecipeId recipeId)
```

**Parameters:**
- `recipeId` - Recipe identifier

**Returns:** `GetDataItemsResult`
- `DataItems` - Array describing data items in the recipe
- `ResultCode` - Operation result

**Example:**
```csharp
var result = schema.GetDataItems(recipeId);
if (result.ResultCode == GetDataItemsResultCode.Success)
{
    foreach (var item in result.DataItems)
    {
        var itemPath = string.Join("/", item.ItemRelativeBrowsePath);
        Log.Info($"Data Item: {itemPath}");
    }
}
```

#### GetRecipeDataItemValue

Retrieves a specific data item value from a recipe.

```csharp
public GetRecipeDataItemValueResult GetRecipeDataItemValue(
    FTOptix.RecipeX.RecipeId recipeId, 
    string[] itemRelativeBrowsePath, 
    string[] dataItemRelativeBrowsePath, 
    FTOptix.Core.ElementAccessStruct elementAccess)
```

**Parameters:**
- `recipeId` - Recipe identifier
- `itemRelativeBrowsePath` - Browse path to item (array of path segments)
- `dataItemRelativeBrowsePath` - Browse path to data item within the item
- `elementAccess` - Array element access for subscripts (if applicable)

**Returns:** `GetRecipeDataItemValueResult`
- `DataItemValue` - Retrieved value (scalar or array)
- `ResultCode` - Operation result

**Example:**
```csharp
var itemPath = new string[] { "Item1" };
var dataItemPath = new string[] { "Property1" };
var elementAccess = new FTOptix.Core.ElementAccessStruct(); // No array index

var result = schema.GetRecipeDataItemValue(recipeId, itemPath, dataItemPath, elementAccess);
if (result.ResultCode == GetRecipeDataItemValueResultCode.Success)
{
    Log.Info($"Value: {result.DataItemValue}");
}
```

#### GetRecipeMetadataValue

Retrieves a single metadata value.

```csharp
public FTOptix.RecipeX.GetRecipeMetadataValueResult GetRecipeMetadataValue(FTOptix.RecipeX.RecipeId recipeId, string metadataName)
```

**Parameters:**
- `recipeId` - Recipe identifier
- `metadataName` - Name of the metadata field

**Returns:** `FTOptix.RecipeX.GetRecipeMetadataValueResult`
- `MetadataValue` - Retrieved metadata
- `ResultCode` - Operation result

**Example:**
```csharp
var result = schema.GetRecipeMetadataValue(recipeId, "Author");
if (result.ResultCode == GetRecipeMetadataValueResultCode.Success)
{
    Log.Info($"Author: {result.MetadataValue.Value}");
}
```

#### GetRecipeMetadataValues

Retrieves all metadata for a recipe.

```csharp
public GetRecipeMetadataValuesResult GetRecipeMetadataValues(FTOptix.RecipeX.RecipeId recipeId)
```

**Parameters:**
- `recipeId` - Recipe identifier

**Returns:** `GetRecipeMetadataValuesResult`
- `MetadataValues` - Array of all metadata fields
- `ResultCode` - Operation result

**Example:**
```csharp
var result = schema.GetRecipeMetadataValues(recipeId);
if (result.ResultCode == GetRecipeMetadataValuesResultCode.Success)
{
    foreach (var metadata in result.MetadataValues)
    {
        Log.Info($"{metadata.Name}: {metadata.Value}");
    }
}
```

### Data Modification Methods

#### SetRecipeDataItemValue

Updates a data item value in a recipe.

```csharp
public FTOptix.RecipeX.SetRecipeDataItemValueResultCode SetRecipeDataItemValue(
    FTOptix.RecipeX.RecipeId recipeId,
    string[] itemRelativeBrowsePath,
    string[] dataItemRelativeBrowsePath,
    FTOptix.Core.ElementAccessStruct elementAccess,
    object value)
```

**Parameters:**
- `recipeId` - Recipe identifier
- `itemRelativeBrowsePath` - Browse path to item
- `dataItemRelativeBrowsePath` - Browse path to data item
- `elementAccess` - Array element access
- `value` - New value to set

**Returns:** `FTOptix.RecipeX.SetRecipeDataItemValueResultCode`
- `Success` - Value set successfully
- `GenericError` - General error occurred
- `StoreError` - Error accessing the store
- `DataItemNotFound` - Data item not found
- `UnexpectedEmptyRecipeName` - Recipe name is empty
- `RecipeNotFound` - Recipe not found

**Example:**
```csharp
var result = schema.SetRecipeDataItemValue(
    recipeId,
    new string[] { "Item1" },
    new string[] { "Property1" },
    new FTOptix.Core.ElementAccessStruct(),
    42);
```

#### SetRecipeMetadataValue

Updates a metadata value.

```csharp
public FTOptix.RecipeX.SetRecipeMetadataValueResultCode SetRecipeMetadataValue(
    FTOptix.RecipeX.RecipeId recipeId,
    string metadataName,
    object value)
```

**Parameters:**
- `recipeId` - Recipe identifier
- `metadataName` - Metadata field name
- `value` - New metadata value

**Returns:** `FTOptix.RecipeX.SetRecipeMetadataValueResultCode`
- `Success` - Metadata set successfully
- `GenericError` - General error occurred
- `StoreError` - Error accessing the store
- `UnexpectedEmptyRecipeName` - Recipe name is empty
- `RecipeNotFound` - Recipe not found
- `MetadataNotFound` - Metadata field not found

**Example:**
```csharp
var result = schema.SetRecipeMetadataValue(recipeId, "Author", "John Doe");
```

### Transfer Methods

Transfer methods move recipes between three locations: **Store** (persistent database), **Edit Model** (temporary working copy), and **Target** (active runtime location).

#### TransferFromStoreToTarget

Loads a recipe from the store and applies it to a target location.

```csharp
public FTOptix.RecipeX.TransferFromStoreToTargetResultCode TransferFromStoreToTarget(
    FTOptix.RecipeX.RecipeId recipeId,
    NodeId targetNodeId,
    FTOptix.RecipeX.ErrorPolicy errorPolicy)
```

**Parameters:**
- `recipeId` - Recipe to transfer
- `targetNodeId` - Destination node
- `errorPolicy` - Error handling mode

**Returns:** `FTOptix.RecipeX.TransferFromStoreToTargetResultCode`
- `Success` - Transfer successful
- `GenericError` - General error occurred
- `StoreError` - Error accessing the store
- `UnexpectedEmptyRecipeName` - Recipe name is empty
- `RecipeNotFound` - Recipe not found
- `TargetNotFound` - Target node not found
- `DataMismatch` - Data structure mismatch

**Example:**
```csharp
var targetNode = hmiProject.GetVariable("TargetObject").Value;
var result = schema.TransferFromStoreToTarget(recipeId, targetNode, FTOptix.RecipeX.ErrorPolicy.Strict);
```

#### TransferFromTargetToStore

Captures the current state of a target and saves it as a recipe.

```csharp
public FTOptix.RecipeX.TransferFromTargetToStoreResultCode TransferFromTargetToStore(
    FTOptix.RecipeX.RecipeId recipeId,
    NodeId targetNodeId,
    bool overwrite,
    FTOptix.RecipeX.ErrorPolicy errorPolicy)
```

**Parameters:**
- `recipeId` - Recipe identifier to create/update
- `targetNodeId` - Source location to capture
- `overwrite` - Whether to overwrite existing recipe
- `errorPolicy` - Error handling mode

**Returns:** `FTOptix.RecipeX.TransferFromTargetToStoreResultCode`
- `SuccessRecipeCreated` - New recipe created
- `SuccessRecipeUpdated` - Existing recipe updated
- `GenericError` - General error
- `StoreError` - Store access error
- `UnexpectedEmptyRecipeName` - Recipe name is empty
- `RecipeAlreadyExists` - Recipe exists and overwrite is false
- `TargetNotFound` - Target node not found
- `DataMismatch` - Data structure mismatch

**Example:**
```csharp
var result = schema.TransferFromTargetToStore(
    recipeId,
    targetNode,
    overwrite: true,
    FTOptix.RecipeX.ErrorPolicy.BestEffort);
```

#### TransferFromStoreToEditModel

Loads a recipe from the store into the session's Edit Model.

```csharp
public FTOptix.RecipeX.TransferFromStoreToEditModelResultCode TransferFromStoreToEditModel(
    NodeId requesterNodeId,
    FTOptix.RecipeX.RecipeId recipeId)
```

**Parameters:**
- `requesterNodeId` - Node requesting the operation
- `recipeId` - Recipe to load

**Returns:** `FTOptix.RecipeX.TransferFromStoreToEditModelResultCode`
- `Success` - Transfer successful
- `GenericError` - General error
- `StoreError` - Store access error
- `RequesterNodeNotFound` - Requester node not found
- `UnexpectedEmptyRecipeName` - Recipe name is empty
- `RecipeNotFound` - Recipe not found
- `EditModelNodeNotFound` - Edit model not found

**Example:**
```csharp
var requester = hmiProject.GetVariable("LogicObject").Value;
var result = schema.TransferFromStoreToEditModel(requester, recipeId);
```

#### TransferFromEditModelToStore

Saves the edit model changes back to the store.

```csharp
public FTOptix.RecipeX.TransferFromEditModelToStoreResultCode TransferFromEditModelToStore(
    NodeId requesterNodeId,
    FTOptix.RecipeX.RecipeId recipeId)
```

**Parameters:**
- `requesterNodeId` - Node requesting the operation
- `recipeId` - Recipe identifier (for new recipes)

**Returns:** `FTOptix.RecipeX.TransferFromEditModelToStoreResultCode`
- `SuccessRecipeCreated` - New recipe created
- `SuccessRecipeUpdated` - Existing recipe updated
- `GenericError` - General error
- `StoreError` - Store access error
- `RequesterNodeNotFound` - Requester node not found
- `UnexpectedEmptyRecipeName` - Recipe name is empty
- `TargetNotFound` - Target not found
- `EditModelNodeNotFound` - Edit model not found

**Example:**
```csharp
var result = schema.TransferFromEditModelToStore(requester, recipeId);
```

#### TransferFromEditModelToTarget

Applies edit model changes to a target location.

```csharp
public FTOptix.RecipeX.TransferFromEditModelToTargetResultCode TransferFromEditModelToTarget(
    NodeId requesterNodeId,
    NodeId targetNodeId,
    FTOptix.RecipeX.ErrorPolicy errorPolicy)
```

**Parameters:**
- `requesterNodeId` - Node requesting the operation
- `targetNodeId` - Destination node
- `errorPolicy` - Error handling mode

**Returns:** `FTOptix.RecipeX.TransferFromEditModelToTargetResultCode`
- `Success` - Transfer successful
- `GenericError` - General error
- `StoreError` - Store access error
- `RequesterNodeNotFound` - Requester node not found
- `TargetNotFound` - Target node not found
- `EditModelNodeNotFound` - Edit model not found
- `DataMismatch` - Data structure mismatch

#### TransferFromTargetToEditModel

Loads target state into edit model for editing.

```csharp
public FTOptix.RecipeX.TransferFromTargetToEditModelResultCode TransferFromTargetToEditModel(
    NodeId requesterNodeId,
    NodeId targetNodeId,
    FTOptix.RecipeX.ErrorPolicy errorPolicy)
```

**Parameters:**
- `requesterNodeId` - Node requesting the operation
- `targetNodeId` - Source location
- `errorPolicy` - Error handling mode

**Returns:** `FTOptix.RecipeX.TransferFromTargetToEditModelResultCode`
- `Success` - Transfer successful
- `GenericError` - General error
- `StoreError` - Store access error
- `RequesterNodeNotFound` - Requester node not found
- `TargetNotFound` - Target node not found
- `EditModelNodeNotFound` - Edit model not found
- `DataMismatch` - Data structure mismatch

### Edit Model Methods

#### CreateEditModel

Creates a working copy of a recipe for editing.

```csharp
public FTOptix.RecipeX.CreateEditModelResult CreateEditModel(
    NodeId parentNodeId,
    NodeId requesterNodeId,
    NodeId targetNodeId,
    FTOptix.RecipeX.RecipeId recipeId)
```

**Parameters:**
- `parentNodeId` - Parent node for the edit model
- `requesterNodeId` - Node requesting the operation
- `targetNodeId` - Target structure reference
- `recipeId` - Recipe to edit

**Returns:** `FTOptix.RecipeX.CreateEditModelResult`
- `EditModelNodeId` - NodeId of created edit model
- `ResultCode` - Operation result

> [!TIP]
> Once you have the `EditModelNodeId`, you can resolve it to the actual `FTOptix.RecipeX.EditModel` node and cast it to `FTOptix.RecipeX.EditModel`. The cast exposes additional members not available through `FTOptix.RecipeX.RecipeSchema`, including:
> - `PendingChanges` — indicates whether the edit model has unsaved changes relative to the recipe in the store.
> - Transfer methods directly on the `FTOptix.RecipeX.EditModel` instance (e.g. applying to target, committing to store) that operate implicitly on this edit model without having to pass the requester/node again.

> [!NOTE]
> `CreateEditModel` behaves as **get-or-create**: if an edit model already exists for the given `parentNodeId`/`requesterNodeId` pair, it returns the existing instance instead of creating a new one. This is the correct way to access an edit model that was created and is managed by a `ListView` widget — pass the `ListView.NodeId` as both `parentNodeId` and `requesterNodeId`, and you will get back the live edit model the ListView is currently using.

**Example:**
```csharp
var parent = hmiProject.GetVariable("Parent").Value;
var requester = hmiProject.GetVariable("Requester").Value;
var target = hmiProject.GetVariable("Target").Value;

var result = schema.CreateEditModel(parent, requester, target, recipeId);
if (result.ResultCode == CreateEditModelResultCode.Success)
{
    // Cast to FTOptix.RecipeX.EditModel to access PendingChanges and FTOptix.RecipeX.EditModel-level transfer methods
    var editModel = (FTOptix.RecipeX.EditModel)InformationModel.Get(result.EditModelNodeId);
    Log.Info($"Pending changes: {editModel.PendingChanges}");
}
```

### Maintenance Methods

#### RefactorRecipes

Asynchronously refactors the recipe schema. This is a long-running operation that completes asynchronously. When finished, a `RefactorCompleted` event is fired.

```csharp
public void RefactorRecipes()
```

**Notes:**
- This is the only async method
- Does not block the calling thread
- Listen for `RefactorCompleted` event for completion notification

**Example:**
```csharp
schema.RefactorRecipes();
// Method returns immediately
// Listen for RefactorCompleted event to know when complete
```

## Events

### RefactorCompleted

Fired when the `RefactorRecipes()` async method completes.

**Example:**
```csharp
public class RecipeRefactorLogic : BaseNetLogic
{
    public override void Start()
    {
        var schema = LogicObject.Owner.GetObject("RecipeSchema1");
        if (schema != null)
        {
            // Subscribe to event
            schema.EventOccurred += OnRefactorCompleted;
        }
    }

    private void OnRefactorCompleted(IUANode sender, UAEventArgs args)
    {
        if (args.Event.BrowseName.Name == "RefactorCompleted")
        {
            Log.Info("Recipe refactoring completed");
        }
    }
}
```

### RecipeApplicationEvent

Fired when `TransferFromStoreToTarget` or `TransferFromEditModelToTarget` operations complete successfully.

## Database Schema

RecipeX uses database tables to persist recipe data. Understanding this structure is useful for direct database queries.

### Recipes Table

```sql
CREATE TABLE Recipes (
    Id,                         -- PRIMARY KEY [UInt32]
    Name,                       -- [String] Recipe name
    Version,                    -- [String] Recipe version
    RecipeSchemaName,           -- [String] Schema name
    RecipeSchemaNamespaceUri,   -- [String] Namespace URI
    RecipeSchemaTimestamp,      -- [DateTime] Schema timestamp
    CreatedAt,                  -- [DateTime] Creation time
    ModifiedAt                  -- [DateTime] Last modification
)
```

### RecipeItems Table

```sql
CREATE TABLE RecipeItems (
    Id,                         -- PRIMARY KEY [UInt32]
    FTOptix.RecipeX.RecipeId,                   -- [UInt32] Foreign key to Recipes
    RelativeBrowsePath,         -- [String] Item path
    TypeId                      -- [String] OPC UA type identifier
)
```

### RecipeDataItems Table

```sql
CREATE TABLE RecipeDataItems (
    Id,                         -- PRIMARY KEY [UInt32]
    RecipeItemId,               -- [UInt32] Foreign key to RecipeItems
    RelativeBrowsePath,         -- [String] Data item path
    ElementAccess,              -- [String] Array element access
    DataTypeId,                 -- [String] OPC UA data type
    Value                       -- [String] Stored value
)
```

### RecipeMetadata Table

Dynamically created as `RecipeMetadata_<RecipeSchemaName>` per schema.

```sql
-- Example: RecipeMetadata_RecipeSchema1
CREATE TABLE RecipeMetadata_RecipeSchema1 (
    FTOptix.RecipeX.RecipeId,                   -- Foreign key to Recipes
    Metadata1,                  -- [UserDefined] Custom metadata
    Metadata2,                  -- [UserDefined] Custom metadata
    -- ... additional metadata columns as defined in schema
)
```

**Sample Query:**
```sql
SELECT R.*, M.* 
FROM Recipes AS R 
LEFT JOIN "RecipeMetadata_RecipeSchema1" AS M 
    ON R.Id = M.FTOptix.RecipeX.RecipeId 
WHERE R.RecipeSchemaName = 'RecipeSchema1' 
ORDER BY R.Name
```

## Common Usage Patterns

### Complete Recipe Workflow

```csharp
public class RecipeManager : BaseNetLogic
{
    public override void Start()
    {
        var schema = (FTOptix.RecipeX.RecipeSchema)LogicObject.Owner.GetObject("RecipeSchema1");
        
        // Create recipe
        var recipeId = new FTOptix.RecipeX.RecipeId { Name = "Configuration1", Version = "1.0" };
        var createResult = schema.CreateRecipe(recipeId);
        
        if (createResult == CreateRecipeResultCode.Success)
        {
            // Set metadata
            schema.SetRecipeMetadataValue(recipeId, "Author", "Engineering");
            schema.SetRecipeMetadataValue(recipeId, "Department", "Manufacturing");
            
            // Set data values
            var itemPath = new string[] { "Settings" };
            var dataItemPath = new string[] { "Parameter1" };
            var elementAccess = new FTOptix.Core.ElementAccessStruct();
            
            schema.SetRecipeDataItemValue(recipeId, itemPath, dataItemPath, elementAccess, 100);
            
            Log.Info($"Recipe '{recipeId.Name}' created and configured");
        }
    }
}
```

### Loading and Applying a Recipe

```csharp
public class RecipeLoader : BaseNetLogic
{
    [ExportMethod]
    public void ApplyRecipe(string recipeName, string version)
    {
        var schema = (FTOptix.RecipeX.RecipeSchema)LogicObject.Owner.GetObject("RecipeSchema1");
        var targetNode = LogicObject.Owner.GetVariable("TargetObject").Value;
        
        var recipeId = new FTOptix.RecipeX.RecipeId { Name = recipeName, Version = version };
        
        var result = schema.TransferFromStoreToTarget(
            recipeId,
            targetNode,
            FTOptix.RecipeX.ErrorPolicy.Strict);
        
        if (result == FTOptix.RecipeX.TransferFromStoreToTargetResultCode.Success)
        {
            Log.Info($"Recipe '{recipeName}' applied to target");
        }
        else
        {
            Log.Error($"Failed to apply recipe: {result}");
        }
    }
}
```

### Capturing Configuration as Recipe

```csharp
public class ConfigurationCapture : BaseNetLogic
{
    [ExportMethod]
    public void SaveCurrentConfiguration(string recipeName)
    {
        var schema = (FTOptix.RecipeX.RecipeSchema)LogicObject.Owner.GetObject("RecipeSchema1");
        var targetNode = LogicObject.Owner.GetVariable("SourceObject").Value;
        
        var recipeId = new FTOptix.RecipeX.RecipeId { Name = recipeName, Version = "1.0" };
        
        var result = schema.TransferFromTargetToStore(
            recipeId,
            targetNode,
            overwrite: true,
            FTOptix.RecipeX.ErrorPolicy.BestEffort);
        
        if (result == FTOptix.RecipeX.TransferFromTargetToStoreResultCode.SuccessRecipeUpdated ||
            result == FTOptix.RecipeX.TransferFromTargetToStoreResultCode.SuccessRecipeCreated)
        {
            Log.Info($"Configuration saved as '{recipeName}'");
        }
    }
}
```

### Listing All Recipes

```csharp
public class RecipeList : BaseNetLogic
{
    [ExportMethod]
    public void ListAllRecipes()
    {
        var schema = (FTOptix.RecipeX.RecipeSchema)LogicObject.Owner.GetObject("RecipeSchema1");
        
        var result = schema.GetRecipes();
        
        if (result.ResultCode == GetRecipesResultCode.Success)
        {
            foreach (var recipe in result.Recipes)
            {
                var metadata = schema.GetRecipeMetadataValues(recipe.FTOptix.RecipeX.RecipeId);
                
                Log.Info($"Recipe: {recipe.FTOptix.RecipeX.RecipeId.Name} v{recipe.FTOptix.RecipeX.RecipeId.Version}");
                Log.Info($"  Created: {recipe.CreatedAt}");
                Log.Info($"  Modified: {recipe.ModifiedAt}");
                
                if (metadata.ResultCode == GetRecipeMetadataValuesResultCode.Success)
                {
                    foreach (var meta in metadata.MetadataValues)
                    {
                        Log.Info($"    {meta.Name}: {meta.Value}");
                    }
                }
            }
        }
    }
}
```

## Practical Examples

### Checking if a Recipe Exists

Verify whether a recipe exists before attempting operations:

```csharp
[ExportMethod]
public bool RecipeExists(string recipeName, string version)
{
    var schema = (FTOptix.RecipeX.RecipeSchema)LogicObject.Owner.GetObject("RecipeSchema1");
    if (schema == null || string.IsNullOrEmpty(recipeName))
        return false;
    
    var result = schema.GetRecipes();
    if (result.ResultCode != GetRecipesResultCode.Success)
        return false;
    
    // Check if recipe matches
    foreach (var recipe in result.Recipes)
    {
        if (recipe.FTOptix.RecipeX.RecipeId.Name == recipeName && 
            recipe.FTOptix.RecipeX.RecipeId.Version == version)
            return true;
    }
    
    return false;
}
```

### Safe Recipe Application with Error Handling

Apply a recipe with comprehensive error handling and logging:

```csharp
[ExportMethod]
public void SafeApplyRecipe(string recipeName, string version)
{
    var schema = (FTOptix.RecipeX.RecipeSchema)LogicObject.Owner.GetObject("RecipeSchema1");
    if (schema == null)
    {
        Log.Error("Recipe schema not found");
        return;
    }
    
    var target = LogicObject.Owner.GetVariable("TargetObject").Value;
    if (target == null)
    {
        Log.Error("Target object not found");
        return;
    }
    
    var recipeId = new FTOptix.RecipeX.RecipeId 
    { 
        Name = recipeName, 
        Version = version 
    };
    
    // Check if recipe exists first
    var recipesResult = schema.GetRecipes();
    bool recipeExists = false;
    
    if (recipesResult.ResultCode == GetRecipesResultCode.Success)
    {
        recipeExists = recipesResult.Recipes.Any(r => 
            r.FTOptix.RecipeX.RecipeId.Name == recipeName && 
            r.FTOptix.RecipeX.RecipeId.Version == version);
    }
    
    if (!recipeExists)
    {
        Log.Error($"Recipe '{recipeName}' v{version} not found");
        return;
    }
    
    // Apply the recipe
    var result = schema.TransferFromStoreToTarget(
        recipeId,
        target,
        FTOptix.RecipeX.ErrorPolicy.Strict);
    
    if (result == FTOptix.RecipeX.TransferFromStoreToTargetResultCode.Success)
    {
        Log.Info($"Recipe '{recipeName}' applied successfully");
    }
    else
    {
        switch (result)
        {
            case FTOptix.RecipeX.TransferFromStoreToTargetResultCode.RecipeNotFound:
                Log.Error("Recipe not found in store");
                break;
            case FTOptix.RecipeX.TransferFromStoreToTargetResultCode.TargetNotFound:
                Log.Error("Target node not found");
                break;
            case FTOptix.RecipeX.TransferFromStoreToTargetResultCode.DataMismatch:
                Log.Error("Recipe structure does not match target");
                break;
            case FTOptix.RecipeX.TransferFromStoreToTargetResultCode.StoreError:
                Log.Error("Error accessing recipe store");
                break;
            default:
                Log.Error($"Failed to apply recipe: {result}");
                break;
        }
    }
}
```

### Create Recipe from Current Target State

Capture current configuration as a recipe:

```csharp
[ExportMethod]
public void CaptureAsRecipe(string recipeName)
{
    var schema = (FTOptix.RecipeX.RecipeSchema)LogicObject.Owner.GetObject("RecipeSchema1");
    var source = LogicObject.Owner.GetVariable("SourceObject").Value;
    
    if (schema == null || source == null)
    {
        Log.Error("Recipe schema or source not found");
        return;
    }
    
    var recipeId = new FTOptix.RecipeX.RecipeId 
    { 
        Name = recipeName, 
        Version = DateTime.UtcNow.ToString("yyyyMMdd_HHmmss") 
    };
    
    // Capture and save
    var result = schema.TransferFromTargetToStore(
        recipeId,
        source,
        overwrite: false,  // Don't overwrite existing
        FTOptix.RecipeX.ErrorPolicy.Strict);
    
    switch (result)
    {
        case FTOptix.RecipeX.TransferFromTargetToStoreResultCode.SuccessRecipeCreated:
            Log.Info($"Recipe '{recipeName}' created and saved");
            // Optionally set metadata
            schema.SetRecipeMetadataValue(recipeId, "CaptureTime", DateTime.UtcNow);
            schema.SetRecipeMetadataValue(recipeId, "CapturedBy", "AutoCapture");
            break;
            
        case FTOptix.RecipeX.TransferFromTargetToStoreResultCode.SuccessRecipeUpdated:
            Log.Info($"Recipe '{recipeName}' updated");
            break;
            
        case FTOptix.RecipeX.TransferFromTargetToStoreResultCode.RecipeAlreadyExists:
            Log.Warning($"Recipe '{recipeName}' already exists");
            break;
            
        case FTOptix.RecipeX.TransferFromTargetToStoreResultCode.DataMismatch:
            Log.Error("Source structure does not match recipe schema");
            break;
            
        default:
            Log.Error($"Failed to capture recipe: {result}");
            break;
    }
}
```

### Batch Duplicate Recipe with Metadata

Duplicate a recipe while updating metadata:

```csharp
[ExportMethod]
public void DuplicateRecipeWithMetadata(string sourceRecipe, string targetRecipe)
{
    var schema = (FTOptix.RecipeX.RecipeSchema)LogicObject.Owner.GetObject("RecipeSchema1");
    if (schema == null)
        return;
    
    // Get source metadata
    var sourceId = new FTOptix.RecipeX.RecipeId { Name = sourceRecipe, Version = "1.0" };
    var metadataResult = schema.GetRecipeMetadataValues(sourceId);
    
    if (metadataResult.ResultCode != GetRecipeMetadataValuesResultCode.Success)
    {
        Log.Error("Failed to retrieve source metadata");
        return;
    }
    
    // Duplicate the recipe
    var targetId = new FTOptix.RecipeX.RecipeId { Name = targetRecipe, Version = "1.0" };
    var dupResult = schema.DuplicateRecipe(sourceId, targetId);
    
    if (dupResult != DuplicateRecipeResultCode.Success)
    {
        Log.Error($"Failed to duplicate recipe: {dupResult}");
        return;
    }
    
    // Copy metadata from source
    foreach (var metadata in metadataResult.MetadataValues)
    {
        schema.SetRecipeMetadataValue(targetId, metadata.Name, metadata.Value);
    }
    
    // Update some fields
    schema.SetRecipeMetadataValue(targetId, "DuplicatedFrom", sourceRecipe);
    schema.SetRecipeMetadataValue(targetId, "DuplicatedAt", DateTime.UtcNow);
    
    Log.Info($"Recipe '{targetRecipe}' created as copy of '{sourceRecipe}'");
}
```

### Interactive Recipe Selection and Application

Create an interactive experience for selecting and applying recipes:

```csharp
[ExportMethod]
public string[] GetAvailableRecipes()
{
    var schema = (FTOptix.RecipeX.RecipeSchema)LogicObject.Owner.GetObject("RecipeSchema1");
    if (schema == null)
        return new string[] { };
    
    var result = schema.GetRecipes();
    if (result.ResultCode != GetRecipesResultCode.Success)
        return new string[] { };
    
    return result.Recipes
        .Select(r => $"{r.FTOptix.RecipeX.RecipeId.Name} (v{r.FTOptix.RecipeX.RecipeId.Version})")
        .ToArray();
}

[ExportMethod]
public void ApplySelectedRecipe(string recipeName, string version)
{
    var schema = (FTOptix.RecipeX.RecipeSchema)LogicObject.Owner.GetObject("RecipeSchema1");
    var target = LogicObject.Owner.GetVariable("TargetObject").Value;
    
    if (schema == null || target == null)
        return;
    
    var recipeId = new FTOptix.RecipeX.RecipeId { Name = recipeName, Version = version };
    var result = schema.TransferFromStoreToTarget(recipeId, target, FTOptix.RecipeX.ErrorPolicy.Strict);
    
    if (result == FTOptix.RecipeX.TransferFromStoreToTargetResultCode.Success)
    {
        Log.Info($"Applied recipe: {recipeName}");
        // Update UI display variable
        LogicObject.Owner.GetVariable("CurrentRecipe").Value = recipeName;
    }
    else
    {
        Log.Error($"Application failed: {result}");
    }
}

[ExportMethod]
public Dictionary<string, object> GetRecipeDetails(string recipeName, string version)
{
    var schema = (FTOptix.RecipeX.RecipeSchema)LogicObject.Owner.GetObject("RecipeSchema1");
    var details = new Dictionary<string, object>();
    
    var recipeId = new FTOptix.RecipeX.RecipeId { Name = recipeName, Version = version };
    
    // Get general recipe info
    var recipesResult = schema.GetRecipes();
    if (recipesResult.ResultCode == GetRecipesResultCode.Success)
    {
        var recipe = recipesResult.Recipes.FirstOrDefault(r => 
            r.FTOptix.RecipeX.RecipeId.Name == recipeName && 
            r.FTOptix.RecipeX.RecipeId.Version == version);
        
        if (recipe != null)
        {
            details["Name"] = recipe.FTOptix.RecipeX.RecipeId.Name;
            details["Version"] = recipe.FTOptix.RecipeX.RecipeId.Version;
            details["CreatedAt"] = recipe.CreatedAt;
            details["ModifiedAt"] = recipe.ModifiedAt;
        }
    }
    
    // Get metadata
    var metaResult = schema.GetRecipeMetadataValues(recipeId);
    if (metaResult.ResultCode == GetRecipeMetadataValuesResultCode.Success)
    {
        var metadata = new Dictionary<string, object>();
        foreach (var meta in metaResult.MetadataValues)
        {
            metadata[meta.Name] = meta.Value;
        }
        details["Metadata"] = metadata;
    }
    
    // Get data items count
    var itemsResult = schema.GetDataItems(recipeId);
    if (itemsResult.ResultCode == GetDataItemsResultCode.Success)
    {
        details["DataItemCount"] = itemsResult.DataItems.Length;
    }
    
    return details;
}
```

### Recipe Validation and Consistency Check

Verify recipe consistency and values:

```csharp
[ExportMethod]
public bool ValidateRecipe(string recipeName, string version)
{
    var schema = (FTOptix.RecipeX.RecipeSchema)LogicObject.Owner.GetObject("RecipeSchema1");
    if (schema == null)
        return false;
    
    var recipeId = new FTOptix.RecipeX.RecipeId { Name = recipeName, Version = version };
    
    // Check recipe exists
    var recipesResult = schema.GetRecipes();
    if (recipesResult.ResultCode != GetRecipesResultCode.Success)
    {
        Log.Error("Failed to retrieve recipes");
        return false;
    }
    
    var recipeExists = recipesResult.Recipes.Any(r => 
        r.FTOptix.RecipeX.RecipeId.Name == recipeName && 
        r.FTOptix.RecipeX.RecipeId.Version == version);
    
    if (!recipeExists)
    {
        Log.Error($"Recipe '{recipeName}' not found");
        return false;
    }
    
    // Validate data items
    var itemsResult = schema.GetDataItems(recipeId);
    if (itemsResult.ResultCode != GetDataItemsResultCode.Success)
    {
        Log.Error("Failed to retrieve data items");
        return false;
    }
    
    if (itemsResult.DataItems.Length == 0)
    {
        Log.Warning("Recipe contains no data items");
    }
    
    // Validate individual values
    bool allValid = true;
    foreach (var item in itemsResult.DataItems)
    {
        var valueResult = schema.GetRecipeDataItemValue(
            recipeId,
            item.ItemRelativeBrowsePath,
            item.DataItemRelativeBrowsePath,
            item.ElementAccess);
        
        if (valueResult.ResultCode != GetRecipeDataItemValueResultCode.Success)
        {
            Log.Warning($"Could not retrieve value for {string.Join("/", item.ItemRelativeBrowsePath)}");
            allValid = false;
        }
    }
    
    // Validate metadata
    var metaResult = schema.GetRecipeMetadataValues(recipeId);
    if (metaResult.ResultCode != GetRecipeMetadataValuesResultCode.Success)
    {
        Log.Warning("Could not retrieve metadata");
        allValid = false;
    }
    
    return allValid;
}
```

### Delete Recipe with Confirmation

This method implements a **two-step confirmation pattern** to prevent accidental deletion. It is designed to be called from a UI button via `ExportMethod`:

1. **First call** (`confirmed = false`, the default): performs a dry run. It looks up the recipe and logs the details of what *would* be deleted, then returns without making any changes. This gives the operator a chance to review before committing.
2. **Second call** (`confirmed = true`): performs the actual deletion.

This pattern is useful in HMI screens where a "Delete" button can first show a confirmation prompt (`confirmed` parameter set to `false`) and only proceed when the operator explicitly confirms the action (`confirmed` parameter set to `true`).

```csharp
[ExportMethod]
public void DeleteRecipeWithConfirmation(string recipeName, string version, bool confirmed)
{
    var schema = (FTOptix.RecipeX.RecipeSchema)LogicObject.Owner.GetObject("RecipeSchema1");
    if (schema == null)
        return;
    
    var recipeId = new FTOptix.RecipeX.RecipeId { Name = recipeName, Version = version };
    
    if (!confirmed)
    {
        // First call: log what would be deleted
        var result = schema.GetRecipes();
        if (result.ResultCode == GetRecipesResultCode.Success)
        {
            var recipe = result.Recipes.FirstOrDefault(r => 
                r.FTOptix.RecipeX.RecipeId.Name == recipeName && 
                r.FTOptix.RecipeX.RecipeId.Version == version);
            
            if (recipe != null)
            {
                Log.Info($"Would delete recipe: {recipe.FTOptix.RecipeX.RecipeId.Name} v{recipe.FTOptix.RecipeX.RecipeId.Version} " +
                         $"(modified: {recipe.ModifiedAt})");
                return;
            }
        }
        
        Log.Warning("Recipe not found");
        return;
    }
    
    // Second call with confirmed=true: actually delete
    var deleteResult = schema.DeleteRecipe(recipeId);
    
    if (deleteResult == DeleteRecipeResultCode.Success)
    {
        Log.Info($"Recipe '{recipeName}' deleted successfully");
    }
    else
    {
        Log.Error($"Failed to delete recipe: {deleteResult}");
    }
}
```

### Edit Recipe (Edit Model Workflow)

Complete workflow for editing a recipe using edit models:

> [!NOTE]
> The Edit Model session is tied to the lifecycle of its **parent node** (`parentNodeId` passed to `CreateEditModel`). If the parent node is deleted, the Edit Model node created underneath it is automatically deleted as well. This means you can scope Edit Model cleanup to session or screen lifecycle by controlling the parent node's lifetime rather than managing the Edit Model node directly.

```csharp
[ExportMethod]
public NodeId StartEditingRecipe(string recipeName, string version)
{
    var schema = (FTOptix.RecipeX.RecipeSchema)LogicObject.Owner.GetObject("RecipeSchema1");
    var parentNode = LogicObject.Owner.GetVariable("EditModelsParent").Value;
    var requester = LogicObject.GetNodeId();
    var target = LogicObject.Owner.GetVariable("TargetObject").Value;
    
    if (schema == null)
        return NodeId.Empty;
    
    var recipeId = new FTOptix.RecipeX.RecipeId { Name = recipeName, Version = version };
    
    // Create edit model
    var createResult = schema.CreateEditModel(parentNode, requester, target, recipeId);
    
    if (createResult.ResultCode != CreateEditModelResultCode.Success)
    {
        Log.Error($"Failed to create edit model: {createResult.ResultCode}");
        return NodeId.Empty;
    }
    
    // Load recipe into edit model
    var transferResult = schema.TransferFromStoreToEditModel(requester, recipeId);
    
    if (transferResult != FTOptix.RecipeX.TransferFromStoreToEditModelResultCode.Success)
    {
        Log.Error($"Failed to load recipe into edit model: {transferResult}");
        return NodeId.Empty;
    }
    
    // Cast to FTOptix.RecipeX.EditModel to access additional members such as PendingChanges
    // and transfer methods that operate directly on this edit model instance.
    var editModel = (FTOptix.RecipeX.EditModel)InformationModel.Get(createResult.EditModelNodeId);
    Log.Info($"Recipe '{recipeName}' loaded for editing. Pending changes: {editModel.PendingChanges}");
    return createResult.EditModelNodeId;
}

[ExportMethod]
public void CancelEditingRecipe(NodeId editModelId)
{
    // Simply delete the edit model
    var editModel = InformationModel.Get(editModelId);
    if (editModel != null)
    {
        editModel.Delete();
        Log.Info("Edit model discarded");
    }
}

[ExportMethod]
public void SaveEditedRecipe(NodeId editModelId, string recipeName, string version, 
    NodeId requesterNodeId)
{
    var schema = (FTOptix.RecipeX.RecipeSchema)LogicObject.Owner.GetObject("RecipeSchema1");
    if (schema == null)
        return;
    
    var recipeId = new FTOptix.RecipeX.RecipeId { Name = recipeName, Version = version };
    
    // Save changes back to store
    var result = schema.TransferFromEditModelToStore(requesterNodeId, recipeId);
    
    if (result == FTOptix.RecipeX.TransferFromEditModelToStoreResultCode.SuccessRecipeUpdated ||
        result == FTOptix.RecipeX.TransferFromEditModelToStoreResultCode.SuccessRecipeCreated)
    {
        Log.Info($"Recipe '{recipeName}' saved successfully");
        
        // Clean up edit model
        var editModel = InformationModel.Get(editModelId);
        if (editModel != null)
            editModel.Delete();
    }
    else
    {
        Log.Error($"Failed to save recipe: {result}");
    }
}

[ExportMethod]
public void ApplyAndClose(NodeId editModelId, NodeId targetNodeId, string recipeName,
    NodeId requesterNodeId, FTOptix.RecipeX.ErrorPolicy errorPolicy = FTOptix.RecipeX.ErrorPolicy.Strict)
{
    var schema = (FTOptix.RecipeX.RecipeSchema)LogicObject.Owner.GetObject("RecipeSchema1");
    if (schema == null)
        return;
    
    // Apply edit model directly to target
    var applyResult = schema.TransferFromEditModelToTarget(requesterNodeId, targetNodeId, errorPolicy);
    
    if (applyResult == FTOptix.RecipeX.TransferFromEditModelToTargetResultCode.Success)
    {
        Log.Info($"Recipe '{recipeName}' applied to target");
        
        // Clean up
        var editModel = InformationModel.Get(editModelId);
        if (editModel != null)
            editModel.Delete();
    }
    else
    {
        Log.Error($"Failed to apply recipe to target: {applyResult}");
    }
}
```

### Accessing an Edit Model Managed by a ListView

When the **Recipes Editor** widget from the TemplateLibrary is used, FactoryTalk Optix automatically creates an `FTOptix.RecipeX.EditModel` to back the `ListView`. This edit model is not exposed as a visible project node, it exists internally and is not directly accessible as children of the `ListView` instance. To programmatically obtain a reference to it, call `CreateEditModel` passing the `ListView.NodeId` as both `parentNodeId` and `requesterNodeId`. Because the edit model already exists for that parent/requester pair, the call returns the existing instance rather than creating a new one.

This pattern is useful when you want to run server-side logic against the values the user is currently editing, for example to validate ranges or cross-field constraints before the recipe is saved.

```csharp
[ExportMethod]
public void ValidateListViewRecipe(NodeId listViewNodeId, out bool isValidRecipe)
{
    isValidRecipe = true;

    var listView = InformationModel.Get<ListView>(listViewNodeId);
    if (listView == null)
    {
        Log.Error("ListView not found");
        isValidRecipe = false;
        return;
    }

    var recipeSchema = Project.Current.Get<FTOptix.RecipeX.RecipeSchema>("Recipes/RecipeSchema1");
    if (recipeSchema == null)
    {
        Log.Error("FTOptix.RecipeX.RecipeSchema not found");
        isValidRecipe = false;
        return;
    }

    // Use get-or-create: if the ListView already owns an edit model this returns it,
    // otherwise a new one is created. Pass NodeId.Empty and an empty FTOptix.RecipeX.RecipeId so the
    // call does not load a different recipe on top of the one already active.
    var createResult = recipeSchema.CreateEditModel(
        listView.NodeId,    // parentNodeId — same as the ListView used when it created the edit model
        listView.NodeId,    // requesterNodeId — same as the ListView used when it created the edit model
        NodeId.Empty,       // targetNodeId — not needed; we are accessing an existing model
        new FTOptix.RecipeX.RecipeId());    // recipeId — empty; the loaded recipe is already in the model

    if (createResult.ResultCode != CreateEditModelResultCode.Success)
    {
        Log.Error($"Failed to access edit model: {createResult.ResultCode}");
        isValidRecipe = false;
        return;
    }

    var editModel = (FTOptix.RecipeX.EditModel)InformationModel.Get(createResult.EditModelNodeId);
    if (editModel == null)
    {
        Log.Error("FTOptix.RecipeX.EditModel not found");
        isValidRecipe = false;
        return;
    }

    // Retrieve the data items defined in the recipe that is currently loaded
    var dataItemsResult = recipeSchema.GetDataItems(editModel.RecipeId);
    if (dataItemsResult.ResultCode != GetDataItemsResultCode.Success)
    {
        Log.Error($"Failed to retrieve data items: {dataItemsResult.ResultCode}");
        isValidRecipe = false;
        return;
    }

    // Iterate every field and apply validation rules
    foreach (var dataItem in dataItemsResult.DataItems)
    {
        var valueResult = recipeSchema.GetRecipeDataItemValue(
            editModel.RecipeId,
            dataItem.ItemRelativeBrowsePath,
            dataItem.DataItemRelativeBrowsePath,
            dataItem.ElementAccess);

        if (valueResult.ResultCode != GetRecipeDataItemValueResultCode.Success)
        {
            Log.Warning($"Could not read '{string.Join("/", dataItem.ItemRelativeBrowsePath)}': {valueResult.ResultCode}");
            isValidRecipe = false;
            continue;
        }

        var value = valueResult.DataItemValue;
        var fieldName = string.Join("/", dataItem.ItemRelativeBrowsePath);

        if (value == null)
        {
            Log.Warning($"Field '{fieldName}' has a null value");
            isValidRecipe = false;
            continue;
        }

        // Add your custom validation logic here.
        // Examples:
        //   if (value is double d && (d < 0 || d > 100)) { ... isValidRecipe = false; }
        //   if (fieldName == "Motor/MaxSpeed" && (int)value > 3000) { ... isValidRecipe = false; }
        Log.Info($"Field '{fieldName}' = {value} — OK");
    }

    Log.Info(isValidRecipe
        ? $"Recipe '{editModel.RecipeId.Name}' passed validation"
        : $"Recipe '{editModel.RecipeId.Name}' failed validation");
}
```

> [!NOTE]
> Do not delete or commit the edit model inside this method. Lifecycle management of the edit model (saving, discarding) belongs to the ListView. This method is read-only with respect to the edit model — it only inspects the current values.

### Validating Edit Model Values Before Save

The example above reads the recipe values from the **store** via `RecipeSchema.GetRecipeDataItemValue`, which does not reflect changes the user has typed into a `ListView` but not yet saved. To validate the **live, unsaved values** currently held by the Edit Model, use `EditModel.GetDataItemValue` instead. This is the recommended check to run just before calling `TransferFromEditModelToStore` or `TransferFromEditModelToTarget`, so invalid input can be rejected before it is persisted or applied.

Key differences from `RecipeSchema.GetRecipeDataItemValue`:
- It is called on the resolved `FTOptix.RecipeX.EditModel` instance, not on the `RecipeSchema`.
- It takes the resolved **item node** (`NodeId`) rather than `ItemRelativeBrowsePath` — resolve it once via `editModel.TargetNodeId` and `IUANode.Get(itemRelativePath)`.
- It returns the value currently in the Edit Model, including any pending, unsaved edits.

```csharp
[ExportMethod]
public void ValidateEditedParameterRange(double minValue, double maxValue, out bool isValidRecipe, out string validationMessage)
{
    isValidRecipe = false;
    validationMessage = string.Empty;

    if (minValue > maxValue)
    {
        SetFailure("Minimum value cannot be greater than maximum value.", ref isValidRecipe, ref validationMessage);
        return;
    }

    var parametersListView = Owner?.Owner?.Owner?.Get<ListView>("RecipeParameters/ParametersListView");
    if (parametersListView == null)
    {
        SetFailure("Recipe parameters ListView was not found.", ref isValidRecipe, ref validationMessage);
        return;
    }

    var recipeSchema = Project.Current.Get<FTOptix.RecipeX.RecipeSchema>("Recipes/RecipeSchema1");
    if (recipeSchema == null)
    {
        SetFailure("Recipe schema not found at 'Recipes/RecipeSchema1'.", ref isValidRecipe, ref validationMessage);
        return;
    }

    // Get-or-create: accesses the edit model already owned by the ListView
    var createResult = recipeSchema.CreateEditModel(
        parametersListView.NodeId,
        parametersListView.NodeId,
        NodeId.Empty,
        new FTOptix.RecipeX.RecipeId { Name = string.Empty, Version = string.Empty });

    if (createResult.ResultCode != FTOptix.RecipeX.CreateEditModelResultCode.Success)
    {
        SetFailure($"Failed to access ListView edit model: {createResult.ResultCode}.", ref isValidRecipe, ref validationMessage);
        return;
    }

    var editModel = InformationModel.Get<FTOptix.RecipeX.EditModel>(createResult.EditModelNodeId);
    var targetNode = InformationModel.Get(editModel.TargetNodeId);
    if (editModel == null || targetNode == null)
    {
        SetFailure("Edit model or its target node could not be resolved.", ref isValidRecipe, ref validationMessage);
        return;
    }

    var dataItemsResult = recipeSchema.GetDataItems(editModel.FTOptix.RecipeX.RecipeId);
    if (dataItemsResult.ResultCode != FTOptix.RecipeX.GetDataItemsResultCode.Success || dataItemsResult.DataItems.Length == 0)
    {
        SetFailure($"Unable to retrieve recipe data items: {dataItemsResult.ResultCode}.", ref isValidRecipe, ref validationMessage);
        return;
    }

    var invalidFields = 0;

    foreach (var dataItem in dataItemsResult.DataItems)
    {
        var itemRelativePath = string.Join("/", dataItem.ItemRelativeBrowsePath ?? Array.Empty<string>());
        var fullPath = string.Join("/", (dataItem.ItemRelativeBrowsePath ?? Array.Empty<string>())
            .Concat(dataItem.DataItemRelativeBrowsePath ?? Array.Empty<string>()));

        // Resolve the item node reached by ItemRelativeBrowsePath from the Edit Model's target
        var itemNodeId = targetNode.NodeId;
        if (!string.IsNullOrWhiteSpace(itemRelativePath))
        {
            var itemNode = targetNode.Get(itemRelativePath);
            if (itemNode == null)
            {
                invalidFields++;
                Log.Warning("RecipeValidation", $"Unable to resolve item node for '{itemRelativePath}'.");
                continue;
            }
            itemNodeId = itemNode.NodeId;
        }

        // Read the current, unsaved value directly from the Edit Model
        var valueResult = editModel.GetDataItemValue(
            itemNodeId,
            dataItem.DataItemRelativeBrowsePath,
            dataItem.ElementAccess,
            dataItem.DataTypeId);

        if (valueResult.ResultCode != FTOptix.RecipeX.GetDataItemValueResultCode.Success || valueResult.DataItemValue == null)
        {
            invalidFields++;
            Log.Warning("RecipeValidation", $"Unable to read edited value for '{fullPath}': {valueResult.ResultCode}");
            continue;
        }

        if (!TryConvertToDouble(valueResult.DataItemValue, out var numericValue) || numericValue < minValue || numericValue > maxValue)
        {
            invalidFields++;
            Log.Warning("RecipeValidation", $"Edited value for '{fullPath}' is out of range [{minValue}, {maxValue}].");
            continue;
        }
    }

    isValidRecipe = invalidFields == 0;
    validationMessage = isValidRecipe
        ? "Validation passed: all fields are within range."
        : $"Validation failed: {invalidFields} of {dataItemsResult.DataItems.Length} fields are invalid or out of range.";
}

private void SetFailure(string message, ref bool isValidRecipe, ref string validationMessage)
{
    isValidRecipe = false;
    validationMessage = message;
    Log.Warning("RecipeValidation", message);
}

// Converts numeric-like UAValue/CLR types to double for range comparison; returns false for non-numeric values
private static bool TryConvertToDouble(object value, out double numericValue)
{
    if (value is UAValue uaValue)
        value = uaValue.Value;

    switch (value)
    {
        case double d: numericValue = d; return true;
        case float f: numericValue = f; return true;
        case int i: numericValue = i; return true;
        case long l: numericValue = l; return true;
        case string str when double.TryParse(str, NumberStyles.Float, CultureInfo.InvariantCulture, out var parsed):
            numericValue = parsed;
            return true;
        default:
            numericValue = 0;
            return false;
    }
}
```

> [!TIP]
> Because `GetDataItemValue` reads directly from the Edit Model, this pattern lets you reject invalid input (e.g. out-of-range values) before calling `TransferFromEditModelToStore`/`TransferFromEditModelToTarget`, preventing bad data from ever reaching the store or the running target.

### Bulk Recipe Operations

Perform operations on multiple recipes:

```csharp
[ExportMethod]
public void RenameAllRecipesWithPrefix(string oldPrefix, string newPrefix)
{
    var schema = (FTOptix.RecipeX.RecipeSchema)LogicObject.Owner.GetObject("RecipeSchema1");
    if (schema == null)
        return;
    
    var recipesResult = schema.GetRecipes();
    if (recipesResult.ResultCode != GetRecipesResultCode.Success)
    {
        Log.Error("Failed to retrieve recipes");
        return;
    }
    
    int renamed = 0;
    foreach (var recipe in recipesResult.Recipes)
    {
        if (recipe.RecipeId.Name.StartsWith(oldPrefix))
        {
            var newName = recipe.RecipeId.Name.Replace(oldPrefix, newPrefix);
            var result = schema.RenameRecipe(recipe.RecipeId, newName);
            
            if (result.ResultCode == RenameRecipeResultCode.Success)
            {
                renamed++;
                Log.Info($"Renamed: {recipe.RecipeId.Name} -> {newName}");
            }
        }
    }
    
    Log.Info($"Renamed {renamed} recipes");
}

[ExportMethod]
public void DeleteRecipesOlderThan(int daysOld)
{
    var schema = (FTOptix.RecipeX.RecipeSchema)LogicObject.Owner.GetObject("RecipeSchema1");
    if (schema == null)
        return;
    
    var recipesResult = schema.GetRecipes();
    if (recipesResult.ResultCode != GetRecipesResultCode.Success)
        return;
    
    var cutoffDate = DateTime.UtcNow.AddDays(-daysOld);
    int deleted = 0;
    
    foreach (var recipe in recipesResult.Recipes)
    {
        if (recipe.ModifiedAt < cutoffDate)
        {
            var result = schema.DeleteRecipe(recipe.FTOptix.RecipeX.RecipeId);
            if (result == DeleteRecipeResultCode.Success)
            {
                deleted++;
                Log.Info($"Deleted old recipe: {recipe.FTOptix.RecipeX.RecipeId.Name}");
            }
        }
    }
    
    Log.Info($"Deleted {deleted} old recipes");
}
```

## Important Considerations

### Schema Compatibility

- Legacy recipe schemas and new RecipeX schemas can coexist in the same project but they have different structures, so they cannot easily interoperate.
- Recipe parameters represent Items or DataItems selected in the FTOptix.RecipeX.RecipeSchema editor.
- Recipe parameters displayed in a ListView are always presented as read-only.

### Error Handling Best Practices

- Always check result codes after operations; they indicate specific error conditions.
- Use `FTOptix.RecipeX.ErrorPolicy.Strict` for critical operations and `FTOptix.RecipeX.ErrorPolicy.BestEffort` for optional operations.
- Log failures with the specific result code for easier diagnosis.
