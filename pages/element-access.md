# ElementAccess

## Overview

**ElementAccess** is a core FactoryTalk Optix mechanism that enables precise navigation and access to values within complex data structures. It allows you to navigate through nested structures, access array elements, and target specific fields within structured data types across the entire platform.

## Understanding ElementAccess Structure

### What is ElementAccess?

ElementAccess is a compound navigation path that combines two navigation strategies:

1. **ArrayIndex** - Direct array subscripts applied at the root level
2. **FieldIndexes** - Hierarchical field navigation through nested structures

Together, they provide a complete path to navigate from a starting node to any value within nested structures and arrays.

```csharp
public class ElementAccessStruct
{
    public uint[] ArrayIndex { get; set; }              // Root-level array indices
    public FieldIndexStruct[] FieldIndexes { get; set; } // Field navigation path
}
```

### ArrayIndex

The **ArrayIndex** property is an array of unsigned integers that specifies array subscripts to apply at the root level before diving into the structure.

**Examples:**

- `ArrayIndex = new uint[] { }` - No array indexing (access root directly)
- `ArrayIndex = new uint[] { 0 }` - Access first element of array
- `ArrayIndex = new uint[] { 0, 1 }` - Multi-dimensional array access (rarely used in recipes)

```csharp
// Access first element of an array of structures
var elementAccess = new ElementAccessStruct
{
    ArrayIndex = new uint[] { 0 },
    FieldIndexes = new FieldIndexStruct[] { }
};
// Equivalent to: targetArray[0]
```

### FieldIndexes

The **FieldIndexes** property is an array of `FieldIndexStruct` instances that define the hierarchical path through nested structures.

```csharp
public class FieldIndexStruct
{
    public uint FieldPos { get; set; }        // Position of field in parent structure
    public uint[] ArrayIndex { get; set; }    // Array indices within this field
}
```

**FieldPos** - Field Position within Structure

Each structure type has its fields in a defined order. `FieldPos` specifies which field to access:
- `FieldPos = 0` - Access first field
- `FieldPos = 1` - Access second field
- `FieldPos = 2` - Access third field
- And so on...

**ArrayIndex within FieldIndexStruct** - Array Access at This Level

If the field itself is an array or multi-dimensional structure, specify array indices:
- `ArrayIndex = new uint[] { }` - No array indexing at this level
- `ArrayIndex = new uint[] { 2 }` - Access third element of field array
- `ArrayIndex = new uint[] { 1, 3 }` - Multi-dimensional access

### Visual Examples

#### Example 1: Simple Nested Structure

```csharp
// Model structure:
Settings (structure)
├── Motor (instance of Motor structure, field 0 of Settings)
│   ├── Speed (field 0 of Motor)
│   └── Torque (field 1 of Motor)
└── Conveyor (instance of Conveyor structure, field 1 of Settings)
    └── Enable (field 0 of Conveyor)

// To access Settings.Motor.Torque:
var elementAccess = new ElementAccessStruct
{
    ArrayIndex = new uint[] { },
    FieldIndexes = new FieldIndexStruct[]
    {
        new FieldIndexStruct { FieldPos = 0, ArrayIndex = new uint[] { } },  // Navigate to Motor (field 0)
        new FieldIndexStruct { FieldPos = 1, ArrayIndex = new uint[] { } }   // Navigate to Torque (field 1 of Motor)
    }
};
```

#### Example 2: Array with Nested Structures

```csharp
// Model structure:
Motors (array of Motor structures)
├── [0] Motor (instance of Motor structure, element 0 of the array)
│   ├── Speed (field 0 of Motor)
│   └── Torque (field 1 of Motor)
├── [1] Motor (instance of Motor structure, element 1 of the array)
│   ├── Speed (field 0 of Motor)
│   └── Torque (field 1 of Motor)
└── ...

// To access Motors[1].Speed:
var elementAccess = new ElementAccessStruct
{
    ArrayIndex = new uint[] { 1 },                    // Access second motor in array
    FieldIndexes = new FieldIndexStruct[]
    {
        new FieldIndexStruct { FieldPos = 0, ArrayIndex = new uint[] { } }  // Navigate to Speed (field 0)
    }
};
```

#### Example 3: Array Within Structure

```csharp
// Model structure:
Settings (structure)
├── Profiles (array of Profile structures, field 0 of Settings)
│   ├── [0] Profile (instance of Profile structure, element 0 of the array)
│   │   ├── Name (field 0 of Profile)
│   │   └── Values (array of scalars, field 1 of Profile)
│   │       ├── [0] = 10
│   │       ├── [1] = 20
│   │       └── [2] = 30
│   └── ...

// To access Settings.Profiles[2].Values[1]:
var elementAccess = new ElementAccessStruct
{
    ArrayIndex = new uint[] { },                      // No root-level array
    FieldIndexes = new FieldIndexStruct[]
    {
        new FieldIndexStruct { FieldPos = 0, ArrayIndex = new uint[] { } },    // Navigate to Profiles (field 0)
        new FieldIndexStruct { FieldPos = 1, ArrayIndex = new uint[] { 2, 1 } } // Profiles[2].Values[1] (field 1)
    }
};
```

#### Example 4: Deeply Nested Structure

```csharp
// Model structure:
System (structure)
├── Configuration (instance of Configuration structure, field 0 of System)
│   └── Industrial (instance of Industrial structure, field 0 of Configuration)
│       └── Machine (instance of Machine structure, field 0 of Industrial)
│           ├── Motor1 (instance of Motor structure, field 0 of Machine)
│           │   └── Speed (field 0 of Motor1)
│           └── Motor2 (instance of Motor structure, field 1 of Machine)
│               └── Speed (field 0 of Motor2)

// To access System.Configuration.Industrial.Machine.Motor1.Speed:
var elementAccess = new ElementAccessStruct
{
    ArrayIndex = new uint[] { },
    FieldIndexes = new FieldIndexStruct[]
    {
        new FieldIndexStruct { FieldPos = 0, ArrayIndex = new uint[] { } },  // Configuration
        new FieldIndexStruct { FieldPos = 0, ArrayIndex = new uint[] { } },  // Industrial
        new FieldIndexStruct { FieldPos = 0, ArrayIndex = new uint[] { } },  // Machine
        new FieldIndexStruct { FieldPos = 0, ArrayIndex = new uint[] { } },  // Motor1
        new FieldIndexStruct { FieldPos = 0, ArrayIndex = new uint[] { } }   // Speed
    }
};
```

## ElementAccess Usage

### Structured Variables

The **Core** module provides the foundation for all ElementAccess operations through variable access methods. The public API enables reading and writing values at specific ElementAccess locations within complex data structures.

**Typical Usage Pattern:**

```csharp
// Access variable
var variable = LogicObject.Owner.GetVariable("MyVariable");

// Create ElementAccess for first element of array
var elementAccess = new ElementAccess(new uint[] { 0 });

// Read value at ElementAccess location
var currentValue = variable.GetValue(elementAccess);

// Write value at ElementAccess location
variable.SetValue(newValue, elementAccess);
```

### DynamicLink

The **DynamicLink** system uses ElementAccess for parent element navigation:

```csharp
public class DynamicLink
{
    // ElementAccess to navigate from child to parent element
    public Core.ElementAccess ParentElementAccess { get; set; }
    
    // Variable storing parent ElementAccess as OPC UA value
    public IUAVariable ParentElementAccessVariable { get; set; }
}

public class DynamicLinkType
{
    // Type definition includes parent ElementAccess
    public Core.ElementAccess ParentElementAccess { get; set; }
}
```

**Use Cases:**

- Resolving parent elements in hierarchical structured variables
- Dynamic binding to parent fields based on computed ElementAccess
- Cross-module variable linking

### RecipesX

The **RecipesX** module (primary focus of this guide) uses ElementAccess for:

```csharp
// RecipeSchema methods with ElementAccess
public GetRecipeDataItemValueResult GetRecipeDataItemValue(
    RecipeId recipeId,
    string[] itemRelativeBrowsePath,
    string[] dataItemRelativeBrowsePath,
    ElementAccessStruct elementAccess)  // <-- ElementAccess parameter

public SetRecipeDataItemValueResultCode SetRecipeDataItemValue(
    RecipeId recipeId,
    string[] itemRelativeBrowsePath,
    string[] dataItemRelativeBrowsePath,
    ElementAccessStruct elementAccess,  // <-- ElementAccess parameter
    object value)
```

**Use Cases:**

- Accessing array elements in recipe data items
- Navigating nested structures in recipe schemas
- Transferring specific fields between recipes


#### RecipesX Methods Using ElementAccess

##### GetRecipeDataItemValue

Retrieve a specific value from a recipe using ElementAccess.

```csharp
public GetRecipeDataItemValueResult GetRecipeDataItemValue(
    RecipeId recipeId,
    string[] itemRelativeBrowsePath,
    string[] dataItemRelativeBrowsePath,
    ElementAccessStruct elementAccess)
```

**Parameters:**

- `elementAccess` - Navigation path to the specific value

**Example:**

```csharp
// Model structure:
//
// Motors (array of Motor structures, recipe item)
// ├── [0] Motor
// │   ├── Speed  (field 0 of Motor, float)    <-- target
// │   └── Torque (field 1 of Motor, float)
// ├── [1] Motor
// │   ├── Speed  (field 0 of Motor, float)
// │   └── Torque (field 1 of Motor, float)
// └── ...
//
// Goal: read Motors[0].Speed

var schema = (RecipeSchema)LogicObject.Owner.GetObject("RecipeSchema1");
var recipeId = new RecipeId { Name = "Config1", Version = "1.0" };

var elementAccess = new ElementAccessStruct
{
    ArrayIndex = new uint[] { 0 },                                          // Motors[0]
    FieldIndexes = new FieldIndexStruct[]
    {
        new FieldIndexStruct { FieldPos = 0, ArrayIndex = new uint[] { } }  // Motor -> Speed
    }
};

var result = schema.GetRecipeDataItemValue(
    recipeId,
    new string[] { "Motors" },
    new string[] { "Speed" },
    elementAccess);

if (result.ResultCode == GetRecipeDataItemValueResultCode.Success)
{
    Log.Info($"Motor 0 speed: {result.DataItemValue}");
}
```

#### SetRecipeDataItemValue

Set a specific value in a recipe using ElementAccess.

```csharp
public SetRecipeDataItemValueResultCode SetRecipeDataItemValue(
    RecipeId recipeId,
    string[] itemRelativeBrowsePath,
    string[] dataItemRelativeBrowsePath,
    ElementAccessStruct elementAccess,
    object value)
```

**Parameters:**

- `elementAccess` - Navigation path to the target location

**Example:**

```csharp
// Model structure:
//
// Motors (array of Motor structures, recipe item)
// ├── [0] Motor
// │   ├── Speed  (field 0 of Motor, float)
// │   └── Torque (field 1 of Motor, float)
// ├── [1] Motor
// │   ├── Speed  (field 0 of Motor, float)    <-- target
// │   └── Torque (field 1 of Motor, float)
// └── ...
//
// Goal: set Motors[1].Speed to 3500

var elementAccess = new ElementAccessStruct
{
    ArrayIndex = new uint[] { 1 },                                          // Motors[1]
    FieldIndexes = new FieldIndexStruct[]
    {
        new FieldIndexStruct { FieldPos = 0, ArrayIndex = new uint[] { } }  // Motor -> Speed
    }
};

var result = schema.SetRecipeDataItemValue(
    recipeId,
    new string[] { "Motors" },
    new string[] { "Speed" },
    elementAccess,
    3500);

if (result == SetRecipeDataItemValueResultCode.Success)
{
    Log.Info("Motor 1 speed updated");
}
```

## Practical Usage Patterns

### Pattern 1: Access Simple Scalar Values

When the data item is a simple scalar (not an array, not nested inside a structure), both `ArrayIndex` and `FieldIndexes` are empty. The `itemRelativeBrowsePath` and `dataItemRelativeBrowsePath` parameters alone identify the value.

```csharp
// Model structure:
//
// Settings (structure, recipe item)
// └── Temperature (scalar float, data item)    <-- target
//
// Goal: read Settings.Temperature

[ExportMethod]
public void ReadScalarValue(string recipeName)
{
    var schema = (RecipeSchema)LogicObject.Owner.GetObject("RecipeSchema1");
    var recipeId = new RecipeId { Name = recipeName, Version = "1.0" };
    
    // Empty ElementAccess: no array indexing, no field navigation needed
    var elementAccess = new ElementAccessStruct
    {
        ArrayIndex = new uint[] { },
        FieldIndexes = new FieldIndexStruct[] { }
    };
    
    var result = schema.GetRecipeDataItemValue(
        recipeId,
        new string[] { "Settings" },
        new string[] { "Temperature" },
        elementAccess);
}
```

### Pattern 2: Access Array Elements

When the data item is an array of scalars, use `ArrayIndex` to select a specific element. No `FieldIndexes` are needed because there is no structure to navigate into.

```csharp
// Model structure:
//
// SensorArray (structure, recipe item)
// └── Reading (array of float scalars, data item)
//     ├── [0] = 23.5
//     ├── [1] = 24.1
//     ├── [2] = 22.8
//     └── ...
//
// Goal: read SensorArray.Reading[elementIndex]

[ExportMethod]
public void ReadArrayElement(string recipeName, uint elementIndex)
{
    var schema = (RecipeSchema)LogicObject.Owner.GetObject("RecipeSchema1");
    var recipeId = new RecipeId { Name = recipeName, Version = "1.0" };
    
    // ArrayIndex selects the element; no FieldIndexes needed for a scalar array
    var elementAccess = new ElementAccessStruct
    {
        ArrayIndex = new uint[] { elementIndex },                // Reading[elementIndex]
        FieldIndexes = new FieldIndexStruct[] { }
    };
    
    var result = schema.GetRecipeDataItemValue(
        recipeId,
        new string[] { "SensorArray" },
        new string[] { "Reading" },
        elementAccess);
    
    if (result.ResultCode == GetRecipeDataItemValueResultCode.Success)
    {
        Log.Info($"Sensor {elementIndex} reading: {result.DataItemValue}");
    }
}
```

### Pattern 3: Access Nested Structure Fields

When a data item is a structure containing other structures, use `FieldIndexes` to navigate level by level to the target field. Each entry in `FieldIndexes` descends one level.

```csharp
// Model structure:
//
// System (recipe item)
// └── Configuration (data item, instance of Configuration structure)
//     ├── MotorSettings (field 0 of Configuration, instance of MotorSettings structure)
//     │   ├── MinSpeed  (field 0 of MotorSettings, float)
//     │   └── MaxSpeed  (field 1 of MotorSettings, float)    <-- target
//     └── ConveyorSettings (field 1 of Configuration, instance of ConveyorSettings structure)
//         └── ...
//
// Goal: read System.Configuration -> MotorSettings.MaxSpeed

[ExportMethod]
public void ReadNestedProperty(string recipeName)
{
    var schema = (RecipeSchema)LogicObject.Owner.GetObject("RecipeSchema1");
    var recipeId = new RecipeId { Name = recipeName, Version = "1.0" };
    
    var elementAccess = new ElementAccessStruct
    {
        ArrayIndex = new uint[] { },
        FieldIndexes = new FieldIndexStruct[]
        {
            new FieldIndexStruct 
            { 
                FieldPos = 0,                      // Configuration -> MotorSettings
                ArrayIndex = new uint[] { }
            },
            new FieldIndexStruct 
            { 
                FieldPos = 1,                      // MotorSettings -> MaxSpeed
                ArrayIndex = new uint[] { }
            }
        }
    };
    
    var result = schema.GetRecipeDataItemValue(
        recipeId,
        new string[] { "System", "Configuration" },
        new string[] { },
        elementAccess);
}
```

### Pattern 4: Access Array of Structures with Nested Fields

When the root data item is an array of structures and the target field is inside a nested structure, combine `ArrayIndex` (to pick the array element) with `FieldIndexes` (to navigate into the nested structure).

```csharp
// Model structure:
//
// Motors (array of Motor structures, recipe item)
// ├── [0] Motor
// │   ├── Name   (field 0 of Motor, string)
// │   └── Config (field 1 of Motor, instance of MotorConfig structure)
// │       ├── MaxRPM   (field 0 of MotorConfig, int)    <-- target
// │       └── MaxTorque (field 1 of MotorConfig, float)
// ├── [1] Motor
// │   └── ...
// └── ...
//
// Goal: read Motors[motorIndex].Config.MaxRPM

[ExportMethod]
public void ReadArrayElementProperty(string recipeName, uint motorIndex)
{
    var schema = (RecipeSchema)LogicObject.Owner.GetObject("RecipeSchema1");
    var recipeId = new RecipeId { Name = recipeName, Version = "1.0" };
    
    var elementAccess = new ElementAccessStruct
    {
        ArrayIndex = new uint[] { motorIndex },                                // Motors[motorIndex]
        FieldIndexes = new FieldIndexStruct[]
        {
            new FieldIndexStruct 
            { 
                FieldPos = 1,                                                  // Motor -> Config
                ArrayIndex = new uint[] { }
            },
            new FieldIndexStruct 
            { 
                FieldPos = 0,                                                  // MotorConfig -> MaxRPM
                ArrayIndex = new uint[] { }
            }
        }
    };
    
    var result = schema.GetRecipeDataItemValue(
        recipeId,
        new string[] { "Motors" },
        new string[] { },
        elementAccess);
    
    if (result.ResultCode == GetRecipeDataItemValueResultCode.Success)
    {
        Log.Info($"Motor {motorIndex} max RPM: {result.DataItemValue}");
    }
}
```

### Pattern 5: Access Elements Within Nested Arrays

When a structure field is itself an array (not the root variable), use the `ArrayIndex` property inside the corresponding `FieldIndexStruct` to select an element at that level.

```csharp
// Model structure:
//
// Configuration (structure, recipe item)
// └── Profiles (array of Profile structures, field 0 of Configuration)
//     ├── [0] Profile
//     │   ├── Name   (field 0 of Profile, string)
//     │   └── Values (array of float scalars, field 1 of Profile)
//     │       ├── [0] = 10.0
//     │       ├── [1] = 20.0
//     │       └── [2] = 30.0
//     ├── [1] Profile
//     │   └── ...
//     └── ...
//
// Goal: read Configuration.Profiles[profileIndex].Values[valueIndex]

[ExportMethod]
public void ReadMultiArrayValue(string recipeName, uint profileIndex, uint valueIndex)
{
    var schema = (RecipeSchema)LogicObject.Owner.GetObject("RecipeSchema1");
    var recipeId = new RecipeId { Name = recipeName, Version = "1.0" };
    
    var elementAccess = new ElementAccessStruct
    {
        ArrayIndex = new uint[] { },
        FieldIndexes = new FieldIndexStruct[]
        {
            new FieldIndexStruct 
            { 
                FieldPos = 0,                            // Configuration -> Profiles
                ArrayIndex = new uint[] { profileIndex }  // Profiles[profileIndex]
            },
            new FieldIndexStruct 
            { 
                FieldPos = 1,                            // Profile -> Values
                ArrayIndex = new uint[] { valueIndex }    // Values[valueIndex]
            }
        }
    };
    
    var result = schema.GetRecipeDataItemValue(
        recipeId,
        new string[] { "Configuration" },
        new string[] { },
        elementAccess);
}
```

### Pattern 6: Batch Read Multiple Array Elements

Loop through array elements to read all values from an array of structures.

```csharp
// Model structure:
//
// Motors (array of Motor structures, recipe item)
// ├── [0] Motor
// │   ├── Speed  (field 0 of Motor, float)    <-- target for each element
// │   └── Torque (field 1 of Motor, float)
// ├── [1] Motor
// │   └── ...
// └── ... (10 elements total)
//
// Goal: read Speed from every motor

[ExportMethod]
public void ReadAllMotorSpeeds(string recipeName)
{
    var schema = (RecipeSchema)LogicObject.Owner.GetObject("RecipeSchema1");
    var recipeId = new RecipeId { Name = recipeName, Version = "1.0" };
    
    for (uint i = 0; i < 10; i++)
    {
        var elementAccess = new ElementAccessStruct
        {
            ArrayIndex = new uint[] { i },                                          // Motors[i]
            FieldIndexes = new FieldIndexStruct[] { }
        };
        
        var result = schema.GetRecipeDataItemValue(
            recipeId,
            new string[] { "Motors" },
            new string[] { "Speed" },
            elementAccess);
        
        if (result.ResultCode == GetRecipeDataItemValueResultCode.Success)
        {
            Log.Info($"Motor {i} speed: {result.DataItemValue}");
        }
    }
}
```

### Pattern 7: Batch Write to Array Elements

Loop through array elements to write the same value to multiple entries.

```csharp
// Model structure (same as Pattern 6):
//
// Motors (array of Motor structures, recipe item)
// ├── [0] Motor
// │   ├── Speed  (field 0 of Motor, float)    <-- target for each element
// │   └── Torque (field 1 of Motor, float)
// ├── [1] Motor
// │   └── ...
// └── ...
//
// Goal: set Speed to newSpeed for motors 0, 1, 2, and 3

[ExportMethod]
public void SetAllMotorSpeeds(string recipeName, uint newSpeed)
{
    var schema = (RecipeSchema)LogicObject.Owner.GetObject("RecipeSchema1");
    var recipeId = new RecipeId { Name = recipeName, Version = "1.0" };
    
    uint[] motorIndices = { 0, 1, 2, 3 };
    
    foreach (var index in motorIndices)
    {
        var elementAccess = new ElementAccessStruct
        {
            ArrayIndex = new uint[] { index },                                      // Motors[index]
            FieldIndexes = new FieldIndexStruct[] { }
        };
        
        var result = schema.SetRecipeDataItemValue(
            recipeId,
            new string[] { "Motors" },
            new string[] { "Speed" },
            elementAccess,
            newSpeed);
        
        if (result == SetRecipeDataItemValueResultCode.Success)
        {
            Log.Info($"Motor {index} speed set to {newSpeed}");
        }
        else
        {
            Log.Error($"Failed to set motor {index} speed: {result}");
        }
    }
}
```

### Pattern 8: Comparing Array Element Values

Read the same field from two different array elements and compare them.

```csharp
// Model structure (same as Patterns 6 and 7):
//
// Motors (array of Motor structures, recipe item)
// ├── [0] Motor
// │   ├── Speed  (field 0 of Motor, float)    <-- compared
// │   └── Torque (field 1 of Motor, float)
// ├── [1] Motor
// │   ├── Speed  (field 0 of Motor, float)    <-- compared
// │   └── Torque (field 1 of Motor, float)
// └── ...
//
// Goal: compare Speed of Motors[motor1] with Motors[motor2]

[ExportMethod]
public bool CompareMotorSettings(string recipeName, uint motor1, uint motor2)
{
    var schema = (RecipeSchema)LogicObject.Owner.GetObject("RecipeSchema1");
    var recipeId = new RecipeId { Name = recipeName, Version = "1.0" };
    
    // Read Speed from Motors[motor1]
    var ea1 = new ElementAccessStruct
    {
        ArrayIndex = new uint[] { motor1 },                                         // Motors[motor1]
        FieldIndexes = new FieldIndexStruct[] { }
    };
    
    var result1 = schema.GetRecipeDataItemValue(
        recipeId,
        new string[] { "Motors" },
        new string[] { "Speed" },
        ea1);
    
    // Read Speed from Motors[motor2]
    var ea2 = new ElementAccessStruct
    {
        ArrayIndex = new uint[] { motor2 },                                         // Motors[motor2]
        FieldIndexes = new FieldIndexStruct[] { }
    };
    
    var result2 = schema.GetRecipeDataItemValue(
        recipeId,
        new string[] { "Motors" },
        new string[] { "Speed" },
        ea2);
    
    if (result1.ResultCode == GetRecipeDataItemValueResultCode.Success &&
        result2.ResultCode == GetRecipeDataItemValueResultCode.Success)
    {
        bool match = Equals(result1.DataItemValue, result2.DataItemValue);
        Log.Info($"Motor {motor1} speed: {result1.DataItemValue}");
        Log.Info($"Motor {motor2} speed: {result2.DataItemValue}");
        Log.Info($"Match: {match}");
        return match;
    }
    
    return false;
}
```

## Common Mistakes and Troubleshooting

### Mistake 1: Using FieldPos as a Global Index

`FieldPos` is always relative to the parent structure at that level, not a global position across the entire model. Each `FieldIndexStruct` in the array navigates one level deeper, and its `FieldPos` refers to the field order within that specific parent structure.

```csharp
// Given this model:
//
// Plant (structure)                        
// ├── Name       (field 0 of Plant, string)
// ├── Location   (field 1 of Plant, string)
// └── Line       (field 2 of Plant, instance of Line structure)
//     ├── Id     (field 0 of Line, int)
//     └── Speed  (field 1 of Line, float)
//
// Goal: access Plant.Line.Speed

// WRONG: Treating FieldPos as a flat, global index across the whole model.
// FieldPos = 2 correctly reaches Line, but FieldPos = 4 does not exist
// inside the Line structure (Line only has fields 0 and 1).
var wrong = new ElementAccessStruct
{
    ArrayIndex = new uint[] { },
    FieldIndexes = new FieldIndexStruct[]
    {
        new FieldIndexStruct { FieldPos = 2, ArrayIndex = new uint[] { } },  // Line (correct)
        new FieldIndexStruct { FieldPos = 4, ArrayIndex = new uint[] { } }   // Nothing at index 4 in Line
    }
};

// CORRECT: Each FieldPos is relative to its parent.
// First entry navigates within Plant -> field 2 is Line.
// Second entry navigates within Line -> field 1 is Speed.
var correct = new ElementAccessStruct
{
    ArrayIndex = new uint[] { },
    FieldIndexes = new FieldIndexStruct[]
    {
        new FieldIndexStruct { FieldPos = 2, ArrayIndex = new uint[] { } },  // Plant -> Line
        new FieldIndexStruct { FieldPos = 1, ArrayIndex = new uint[] { } }   // Line  -> Speed
    }
};
```

### Mistake 2: Putting Array Indexes at the Wrong Level

The root-level `ArrayIndex` property and the `ArrayIndex` inside each `FieldIndexStruct` serve different purposes. The root `ArrayIndex` selects an element when the variable itself is an array. The `ArrayIndex` inside a `FieldIndexStruct` selects an element when a specific field within the structure is an array. Mixing them up accesses the wrong data or fails entirely.

```csharp
// Given this model:
//
// Motors (array of Motor structures)        <-- root variable is an array
// ├── [0] Motor
// │   ├── Name   (field 0 of Motor, string)
// │   └── Speed  (field 1 of Motor, float)
// ├── [1] Motor
// │   ├── Name   (field 0 of Motor, string)
// │   └── Speed  (field 1 of Motor, float)
// └── ...
//
// Goal: access Motors[0].Speed

// WRONG: The root variable (Motors) is the array, so the array index
// belongs in the root-level ArrayIndex. Placing it inside a FieldIndexStruct
// would attempt to index the Name field as if Name were an array.
var wrong = new ElementAccessStruct
{
    ArrayIndex = new uint[] { },
    FieldIndexes = new FieldIndexStruct[]
    {
        new FieldIndexStruct { FieldPos = 0, ArrayIndex = new uint[] { 0 } },  // Tries Name[0], not Motors[0]
        new FieldIndexStruct { FieldPos = 1, ArrayIndex = new uint[] { } }
    }
};

// CORRECT: Use root-level ArrayIndex to select Motors[0],
// then FieldIndexes to navigate within the selected Motor structure.
var correct = new ElementAccessStruct
{
    ArrayIndex = new uint[] { 0 },                                             // Motors[0]
    FieldIndexes = new FieldIndexStruct[]
    {
        new FieldIndexStruct { FieldPos = 1, ArrayIndex = new uint[] { } }     // Motor -> Speed
    }
};
```

### Mistake 3: Skipping Intermediate Structure Levels

Each level of structure nesting requires its own `FieldIndexStruct` entry. If a structure contains another structure that in turn contains the target field, every intermediate structure must appear in the path. Skipping a level lands on the wrong field or produces an error.

```csharp
// Given this model:
//
// Motors (array of Motor structures)
// ├── [0] Motor
// │   ├── Name          (field 0 of Motor, string)
// │   └── Config        (field 1 of Motor, instance of MotorConfig structure)
// │       ├── MaxSpeed   (field 0 of MotorConfig, float)
// │       └── MaxTorque  (field 1 of MotorConfig, float)
// └── ...
//
// Goal: access Motors[0].Config.MaxSpeed

// WRONG: Jumps directly from the Motor level to MaxSpeed, skipping Config.
// FieldPos = 0 inside Motor is Name, not MaxSpeed. The path never enters Config.
var wrong = new ElementAccessStruct
{
    ArrayIndex = new uint[] { 0 },                                             // Motors[0]
    FieldIndexes = new FieldIndexStruct[]
    {
        new FieldIndexStruct { FieldPos = 0, ArrayIndex = new uint[] { } }     // Motor -> Name (not MaxSpeed!)
    }
};

// CORRECT: Navigate each level explicitly.
// First entry enters Config (field 1 of Motor).
// Second entry enters MaxSpeed (field 0 of MotorConfig).
var correct = new ElementAccessStruct
{
    ArrayIndex = new uint[] { 0 },                                             // Motors[0]
    FieldIndexes = new FieldIndexStruct[]
    {
        new FieldIndexStruct { FieldPos = 1, ArrayIndex = new uint[] { } },    // Motor      -> Config
        new FieldIndexStruct { FieldPos = 0, ArrayIndex = new uint[] { } }     // MotorConfig -> MaxSpeed
    }
};
```

## Debugging ElementAccess

### Verify Your Structure

Display the structure to understand field positions:

```csharp
[ExportMethod]
public void PrintRecipeStructure(string recipeName, string version)
{
    var schema = (RecipeSchema)LogicObject.Owner.GetObject("RecipeSchema1");
    var recipeId = new RecipeId { Name = recipeName, Version = version };
    
    var dataItemsResult = schema.GetDataItems(recipeId);
    if (dataItemsResult.ResultCode != GetDataItemsResultCode.Success)
    {
        Log.Error("Failed to get data items");
        return;
    }
    
    Log.Info($"Recipe '{recipeName}' contains {dataItemsResult.DataItems.Length} data items:");
    
    foreach (var item in dataItemsResult.DataItems)
    {
        var itemPath = string.Join("/", item.ItemRelativeBrowsePath);
        var dataPath = string.Join("/", item.DataItemRelativeBrowsePath);
        var arrayIndices = string.Join(",", item.ElementAccess.ArrayIndex);
        var fieldIndices = string.Join(";", item.ElementAccess.FieldIndexes.Select(f => 
            $"Pos:{f.FieldPos}[{string.Join(",", f.ArrayIndex)}]"));
        
        Log.Info($"  Item: {itemPath}");
        Log.Info($"    DataPath: {dataPath}");
        Log.Info($"    ArrayIndex: [{arrayIndices}]");
        Log.Info($"    FieldIndexes: [{fieldIndices}]");
    }
}
```

### Test Single Values

Write a helper to test ElementAccess values:

```csharp
[ExportMethod]
public void TestElementAccess(string recipeName, uint[] arrayIndex, uint[] fieldPositions)
{
    var schema = (RecipeSchema)LogicObject.Owner.GetObject("RecipeSchema1");
    var recipeId = new RecipeId { Name = recipeName, Version = "1.0" };
    
    // Build ElementAccess from inputs
    var fieldIndexes = fieldPositions.Select(pos => 
        new FieldIndexStruct { FieldPos = pos, ArrayIndex = new uint[] { } }
    ).ToArray();
    
    var elementAccess = new ElementAccessStruct
    {
        ArrayIndex = arrayIndex,
        FieldIndexes = fieldIndexes
    };
    
    var result = schema.GetRecipeDataItemValue(
        recipeId,
        new string[] { "Data" },
        new string[] { },
        elementAccess);
    
    if (result.ResultCode == GetRecipeDataItemValueResultCode.Success)
    {
        Log.Info($"Value found: {result.DataItemValue}");
    }
    else
    {
        Log.Error($"Failed to access: {result.ResultCode}");
    }
}
```

## Migration Guide - From Array Index to ElementAccess

Several APIs have been deprecated in favor of ElementAccess-based versions. This section helps you migrate existing code to use the new ElementAccess API.

### DynamicLink: ParentArrayIndex → ParentElementAccess

**Deprecated Properties:**

- `ParentArrayIndex` (uint[])
- `ParentScalarIndex` (uint)
- `ParentArrayIndexVariable` (IUAVariable)

**New Properties:**

- `ParentElementAccess` (ElementAccess)
- `ParentElementAccessVariable` (IUAVariable)

**Migration Example:**

```csharp
// DEPRECATED: Old way using ParentArrayIndex
var dynamicLink = (DynamicLink)LogicObject.Owner.GetObject("MyDynamicLink");
uint[] oldArrayIndex = dynamicLink.ParentArrayIndex;
dynamicLink.ParentArrayIndex = new uint[] { 0, 1 };

// Or set a single value
dynamicLink.ParentScalarIndex = 2;

// Access the variable
var oldVariable = dynamicLink.ParentArrayIndexVariable;

// NEW: Use ParentElementAccess instead
var dynamicLink = (DynamicLink)LogicObject.Owner.GetObject("MyDynamicLink");

// Read ParentElementAccess
var elementAccess = dynamicLink.ParentElementAccess;
var arrayIndex = elementAccess.ArrayIndex;  // Get array portion

// Set ParentElementAccess with field navigation
dynamicLink.ParentElementAccess = new ElementAccess(new uint[] { 0, 1 }, new FieldIndex[] { });

// Set ParentElementAccess with single array index
dynamicLink.ParentElementAccess = new ElementAccess(new uint[] { 2 });

// Set with nested field navigation
dynamicLink.ParentElementAccess = new ElementAccess(
    new uint[] { },
    new[] {
        new FieldIndex { FieldPos = 0, ArrayIndex = new uint[] { } },
        new FieldIndex { FieldPos = 1, ArrayIndex = new uint[] { 2 } }
    });

// Access the variable
var newVariable = dynamicLink.ParentElementAccessVariable;
```

**Why This Changed:**

- **ParentArrayIndex was limited** - Only supported array indices, not nested field navigation
- **ElementAccess is comprehensive** - Supports both array indices and field navigation in a single structure
- **Consistency** - ElementAccess is the standard across all FactoryTalk Optix modules

**Specific Migration Paths:**

| Old Code | New Code | Notes |
|----------|----------|-------|
| `link.ParentArrayIndex` | `link.ParentElementAccess.ArrayIndex` | Read array index |
| `link.ParentArrayIndex = new uint[] { 0 }` | `link.ParentElementAccess = new ElementAccess(new uint[] { 0 })` | Set array index only |
| `link.ParentScalarIndex = 5` | `link.ParentElementAccess = new ElementAccess(new uint[] { 5 })` | Set single array index |
| `link.ParentArrayIndexVariable` | `link.ParentElementAccessVariable` | Access underlying variable |

### Event Handlers: uint[] Indexes to ElementAccess

**Deprecated Interface:**

```csharp
// DEPRECATED: IVariableChangeObserver with uint[] indexes
public interface IVariableChangeObserver : IEventObserver
{
    [Obsolete("Use OnVariableChanged with ElementAccess parameter instead.")]
    void OnVariableChanged(IUAVariable variable, UAValue newValue, UAValue oldValue, 
                          uint[] indexes,  // <-- Array index only
                          ulong senderId) { }
}
```

**New Interface:**

```csharp
// NEW: IVariableChangeObserver with ElementAccess
public interface IVariableChangeObserver : IEventObserver
{
    void OnVariableChanged(IUAVariable variable, UAValue newValue, UAValue oldValue, 
                          ElementAccess elementAccess,  // <-- Full ElementAccess
                          ulong senderId);
}
```

**Migration Example:**

```csharp
// DEPRECATED: Old handler using uint[] indexes
public class OldVariableMonitor : IVariableChangeObserver
{
    public void OnVariableChanged(IUAVariable variable, UAValue newValue, UAValue oldValue, 
                                  uint[] indexes, ulong senderId)
    {
        if (indexes.Length > 0)
        {
            Log.Info($"Array element {indexes[0]} changed");
        }
    }
}

// NEW: Handler using ElementAccess
public class NewVariableMonitor : IVariableChangeObserver
{
    public void OnVariableChanged(IUAVariable variable, UAValue newValue, UAValue oldValue, 
                                  ElementAccess elementAccess, ulong senderId)
    {
        var arrayIndex = elementAccess.ArrayIndex;
        var fieldIndexes = elementAccess.FieldIndexes;
        
        if (arrayIndex.Length > 0)
        {
            Log.Info($"Array element {arrayIndex[0]} changed");
        }
        
        if (fieldIndexes.Length > 0)
        {
            Log.Info($"Field at position {fieldIndexes[0].FieldPos} changed");
        }
    }
}
```

### Callback Observer: Constructor Change

**Deprecated Constructor:**

```csharp
// DEPRECATED: CallbackVariableChangeObserver with uint[] action
var observer = new CallbackVariableChangeObserver(
    (variable, newValue, oldValue, indexes, senderId) => {
        Log.Info($"Index: {indexes}");
    }
);
```

**New Constructor:**

```csharp
// NEW: CallbackVariableChangeObserver with ElementAccess action
var observer = new CallbackVariableChangeObserver(
    (variable, newValue, oldValue, elementAccess, senderId) => {
        Log.Info($"ArrayIndex: {elementAccess.ArrayIndex}");
        Log.Info($"FieldIndexes: {elementAccess.FieldIndexes}");
    }
);
```

**Migration Example:**

```csharp
// OLD: Register callback with array indices
LogicObject.Owner.Owner.GetVariable("MyVariable").Subscribe(
    new CallbackVariableChangeObserver(
        (variable, newValue, oldValue, indexes, senderId) => {
            var index = indexes.Length > 0 ? indexes[0] : 0;
            UpdateUI(index, newValue);
        }
    )
);

// NEW: Register callback with ElementAccess
LogicObject.Owner.Owner.GetVariable("MyVariable").Subscribe(
    new CallbackVariableChangeObserver(
        (variable, newValue, oldValue, elementAccess, senderId) => {
            var index = elementAccess.ArrayIndex.Length > 0 ? elementAccess.ArrayIndex[0] : 0;
            var fieldInfo = elementAccess.FieldIndexes.Length > 0 ? elementAccess.FieldIndexes[0] : null;
            UpdateUI(index, newValue, fieldInfo);
        }
    )
);
```

### Event Args Properties: Indexes to ElementAccess

**Deprecated Properties:**

```csharp
// DEPRECATED properties in event argument classes
public class VariableChangeEventArgs
{
    [Obsolete("Use ElementAccess.ArrayIndex instead.")]
    public uint[] Indexes => ElementAccess.ArrayIndex;
    
    public ElementAccess ElementAccess { get; }
}

public class PathResolverResult
{
    [Obsolete("Use ElementAccess.ArrayIndex instead.")]
    public uint[] Indexes => ElementAccess.ArrayIndex;
    
    public ElementAccess ElementAccess { get; }
}
```

**Migration Example:**

```csharp
// OLD: Access through Indexes property
var indexes = eventArgs.Indexes;
var elementIndex = indexes.Length > 0 ? indexes[0] : 0;

// OR in path resolver
var indexes = resolverResult.Indexes;
if (indexes != null)
{
    Log.Info($"Array index: {indexes[0]}");
}

// NEW: Access through ElementAccess
var elementAccess = eventArgs.ElementAccess;
var arrayIndex = elementAccess.ArrayIndex;
var elementIndex = arrayIndex.Length > 0 ? arrayIndex[0] : 0;

// OR in path resolver
var elementAccess = resolverResult.ElementAccess;
if (elementAccess.ArrayIndex.Length > 0)
{
    Log.Info($"Array index: {elementAccess.ArrayIndex[0]}");
}

// Access field navigation too (not available in old Indexes property)
if (elementAccess.FieldIndexes.Length > 0)
{
    Log.Info($"Field position: {elementAccess.FieldIndexes[0].FieldPos}");
}
```
