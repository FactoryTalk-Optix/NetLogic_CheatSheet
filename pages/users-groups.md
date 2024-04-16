# Users and groups

## Creating a new Group

```csharp
[ExportMethod]
public void CreateNewGroup(string groupName) {
    // Get to the base folder to create groups
    Folder userFolder = Project.Current.Get<Folder>("Security/Groups");
    if (userFolder == null) {
        Log.Error("CreateNewGroup", "Cannot find Groups folder");
        return;
    } else if (string.IsNullOrEmpty(groupName)) {
        Log.Error("CreateNewGroup", "Cannot create group with empty name");
        return;
    } else if (userFolder.Get<Group>(groupName) != null) {
        Log.Error("CreateNewGroup", "Group already exists!");
        return;
    }
    // Create a new group
    var newGroup = InformationModel.Make<Group>(groupName);
    // Add the group to the folder
    userFolder.Add(newGroup);
}
```