# JFR command checklist

Use this checklist when shell access is available and the user asks for a runtime diagnosis grounded in `.jfr` artifacts.

The exact command set should stay proportional to the question. For a quick answer, prefer the smallest subset that proves or disproves the main hypothesis. For a report or comparison, collect the full baseline.

> **Tip:** Use `jfr view --verbose <view-name> <file>` on any view to see the underlying query. Use `--width`, `--cell-height`, and `--truncate` to control output formatting for readability.

## 1. Recording coverage and metadata

Use these first:

```bash
jfr summary run.jfr
```

Look for:

- recording duration
- event counts
- whether the file is obviously partial relative to the log timeline

If logs mention profiler startup settings, capture them as well. Pay attention to bounded recordings such as `maxsize`.

To list all available views in the recording:

```bash
jfr view all-views run.jfr
```

## 2. GC and memory pressure

Recommended commands:

```bash
jfr view gc-pauses run.jfr
jfr print --events jdk.ResidentSetSize run.jfr
jfr print --events jdk.GCHeapSummary run.jfr
jfr print --events jdk.GarbageCollection run.jfr
jfr print --events jdk.GCConfiguration run.jfr
```

Extract:

- total GC pauses
- pause time, P99, and max pause
- dominant GC causes
- peak RSS
- peak heap used and committed size
- GC algorithm and heap configuration

When the output is large, use targeted filtering to compute peaks or cause counts.

## 3. Native memory (off-heap)

Recommended commands (JDK 21+):

```bash
jfr view native-memory-committed run.jfr
jfr view native-memory-reserved run.jfr
jfr print --events jdk.NativeMemoryUsage run.jfr
```

Use them to answer:

- is off-heap memory growing independently of heap
- which NMT categories dominate: thread stacks, code cache, metaspace, direct buffers, or internal
- does native memory growth correlate with specific application phases

Note: Fine-grained NMT data requires the JVM to be started with `-XX:NativeMemoryTracking=detail`.

## 4. CPU, threads, and scheduling

Recommended commands:

```bash
jfr view cpu-load run.jfr
jfr view hot-methods run.jfr
jfr view thread-cpu-load run.jfr
jfr view thread-allocation run.jfr
jfr view thread-count run.jfr
```

Use them to answer:

- was CPU saturated or not
- which methods consumed the most CPU samples
- which threads allocated the most
- how many threads were active throughout the recording
- whether concurrency changed the distribution of work or the root cause

For raw CPU profiling samples:

```bash
jfr print --events jdk.ExecutionSample run.jfr
```

If thread count or lock behavior matters, inspect the relevant events available in the recording rather than guessing from CPU load alone.

## 5. Allocation hotspots

Recommended commands:

```bash
jfr view allocation-by-class run.jfr
jfr view allocation-by-site run.jfr
jfr view allocation-by-thread run.jfr
jfr view tlabs run.jfr
```

Use them to identify:

- dominant classes, especially `byte[]`, `char[]`, `String`, and large framework objects
- dominant allocation sites
- per-thread allocation distribution
- TLAB waste and refill frequency
- whether the profile points to application code, framework code, or both

If framework methods dominate, walk up the stack and correlate them with the first meaningful application-owned method.

## 6. Lock contention and I/O

Recommended commands for lock analysis:

```bash
jfr print --events jdk.JavaMonitorEnter run.jfr
jfr print --events jdk.JavaMonitorWait run.jfr
jfr print --events jdk.ThreadPark run.jfr
```

Recommended commands for I/O analysis:

```bash
jfr print --events jdk.FileRead run.jfr
jfr print --events jdk.FileWrite run.jfr
jfr print --events jdk.SocketRead run.jfr
jfr print --events jdk.SocketWrite run.jfr
```

Use them to identify:

- monitors with high contention (long wait times or high entry counts)
- threads blocked on I/O for disproportionate durations
- whether latency is caused by lock convoys, I/O waits, or both

## 7. Virtual thread diagnostics (JDK 21+)

Recommended commands:

```bash
jfr view pinned-threads run.jfr
jfr print --events jdk.VirtualThreadPinned run.jfr
jfr print --events jdk.VirtualThreadSubmitFailed run.jfr
```

Use them to identify:

- carrier thread pinning events, their duration, and their stack traces
- whether pinning is caused by `synchronized` blocks (JDK 21–23) or native method calls
- failed virtual thread submissions indicating carrier pool exhaustion

On JDK 21–23, the most common fix for pinning is replacing `synchronized` with `ReentrantLock` on code paths that perform blocking I/O. On JDK 24+, `synchronized` no longer causes pinning (JEP 491).

## 8. Safepoints and JIT compilation

Recommended commands:

```bash
jfr view safepoints run.jfr
jfr view compiler-statistics run.jfr
```

Use them to identify:

- stop-the-world pauses that are not GC-related
- whether safepoint frequency is distorting CPU profiling samples (safepoint bias)
- JIT compilation overhead during warmup

If CPU profiles seem biased, use `-XX:+UnlockDiagnosticVMOptions -XX:+DebugNonSafepoints` for future recordings to reduce safepoint sampling bias.

## 9. Exception analysis

Recommended commands:

```bash
jfr view exception-count run.jfr
jfr print --events jdk.JavaExceptionThrow run.jfr
```

Use them to identify:

- whether exception creation rate is high enough to create measurable overhead (stack trace filling is expensive)
- which exception types dominate and whether they represent expected control flow or genuine errors

## 10. Container-aware diagnostics (JDK 21+)

Recommended commands:

```bash
jfr view container-memory-usage run.jfr
jfr view container-configuration run.jfr
jfr view network-utilization run.jfr
```

Use them to answer:

- what are the cgroup memory and CPU limits
- is JVM RSS approaching the container memory limit (OOM-kill risk)
- does the JVM correctly detect container constraints (`-XX:+UseContainerSupport`)
- is network I/O a bottleneck in containerized microservice architectures

## 11. JMC HTML corroboration

When an automated Java Mission Control report is available as HTML, inspect sections such as:

- Threads Allocating
- Allocated Classes
- Method Profiling
- GC Pressure
- GC Stall
- Application Halts
- CPU Load

Use the HTML to corroborate or challenge the raw JFR interpretation. Do not let it outrank direct evidence from the recording itself.

## 12. Comparison rules for multiple runs

Before comparing runs:

1. Confirm the workload is equivalent.
2. Confirm the recordings have comparable coverage.
3. Capture the same metric set for each run.
4. Compare both absolute values and deltas.

Good comparison dimensions:

- wall-clock duration
- step duration or stage duration from logs
- peak RSS
- peak heap used and committed size
- native memory committed (if available)
- GC pause count and total time
- pause percentile and max pause
- top allocation classes and sites
- thread allocation distribution
- average JVM CPU load
- hot methods by CPU sample count
- virtual thread pinning events (if applicable)
- exception rates

If one run is partial, keep it in the report but label it as non-symmetric evidence.

## 13. Code correlation checklist

After identifying hotspots, read code around the implicated methods and ask:

1. Is this path on the critical data path or just a cleanup path?
2. Is the expensive behavior required for correctness or only for convenience?
3. Is a resilience guarantee involved, such as restart safety or idempotency?
4. Is the hot path dominated by copying, re-encoding, or oversized state snapshots?
5. Is a framework or storage engine amplifying an application-owned design?
6. Is there lock contention or I/O blocking that explains the latency independently of CPU or allocation?
7. Are virtual threads pinned inside this code path?

Your diagnosis should name:

- the root cause
- the amplifier
- the ruled-out suspects

## 14. Reporting checklist

A good final report should make these explicit:

- what artifacts were analyzed
- what is complete versus partial
- what metrics support the conclusion
- what code paths explain the metrics
- what uncertainty remains
- what changes are likely to reduce the problem without breaking the guarantee the code was trying to preserve
- a confidence level for each conclusion (High, Medium, Low, or Ruled Out)
