# Managing Aliases

## Delete the value from an Alias

```csharp
Project.Current.Get("UI/Pages/MainPage").SetAlias("MainBoilerAlias", NodeId.Empty);
```

## Delete the Kind property of an Alias

```csharp
Project.Current.Get<Alias>("UI/Pages/MainPage/MainBoilerAlias").Kind = NodeId.Empty;
```

## Connect an Alias to a node

```csharp
var targetVar = Project.Current.Get<Tag>("CommDriver/RAEthernetIP/Station1/InletPump");
Project.Current.Get("UI/Pages/MainPage").SetAlias("MainBoilerAlias", targetVar.NodeId);
```
