---
layout: post
title: Tracing Linux Kernel Execution with ftrace on Yocto QEMU ARM64
date: 2026-10-23
tag: [linux, yocto, ftrace]
categories: Linux Kernel
---

## 1. Introduction

Understanding what happens inside the Linux kernel is essential when debugging embedded systems, especially when investigating performance bottlenecks, unexpected interrupts, scheduling behavior, or device driver issues.

Traditional debugging tools such as GDB allow developers to inspect kernel execution, but breakpoints can interrupt timing-sensitive operations.

Linux provides **ftrace**, a built-in tracing framework that records kernel execution without requiring the system to stop at every function call.

In this experiment, I explore ftrace on an ARM64 Linux system built with Yocto and running under QEMU.

The focus is on the `function` and `function_graph` tracers, with particular attention to kernel execution paths and interrupt handling.

## 2. Test Environment

| Component         | Configuration                |
| ----------------- | ---------------------------- |
| Host              | Fedora Linux (x86_64)        |
| Yocto             | 5.0 Scarthgap                |
| Machine           | qemuarm64                    |
| Kernel            | Linux 6.6.151-yocto-standard |
| Virtual platform  | QEMU ARM64 virt              |
| CPU               | 4 × Cortex-A57               |
| Memory            | 2 GiB                        |
| Tracing interface | tracefs                      |

Start the QEMU environment:

```bash
runqemu qemuarm64 nographic
```

All subsequent tracing commands are executed inside the QEMU guest as root.

## 3. Enabling ftrace in the Kernel

The following kernel configuration options are relevant:

```ini
CONFIG_FTRACE=y
CONFIG_FUNCTION_TRACER=y
CONFIG_FUNCTION_GRAPH_TRACER=y
```

Depending on the kernel architecture and configuration, additional options such as `CONFIG_DYNAMIC_FTRACE` may be enabled.

For Yocto, these options can be managed through a kernel configuration fragment.

After building and booting the kernel, verify the configuration:

```bash
zcat /proc/config.gz | grep -E \
'CONFIG_(FTRACE|FUNCTION_TRACER|FUNCTION_GRAPH_TRACER)='
```

If `/proc/config.gz` is unavailable, inspect the generated kernel `.config` in the Yocto build directory.

## 4. Accessing the ftrace Interface

Modern Linux kernels expose tracing controls through **tracefs**.

Mount it if necessary:

```bash
mount -t tracefs nodev /sys/kernel/tracing
cd /sys/kernel/tracing
```

Check the available tracers:

```bash
cat available_tracers
```

An example output might be:

```text
blk function_graph function nop
```

The exact list depends on the kernel configuration.

Important control files include:

| File                         | Purpose                             |
| ---------------------------- | ----------------------------------- |
| `available_tracers`          | Lists supported tracers             |
| `current_tracer`             | Selects the active tracer           |
| `tracing_on`                 | Enables or disables recording       |
| `trace`                      | Reads the trace buffer              |
| `trace_pipe`                 | Streams and consumes trace events   |
| `set_ftrace_filter`          | Filters traced functions            |
| `set_graph_function`         | Selects function graph entry points |
| `available_filter_functions` | Lists traceable functions           |

The `trace` interface reads a snapshot of the ring buffer, whereas `trace_pipe` consumes records as they are read.

## 5. Tracing Kernel Function Calls

The simplest function tracer records function entry events.

Enable it (`vfs_read`):

```bash
cd /sys/kernel/tracing

echo 0 > tracing_on
echo nop > current_tracer

echo > set_ftrace_filter
echo > set_graph_function
echo vfs_read > set_ftrace_filter

echo function > current_tracer
echo > trace
echo 1 > tracing_on

cat /proc/version > /dev/null

echo 0 > tracing_on
cat trace
```

A function trace record contains information such as:

```text
# tracer: function
#
# entries-in-buffer/entries-written: 54/54   #P:4
#
#                                _-----=> irqs-off/BH-disabled
#                               / _----=> need-resched
#                              | / _---=> hardirq/softirq
#                              || / _--=> preempt-depth
#                              ||| / _-=> migrate-disable
#                              |||| /     delay
#           TASK-PID     CPU#  |||||  TIMESTAMP  FUNCTION
```

For example, a record might look like:

```text
              sh-344     [003] .....  2033.435763: vfs_read <-ksys_read
             cat-361     [001] .....  2033.928046: vfs_read <-ksys_read
```

This example is illustrative rather than captured from the test system.

The function tracer records which functions are entered and can show the calling function. However, it does not provide the same nested execution-time view as `function_graph`.

## 6. Using the Function Graph Tracer

The `function_graph` tracer records function entry and return events, making it possible to reconstruct nested function calls and estimate their execution duration.

Enable it (`vfs_read`):

```bash
echo 0 > tracing_on
echo nop > current_tracer

echo > set_ftrace_filter
echo > set_graph_function
echo vfs_read > set_graph_function

echo function_graph > current_tracer
echo > trace
echo 1 > tracing_on

cat /proc/version > /dev/null

echo 0 > tracing_on
cat trace
```

A simplified example:

```text
3)               |  vfs_read() {
3)               |    rw_verify_area() {
3)               |      security_file_permission() {
3)   4.496 us    |        __fsnotify_parent();
3) + 15.088 us   |      }
3) + 20.560 us   |    }
3)   2.720 us    |    __get_task_ioprio();
3)               |    tty_read() {
3)               |      tty_ldisc_ref_wait() {
3)   2.048 us    |        ldsem_down_read();
3)   6.176 us    |      }
3)               |      n_tty_read() {
3)               |        __tty_check_change() {
3)               |          __tty_check_change.part.0() {
3)   1.824 us    |            __rcu_read_lock();
3)   2.160 us    |            _raw_spin_lock_irqsave();
3)   2.112 us    |            _raw_spin_unlock_irqrestore();
3)   1.984 us    |            __rcu_read_unlock();
3) + 19.328 us   |          }
3) + 24.192 us   |        }
3)   2.704 us    |        mutex_lock_interruptible();
3)   2.608 us    |        down_read();
3)               |        add_wait_queue() {
3)   2.032 us    |          _raw_spin_lock_irqsave();
3)   2.096 us    |          _raw_spin_unlock_irqrestore();
3) + 11.008 us   |        }
3)   3.824 us    |        copy_from_read_buf();
3)               |        n_tty_check_unthrottle() {
3)   2.432 us    |          n_tty_kick_worker();
3)               |          tty_unthrottle_safe() {
3)   2.336 us    |            mutex_lock();
3)   2.464 us    |            mutex_unlock();
3) + 11.744 us   |          }
3) + 22.160 us   |        }
3)   2.128 us    |        n_tty_kick_worker();
3)   2.400 us    |        up_read();
3)               |        remove_wait_queue() {
3)   2.144 us    |          _raw_spin_lock_irqsave();
3)   2.144 us    |          _raw_spin_unlock_irqrestore();
3) + 10.304 us   |        }
3)   2.176 us    |        mutex_unlock();
3) ! 116.080 us  |      }
3)               |      tty_ldisc_deref() {
3)   2.064 us    |        ldsem_up_read();
3)   6.128 us    |      }
3)               |      tty_update_time() {
3)   2.016 us    |        ktime_get_real_seconds();
3)   2.160 us    |        _raw_spin_lock();
3)   2.272 us    |        _raw_spin_unlock();
3) + 18.080 us   |      }
3) ! 161.600 us  |    }
3)   2.160 us    |    __fsnotify_parent();
3) ! 216.080 us  |  }
```

The output resembles nested C function calls.

The duration is displayed in microseconds.

Unlike a static call graph, ftrace records actual runtime execution paths.

This is especially useful when a function takes longer than expected and we need to identify which nested operations contribute to its execution time.

## 7. Filtering Specific Functions

Tracing the entire kernel produces a large amount of data and adds overhead.

A more practical approach is to focus on a particular execution path.

First, check whether a function is traceable:

```bash
grep -w vfs_read available_filter_functions
```

For the function tracer, select a function:

```bash
echo 0 > tracing_on
echo vfs_read > set_ftrace_filter
echo function > current_tracer
echo > trace
echo 1 > tracing_on

cat /proc/version > /dev/null

echo 0 > tracing_on
cat trace
```

For the function graph tracer, use `set_graph_function` to trace a function and its descendants:

```bash
echo 0 > tracing_on
echo nop > current_tracer

echo > set_ftrace_filter
echo > set_graph_function

echo vfs_read > set_graph_function
echo function_graph > current_tracer
echo > trace
echo 1 > tracing_on

cat /proc/version > /dev/null

echo 0 > tracing_on
cat trace
```

This approach makes the output much easier to interpret.

The function name must be present in `available_filter_functions`; compiler optimization and inlining can affect which functions are traceable.

### Making a Function Available for Tracing

Before tracing a kernel function, check whether it is listed in `available_filter_functions`:

```bash
grep -w vfs_read available_filter_functions
```

If the function is available, it can be selected for tracing.

For the `function` tracer:

```bash
echo vfs_read > set_ftrace_filter
echo function > current_tracer
```

For the `function_graph` tracer:

```bash
echo vfs_read > set_graph_function
echo function_graph > current_tracer
```

However, not every kernel function is necessarily traceable.

A function may be missing from `available_filter_functions` for several reasons:

- **Inlining:** The compiler may inline the function, eliminating its standalone entry point.
- **Compiler optimization:** Functions may be optimized away or transformed.
- **Instrumentation exclusions:** Functions marked with `notrace` or otherwise excluded from ftrace instrumentation cannot be traced directly.
- **Kernel configuration:** Function tracing support may not be enabled in the kernel build.

To make a function traceable, first ensure that the kernel is built with function tracing support:

```ini
CONFIG_FTRACE=y
CONFIG_FUNCTION_TRACER=y
CONFIG_FUNCTION_GRAPH_TRACER=y
```

For a function in kernel source code that is being optimized or inlined, it may be necessary to prevent inlining:

```c
noinline int my_function(void)
{
    /* Function implementation */
    return 0;
}
```

The kernel must then be rebuilt with the modified source code.

The `noinline` attribute prevents inlining but does not guarantee that the function will appear in `available_filter_functions`. It must also be eligible for ftrace instrumentation.

Functions deliberately excluded from instrumentation, such as those marked `notrace`, should not have that restriction removed without understanding the reason. Some low-level functions must remain uninstrumented to avoid recursion or other tracing problems.

**Important:** `available_filter_functions` is not a list of functions currently being traced. It lists functions that the running kernel can potentially trace. Selecting a function through `set_ftrace_filter` or `set_graph_function` does not modify the kernel binary.

## 8. Investigating the ARM64 Idle Path

In an earlier QEMU ARM64 debugging experiment, I inspected the kernel's idle execution path using GDB.

Functions of interest included:

```text
cpu_startup_entry
do_idle
default_idle_call
```

ftrace provides another way to investigate this behavior while the system is running.

First, inspect available functions:

```bash
grep -E '^(do_idle|default_idle_call)$' \
  available_filter_functions
```

If `do_idle` is traceable:

```bash
cd /sys/kernel/tracing

echo 0 > tracing_on
echo nop > current_tracer
echo 0 > events/enable

# Check whether CPU idle tracepoints are available
ls events/power/cpu_idle

# Enable CPU idle events
echo 1 > events/power/cpu_idle/enable

echo > trace
echo 1 > tracing_on

sleep 2

echo 0 > tracing_on
cat trace
```

This can reveal which functions execute inside the idle path and how interrupts affect execution.

However, the function graph tracer does not necessarily represent time spent in low-power states in the same way as CPU idle trace events.

For deeper CPU idle analysis, tracepoints such as `power:cpu_idle` may be more appropriate.

### Analyzing CPU Idle Trace Events

The `power:cpu_idle` tracepoint was successfully captured on all four ARM64 virtual CPUs.

A sample from the actual trace:

```text
          <idle>-0 [000] ... 2977.287427: cpu_idle: state=1 cpu_id=0
          <idle>-0 [001] ... 2977.287508: cpu_idle: state=1 cpu_id=1
          <idle>-0 [000] ... 2977.291034: cpu_idle: state=4294967295 cpu_id=0
          <idle>-0 [001] ... 2977.291041: cpu_idle: state=4294967295 cpu_id=1
```

The `state` field indicates the idle state transition:

- `state=1`: Enter idle state 1.
- `state=4294967295`: Exit the idle state (`-1` represented as an unsigned 32-bit integer).

For CPU 0, the idle residency time is approximately:

```text
2977.291034 - 2977.287427 = 0.003607 seconds
                              = 3.607 ms
```

Across the captured trace, all four virtual CPUs repeatedly entered and exited idle, with average recorded idle intervals of approximately 3.7–3.8 ms.

This demonstrates that tracepoints can reveal CPU idle behavior without instrumenting the entire kernel function call graph.

However, these events alone do not identify the exact wake-up source. Correlating CPU idle events with timer and IRQ tracepoints would provide further insight into why the virtual CPUs wake up.

## 9. Correlating CPU Idle with Interrupt Activity

In the previous experiment, all four ARM64 virtual CPUs repeatedly entered and exited idle states. Many of the observed idle intervals lasted approximately 3–4 milliseconds.

However, the `power:cpu_idle` tracepoint only tells us **when a CPU enters or exits an idle state**. It does not directly identify what caused the CPU to wake up.

To investigate further, we can combine CPU idle events with IRQ tracepoints.

### 9.1 Enabling CPU Idle and IRQ Tracepoints

First, reset the tracing configuration to avoid interference from previous experiments:

```bash
cd /sys/kernel/tracing

echo 0 > tracing_on
echo nop > current_tracer
echo 0 > events/enable

echo > set_ftrace_filter
echo > set_graph_function

# Enable CPU idle events
echo 1 > events/power/cpu_idle/enable

# Enable IRQ handler events
echo 1 > events/irq/irq_handler_entry/enable
echo 1 > events/irq/irq_handler_exit/enable

# Start tracing
echo > trace
echo 1 > tracing_on

sleep 1

# Stop tracing and save the result
echo 0 > tracing_on
cat trace > /tmp/idle-irq-trace.txt
```

This configuration captures three types of events:

| Tracepoint              | Purpose                                        |
| ----------------------- | ---------------------------------------------- |
| `power:cpu_idle`        | Records CPU idle state transitions             |
| `irq:irq_handler_entry` | Records the beginning of a traced IRQ handler  |
| `irq:irq_handler_exit`  | Records the completion of a traced IRQ handler |

Unlike the `function` and `function_graph` tracers, tracepoints record specific kernel events without tracing every function call.

### 9.2 Analyzing the Actual IRQ Trace

The combined trace revealed a strong correlation between CPU idle transitions and ARM64 architectural timer interrupts.

For example, CPU 2 produced the following sequence:

```text
          <idle>-0       [002] d..1.   459.402537: cpu_idle: state=4294967295 cpu_id=2
          <idle>-0       [002] d.h1.   459.402568: irq_handler_entry: irq=11 name=arch_timer
          <idle>-0       [002] d.h1.   459.402719: irq_handler_exit: irq=11 ret=handled
          <idle>-0       [002] d..1.   459.402744: cpu_idle: state=1 cpu_id=2
```

The CPU exited its idle state, executed the `arch_timer` interrupt handler, and returned to idle.

The recorded interval between IRQ handler entry and exit was approximately 151 microseconds, while the entire interval outside the idle state was approximately 207 microseconds.

Repeated observations showed timer-related wake-ups occurring roughly every four milliseconds across the virtual CPUs.

The trace also captured an inter-processor interrupt on CPU 3:

```text
          <idle>-0       [003] d.h1.   459.395684: irq_handler_entry: irq=2 name=IPI
          <idle>-0       [003] dNh1.   459.395839: irq_handler_exit: irq=2 ret=handled
```

This demonstrates that the same tracing mechanism can observe both architectural timer interrupt handlers and certain SMP inter-processor interrupt handlers.

**Key finding:** By correlating `power:cpu_idle` and `irq:irq_handler_*` tracepoints, we identified architectural timer interrupts as a likely cause of the periodic idle exits observed in the QEMU ARM64 environment.

The trace establishes a strong temporal relationship, although it does not independently prove every wake-up source or measure pure hardware interrupt latency.

### 9.3 Understanding ARM64 Interrupt Handling

On ARM64, hardware interrupts are typically delivered through the Generic Interrupt Controller (GIC).

A simplified interrupt handling path is:

```text
ARM64 CPU
    |
    v
Exception Vector
    |
    v
ARM64 IRQ Entry
    |
    v
GIC Interrupt Handling
    |
    v
Linux IRQ Subsystem
    |
    v
Registered IRQ Handler
```

In our QEMU ARM64 experiment, the trace captured two interrupt handlers:

- **`arch_timer` (IRQ 11):** Handles architectural timer interrupts, which were strongly correlated with the periodic CPU idle exits.
- **`IPI` (IRQ 2):** Handles inter-processor interrupts used for communication between CPUs in an SMP system.

These IRQ numbers are Linux virtual IRQ identifiers, not necessarily GIC hardware interrupt IDs.

The `irq_handler_entry` and `irq_handler_exit` tracepoints are emitted around registered IRQ handler execution. They do not capture the complete exception entry and exit path.

For example, the time between a hardware interrupt being signaled and the corresponding `irq_handler_entry` event is not directly measured by these tracepoints.

This distinction is important when analyzing interrupt latency: **IRQ handler execution time is not the same as total interrupt response latency.**

### 9.4 What We Learned

By combining CPU idle and IRQ tracepoints, we were able to correlate periodic CPU wake-ups with interrupt handler activity on our QEMU ARM64 system.

The experiment revealed several important findings:

- All four virtual CPUs repeatedly entered and exited idle states.
- The idle transitions occurred at intervals of approximately 4 milliseconds.
- The `arch_timer` interrupt handler (IRQ 11) was consistently observed immediately after CPUs exited idle.
- An inter-processor interrupt (IPI, IRQ 2) was also captured, demonstrating interrupt activity associated with SMP operation.
- The time spent executing an IRQ handler is different from the total interrupt response latency.

These observations demonstrate how multiple ftrace tracepoints can be combined to investigate kernel behavior beyond individual function calls.

Although the trace strongly suggests that architectural timer interrupts are responsible for the periodic wake-ups, identifying the exact wake-up mechanism and measuring end-to-end interrupt latency would require further investigation.

## 10. Understanding the ftrace Ring Buffer

ftrace stores tracing events in an in-memory ring buffer. Each CPU has its own tracing buffer, allowing events to be recorded with relatively low synchronization overhead.

When tracing generates more data than the buffer can hold, older events may be overwritten in the default overwrite mode.

This is particularly relevant to our CPU idle experiment, where four virtual CPUs generated frequent idle transitions and interrupt events.

### 10.1 Checking the Buffer Configuration

The current buffer size can be inspected with:

```bash
cd /sys/kernel/tracing

cat buffer_size_kb
cat buffer_total_size_kb
```

`buffer_size_kb` specifies the buffer size per CPU, while `buffer_total_size_kb` reports the total allocated buffer size across CPUs.

To increase the buffer size to 4 MiB per CPU:

```bash
echo 4096 > buffer_size_kb
```

With four CPUs, this requests approximately 16 MiB of tracing buffer space in total.

### 10.2 Checking for Lost Events

The `per_cpu/cpu*/stats` files provide information about tracing buffer usage and event loss:

```bash
cat per_cpu/cpu0/stats
```

Useful fields include:

- `entries`: Number of events currently stored in the buffer.
- `overrun`: Number of events lost because older entries were overwritten.
- `dropped events`: Events discarded when recording could not proceed, such as in non-overwrite mode.

These statistics help determine whether the captured trace is complete enough for analysis.

### 10.3 Reading the Trace Buffer

There are two common ways to read trace data:

```bash
cat trace
```

This provides a snapshot of the trace without consuming the stored events.

Alternatively:

```bash
cat trace_pipe
```

`trace_pipe` streams events as they become available and consumes them from the buffer.

For continuous collection, the output can be redirected to a file:

```bash
cat trace_pipe > /tmp/kernel-trace.log
```

The command continues running until interrupted.

For short experiments, stopping tracing and reading `trace` is usually sufficient. For longer experiments, streaming through `trace_pipe` can help avoid losing older events due to buffer overwrites.

### 10.4 Practical Considerations

When collecting high-frequency events such as CPU idle transitions or IRQ activity:

- Increase the buffer size when necessary.
- Keep the tracing window as short as practical.
- Check per-CPU statistics for overwritten or dropped events.
- Disable unnecessary tracepoints to reduce tracing overhead.
- Save the captured data before shutting down QEMU.

The ftrace ring buffer makes kernel tracing efficient, but buffer capacity and tracing overhead must be considered when interpreting the results.

## 11. ftrace vs. GDB vs. Kdump

These tools solve different debugging problems.

| Tool          | Primary purpose                                       | Stops execution?      |
| ------------- | ----------------------------------------------------- | --------------------- |
| GDB           | Interactive debugging, registers, memory, breakpoints | Usually               |
| ftrace        | Runtime function and event tracing                    | No, but adds overhead |
| Kdump + crash | Postmortem kernel crash analysis                      | Runs after a crash    |

For example:

- Use **GDB** to investigate register values and instruction-level behavior during driver initialization.
- Use **ftrace** to identify unexpected execution paths or long-running kernel functions.
- Use **kdump** to preserve system state after a fatal kernel error.

They complement rather than replace each other.

## 12. Limitations and Practical Considerations

ftrace is powerful, but it has several limitations.

**Tracing overhead:** Instrumenting too many functions can affect system timing and performance.

**Limited visibility:** Inlined functions and functions excluded from instrumentation may not appear in the trace.

**Buffer capacity:** Trace records may be overwritten if the buffer fills.

**Timing accuracy:** Function graph durations include instrumentation overhead and may be affected by interrupts, preemption, and scheduling.

**Crash persistence:** Normal ftrace buffers are stored in RAM and are not automatically preserved across a reboot. Persistent tracing requires additional mechanisms.

For performance analysis, it is important to minimize tracing scope and compare results under controlled conditions.

## 13. Conclusion

This experiment provides a practical introduction to Linux kernel tracing on a Yocto-based ARM64 virtual platform.

The key capabilities explored include:

- Enabling ftrace through kernel configuration.
- Recording function entry events.
- Reconstructing nested function calls using `function_graph`.
- Filtering specific kernel execution paths.
- Investigating CPU idle and interrupt behavior.
- Understanding trace buffers and tracing overhead.

Together with GDB and kdump, ftrace provides another important tool for debugging embedded Linux systems.

The next step is to apply these techniques to a real driver or subsystem, such as interrupt handling, device initialization, or DMA completion latency.

## References

- [Linux Kernel Documentation — ftrace](https://docs.kernel.org/trace/ftrace.html)
- [Linux Kernel 6.6 — ftrace Documentation](https://www.kernel.org/doc/html/v6.6/trace/ftrace.html)
- [Yocto Project — Kernel Development](https://docs.yoctoproject.org/kernel-dev/common.html)
