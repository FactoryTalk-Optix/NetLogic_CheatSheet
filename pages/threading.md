# FT Optix Runtime – Threading Model

## Overview

The FT Optix Runtime is built around a **shared, priority-based thread pool** combined with an **affinity-based task serializer**. This design allows many independent subsystems to execute concurrently without requiring each one to own a dedicated OS thread, while still giving every behavior object a strong single-threaded execution guarantee.

Understanding this model is important when reasoning about the relationship between **CPU clock speed** and **CPU core count** on overall application performance.



## Core Thread Pools

### Main dispatch pool

At startup, the runtime creates one shared thread pool containing **one worker thread per logical CPU** (determined via `std::thread::hardware_concurrency()`). On a 4-core/8-thread CPU this gives 8 workers; on a 32-core/64-thread server it gives 64.

All asynchronous callbacks for behavior objects - variable change notifications, OPC UA events, and user-defined dispatch calls - are served from this single pool.

The pool uses a six-level priority queue, so time-sensitive work is always promoted ahead of background tasks:

| Priority | Intended use |
|---|---|
| Blocking | Tasks that must start immediately to unblock a caller |
| Highest | Critical-path control actions |
| High | Time-sensitive HMI updates |
| Normal | Default for most dispatched work |
| Low | Background data logging, analytics |
| Lowest | Maintenance sweeps |

### Timer pool

One-shot and periodic timers are managed by a separate timer subsystem backed by its own **fixed-size thread pool** (independent of the main dispatch pool). This ensures that a slow or blocked timer callback cannot starve other dispatched work, and vice versa.

Timers are added programmatically and are represented by a RAII handle: dropping the handle cancels the timer. A blocking removal API is also available that waits for any currently executing callback to finish before returning.

## Affinity-Based Dispatch

### The affinity concept

Every behavior object in the runtime is assigned a unique numeric **affinity ID** at construction time. The task scheduler guarantees that **no two tasks sharing the same affinity ID ever run concurrently**, regardless of how many pool threads are available.

This is the primary concurrency safety mechanism in the runtime. Behavior code can access its own state without locks because only one task bearing its affinity ID will ever run at a time. Mutexes are only required when data is shared *across* affinity boundaries.

### Task lifecycle

```
  Trigger (variable change / OPC UA event / timer expiry)
           │
           ▼
    Dispatch(callback, affinityId)
           │
      ┌────┴─────┐
      │ Is the   │ Yes ──► park task until current task for
      │ affinity │         this affinity completes
      │ busy?    │
      └────┬─────┘
           │ No
           ▼
    Mark affinity as "in-flight"
           │
           ▼
    Submit to thread pool ──► picked up by an idle worker thread
           │
           ▼
    Execute callback on pool thread
           │
           ▼
    On completion: promote next parked task for this affinity (if any)
```

### Shutdown guards

Three lifecycle guards coordinate ordered shutdown of a behavior:

| Guard | Effect |
|---|---|
| Suspend dispatch | New tasks queue up; the current in-flight task runs to completion |
| Suspend on stop | Same, but auto-resumes when the guard scope exits |
| Terminate on stop | All queued tasks for this affinity are abandoned when the guard scope exits |

## Per-Subsystem Dedicated Threads

Beyond the shared pool, several subsystems own their own dedicated OS thread for specialized I/O serialization:

| Subsystem | Thread count | Purpose |
|---|---|---|
| Communication driver scheduler | One per configured driver station | Runs timed poll tasks in priority order (read/write priority levels); also handles on-demand requests enqueued from other threads |
| Persistent store I/O | One per store node | Serializes all database reads and writes; exceptions propagate back to the calling thread synchronously |
| UI rendering queue | One | Batch consumer queue for layout and rendering operations that must not block OPC UA callbacks |
| Write batching | One | Collects write values from any thread, batches them, and forwards them on its own thread to reduce protocol round-trips |
| Store-and-forward retry | Timer-driven | Uses the timer subsystem for flush and retry intervals; no permanently blocked thread |

## Communication Driver Threading

Each configured **driver station** runs its own scheduler thread. This thread:

- Maintains a sorted queue of tasks keyed by **priority level** (Read / Write) and **next execution time**.
- Pulls the next due task, executes it, then reschedules it if it is periodic (configured with a period in milliseconds).
- Accepts on-demand tasks (single-execution jobs) from any thread via an enqueue API; these are prioritized above or below cyclic poll tasks depending on their priority level.
- Transitions through a communication state machine (connecting → connected → error → reconnecting) independently of any other station's thread.

Because each station has its own thread, **N configured stations require N dedicated threads** regardless of pool size. Stations are fully isolated: a network hang on station A does not affect the scheduler thread of station B.

### Threads vs CPU cores - over-subscription

Having more driver stations than CPU cores is normal and supported. A scheduler thread spends the vast majority of its time **sleeping** (blocked on a timed wait until the next task is due). Sleeping threads do not consume a CPU slot - the OS only schedules them onto a core when they have actual work to do. Twenty stations on an 8-core machine is a common and well-functioning configuration.

Over-subscription only becomes noticeable when **many stations have work ready at the same instant**:

- If N stations all have a short-period poll task that fires simultaneously, and N is greater than the number of logical CPUs, the OS will time-slice. Some stations will be delayed by one OS scheduling quantum while waiting for a free CPU slot.
- The magnitude of this delay is typically in the range of a few hundred microseconds to a few milliseconds, depending on OS and system load.
- This introduces **jitter** into the actual poll period - the configured period is a minimum interval, not a hard real-time guarantee.

**When this matters in practice:**

| Scenario | Impact |
|---|---|
| Poll periods ≥ 100 ms (typical SCADA/HMI) | Negligible - a 1–2 ms jitter on a 500 ms cycle is imperceptible |
| Poll periods 10–100 ms with many stations | May cause occasional late scans under load; monitor with task execution counters |
| Poll periods < 10 ms with many stations | Requires careful sizing; use more physical cores or reduce the number of concurrently active stations |

**Mitigations if over-subscription causes issues:**

- Increase the number of physical CPU cores.
- Stagger station start times to spread wake-up patterns naturally (stations started at different times will tend to drift into different scheduling slots).
- Increase poll periods where sub-100 ms scanning is not strictly required.
- Run the runtime process at elevated OS scheduling priority to reduce competition from other processes on the same machine.

## NetLogic Execution Model

### Default execution (short-lived operations)

C# NetLogic methods (`Start()`, UA method callbacks) execute on a **pool thread from the main dispatch pool**, within the affinity context of the owning NetLogic node. This means:

- `Start()` and method callbacks are serialized with respect to that node - no locks are needed for node-local state.
- A `Start()` or method callback that runs longer than **2 seconds** triggers a runtime warning in the log.
- Any computation or I/O that may take longer must be explicitly offloaded to a `LongRunningTask`.

### Long-running tasks

`LongRunningTask` allocates a **dedicated OS thread** for the duration of the task (using .NET's `TaskCreationOptions.LongRunning`). This prevents slow user code from occupying a shared pool thread and impacting other nodes.

```csharp
public override void Start()
{
    longTask = new LongRunningTask(RunSlowOperation, LogicObject);
    longTask.Start();
}

private void RunSlowOperation(LongRunningTask task)
{
    // Runs on its own dedicated OS thread.
    // Use task.IsCancellationRequested to cooperate with Stop().
    while (!task.IsCancellationRequested)
    {
        // ... polling loop, database query, external API call, etc.
    }
}

public override void Stop()
{
    longTask?.Dispose(); // requests cancellation and waits for the thread to exit
}
```

**Important:** `LongRunningTask` instances must always be disposed in `Stop()`. Failing to do so will leave an OS thread running after the node is stopped.

## Thread Count at a Glance

```
Main dispatch pool:      hardware_concurrency threads  (all behavior callbacks)
Comm driver scheduler:   1 thread per station          (protocol polling & writes)
Persistent store I/O:    1 thread per store node       (DB serialization)
UI rendering queue:      1 thread                      (layout / paint batching)
Write batching:          1 thread                      (output batching)
LongRunningTask (.NET):  1 thread per active instance  (user long-running logic)
```

## How the Application Benefits from More CPU Threads vs Higher Clock Speed

### Workloads that scale with more CPU cores

Because the main dispatch pool is sized to `hardware_concurrency`, adding more logical CPUs directly increases the number of behavior callbacks, timer tasks, and OPC UA events that can execute simultaneously.

| Workload | Why more cores help |
|---|---|
| **Many independent HMI nodes with data subscriptions** | Each affinity can be scheduled on a different thread. With 100 nodes and 8 cores, up to 8 nodes update in parallel; with 16 cores, up to 16 do. |
| **Multiple communication driver stations** | Each station has its own dedicated scheduler thread. More cores prevent those threads from time-sharing with pool workers. |
| **Multiple store nodes** | Each store node has its own I/O thread. Concurrent database operations proceed in parallel. |
| **Many active `LongRunningTask` instances** | Each occupies a dedicated OS thread. More physical cores allow more of them to run truly in parallel. |
| **High-frequency periodic timers** | The timer pool has a fixed number of threads. If many timers fire at the same millisecond, more idle cores prevent queuing delays. |

**Rule of thumb:** if the application has more independent behavior objects, driver stations, or long-running tasks than the CPU has threads, adding more cores will improve throughput approximately proportionally - until another bottleneck (network, storage, database latency) takes over.

### Workloads that scale with higher clock speed

Some execution paths are natively serial and cannot be parallelized at the dispatch level:

| Workload | Why higher clock helps |
|---|---|
| **Single "hot" behavior receiving events faster than it can process them** | Clock speed directly reduces per-callback latency. Task queue depth stays near zero. |
| **NetLogic `Start()` or method with heavy computation** | These run sequentially within their affinity. A computationally intensive method benefits from more MHz, not more cores. |
| **Communication driver scan cycle** | Each station runs on a single thread. Heavy protocol decoding or tag-mapping logic is bounded by single-thread performance. |
| **Persistent store transactions** | One thread per store node. Reducing query or write time (clock speed, fast SSD) reduces transaction latency. |
| **OPC UA Server session management** | Session-level state is managed on dedicated internal threads. Reducing per-session processing time allows more concurrent clients without queuing. |
| **Low-latency periodic timers (< 10 ms)** | Short-period timers are sensitive to OS scheduling jitter and CPU cache performance, both of which improve with higher base clock. |

### Practical guidance

| Scenario | Recommended focus |
|---|---|
| Faster HMI screen load and rendering | Higher clock speed |
| Multiple HMI sessions (WebPresentationEngine clients) | More cores but with some limitations (see the [hot-path issue](#the-hot-path-issue)) |
| Many communication driver stations | More cores (each needs its own thread) |
| Heavy per-event computation in NetLogic | Higher clock speed |
| Fast scan-rate polling (1–10 ms) | Higher clock + real-time OS configuration |
| Large data logger write throughput | More cores AND fast storage I/O |
| Typical mixed HMI application | 4–8 fast cores is the sweet spot; beyond ~8 cores gives diminishing returns unless the project has a very large number of independent subsystems (see the [hot-path issue](#the-hot-path-issue)) |

### The hot-path issue

A significant benefit of a multi-core CPU is on applications having many independent modules (HMI sessions, driver stations, data stores, long-running tasks). In these cases, adding cores increases throughput as the different modules are assigned to different threads. 

If multiple modules are contending for the same set of items (same PLC tag, same store, etc.), then adding cores will not help, and the application will be limited by the single-threaded performance of the hot path.

### Hyper-Threading / SMT

The main pool creates one thread per **logical** CPU. On a system with Hyper-Threading, two logical threads share one physical core's execution units. Because most runtime pool tasks involve a mix of memory accesses and computation, Hyper-Threading typically provides a 20–40 % throughput improvement over the equivalent physical core count without it. However, Hyper-Threading does **not** reduce latency for a single hot path - only additional physical cores do.

## Key Design Principles

1. **One main pool, many affinities.** All behavior callbacks share the same thread pool. Isolation is achieved through affinity serialization, not through private threads per behavior.
2. **Timer callbacks are isolated from dispatch callbacks.** A blocking timer never starves the main pool.
3. **Communication drivers are fully isolated from each other.** Each station has its own scheduler thread; a network fault on one station does not affect others.
4. **Long user code must be explicit.** NetLogic that takes more than a few milliseconds must be moved to a `LongRunningTask`. Short callbacks keep the pool responsive for all other nodes.
5. **Hardware concurrency drives pool size.** Adding CPU cores directly translates to pool thread count with no configuration change required.
