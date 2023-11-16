# Translations

## Assign a translation key to a Text property

Please note: `myKey` should already exist in the TranslationDictionary

```csharp
Label myLabel = InformationModel.Make<Label>("MyLabelName");
myLabel.LocalizedText = new LocalizedText(myLabel.NodeId.NamespaceIndex, "myKey1");
Owner.Add(myLabel);
```

## Read the translation key of a Text property

```csharp
string myKey = myLabel.LocalizedText.TextId
string myKey = ((LocalizedText)myVariable.Value).TextId
```

## Read the translation content for a specific key

```csharp
LocalizedText alarmKey = new LocalizedText(LogicObject.NodeId.NamespaceIndex, "Alarm_TL_Key");
string alarmMessage = InformationModel.LookupTranslation(alarmKey).Text;
```

## Adding a description to a variable

```csharp
variable.Description = new LocalizedText(Project.Current.NodeId.NamespaceIndex, "descriptionTextId");
```
