# Debugging tips

> [!NOTE]
> The following tips are meant to help you debug your FactoryTalk Optix projects. They might help you find issues in your code or understand how the system works. Use them wisely!

## Debugging page load time

To debug the page load time, the NativeUI and WebUI debug logs can be used, these will provide you detailed information about the page load time, including the time taken to load each component and the time taken to execute each script.

1. Using the Save menu, export the application as an x86 binary anywhere on your PC
1. Move the exported folder manually to the target device
2. Open a command line prompt in the main FTOptixRuntime.exe folder
3. Run the application with:

```
FTOptixRuntime.exe --logfile-size=10000 --log-level=INFO --module-log=urn:FTOptix:NativeUI=DEBUG;urn:FTOptix:WebUI=DEBUG
```

Now start using the app, switching screens, interacting with grids and so on.

When you are satisfied and you passed trough most pages (or the ones that are taking longer to load), get the three log files from the Logs folder so they can be inspected:

1. Open the [Logs Parsing Tool](https://asem-applicationsoftwareengineers.github.io/OptixRuntimeLogsParser/)
2. Select and   the three log files from the Logs folder one by one
3. Click on the "Parse" button

> [!NOTE]
> The logs parsing tool does not store any data anywhere, it only parses the logs and displays the results in a table. The logs are not sent to any server or stored anywhere, they are only used to display the results in the web app. The source code of the tool is available on [GitHub](https://github.com/ASEM-ApplicationSoftwareEngineers/OptixRuntimeLogsParser)

The web app will parse the logs and display the results in a table, showing the time taken to load each component and the time taken to execute each script.

Now, click the tabs to investigate the results:

### UI Timing

This tab shows the time taken to load each component of the page, including the time taken to load the page itself, the time taken to load each component, and the time taken to execute each script.

> [!NOTE]
> The first page being loaded will likely produce odd results (with some crazy loading time), as the page is being loaded for the first time and the components are being initialized. This is normal and should not be taken into account when analyzing the results.

For example:


| Logs Parser Tool output | Description |
| --- | --- |
| Page: Screen1 | The name of  the page being loaded |
| Load time: 1467 ms  | The total time taken to load the page |
| Nodes: 904 (31 ms) | The number of project nodes loaded on the page and the time taken to load them |
| UI Objects: 280 (31 ms)  | The number of UI objects loaded on the page and the time taken to load them |
| DataBinds and Converters: 127 (7 ms)  | The number of data bindings and converters loaded on the page and the time taken to load them |
| UIObjectsModelControllers: 20 (882 ms)  | The number of UI object model controllers loaded on the page and the time taken to load them |
| EventHandlers: 8 (0 ms)   | The number of event handlers loaded on the page and the time taken to load them |
| MethodInvocations: 2 (0 ms)   | The number of method invocations loaded on the page and the time taken to load them |
| NetLogics: 2 (447 ms)   | The number of NetLogics loaded on the page and the time taken to load them |
| Qt create: 172 ms   | The time taken to create the Qt objects |
| Qt model: 2 ms    | The time taken to create the Qt model |
| Qt total: 1398 ms    | The total time taken to create the Qt objects and pass them to the graphics card |

In this example, the page took 1467 ms to load, with the majority of the time being spent on loading the UI objects model controllers (882 ms) and the NetLogics (447 ms). This means that the page is likely to be slow due to the complexity of the UI objects model controllers and the NetLogics, it might be safe to assume that some portion of the UI are waiting for the C# scripts to be completed for the page to be rendered.

### Warnings

Although not directly related to the page load time, this tab shows the warnings that were logged during the page load. This can help you identify potential issues in your code that might be causing the page to load slowly.

> [!TIP]
> If you see any warnings in this tab, it is recommended to investigate them and fix them, as they might be causing the page to load slowly or might cause issues in the future.

## Measure code execution time with Stopwatch

```csharp
// Start stopwatch
Stopwatch stopwatch = new Stopwatch();
stopwatch.Start();

// code to measure

stopwatch.Stop();
Log.Info("Execution time (ms): " + stopwatch.Elapsed.TotalMilliseconds);
```

### Errors

This tab shows the errors that were logged during the page load. This can help you identify potential issues in your code that might be causing the page to fail to load or to load slowly.

Before attempting any app optimization, it is recommended to check this tab and fix any errors that might be causing the page to fail to load or to load slowly.
