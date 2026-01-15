# Sessions

## Getting the number of the WebPresentationEngine sessions

```csharp
 /// <summary>
 /// Retrieves the number of active Web sessions
 /// </summary>
 /// <param name="webPresentationEngineNodeId">NodeId of the WebPresentationEngine node</param>
 /// <param name="activeWebSessionNumber">Counting active web sessions</param>
 [ExportMethod]
 public void GetActiveWebSessionsNumber(NodeId webPresentationEngineNodeId, out int activeWebSessionNumber)
 {
     try
     {
         // Get the current WebPresentationEngine from the project
         PresentationEngine webPresentationEngine = InformationModel.Get<PresentationEngine>(webPresentationEngineNodeId);
         // Call the method to read the active sessions
         activeWebSessionNumber = GetWebPresentationEngineSessions(webPresentationEngine).Count();
     }
     catch (Exception ex)
     {
         Log.Error(ex.Message);
         activeWebSessionNumber = -1;
     }
 }

 // Use reflections to get the sessions (check where user is present (like Anonymous))
 private static IEnumerable<UISession> GetWebPresentationEngineSessions(PresentationEngine webPresentationEngine) => webPresentationEngine.Sessions.Where(s => s.User != null);
```

## Impersonate a session

Every operation being done on the Optix core must be done in the context of a session. When you are invoking code from an asynchronous method, you must always bind to an existing session, even if for a short time.

> [!TIP]
> This is a very edge case, if you need to perform asynchronous operation without having to deal with sessions, it is a better choice to use [async tasks](./async-tasks.md) instead.

If no session is impersonated, the code will likely cause the runtime to crash (even with simple operation like toggling a variable).

```csharp
public void DoSomethingAsynchronously()
{
    // Get the first available session
    var sessionHandler = LogicObject.Context.Sessions.ImpersonateRootTemporary();
    if (sessionHandler != null)
    {
        // Do something in the context of the session
        var someVariable = InformationModel.GetVariable("SomeVariable");
        someVariable.Value = true;        
    }
    // Always dispose the session handler to release the session
    sessionHandler?.Dispose();
}
```
