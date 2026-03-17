# NetLogic Lifecycle

Understanding how FT Optix manages threads is essential to writing reliable NetLogic code, especially in projects with complex pages, many widgets, or heavy processing requirements.

## Overview

FT Optix runtime operates with two primary execution contexts that are always present:

| Thread | Responsible for |
|--------|----------------|
| **UI thread** | Rendering all graphical objects, handling user input events |
| **Behavior/Core thread** | Calling `Start()` and `Stop()` on all behaviors (including NetLogic), sequentially |

These two threads are **independent** from each other. The UI can render while behaviors are being initialized, and vice versa.

## Start() lifecycle

### When is Start() called?

`Start()` is invoked by the FT Optix runtime when a node (page, object, widget) is activated - either at application startup or when a page is navigated to.

The fundamental rule is: **all `Start()` methods are called sequentially, one at a time, on the behavior thread**. There is no parallelism between different NetLogic `Start()` calls.

> [!WARNING]
> The `Start()` method of a Runtime NetLogic should not add or remove nodes from the page tree, as this can cause unpredictable behavior in the loading sequence. If dynamic modification of the tree is needed, it should be done on the page type definition, and then load the page on the session.

### Execution order

The runtime determines the execution order based on the priority of the objects type (DynamicLinks, EURange, NetLogic, etc.) in the page tree. The general principle is: **deeper nodes start before their parents**. This allows child components to initialize before their containers, which is usually desirable.

The practical outcome for a page where multiple NetLogic exist, is: the most deeply nested NetLogic starts first, and execution proceeds upward through the tree until the page-level NetLogic is last. This is the common case.

If multiple NetLogics have the same depth, their order is determined by the tree traversal (i.e., top to bottom as they appear in the project tree), but this cannot be relied upon for deterministic behavior.

```
Example tree (all same priority):

Page  (depth 1)
├── PanelA  (depth 2)
│   └── NetLogic_A  (depth 3)   <- starts FIRST
├── PanelB  (depth 2)
│   ├── NetLogic_B  (depth 3)   <- starts SECOND
│   └── NetLogic_PanelB         <- starts THIRD (same depth as PanelB, ordered by tree traversal)
└── NetLogic_Page  (depth 1)    <- starts LAST
```

### 2-second threshold warning

If a `Start()` method takes longer than **2000 ms** to return, the runtime logs a warning:

```text
Start method on <NodeName> took too long to execute, Long tasks should alternatively be implemented inside a LongRunningTask.
```

This is not an error - the runtime will wait for `Start()` to finish before moving on to the next behavior. However, until all behaviors have started, any other behavior whose `Start()` is queued will be blocked.

> [!WARNING]
> A slow `Start()` on one NetLogic delays the `Start()` of every other NetLogic that comes after it in the queue. On pages with many components, this can cause a visible freeze during page load.

```csharp
// BAD: blocking operation in Start() delays everything after it
public override void Start()
{
    // This blocks the behavior thread for potentially seconds
    var data = File.ReadAllText("/mnt/slow-nfs/data.csv");
    ProcessData(data);
}

// GOOD: offload to LongRunningTask
public override void Start()
{
    loadTask = new LongRunningTask(LoadData, LogicObject);
    loadTask.Start();
}

private void LoadData()
{
    var data = File.ReadAllText("/mnt/slow-nfs/data.csv");
    ProcessData(data);
}

private LongRunningTask loadTask;
```

## Stop() lifecycle

`Stop()` is called in the **exact reverse order** of `Start()`. This is guaranteed by the runtime regardless of how many NetLogics are present on the page.

- `Stop()` is also called sequentially on the behavior thread
- Exceptions thrown in `Stop()` are caught and logged; they do not interrupt the stop sequence of other behaviors
- After `Stop()`, the NetLogic instance is disposed

```csharp
public override void Stop()
{
    // Always dispose tasks and observers here
    myPeriodicTask?.Dispose();
    myLongRunningTask?.Dispose();
    myVariableObserver?.Dispose();
}
```

### Stop() ordering during page navigation

When navigating from **SCREEN1** to **SCREEN2**, the runtime prioritizes loading the new page to minimize time-to-first-content for the user. As a consequence, **`Stop()` on SCREEN1's NetLogics may execute *after* `Start()` has already been called on SCREEN2's NetLogics**.

This is by design: the destruction and cleanup of nodes belonging to the previous page is intentionally deferred in favor of the new page becoming interactive as quickly as possible.

> [!WARNING]
> Do not assume that all `Stop()` calls from the previous page have completed before `Start()` begins on the new page. If SCREEN1 and SCREEN2 share any global/static state, session variables, or external resources, this overlap can cause unexpected behavior.

```
Page transition timeline:

User behavior:      [Navigate to SCREEN2]
                    ↑ user clicks on the NavigationPanel tab

Behavior thread:        [Start NL_A (SCREEN2)][Start NL_B (SCREEN2)]
                        ↑ new page load starts first

                            [Stop NL_A (SCREEN1)][Stop NL_B (SCREEN1)]
                            ↑ old page cleanup follows asynchronously
```

Practical guidelines:
- **Do not rely on SCREEN1's `Stop()` having run** before SCREEN2's `Start()` initializes shared resources.
- **Release exclusive resources** (serial ports, file handles, database connections) as early as possible within `Stop()`, and acquire them as late as possible in `Start()`, to minimize the overlap window.
- If a clean handoff is required, use a shared session variable or a global synchronization flag to signal that SCREEN1 has fully torn down before SCREEN2 proceeds with the conflicting initialization.

## ExportMethod execution

Methods decorated with `[ExportMethod]` are called synchronously on **whichever thread triggered them**:

| Trigger | Thread used |
|---------|-------------|
| Button click / UI interaction | UI thread |
| Called from another NetLogic's `Start()` | Behavior thread |
| Called from a `LongRunningTask` | That task's dedicated thread |
| Called from a `PeriodicTask` | That task's dedicated thread |

```csharp
// BAD: called by a button click, freezes the UI for 5 seconds
[ExportMethod]
public void GenerateReport()
{
    Thread.Sleep(5000); // UI is completely unresponsive during this
    WriteReportFile();
}

// GOOD: return immediately, do the work in background
[ExportMethod]
public void GenerateReport()
{
    reportTask?.Dispose();
    reportTask = new LongRunningTask(WriteReportFile, LogicObject);
    reportTask.Start();
}

private LongRunningTask reportTask;
```

### ExportMethod and thread safety

When an `[ExportMethod]` runs on a thread different from the one that owns a shared variable, concurrent access can occur. The safest pattern is to keep all model-node reads/writes inside the method call itself (the FT Optix information model is thread-safe at the node level)

```csharp
// Sharing a plain C# object between a PeriodicTask and an ExportMethod: use a lock
private readonly object _lock = new object();
private List<double> _buffer = new();

private void SampleData()
{
    lock (_lock)
    {
        _buffer.Add(GetSensorValue());
    }
}

[ExportMethod]
public void FlushBuffer()
{
    List<double> snapshot;
    lock (_lock)
    {
        snapshot = new List<double>(_buffer);
        _buffer.Clear();
    }
    // Work on snapshot safely
}
```

## Pages with many widgets or NetLogics

### What happens at page load

When a page is navigated to, the runtime:

1. Instantiates and renders all graphical widgets on the **UI thread** (independent, non-blocking for the behavior thread)
2. Traverses the page subtree on the **behavior thread** and calls all `Start()` methods in priority order - **sequentially**

These two activities happen **concurrently** but on separate threads. The UI may already be partially visible while NetLogics are still starting up.

### Practical implications for complex pages

- **Many NetLogics + fast `Start()` methods**: no meaningful issue. Each `Start()` completes in milliseconds and the total startup time is the sum of all of them.
- **One slow `Start()`**: blocks the startup of every NetLogic that follows it in the queue. Use `LongRunningTask` to move the slow work off the behavior thread.
- **Many heavy widgets**: the UI thread handles all rendering; a heavy widget (e.g., a large DataGrid populating from a database) may cause the UI to feel sluggish, but it does not block `Start()` on other NetLogics.
  - Having multiple heavy widget might cause the Start() of the NetLogic to be delayed until the node is created, so multiple heavy widgets can cause a longer delay before the some NetLogic starts.

```
Page load timeline (simplified):

UI thread:     [render widget 1][render widget 2]...[render widget N]
Behavior thread:  [Start NL_A][Start NL_B]...[Start NL_N]
                   ↑ these are sequential, not parallel
```

### Avoid cascading Start() delays

If you have, for example, 10 NetLogics each doing a 300 ms blocking operation in `Start()`, total startup time is **3 seconds** during which any NetLogic not yet started is completely inert. This can cause:
- Variables remaining at their default value longer than expected
- Observers not yet registered, missing initial state changes
- Timers not yet running

Offload anything non-trivial to `LongRunningTask`:

```csharp
public override void Start()
{
    // Register observers immediately (fast)
    myVar = LogicObject.GetVariable("MyVar");
    observer = new RemoteVariableSynchronizer();
    observer.Add(myVar);
    observer.VariableChange += OnVarChanged;

    // Defer heavy initialization
    initTask = new LongRunningTask(HeavyInit, LogicObject);
    initTask.Start();
}

private void HeavyInit()
{
    // Connect to database, read config file, etc.
}
```

## Async tasks and their threads

For a detailed reference, see [async-tasks.md](./async-tasks.md). Quick summary:

| Task type | Creates a new thread? | Can access info model? | When to use |
|-----------|----------------------|----------------------|-------------|
| `LongRunningTask` | Yes, dedicated | Yes (thread-safe) | Long/blocking operations |
| `PeriodicTask` | Yes, dedicated | Yes (thread-safe) | Recurring work (polling, watchdog) |
| `DelayedTask` | Yes, dedicated | Yes (thread-safe) | One-shot deferred action |
| `async/await` (C# native) | Uses ThreadPool | **No** - avoid accessing nodes | Pure computation, I/O without node access |

> [!NOTE]
> All FT Optix task types (`LongRunningTask`, `PeriodicTask`, `DelayedTask`) are [information-model](./information-model.md)-safe. Native C# `async/await` or `Task.Run()` should only be used for operations that do not touch any project node.

## DesignTime NetLogic and ExportMethod

DesignTime NetLogics follow a different loading strategy:
- The .NET assembly is **reloaded from disk on every `[ExportMethod]` call** (to pick up freshly compiled changes)
- `Start()` is **never called** for DesignTime NetLogics (and, it is not even created by the IDE when adding a new DesignTime NetLogic, the instance is only created when an `[ExportMethod]` is invoked)
- `[ExportMethod]` runs synchronously on the thread that triggered it (typically the Studio UI thread) - this means blocking operations inside a DesignTime `[ExportMethod]` will freeze Optix Studio until it returns.
  - A DesignTime NetLogic can have multiple `[ExportMethod]`s, but they are all executed only on when the user invokes them by the context menu
  - Asynchronous operations are allowed using `LongRunningTask`, but they will still execute on the same project, so it is advisable not to change the project tree or access project nodes from them to avoid concurrency issues with the IDE thread
