---
name: java-jfr-diagnostics
description: Analyze Java runtime behavior using `.jfr` recordings, application logs, and optional Java Mission Control HTML exports. Use when asked to inspect JFRs, compare runs, diagnose memory, GC, CPU, thread scheduling, virtual thread pinning, native memory, lock contention, or allocation issues, or write an evidence-based performance report from runtime artifacts. Also use for container-aware JVM diagnostics. Do NOT use for non-Java profiling, benchmark design, or optimization advice without recordings or logs.
license: CC-BY-4.0
metadata:
  author: Erik da Rosa Rodriguez (erik@rodgz.com)
  version: '2.0.0'
---

# Java JFR Diagnostics

Use this skill to turn Java runtime artifacts into a defensible diagnosis. The goal is not to paraphrase profiler output; the goal is to establish what happened, how confident the evidence is, and which code paths most plausibly caused it.

## Instructions

### Step 1: Inventory and validate the artifacts

Start by listing the available evidence:

- application logs
- one or more `.jfr` recordings
- optional Java Mission Control HTML exports
- optional code paths or tests already suspected by the user

Before diagnosing anything, establish whether the artifacts are complete and comparable:

1. Match scenario names, dates, and run timestamps across files.
2. Compare log start time and job duration with the JFR recording start and duration.
3. Check whether the recording was truncated by ring buffer settings such as `maxsize`.
4. Extract runtime configuration that materially affects interpretation, such as concurrency, `-Xmx`, GC, environment mode, and workload size.
5. Identify the JDK version to determine which JFR events and views are available.
6. If the application runs in a container, note cgroup memory limits and CPU quotas.

If one recording is partial, say so explicitly and downgrade it to supporting evidence instead of treating it as a symmetric comparison baseline.

### Step 2: Build a baseline from raw JFR data

Use the `jfr` CLI as the primary evidence source. Before running commands, read [references/command-checklist.md](references/command-checklist.md) and execute the smallest set needed for the available artifacts.

For each recording, extract enough data to answer these questions:

- How long did the recording cover?
- What was the peak RSS and peak heap usage?
- How much time was lost to GC, and what were the dominant causes?
- Which classes, sites, and threads dominated allocation?
- Was the pressure mostly heap, native, CPU, thread scheduling, lock contention, or I/O?
- What were the hottest methods by CPU sample count?
- Was native (off-heap) memory growing independently of heap?

Do not stop at one summary view. Cross-check aggregate views with event-level details before forming conclusions.

When formatting `jfr` output for analysis, use `--width`, `--cell-height`, and `--truncate` flags to keep output concise and parseable. Use `--verbose` on any view to inspect the underlying query when results look unexpected.

### Step 3: Use logs to establish execution semantics

Logs tell you what the program thought it was doing. Use them to capture:

- start and end timestamps
- job and step durations
- workload counts
- concurrency configuration
- retries, failures, and fallbacks
- warnings about profiler setup or runtime configuration

Use exact values and timestamps when comparing scenarios. If the user says "today" or "the latest run" but the files point to a specific date, anchor the report on the concrete date from the artifacts.

### Step 4: Use JMC HTML as corroboration, not as the only source

If Java Mission Control exported an automated HTML report, use it to quickly validate or challenge the raw JFR readout.

Prioritize sections such as:

- Threads Allocating
- Allocated Classes
- Method Profiling
- GC Pressure
- GC Stall or Concurrent Mode failures
- Application Halts
- CPU Load

Treat the HTML as corroborating evidence. If it conflicts with raw JFR data, trust the raw recording and explain the discrepancy.

### Step 5: Correlate hotspots with code and tests

Map the hottest allocation sites, stack frames, or methods back to the codebase.

Read enough surrounding code to classify each hotspot as one of:

- root cause
- amplifier
- secondary victim or symptom
- unrelated but noisy

Look for implementation patterns that commonly explain JFR findings:

**Allocation and memory patterns:**

- repeated full-buffer copies
- Base64 or JSON re-encoding loops
- large `String` or `byte[]` churn
- oversized chunk or batch state
- checkpoint or persistence of heavy blobs
- caches with weak eviction discipline
- large object graphs created for convenience, not necessity
- framework or database layers amplifying an application-level design choice

**Concurrency and scheduling patterns:**

- thread pool exhaustion or queue backpressure from unbounded work submission
- lock ordering inversions or convoy effects from coarse-grained synchronization
- virtual thread carrier pinning from `synchronized` blocks during blocking I/O (JDK 21–23)
- connection pool starvation masquerading as slow downstream services

**Lifecycle and resource leak patterns:**

- classloader leaks from retained references across redeployment boundaries
- finalizer or `Cleaner` queue buildup delaying native resource release
- unclosed I/O streams or database connections accumulating under error paths

**Profiling artifact patterns:**

- safepoint bias distorting CPU sample distribution toward methods near safepoint polls
- JIT compilation churn inflating early-run method profiling data

If resilience or restart behavior seems to justify a heavy design, inspect tests and documentation. Separate the real requirement from the accidental implementation detail.

### Step 6: Build an explicit causal model

State the diagnosis as a chain of causality, not as disconnected facts.

Your answer should make these distinctions explicit:

- symptom: what the JVM exhibited
- mechanism: what kind of allocations, pauses, or contention created it
- code path: where that mechanism lives
- root cause: the design or behavior that repeatedly triggers it
- amplifier: infrastructure or framework behavior that makes it worse
- non-cause: suspects ruled out by the evidence

Examples:

- "Concurrency redistributed allocation across worker threads, but the peak heap stayed the same."
- "The embedded database appears in the profile, but its allocation share is too small to be the primary driver."
- "The restart requirement is real; the Base64 representation is not."
- "Virtual threads are pinned inside `synchronized` JDBC calls, starving the carrier pool. The fix is to replace `synchronized` with `ReentrantLock`, not to reduce virtual thread count."
- "Native memory grew 3× faster than heap, pointing to a direct buffer leak rather than GC pressure."

### Step 7: Produce a report that is clear about confidence and limits

When the user asks for a report, structure it around evidence and decisions. Use the template in [references/report-template.md](references/report-template.md) as a starting point:

1. Executive summary
2. Scope and limitations
3. Artifact inventory
4. Comparative evidence by scenario
5. Technical diagnosis tied to code
6. Root cause versus amplifiers versus ruled-out suspects
7. Recommended next steps

Use tables when comparing multiple runs. Quantify deltas. Name files and scenarios exactly. If some artifacts are incomplete, say what they support and what they cannot prove.

#### Confidence-level heuristics

Assign one of the following confidence levels to each conclusion in the report, and justify the assignment:

| Level | Criteria |
|:---|:---|
| **High** | Multiple independent evidence sources agree (JFR + logs + code inspection). The causal chain is complete from symptom to root cause. No contradicting data. |
| **Medium** | Two evidence sources agree, or one strong source (raw JFR) supports the conclusion, but one link in the causal chain is inferred rather than directly observed. |
| **Low** | Only one source provides weak signal. Key data is missing, partial, or contradictory. The conclusion is plausible but should be validated with a targeted experiment. |
| **Ruled out** | The evidence explicitly contradicts this hypothesis. State what evidence disproved it. |

When recommending changes, preserve whatever guarantee the code was designed to protect, such as restart safety, ordering, or multipart correctness. Do not recommend deleting safeguards until you have identified the real requirement they serve.

### Step 8: Container and environment-aware adjustments

When the application runs inside a container (Docker, Kubernetes, etc.), apply these additional checks:

1. Use `jfr view container-memory-usage` and `container-configuration` to capture cgroup limits.
2. Compare JVM RSS against the container memory limit. If RSS approaches the limit, the OOM killer may terminate the process before GC can recover.
3. Check whether `-XX:+UseContainerSupport` is active and whether `-XX:MaxRAMPercentage` is set. If not, the JVM may be ignoring cgroup constraints.
4. Compare JVM CPU load against container CPU quota. A 2-core quota on a 64-core host will throttle differently than what `jfr view cpu-load` suggests.
5. Use `jfr view network-utilization` when I/O-bound behavior is suspected in containerized microservices.

## Examples

### Example 1: Single-run memory diagnosis

User says: "Analyze this JFR and tell me why the process hit 12 GB of RAM."

Actions:

1. Check whether the `.jfr` covers the full run and whether logs include JVM flags or workload size.
2. Extract RSS, heap peaks, GC causes, top allocation classes, top allocation sites, and thread allocation.
3. Check native memory with `jfr view native-memory-committed` to distinguish heap from off-heap growth.
4. Map the hottest allocation sites to code.
5. Distinguish heap-driven growth from native or off-heap growth.
6. Report the likely root cause and the confidence level.

Result: A diagnosis that says whether the memory peak was caused by application heap churn, native memory, framework behavior, or an external factor.

### Example 2: Sequential versus concurrent comparison

User says: "Compare these sequential and concurrent runs and tell me if concurrency caused the memory issue."

Actions:

1. Confirm both runs processed the same workload.
2. Extract comparable runtime, RSS, heap, GC, CPU, and allocation metrics from both recordings.
3. Use logs to capture exact concurrency settings and step timings.
4. Compare which metrics changed and which stayed flat.
5. Explain whether concurrency changed the cause or only changed the shape of the pressure.

Result: A report that quantifies what concurrency improved, what it did not change, and whether the memory issue is fundamentally independent of thread count.

### Example 3: Resilience-driven hotspot analysis

User says: "Is this expensive code path technically necessary for restart safety?"

Actions:

1. Identify the hot methods from JFR and JMC.
2. Read the implementation, related persistence model, and restart tests.
3. Separate the genuine resilience requirement from the current storage or encoding format.
4. Explain which parts must remain and which parts are implementation choices.

Result: A diagnosis that preserves the operational guarantee while still identifying what can be refactored away.

### Example 4: Virtual thread pinning diagnosis

User says: "My virtual thread application is slower than expected. Analyze the JFR."

Actions:

1. Check for `jdk.VirtualThreadPinned` events using `jfr print --events jdk.VirtualThreadPinned run.jfr`.
2. Use `jfr view pinned-threads` to identify which carrier threads are blocked and for how long.
3. Check for `jdk.VirtualThreadSubmitFailed` events indicating carrier pool exhaustion.
4. Trace pinning stack frames back to `synchronized` blocks or native method calls.
5. If JDK version is 21–23, recommend replacing `synchronized` with `ReentrantLock` for blocking operations. If JDK 24+, check for native method calls instead.

Result: A diagnosis identifying whether pinning is the bottleneck, which code paths cause it, and the migration path.

### Example 5: Container OOM investigation

User says: "Our pods keep getting OOM-killed but the JVM heap looks fine."

Actions:

1. Compare container memory limit from `jfr view container-configuration` against JVM RSS from `jfr print --events jdk.ResidentSetSize`.
2. Check `jfr view native-memory-committed` for off-heap growth.
3. Verify `-XX:+UseContainerSupport` and `-XX:MaxRAMPercentage` settings.
4. Look for direct buffer allocation, memory-mapped files, or JNI-allocated memory outside heap control.
5. Check if metaspace, code cache, or thread stacks are growing unconstrained.

Result: A diagnosis explaining the gap between heap usage and total RSS, with specific recommendations for memory budget allocation.

## Troubleshooting

### Problem: The JFR is partial or truncated

Cause: The recording used a bounded ring buffer or a size cap and no longer contains the beginning of the run.

What to do:

- Compare log timestamps with recording coverage.
- Use the partial recording only as supporting evidence.
- Avoid direct metric comparisons against full recordings unless you clearly label the asymmetry.

### Problem: The HTML report looks convincing but the raw JFR disagrees

Cause: The HTML is a derived summary with opinionated heuristics.

What to do:

- Fall back to raw JFR summaries and event-level inspection.
- Use the HTML only to reinforce or qualify the diagnosis.

### Problem: The hottest code path looks intentional

Cause: Performance-heavy code often protects a real guarantee such as restart safety, idempotency, or batching correctness.

What to do:

- Read tests and design docs before calling it a mistake.
- Diagnose the guarantee separately from the current implementation.

### Problem: The profile points at a framework class instead of application code

Cause: The framework is executing on behalf of an application-level design choice.

What to do:

- Walk up the stack to the first meaningful application-owned method.
- Identify whether the framework is the root cause, an amplifier, or just where the cost becomes visible.

### Problem: CPU profile seems biased toward unexpected methods

Cause: Safepoint bias. JFR CPU sampling can only sample threads at safepoints, which may over-represent methods that happen to contain safepoint polls (e.g., counted loops, method returns).

What to do:

- Cross-reference `jfr view hot-methods` with allocation sites and GC data.
- If a method appears hot but has no matching allocation or I/O cost, suspect safepoint bias.
- Use `jfr view safepoints` to check safepoint frequency and duration.
- Consider `-XX:+UnlockDiagnosticVMOptions -XX:+DebugNonSafepoints` for more accurate profiling.

### Problem: Virtual threads are slower than platform threads

Cause: Carrier thread pinning, excessive virtual thread creation, or scheduler contention.

What to do:

- Check `jfr print --events jdk.VirtualThreadPinned` for pinning events.
- Use `jfr view pinned-threads` to quantify the impact.
- Trace pinning to `synchronized` blocks or native calls.
- On JDK 21–23, migrate from `synchronized` to `ReentrantLock` for code paths that perform blocking I/O.
