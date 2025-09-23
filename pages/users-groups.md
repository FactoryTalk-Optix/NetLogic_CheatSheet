# Users, groups and roles

## Users

Users are represented by the `FTOptix.Core.User` type in FactoryTalk Optix. Users can be created, modified, and deleted programmatically using C# scripts.

> [!NOTE]
> Operations on users might require a valid `Session` to interact with the security system. You can simply put the piece of code somewhere in the UI (e.g., a screen) and it will work as expected.

### Creating a new User

```csharp
/// <summary>
/// Creates a new user with the specified username, password, and locale.
/// The method sets the result to a default value (NodeId.Empty) and checks if the username is empty.
/// If the username is empty, it shows a message and logs an error.
/// Otherwise, it generates the user and assigns the result.
/// </summary>
/// <param name="username">The username for the new user.</param>
/// <param name="password">The password for the new user.</param>
/// <param name="locale">The locale setting for the new user.</param>
/// <note>
/// The method uses <see cref="NodeId.Empty"/> as the default value for <see cref="result"/>.
/// </note>
[ExportMethod]
public void CreateUser(string username, string password, string locale, out NodeId result)
{
    result = NodeId.Empty;

    if (string.IsNullOrEmpty(username))
    {
        ShowMessage(1);
        Log.Error("EditUserDetailPanelLogic", "Cannot create user with empty username");
        return;
    }

    result = GenerateUser(username, password, locale);
}

/// <summary>
/// This method creates a new user with the specified username, password, and locale.
/// It checks if the username already exists in the user list and handles various
/// password change result codes.
/// </summary>
/// <param name="username">The username for the new user.</param>
/// <param name="password">The password for the new user.</param>
/// <param name="locale">The locale ID for the user (optional).</param>
/// <returns>
/// The NodeId of the newly created user, or NodeId.Empty if an error occurs.
/// </returns>
private NodeId GenerateUser(string username, string password, string locale)
{
    var users = GetUsers();
    if (users == null)
    {
        Log.Error("EditUserDetailPanelLogic", "Unable to get users");
        return NodeId.Empty;
    }

    foreach (var child in users.Children.OfType<FTOptix.Core.User>())
    {
        if (child.BrowseName.Equals(username, StringComparison.OrdinalIgnoreCase))
        {
            Log.Error("EditUserDetailPanelLogic", "Username already exists");
            return NodeId.Empty;
        }
    }

    var user = InformationModel.MakeObject<FTOptix.Core.User>(username);
    users.Add(user);

    //Apply LocaleId
    if (!string.IsNullOrEmpty(locale))
        user.LocaleId = locale;

    //Apply password
    var result = Session.ChangePassword(username, password, string.Empty);

    switch (result.ResultCode)
    {
        case FTOptix.Core.ChangePasswordResultCode.Success:
            Log.Verbose("EditUserDetailPanelLogic", "User created successfully");
            break;
        case FTOptix.Core.ChangePasswordResultCode.WrongOldPassword:
            //Not applicable
            break;
        case FTOptix.Core.ChangePasswordResultCode.PasswordAlreadyUsed:
            //Not applicable
            break;
        case FTOptix.Core.ChangePasswordResultCode.PasswordChangedTooRecently:
            //Not applicable
            break;
        case FTOptix.Core.ChangePasswordResultCode.PasswordTooShort:
            Log.Error("EditUserDetailPanelLogic", "Password is too short");
            users.Remove(user);
            return NodeId.Empty;
        case FTOptix.Core.ChangePasswordResultCode.UserNotFound:
            //Not applicable
            break;
        case FTOptix.Core.ChangePasswordResultCode.UnsupportedOperation:
            Log.Error("EditUserDetailPanelLogic", "Unsupported operation");
            users.Remove(user);
            return NodeId.Empty;

    }

    return user.NodeId;
}
```

### Change user password

```csharp
/// <summary>
/// Changes the password of a user.
/// </summary>
/// <param name="username">The username of the user whose password is to be changed.</param>
/// <param name="newPassword">The new password to be set for the user.</param>
/// <returns>True if the password was changed successfully, otherwise false.</returns>
[ExportMethod]
public void ChangePassword(string username, string newPassword, out bool result)
{
    result = false;

    if (string.IsNullOrEmpty(username))
    {
        Log.Error("ChangePassword", "Username cannot be empty");
        return;
    }

    var changePasswordResult = Session.ChangePassword(username, newPassword, string.Empty);

    switch (changePasswordResult.ResultCode)
    {
        case FTOptix.Core.ChangePasswordResultCode.Success:
            Log.Verbose("ChangePassword", "Password changed successfully");
            result = true;
            break;
        case FTOptix.Core.ChangePasswordResultCode.WrongOldPassword:
            // Not applicable
            break;
        case FTOptix.Core.ChangePasswordResultCode.PasswordAlreadyUsed:
            Log.Error("ChangePassword", "Password has already been used");
            break;
        case FTOptix.Core.ChangePasswordResultCode.PasswordChangedTooRecently:
            Log.Error("ChangePassword", "Password was changed too recently");
            break;
        case FTOptix.Core.ChangePasswordResultCode.PasswordTooShort:
            Log.Error("ChangePassword", "New password is too short");
            break;
        case FTOptix.Core.ChangePasswordResultCode.UserNotFound:
            Log.Error("ChangePassword", "User not found");
            break;
        case FTOptix.Core.ChangePasswordResultCode.UnsupportedOperation:
            Log.Error("ChangePassword", "Unsupported operation");
            break;
    }
}
```

### Check if user belongs to a specific group or role

Groups and roles are used to manage user permissions in FactoryTalk Optix. Users can be assigned to groups and roles, which can then be used to control access to various parts of the system.

Both Roles and Groups are reference types, meaning that they can be assigned to users by adding a reference between the user and the group or role. This allows for flexible management of user permissions.

For example:

```csharp
// Get all groups the user is part of
var userGroups = newUser.Refs.GetObjects(FTOptix.Core.ReferenceTypes.HasGroup, false);
// Get all roles the user is part of
var userGroups = newUser.Refs.GetObjects(FTOptix.Core.ReferenceTypes.HasRole, false);
```

Roles can be assigned to groups and users, groups can be assigned to roles and users, you can make any combination of these to create a flexible permission system.

### Perform a login

```csharp
/// <summary>
/// This method performs the login operation using the provided username and password.
/// It retrieves the Users alias and PasswordExpiredDialogType from the logic object,
/// attempts to log in using the Session object,
/// and handles the result of the login attempt.
/// If the login is successful, it enables the login button;
/// if the password is expired, it opens a dialog for the user to change their password.
/// If the login fails, it sets an output message on the login form.
/// </summary>
/// <param name="username">The username to log in with.</param>
/// <param name="password">The password to log in with.</param>
[ExportMethod]
public void PerformLogin(string username, string password)
{
    Button loginButton = (Button)Owner;
    loginButton.Enabled = false;

    try
    {
        var loginResult = Session.Login(username, password);
        if (loginResult.ResultCode == ChangeUserResultCode.PasswordExpired)
        {
            var user = Project.Current.Get<User>($"Security/Users/{username}");
            var ownerButton = (Button)Owner;
            Log.Info("LoginButtonLogic", "Password expired, opening change password dialog");
            return;
        }
        else if (loginResult.ResultCode != ChangeUserResultCode.Success)
        {
            Log.Error("LoginButtonLogic", "Authentication failed");
        }
        else if (loginResult.ResultCode == ChangeUserResultCode.Success)
        {
            Log.Info("LoginButtonLogic", "User logged in successfully");
        }
        else
        {
            Log.Error("LoginButtonLogic", "Unknown error during login");
        }
    }
    catch (Exception e)
    {
        Log.Error("LoginButtonLogic", e.Message);
    }
    finally
    {
        loginButton.Enabled = true;
    }
}
```

## Groups

### Creating a new Group

```csharp
/// <summary>
/// Creates a new group with the specified name.
/// The method checks if the group name is empty or if the group already exists,
/// and logs appropriate error messages.
/// If the group is successfully created, it adds the group to the "Security/Groups" folder.
/// </summary>
/// <param name="groupName">The name of the group to be created.</param>
/// <returns>True if the group was created successfully, false otherwise.</returns>
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
/// <summary>
/// Checks if a user is part of a specified group.
/// </summary>
/// <param name="user">The user to check.</param>
/// <param name="group">The group to check against.</param>
/// <returns>True if the user is part of the group, otherwise false.</returns>
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

## Get the list of groups associated to the user which is logging in

Please see: [Register OPC/UA observers](./register-observers.md).
