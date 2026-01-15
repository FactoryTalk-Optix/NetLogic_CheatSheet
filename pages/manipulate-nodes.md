# Manipulate Optix nodes

> [!WARNING]
> This is a very advanced topic and potentially dangerous, only the Optix core should perform these operations.
> This feature is not officially supported and may be subject to changes in future releases.

## Stopping an Optix node

Once nodes are stopped, they stop interacting with the system for example an ODBC database node will stop processing queries, a Trend node will stop logging data, an Alarm node will stop monitoring conditions, etc.

This can be useful in some scenarios to disable some components (like the `Signing Workflow` node) or to temporarily stop some processes (like a `Trend` node) without deleting them.

```csharp
/// <summary>
/// Stops an Optix node if it is started
/// </summary>
/// <param name="node">The node to be stopped</param>
private void StopNode(IUANode node)
{
    // Only objects can be started/stopped
    if (node is IUAObject nodeObject && node.Status == NodeStatus.Started)
    {
        // Stop the node
        nodeObject.Stop();
    }
        
}
```

## Starting an Optix node

```csharp
/// <summary>
/// Starts an Optix node if it is stopped
/// </summary>
/// <param name="node">The node to be started</param>
private void StartNode(IUANode node)
{
    // Only objects can be started/stopped
    if (node is IUAObject nodeObject && node.Status != NodeStatus.Started)
    {
        // Start the node
        nodeObject.Start();
    }
        
}
```

## Restarting an Optix node

This is simply a combination of the previous two methods

```csharp
/// <summary>
/// Restarts an Optix node if it is started
/// </summary>
/// <param name="node">The node to be restarted</param>
private void RestartNode(IUANode node)
{
    // Only objects can be started/stopped
    if (node is IUAObject nodeObject && node.Status == NodeStatus.Started)
    {
        // Restart the node
        nodeObject.Stop();
        nodeObject.Start();
    }
        
}
```
