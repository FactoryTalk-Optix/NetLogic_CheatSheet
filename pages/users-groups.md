# Users and groups

## Groups

### Creating a new Group

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

### Check if user is part of a group

```csharp
// Method to check if a user is part of a specific group
private static bool UserHasGroup(FTOptix.Core.User user, FTOptix.Core.Group group)
{
    // Return false if user is null
    if (user == null)
        return false;
    
    // Return false if group is null
    if (group == null)
        return false;

    // Get all groups the user is part of
    var userGroups = user.Refs.GetObjects(FTOptix.Core.ReferenceTypes.HasGroup, false);

    // Iterate through the groups to check if the user is part of the specified group
    foreach (var userGroup in userGroups)
    {
        if (userGroup.NodeId == group.NodeId)
            return true;
    }
    // Return false if the user is not part of the specified group
    return false;
}
```

### Assign group to a user

```csharp
/// <summary>
/// Assigns a user to a specified group by adding a reference between the user and the group.
/// </summary>
/// <param name="user">The user to be assigned to the group.</param>
/// <param name="group">The group to which the user will be assigned.</param>
private void AssignUserToGroup(FTOptix.Core.User user, FTOptix.Core.Group group)
{
    user.Refs.AddReference(FTOptix.Core.ReferenceTypes.HasGroup, group.NodeId);
}
```

### Remove group from a user

```csharp
/// <summary>
/// Removes a user from the specified group by removing a reference between the user and the group.
/// </summary>
/// <param name="user">The user to be removed from the group.</param>
/// <param name="group">The group to which the user will be removed from.</param>
private void RemoveUserFromGroup(FTOptix.Core.User user, FTOptix.Core.Group group)
{
    user.Refs.RemoveReference(FTOptix.Core.ReferenceTypes.HasGroup, group.NodeId);
}
```

## Roles

### Creating a new Role

```csharp
[ExportMethod]
public void CreateNewRole(string roleName) {
    // Get to the base folder to create roles
    Folder userFolder = Project.Current.Get<Folder>("Security/Roles");
    if (userFolder == null) {
        Log.Error("CreateNewRole", "Cannot find Roles folder");
        return;
    } else if (string.IsNullOrEmpty(roleName)) {
        Log.Error("CreateNewRole", "Cannot create role with empty name");
        return;
    } else if (userFolder.Get<FTOptix.Core.Role>(roleName) != null) {
        Log.Error("CreateNewRole", "Role already exists!");
        return;
    }
    // Create a new role
    var newRole = InformationModel.Make<FTOptix.Core.Role>(roleName);
    // Add the role to the folder
    userFolder.Add(newRole);
}
```

### Check if user is part of a role

```csharp
/// <summary>
/// Checks if a user is part of a specified role.
/// </summary>
/// <param name="user">The user to check.</param>
/// <param name="role">The role to check against.</param>
/// <returns>True if the user is part of the role, otherwise false.</returns>
private static bool UserHasRole(FTOptix.Core.User user, FTOptix.Core.Role role)
{
    // Return false if user is null
    if (user == null)
        return false;
    
    // Return false if role is null
    if (role == null)
        return false;

    // Get all roles the user is part of
    var userRoles = user.Refs.GetObjects(FTOptix.Core.ReferenceTypes.HasRole, false);

    // Iterate through the roles to check if the user is part of the specified role
    foreach (var userRole in userRoles)
    {
        if (userRole.NodeId == role.NodeId)
            return true;
    }
    // Return false if the user is not part of the specified role
    return false;
}
```

### Assign role to a user

```csharp
/// <summary>
/// Assigns a user to a specified role by adding a reference between the user and the role.
/// </summary>
/// <param name="user">The user to be assigned to the role.</param>
/// <param name="role">The role to which the user will be assigned.</param>
private void AssignUserToRole(FTOptix.Core.User user, FTOptix.Core.Role role)
{
    user.Refs.AddReference(FTOptix.Core.ReferenceTypes.HasRole, role.NodeId);
}
```

### Remove role from a user

```csharp
/// <summary>
/// Removes a user from the specified role by removing a reference between the user and the role.
/// </summary>
/// <param name="user">The user to be removed from the role.</param>
/// <param name="role">The role to which the user will be removed from.</param>
private void RemoveUserFromRole(FTOptix.Core.User user, FTOptix.Core.Role role)
{
    user.Refs.RemoveReference(FTOptix.Core.ReferenceTypes.HasRole, role.NodeId);
}
```
