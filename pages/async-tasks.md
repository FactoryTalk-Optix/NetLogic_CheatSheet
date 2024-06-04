# Asynchronous tasks

Asynchronous tasks are very common when executing special or long tasks that may negatively impact the overall execution

## General concepts about Asynchronous tasks

Standard C# language provides multiple ways to execute asynchronous operations (such as the `async` modifier for some methods), these can be used in FT Optix as long as those methods does not access any of the project structure (pages, tags, etc).

- Example:
    - async method to read/write a CSV -> GOOD
    - async method to read a CSV and import it into the TranslationDictionary -> BAD

## Using FT Optix Asynchronous tasks

These asynchronous tasks are InformationModel-safe and can be used to access project nodes without concerns

### PeriodicTask

This kind of task allows a specific method to be executed every `x` milliseconds, for example to blink a LED or to handle a watchdog.

Please note: the specified interval, is actually the delay between two consecutive execution of the method, and the execution is impacted by the length of the instruction to process. The next execution is delayed until the task is completed, for example:

Having a task:
- PeriodicTask declared every 50mS
- Method executed by the PeriodicTask takes 100mS to process

With this timings, the task is actually executed every 150mS, as the interval is actually the delay after which the task is executed again.

```csharp
public override void Start()
{
    // Creates a task that runs every second without impacting the UI
    myPeriodicTask = new PeriodicTask(IncrementVariable, 1000, LogicObject);
    myPeriodicTask.Start();
}

public override void Stop()
{
    // When the NetLogic is disposed, kill the task too
    myPeriodicTask?.Dispose();
}

private void IncrementVariable()
{
    // Action to be executed every tick of the periodic task
    …
}

private PeriodicTask myPeriodicTask;
```

### DelayedTask

This kind of task will trigger a method after `x` milliseconds it is called, this can be useful for example to hide a toast notification after some time

```csharp
public override void Start()
{
    // Create an action that is executed after 10s the page is loaded
    // this could also be a method called by a button or any trigger
    myDelayedTask = new DelayedTask(ResetLabelText, 10000, LogicObject);
    myDelayedTask.Start();
}

public override void Stop()
{
    // Make sure to dispose the task when the NetLogic is stopped
    myDelayedTask?.Dispose();
}

private void ResetLabelText()
{
    // Action to be executed after the delay
    …
}

private DelayedTask myDelayedTask;
```

### LongRunningTask

This is the most similar approach to the `async` modifier of the standard C# scripting, it allows to create a dedicated thread and execute it without affecting the main code execution.
An `async` modifier can also be used, but an async method **cannot** access any project node (it may cause concurrency issues), in such cases, use a LongRunningTask instead.

```csharp
class public override void Start()
{
    // Create a dedicated thread to run a long or demanding process
    // Without stopping the FT Optix Runtime or Studio
    myLongRunningTask = new LongRunningTask(ProcessCSVFile, LogicObject);
    myLongRunningTask.Start();
}

public override void Stop()
{
    // Make sure to dispose the thread once the NetLogic is stopped
    myLongRunningTask?.Dispose();
}

private void ProcessCsvFile(LongRunningTask task)
{
    // Process that takes long time to be executed
    …
}

private LongRunningTask myLongRunningTask;
```

### Some general notes on asynchronous tasks

When a task is called asynchronously (especially in LongRunningTask), then the method itself should check if a cancellation request is pending (see [documentation here](https://learn.microsoft.com/en-us/dotnet/standard/parallel-programming/task-cancellation)) and act accordingly, for example:

```csharp
public class LongRunningLogic : BaseNetLogic
{
    public override void Start()
    {
        // Create the new LongRunningTask and execute it
        myTask = new LongRunningTask(MyMethod, LogicObject);
        myTask.Start();
    }

    public override void Stop()
    {
        // Request a cancellation event to the async task
        myTask?.Dispose();
    }

    public void MyMethod()
    {
        for (int i = 0; i < 1000; i++)
        {
            // Check if the somebody requested to dispose this task
            if (myTask.IsCancellationRequested)
                return;
            // Do some work here
            Thread.Sleep(1000);
            Owner.Get<Led>("LED1").Active = !Owner.Get<Led>("LED1").Active;
        }
    }

    private LongRunningTask myTask;
}
```
