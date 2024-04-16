# Sessions

## Getting the number of the WebPresentationEngine sessions

```csharp
[ExportMethod]
public void GetActiveWebSessionsNumber(NodeId webPresentationEngine, out int activeWebSessionNumber)
{
    try
    {
        // Get the current WebPresentationEngine from the project
        var webPresentationEngine = InformationModel.Get<PresentationEngine>(webPresentationEngine);
        // Call the method to read the active sessions
        activeWebSessionNumber = GetWebPresentationEngineSessions(webPresentationEngine).Count();
    }
    catch(System.Exception)
    {
        Log.Error(e.Message);
        activeWebSessionNumber=-1;
    }
}

// Use reflections to get the sessions
private IEnumerable<UISession> GetWebPresentationEngineSessions(PresentationEngine webPresentationEngine) => webPresentationEngine.Sessions.Where(s=>s.User!=null);
```


