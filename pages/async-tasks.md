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

```csharp
public override void Start()
{
    myPeriodicTask = new PeriodicTask(IncrementVariable, 1000, LogicObject);
    myPeriodicTask.Start();
}

public override void Stop()
{
    myPeriodicTask?.Dispose();
}

private void IncrementVariable()
{
    …
}

private PeriodicTask myPeriodicTask;
```

### DelayedTask

This kind of task will trigger a method after `x` milliseconds it is called, this can be useful for example to hide a toast notification after some time

```csharp
public override void Start()
{
    myDelayedTask = new DelayedTask(ResetLabelText, 10000, LogicObject);
    myDelayedTask.Start();
}

public override void Stop()
{
    myDelayedTask?.Dispose();
}

private void ResetLabelText()
{
    …
}

private DelayedTask myDelayedTask;
```

### LongRunningTask

This is the most similar approach to the `async` modifier of the standard C# scripting, it allows to create a dedicated thread and execute it without affecting the main code execution

```csharp
class public override void Start()
{
    myLongRunningTask = new LongRunningTask(ProcessCSVFile, LogicObject);
    myLongRunningTask.Start();
}

public override void Stop()
{
    myLongRunningTask?.Dispose();
}

private void ProcessCsvFile(LongRunningTask task)
{
    …
}

private LongRunningTask myLongRunningTask;
```
