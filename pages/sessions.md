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
