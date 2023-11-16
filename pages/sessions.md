# Sessions

## Getting the number of the WebPresentationEngine sessions

```csharp
[ExportMethod]
publicvoidGetActiveWebSessionsNumber(NodeIdwebPresentatonEngine, out intactiveWebSessionNumber){
    try
    {
        varwpe=InformationModel.Get<PresentationEngine>(webPresentatonEngine);
        activeWebSessionNumber=GetWebpresentationengingSessions(wpe).Count();
    }
    catch(System.Exception)
    {
        Log.Error(e.Message);
        activeWebSessionNumber=-1;
    }
}
privateIEnumerable<UISession>GetWebpresentationengingSessions(PresentationEnginewebPresentationEngine) =>webPresentationEngine.Sessions.Where(s=>s.User!=null);
```


