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
    // A big dictionary may take a long processing time, usage of a LongRunningTask is recommended
    if (dictionary == null || dictionary.Value.Value == null || valuesToAdd == null || valuesToAdd.Count <= 0)
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
    List<string[]> valuesVerifiedToAdd = new List<string[]>();
    foreach (string[] valueToAdd in valuesToAdd)
    {
        if (valueToAdd == null)
            continue;
        if (valueToAdd.Length != actualDictionaryValues.GetLength(1))
        {
            Log.Warning(LogicObject.BrowseName, "The count of elements to be added is different from the number of columns in the dictionary, skipped");
            continue;
        }
        if (string.IsNullOrEmpty(valueToAdd[0]))
        {
            Log.Warning(LogicObject.BrowseName, "Missing key, skipped");
            continue;
        }
        string checkIfKeyExist = Enumerable
       .Range(0, actualDictionaryValues.GetLength(0))
       .Where(x => actualDictionaryValues[x, 0] == valueToAdd[0])
       .Select(y => actualDictionaryValues[y, 0])
       .FirstOrDefault();
        if (!string.IsNullOrEmpty(checkIfKeyExist))
        {
            Log.Warning(LogicObject.BrowseName, $"Key {valueToAdd[0]} already exist, cannot add to dictionary, use Edit method");
            continue;
        }
        valuesVerifiedToAdd.Add(valueToAdd);
    }        
    string[,] newDictionaryValues = new string[actualDictionaryValues.GetLength(0) + valuesVerifiedToAdd.Count, actualDictionaryValues.GetLength(1)];
    for (int i = 0; i< actualDictionaryValues.GetLength(0); i++) 
    {
        for (int j = 0; j < actualDictionaryValues.GetLength(1);j++)
        {
            newDictionaryValues[i,j] = actualDictionaryValues[i,j];
        }
    }
    int rowOffset = 0;
    foreach (string[] valueToAdd in valuesVerifiedToAdd)
    {
        for (int i = 0; i < valueToAdd.Length; i++)
        {
            int rowIndex = actualDictionaryValues.GetLength(0) + rowOffset;
            newDictionaryValues[rowIndex, i] = valueToAdd[i];
        }
        rowOffset++;
    }
    dictionary.Value = new UAValue(newDictionaryValues);
}
```

### Modify content of a dictionary

This example provide a collection of array to edit multiple records into the dictionary. Into a single array you can only write the language value to modify (take in consideration to place in correct order) and leave other field empty.

```csharp
private void StuffModifyDictionaryTranslations(IUAVariable dictionary, List<string[]> valuesToEdit)
{
    // A big dictionary may take a long processing time, usage of a LongRunningTask is recommended
    if (dictionary == null || dictionary.Value.Value == null || valuesToEdit == null || valuesToEdit.Count <= 0)
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
    foreach (string[] valueToEdit in  valuesToEdit) 
    {
        if (valueToEdit == null)
            continue;
        if (valueToEdit.Length != actualDictionaryValues.GetLength(1))
        {
            Log.Warning(LogicObject.BrowseName, "The count of elements to be edit is different from the number of columns in the dictionary, skipped");
            continue;
        }
        if (string.IsNullOrEmpty(valueToEdit[0]))
        {
            Log.Warning(LogicObject.BrowseName, "Missing key, skipped");
            continue;
        }
        int rowNumberToEdit = Enumerable
       .Range(0, actualDictionaryValues.GetLength(0))
       .Where(x => actualDictionaryValues[x, 0] == valueToEdit[0])
       .FirstOrDefault(-1);
        if (rowNumberToEdit == -1)
        {
            Log.Warning(LogicObject.BrowseName, $"Key {valueToEdit[0]} not exist, cannot edit the dictionary, use Add method");
            continue;
        }
        for (int i = 0; i < valueToEdit.Length; i++) 
        {
            if (!string.IsNullOrEmpty(valueToEdit[i]))
                actualDictionaryValues[rowNumberToEdit,i] = valueToEdit[i];
        }
    }
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
