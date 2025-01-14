# Translations

## Assign a translation key to a Text property

Please note: `myKey` should already exist in the TranslationDictionary

```csharp
// Create a new label
Label myLabel = InformationModel.Make<Label>("MyLabelName");
// Create a new LocalizedText with a specific Key (that should exist in the LocalizationDictionary)
myLabel.LocalizedText = new LocalizedText(myLabel.NodeId.NamespaceIndex, "myKey1");
// Add the label to the project
Owner.Add(myLabel);
```

## Read the translation key of a Text property

```csharp
// Get the LocalizedText variable from the Label
string myKey = myLabel.LocalizedText.TextId
```

```csharp
// Alternative way (same result)
string myKey = ((LocalizedText)myVariable.Value).TextId
```

## Read the translation content for a specific key

```csharp
// Create a new temporary localized text to read the key value from the dictionary
LocalizedText alarmKey = new LocalizedText(LogicObject.NodeId.NamespaceIndex, "Alarm_TL_Key");
// Get the text value from the current language
string alarmMessage = InformationModel.LookupTranslation(alarmKey).Text;
```

## Adding a description to a variable

```csharp
// Set the description of an object to a localized text
variable.Description = new LocalizedText(Project.Current.NodeId.NamespaceIndex, "descriptionTextId");
```

## Create a dictionary

Dictionary is a variable with dataType OpcUa.String and variableType FTOptix.Core.LocalizationDictionary, values are stored in a bidimensional array of string

```csharp
private void StuffMakeNewDictionary(string browseName, string[]languages, IUANode dictionaryOwner) 
{
    if (dictionaryOwner == null || languages.Length <= 0 || string.IsNullOrEmpty(browseName))
        return;
    if (languages[0] != "")
    {
        // The first column needs to be empty as it's the keys list
        string[] firstColumnEmpty = new string[1] {""};
        // Add the languages to the other columns
        languages = firstColumnEmpty.Concat(languages).ToArray();
    }
    List<string> languageVerifiedToAdd = new List<string>();
    foreach (string language in languages) 
    {
        try
        {
            if (CultureInfo.GetCultureInfo(language, false) != null)
                languageVerifiedToAdd.Add(language);
        }
        catch 
        {  
            // In case of culture identifier is not valid, skip to add to dictionary          
        }
    }
    string[,] newDictionaryValues = new string[1, languageVerifiedToAdd.Count];
    for (int i = 0; i < languageVerifiedToAdd.Count; i++) 
    {
        newDictionaryValues[0,i] = languageVerifiedToAdd[i];
    }
    // Create the new dictionary
    IUAVariable newDictionary = InformationModel.MakeVariable(
        browseName, 
        OpcUa.DataTypes.String, 
        FTOptix.Core.VariableTypes.LocalizationDictionary, 
        new uint[2] { 
            (uint)newDictionaryValues.GetLength(0), 
            (uint)newDictionaryValues.GetLength(1) 
        });
    // Set the dictionary content
    newDictionary.Value = new UAValue(newDictionaryValues);
    // Check if the dictionary does not exist already, then add it
    if (dictionaryOwner.Get(browseName) == null) 
        dictionaryOwner.Add(newDictionary);
}
```

## Add/Modify/Remove translation from a dictionary

Dictionary values are stored in a bidimensional array of string, you need to handle this rectangular array and regenerate the UAValue to store into the dictionary.

### Add elements to the dictionary

This example provide a collection of array to add multiple records into the dictionary

```csharp
private void StuffAddDictionaryTranslations(IUAVariable dictionary, List<string[]> valuesToAdd)
{
    // A big dictionary may take a long processing time, so using a LongRunningTask is recommended to avoid blocking the main thread.
    
    // Validate input: if any of the inputs are null or empty, exit early.
    if (dictionary == null || dictionary.Value.Value == null || valuesToAdd == null || valuesToAdd.Count <= 0)
        return;

    string[,] actualDictionaryValues;
    try
    {
        // Attempt to cast the dictionary's value to a 2D string array (dictionary format).
        actualDictionaryValues = dictionary.Value.Value as string[,];
    }
    catch
    {
        // If the dictionary value is not a valid 2D array, stop further execution.
        return;
    }

    // List to store only verified values that can be safely added to the dictionary.
    List<string[]> valuesVerifiedToAdd = new List<string[]>();

    // Iterate through each value set to be added to the dictionary.
    foreach (string[] valueToAdd in valuesToAdd)
    {
        if (valueToAdd == null)
            continue; // Skip null entries.

        // Check if the number of elements in the value matches the number of columns in the dictionary.
        if (valueToAdd.Length != actualDictionaryValues.GetLength(1))
        {
            Log.Warning(LogicObject.BrowseName, "The count of elements to be added is different from the number of columns in the dictionary, skipped");
            continue;
        }

        // Validate that the first column (key) is not null or empty.
        if (string.IsNullOrEmpty(valueToAdd[0]))
        {
            Log.Warning(LogicObject.BrowseName, "Missing key, skipped");
            continue;
        }

        // Check if the key already exists in the current dictionary.
        string checkIfKeyExist = Enumerable
            .Range(0, actualDictionaryValues.GetLength(0)) // Generate row indices.
            .Where(x => actualDictionaryValues[x, 0] == valueToAdd[0]) // Check if the key matches the first column.
            .Select(y => actualDictionaryValues[y, 0]) // Get the matching key.
            .FirstOrDefault(); // Return the first match or null.

        if (!string.IsNullOrEmpty(checkIfKeyExist))
        {
            Log.Warning(LogicObject.BrowseName, $"Key {valueToAdd[0]} already exists, cannot add to dictionary, use Edit method");
            continue;
        }

        // Add the value to the list of verified values for addition.
        valuesVerifiedToAdd.Add(valueToAdd);
    }

    // Create a new dictionary with additional rows for the new entries.
    string[,] newDictionaryValues = new string[actualDictionaryValues.GetLength(0) + valuesVerifiedToAdd.Count, actualDictionaryValues.GetLength(1)];

    // Copy the existing dictionary values to the new dictionary.
    for (int i = 0; i < actualDictionaryValues.GetLength(0); i++)
    {
        for (int j = 0; j < actualDictionaryValues.GetLength(1); j++)
        {
            newDictionaryValues[i, j] = actualDictionaryValues[i, j];
        }
    }

    // Add the verified new values to the new dictionary starting from the next available row.
    int rowOffset = 0;
    foreach (string[] valueToAdd in valuesVerifiedToAdd)
    {
        for (int i = 0; i < valueToAdd.Length; i++)
        {
            int rowIndex = actualDictionaryValues.GetLength(0) + rowOffset; // Calculate the row index for the new entry.
            newDictionaryValues[rowIndex, i] = valueToAdd[i]; // Add the value to the corresponding position.
        }
        rowOffset++; // Move to the next row for subsequent entries.
    }

    // Update the dictionary's value with the new dictionary containing the added values.
    dictionary.Value = new UAValue(newDictionaryValues);
}

```

### Modify content of a dictionary

This example provide a collection of array to edit multiple records into the dictionary. Into a single array you can only write the language value to modify (take in consideration to place in correct order) and leave other field empty.

```csharp
private void StuffModifyDictionaryTranslations(IUAVariable dictionary, List<string[]> valuesToEdit)
{
    // A big dictionary may take a long time to process, so using a LongRunningTask is recommended to avoid blocking execution.
    
    // Validate inputs: check if dictionary or its value is null, or if there are no values to edit.
    if (dictionary == null || dictionary.Value.Value == null || valuesToEdit == null || valuesToEdit.Count <= 0)
        return;

    string[,] actualDictionaryValues;
    try
    {
        // Attempt to cast the dictionary's value to a two-dimensional string array.
        actualDictionaryValues = dictionary.Value.Value as string[,];
    }
    catch
    {
        // If the value cannot be cast, it may not be a dictionary, so stop further execution.
        return;
    }

    // Iterate over each entry in the valuesToEdit list.
    foreach (string[] valueToEdit in valuesToEdit)
    {
        // Skip null entries.
        if (valueToEdit == null)
            continue;

        // Check if the entry length matches the number of dictionary columns.
        if (valueToEdit.Length != actualDictionaryValues.GetLength(1))
        {
            Log.Warning(LogicObject.BrowseName, "The count of elements to be edited is different from the number of columns in the dictionary, skipped");
            continue;
        }

        // Ensure the key (first element) is not null or empty.
        if (string.IsNullOrEmpty(valueToEdit[0]))
        {
            Log.Warning(LogicObject.BrowseName, "Missing key, skipped");
            continue;
        }

        // Find the row in the dictionary corresponding to the key.
        int rowNumberToEdit = Enumerable
            .Range(0, actualDictionaryValues.GetLength(0)) // Iterate over all rows.
            .Where(x => actualDictionaryValues[x, 0] == valueToEdit[0]) // Check if the first column matches the key.
            .FirstOrDefault(-1); // Default to -1 if no match is found.

        // If the key does not exist in the dictionary, log a warning and skip the entry.
        if (rowNumberToEdit == -1)
        {
            Log.Warning(LogicObject.BrowseName, $"Key {valueToEdit[0]} does not exist, cannot edit the dictionary, use Add method");
            continue;
        }

        // Update the corresponding row with the new values.
        for (int i = 0; i < valueToEdit.Length; i++)
        {
            // Only update non-empty values to avoid overwriting existing data with empty values.
            if (!string.IsNullOrEmpty(valueToEdit[i]))
                actualDictionaryValues[rowNumberToEdit, i] = valueToEdit[i];
        }
    }

    // Update the dictionary with the modified values.
    dictionary.Value = new UAValue(actualDictionaryValues);
}

```

### Remove

This example provide a list of keys to delete multiple records into the dictionary.

```csharp
private void StuffRemoveDictionaryTranslations(IUAVariable dictionary, List<string> keysToRemove)
{
    // A big dictionary may take a long processing time, usage of a LongRunningTask is recommended
    if (dictionary == null || dictionary.Value.Value == null || keysToRemove == null || keysToRemove.Count <= 0)
        return;
    string[,] actualDictionaryValues;
    try
    {
        actualDictionaryValues = dictionary.Value.Value as string[,];
    }
    catch
    {
        //Maybe is not a dictionary, so is better to stop execution here
        return;
    }
    List<int> rowsToRemove = new List<int>();
    const int defaultValue = 0;
    foreach(string keyToRemove in keysToRemove) 
    {
        if (string.IsNullOrEmpty(keyToRemove))
        {
            Log.Warning(LogicObject.BrowseName, "Missing key, skipped");
            continue;
        }
        int rowToDelete = Enumerable
       .Range(0, actualDictionaryValues.GetLength(0))
       .Where(x => actualDictionaryValues[x, 0] == keyToRemove)
       .FirstOrDefault(defaultValue);
        if (rowToDelete == defaultValue)
        {
            Log.Warning(LogicObject.BrowseName, $"Key {keyToRemove} not exist, cannot edit the dictionary, use Add method");
            continue;
        }
        rowsToRemove.Add(rowToDelete);
    }
    if (rowsToRemove.Count >= actualDictionaryValues.GetLength(0))
    {
        Log.Error(LogicObject.BrowseName, "The rows to be removed are equal to or more than the dictionary entries with header, impossible to continue");
        return;
    }
    string[,] newDictionaryValues = new string[actualDictionaryValues.GetLength(0) - rowsToRemove.Count, actualDictionaryValues.GetLength(1)];
    int rowToWrite = 0;
    for (int i = 0; i < actualDictionaryValues.GetLength(0); i++) 
    {
        if (!rowsToRemove.Contains(i))
        {
            for (int j = 0; j < actualDictionaryValues.GetLength(1); j++)
            {
                newDictionaryValues[rowToWrite, j] = actualDictionaryValues[i, j];
            }
            rowToWrite++;
        }
    }
    dictionary.Value = new UAValue(newDictionaryValues);
}
```

### Example of previous code snippets

You can build a Design NetLogic with all previous methods to create and populate a Dictionary

```csharp
[ExportMethod]
public void Method1()
{
    // Get to the translations folder
    Folder translation = Project.Current.Get<Folder>("Translations");
    // Create the new dictionary object
    StuffMakeNewDictionary("MyNewDictionary", new string[6] { "", "it-IT", "en-US", "fr-FR", "es-ES","de-DE" }, translation);
    // Set some content to the dictionary
    List<string[]> fillThisDict = new List<string[]>
    {
        new string[6] { "keyHello1", "Ciao", "Hello", "Bonjour", "Hola", "Hallo" },
        new string[6] { "keyHello", "Ciao", "Hello", "Bonjour", "Hola", "Hallo" },
        new string[6] { "keyStart2", "Avviare!", "Start!", "Démarrer!", "Inicio!","Start!" },
        new string[6] { "keyStop2", "Fermare", "Stop", "Arrêter", "Para", "Stopp" },
        new string[6] { "keyStart", "Avviare", "Start", "Démarrer", "Inicio", "Start" },
        new string[6] { "keyStop", "Fermare", "Stop", "Arrêter", "Para", "Stopp" },
        new string[6] { "keyStop3", "Fermare", "Stop", "Arrêter", "Para", "Stopp" }
    };
    StuffAddDictionaryTranslations(translation.GetVariable("MyNewDictionary"), fillThisDict);
    // Add even more lines to the dictionary
    List<string[]> editThisDict = new List<string[]>
    {
        new string[6] { "keyHello", "Ciao!", "Hello!", "Bonjour!", "Hola!", "Hallo!" },
        new string[6] { "keyStop", "Fermati!", "Stop", "Arrêter", "Para", "Stopp" },
        new string[6] { "keyStart2", "Avviare", "Start", "Démarrer", "Inicio", "Start" }
    };
    StuffModifyDictionaryTranslations(translation.GetVariable("MyNewDictionary"), editThisDict);
    // Remove some unused keys
    List<string> removeToThisDict = new List<string>
    {
        "keyHello1",
        "keyStart2",
        "keyStop2",
        "keyStop3"
    };
    StuffRemoveDictionaryTranslations(translation.GetVariable("MyNewDictionary"), removeToThisDict);
}
```
