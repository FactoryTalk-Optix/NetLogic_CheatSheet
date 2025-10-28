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

### Errors

This tab shows the errors that were logged during the page load. This can help you identify potential issues in your code that might be causing the page to fail to load or to load slowly.

Before attempting any app optimization, it is recommended to check this tab and fix any errors that might be causing the page to fail to load or to load slowly.

## Measure code execution time with Stopwatch

Using `DateTime.Now` to measure code execution time is not accurate, as it is affected by system time changes and has a low resolution. Instead, use the `Stopwatch` class from the `System.Diagnostics` namespace, which provides a high-resolution timer specifically designed for measuring elapsed time.

```csharp
// Start stopwatch
Stopwatch stopwatch = new Stopwatch();
stopwatch.Start();

// code to measure

stopwatch.Stop();
Log.Info("Execution time (ms): " + stopwatch.Elapsed.TotalMilliseconds);
```

## Simple profiler

This code can be used to create a simple profiler that logs memory usage, CPU usage, number of variables, tags, sessions, and active alarms every minute to a log file. The log files are rotated when they reach a specified size limit, and a maximum number of log files are kept.

This code can easily be extended to log additional metrics as needed.

```csharp
#region Using directives
using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.IO;
using System.Linq;
using System.Threading.Tasks;
using FTOptix.CommunicationDriver;
using FTOptix.Core;
using FTOptix.CoreBase;
using FTOptix.HMIProject;
using FTOptix.NativeUI;
using FTOptix.NetLogic;
using FTOptix.Retentivity;
using FTOptix.UI;
using UAManagedCore;
using FTOptix.RAEtherNetIP;
using OpcUa = UAManagedCore.OpcUa;
#endregion

public class SimpleProfiler : BaseNetLogic
{
    private FileLogger logger;

    public override void Start()
    {
        // Initialize the logger with a directory path
        // Logs will be stored in the project's ApplicationData folder
        string logDirectory = ResourceUri.FromApplicationRelativePath("").Uri;
        
        // Create logger with:
        // - Log directory path
        // - File prefix "SimpleProfiler"
        // - Max 10 MB per file
        // - Keep maximum 5 log files
        logger = new FileLogger(logDirectory, "Profiler", maxFileSizeMB: 10, maxFileCount: 5);
        
        // Example usage
        logger.WriteLog("SimpleProfiler started");

        // Set up a periodic task to log memory usage every minute
        profilerTask = new PeriodicTask(LogMemoryUsage, 60000, LogicObject);
        profilerTask.Start();
    }

    private void LogMemoryUsage()
    {
        var stringBuilder = new System.Text.StringBuilder();
        var cpuTask = GetMemoryUsageForProcess();
        cpuTask.Wait();
        var memoryUsage = cpuTask.Result;
        stringBuilder.Append($"RAM:{memoryUsage / 1024}KB;");
        var cpuUsageTask = GetCpuUsageForProcess();
        cpuUsageTask.Wait();
        double cpuUsage = cpuUsageTask.Result;
        stringBuilder.Append($"CPU:{cpuUsage:F2}%;");
        long variablesCount = GetInstancesOfTypeRecursive(InformationModel.Get(OpcUa.VariableTypes.BaseVariableType)).Count;
        stringBuilder.Append($"Vars:{variablesCount};");
        long retainedAlarmsCount = GetRetainedAlarmsCount();
        stringBuilder.Append($"ActiveAlarms:{retainedAlarmsCount};");
        long tagsCount = GetInstancesOfTypeRecursive(InformationModel.Get(FTOptix.CommunicationDriver.VariableTypes.Tag)).Count;
        stringBuilder.Append($"Tags:{tagsCount};");
        long sessionsCount = GetInstancesOfTypeRecursive(InformationModel.Get(FTOptix.Core.ObjectTypes.Session)).Count;
        stringBuilder.Append($"Sessions:{sessionsCount};");
        logger.WriteLog(stringBuilder.ToString());
    }

    public override void Stop()
    {
        // Log when stopping
        if (logger != null)
        {
            logger.WriteLog("SimpleProfiler stopped");
        }
        
        profilerTask?.Dispose();
        logger = null;
        profilerTask = null;
    }

    /// <summary>
    /// Gets the total count of retained (active) alarms in the system
    /// </summary>
    /// <returns>The number of retained alarms, or 0 if unable to retrieve the count</returns>
    private long GetRetainedAlarmsCount()
    {
        try
        {
            var retainedAlarmsObject = LogicObject.Context.GetNode(FTOptix.Alarm.Objects.RetainedAlarms);
            if (retainedAlarmsObject != null)
            {
                var localizedAlarmsVariable = retainedAlarmsObject.GetVariable("LocalizedAlarms");
                if (localizedAlarmsVariable != null)
                {
                    var localizedAlarmsNodeId = (NodeId)localizedAlarmsVariable.Value;
                    if (localizedAlarmsNodeId != null && !localizedAlarmsNodeId.IsEmpty)
                    {
                        var localizedAlarmsContainer = LogicObject.Context.GetNode(localizedAlarmsNodeId);
                        if (localizedAlarmsContainer != null)
                        {
                            return localizedAlarmsContainer.Children.Count;
                        }
                    }
                }
            }
        }
        catch (Exception ex)
        {
            Log.Warning("SimpleProfiler", $"Failed to get retained alarms count: {ex.Message}");
        }
        
        return 0;
    }

    /// <summary>
    /// Recursively gets all instances of a type and its subtypes
    /// </summary>
    /// <param name="typeNode">The type node to get instances from</param>
    /// <returns>List of all instances of the type and its subtypes</returns>
    private List<IUANode> GetInstancesOfTypeRecursive(IUANode typeNode)
    {
        // Get all instances that have this type as their TypeDefinition
        var result = typeNode.InverseRefs.GetNodes(OpcUa.ReferenceTypes.HasTypeDefinition, false).ToList();

        // Get all subtypes of this type
        var subtypes = typeNode.Refs.GetNodes(OpcUa.ReferenceTypes.HasSubtype, false);
        
        // Recursively get instances from each subtype
        foreach (var subtype in subtypes)
        {
            var subtypeInstances = GetInstancesOfTypeRecursive(subtype);
            result.AddRange(subtypeInstances);
        }

        return result;
    }

    /// <summary>
    /// Asynchronously calculates the CPU usage percentage for the current process over a short interval.
    /// </summary>
    /// <remarks>This method measures the CPU usage of the current process by sampling the total processor
    /// time over a fixed delay period. The result represents the percentage of CPU time used by the process relative to
    /// the total available CPU time across all cores.</remarks>
    /// <returns>A <see cref="double"/> representing the CPU usage percentage of the current process. The value ranges from 0 to
    /// 100, where 100 indicates full utilization of all CPU cores.</returns>
    private async Task<double> GetCpuUsageForProcess()
    {
        var stopWatch = new Stopwatch();
        stopWatch.Start();
        var startCpuUsage = Process.GetCurrentProcess().TotalProcessorTime;
        await Task.Delay(500);

        stopWatch.Stop();
        var endCpuUsage = Process.GetCurrentProcess().TotalProcessorTime;
        var cpuUsedMs = (endCpuUsage - startCpuUsage).TotalMilliseconds;
        var cpuUsageTotal = cpuUsedMs / (Environment.ProcessorCount * stopWatch.Elapsed.TotalMilliseconds);
        return cpuUsageTotal * 100;
    }

    private async Task<double> GetMemoryUsageForProcess()
    {
        return await Task.FromResult(Process.GetCurrentProcess().PrivateMemorySize64);
    }

    private PeriodicTask profilerTask;
}

public class FileLogger
{
    private readonly string _logDirectory;
    private readonly string _logFilePrefix;
    private readonly int _maxFileSizeMB;
    private readonly int _maxFileCount;
    private readonly object _lockObject = new object();
    private string _currentLogFile;

    /// <summary>
    /// Initializes a new instance of the FileLogger class
    /// </summary>
    /// <param name="logDirectory">Directory where log files will be stored</param>
    /// <param name="logFilePrefix">Prefix for log file names (default: "log")</param>
    /// <param name="maxFileSizeMB">Maximum size of each log file in MB (default: 10)</param>
    /// <param name="maxFileCount">Maximum number of log files to keep (default: 5)</param>
    public FileLogger(string logDirectory, string logFilePrefix = "log", int maxFileSizeMB = 10, int maxFileCount = 5)
    {
        _logDirectory = logDirectory;
        _logFilePrefix = logFilePrefix;
        _maxFileSizeMB = maxFileSizeMB;
        _maxFileCount = maxFileCount;

        // Ensure log directory exists
        if (!Directory.Exists(_logDirectory))
        {
            Directory.CreateDirectory(_logDirectory);
        }

        // Initialize current log file
        _currentLogFile = GetCurrentLogFilePath();
    }

    /// <summary>
    /// Writes a message to the log file with timestamp
    /// </summary>
    /// <param name="message">The message to log</param>
    public void WriteLog(string message)
    {
        lock (_lockObject)
        {
            try
            {
                // Check if rotation is needed
                CheckAndRotateLogFile();

                // Write the log entry with timestamp
                string logEntry = $"[{DateTime.Now:yyyy-MM-dd HH:mm:ss.fff}] {message}";
                File.AppendAllText(_currentLogFile, logEntry + Environment.NewLine);
            }
            catch (Exception ex)
            {
                // Fallback to debug output if file write fails
                Log.Error($"FileLogger - Failed to write log: {ex.Message}");
            }
        }
    }

    /// <summary>
    /// Writes a message to the log file with a specified log level
    /// </summary>
    /// <param name="level">Log level (INFO, WARNING, ERROR, etc.)</param>
    /// <param name="message">The message to log</param>
    public void WriteLog(string level, string message)
    {
        WriteLog($"[{level}] {message}");
    }

    /// <summary>
    /// Checks if the current log file exceeds the size limit and rotates if necessary
    /// </summary>
    private void CheckAndRotateLogFile()
    {
        if (File.Exists(_currentLogFile))
        {
            FileInfo fileInfo = new FileInfo(_currentLogFile);
            long maxSizeBytes = _maxFileSizeMB * 1024 * 1024;

            if (fileInfo.Length >= maxSizeBytes)
            {
                RotateLogFiles();
            }
        }
    }

    /// <summary>
    /// Rotates log files by renaming them with incremental numbers
    /// </summary>
    private void RotateLogFiles()
    {
        try
        {
            // Get all existing log files with numbers in their names
            var logFiles = Directory.GetFiles(_logDirectory, $"{_logFilePrefix}*.txt")
                .Select(f => new FileInfo(f))
                .ToList();

            // Parse file numbers and find the highest number
            int highestNumber = 0;
            var numberedFiles = new List<(FileInfo file, int number)>();

            foreach (var file in logFiles)
            {
                string nameWithoutExtension = Path.GetFileNameWithoutExtension(file.Name);
                
                // Check if this is the base file (no number)
                if (nameWithoutExtension == _logFilePrefix)
                {
                    numberedFiles.Add((file, 0));
                }
                else if (nameWithoutExtension.StartsWith(_logFilePrefix + "_"))
                {
                    string numberPart = nameWithoutExtension.Substring(_logFilePrefix.Length + 1);
                    if (int.TryParse(numberPart, out int number))
                    {
                        numberedFiles.Add((file, number));
                        if (number > highestNumber)
                        {
                            highestNumber = number;
                        }
                    }
                }
            }

            // Sort by number (descending) to rename from highest to lowest
            numberedFiles = numberedFiles.OrderByDescending(x => x.number).ToList();

            // Delete files that exceed the max count
            if (numberedFiles.Count >= _maxFileCount)
            {
                var filesToDelete = numberedFiles.Skip(_maxFileCount - 1).ToList();
                foreach (var (file, _) in filesToDelete)
                {
                    try
                    {
                        file.Delete();
                    }
                    catch (Exception ex)
                    {
                        Log.Error($"FileLogger - Failed to delete old log file: {ex.Message}");
                    }
                }
                // Remove deleted files from the list
                numberedFiles = numberedFiles.Take(_maxFileCount - 1).ToList();
            }

            // Rename existing files (shift numbers up)
            foreach (var (file, currentNumber) in numberedFiles)
            {
                if (file.Exists)
                {
                    int newNumber = currentNumber + 1;
                    string newFileName = Path.Combine(_logDirectory, $"{_logFilePrefix}_{newNumber}.txt");

                    try
                    {
                        // If target file exists, we need to find the next available number
                        while (File.Exists(newFileName))
                        {
                            newNumber++;
                            newFileName = Path.Combine(_logDirectory, $"{_logFilePrefix}_{newNumber}.txt");
                        }
                        
                        file.MoveTo(newFileName);
                    }
                    catch (Exception ex)
                    {
                        Log.Error($"FileLogger - Failed to rotate log file: {ex.Message}");
                    }
                }
            }

            // Current log file path remains the same (base name without number)
            // This ensures we always append to a new file after rotation
            _currentLogFile = GetCurrentLogFilePath();
        }
        catch (Exception ex)
        {
            Log.Error($"FileLogger - Failed to rotate log files: {ex.Message}");
        }
    }

    /// <summary>
    /// Gets the path for the current log file
    /// </summary>
    /// <returns>Full path to the current log file</returns>
    private string GetCurrentLogFilePath()
    {
        return Path.Combine(_logDirectory, $"{_logFilePrefix}.txt");
    }

    /// <summary>
    /// Clears all log files in the log directory
    /// </summary>
    public void ClearAllLogs()
    {
        lock (_lockObject)
        {
            try
            {
                var logFiles = Directory.GetFiles(_logDirectory, $"{_logFilePrefix}*.txt");
                foreach (var file in logFiles)
                {
                    try
                    {
                        File.Delete(file);
                    }
                    catch (Exception ex)
                    {
                        Log.Error($"FileLogger - Failed to delete log file {file}: {ex.Message}");
                    }
                }

                // Recreate current log file path (file will be created on first write)
                _currentLogFile = GetCurrentLogFilePath();
            }
            catch (Exception ex)
            {
                Log.Error($"FileLogger - Failed to clear log files: {ex.Message}");
            }
        }
    }
}
```