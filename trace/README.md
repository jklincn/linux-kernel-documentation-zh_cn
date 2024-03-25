> 原文：[Linux Tracing Technologies — The Linux Kernel documentation](https://docs.kernel.org/trace/index.html)
>
> 译者：jklincn \<jklincn@outlook.com\>
>
> 日期：2024.03.23
>
> 修订：
>
> **已知问题：二级目录需要右键-在新标签页中打开链接才能正确跳转**

# Linux 跟踪技术

- Function Tracer Design
  - [Introduction](https://docs.kernel.org/trace/ftrace-design.html#introduction)
  - [Prerequisites](https://docs.kernel.org/trace/ftrace-design.html#prerequisites)
  - [HAVE_FUNCTION_TRACER](https://docs.kernel.org/trace/ftrace-design.html#have-function-tracer)
  - [HAVE_FUNCTION_GRAPH_TRACER](https://docs.kernel.org/trace/ftrace-design.html#have-function-graph-tracer)
  - [HAVE_FUNCTION_GRAPH_FP_TEST](https://docs.kernel.org/trace/ftrace-design.html#have-function-graph-fp-test)
  - [HAVE_FUNCTION_GRAPH_RET_ADDR_PTR](https://docs.kernel.org/trace/ftrace-design.html#have-function-graph-ret-addr-ptr)
  - [HAVE_SYSCALL_TRACEPOINTS](https://docs.kernel.org/trace/ftrace-design.html#have-syscall-tracepoints)
  - [HAVE_FTRACE_MCOUNT_RECORD](https://docs.kernel.org/trace/ftrace-design.html#have-ftrace-mcount-record)
  - [HAVE_DYNAMIC_FTRACE](https://docs.kernel.org/trace/ftrace-design.html#have-dynamic-ftrace)
  - [HAVE_DYNAMIC_FTRACE + HAVE_FUNCTION_GRAPH_TRACER](https://docs.kernel.org/trace/ftrace-design.html#have-dynamic-ftrace-have-function-graph-tracer)
- [使用事件和跟踪点分析行为的注意事项](tracepoint-analysis.md)
  - [1. 引言](tracepoint-analysis.md#1-引言)
  - [2. 列出可用事件](tracepoint-analysis.md#2-列出可用事件)
  - [3. 启用事件](tracepoint-analysis.md#3-启用事件)
  - [4. 事件过滤](tracepoint-analysis.md#4-事件过滤)
  - [5. 使用 PCL 分析事件差异](tracepoint-analysis.md#5-使用-pcl-分析事件差异)
  - [6. 使用辅助脚本分析高级事件](tracepoint-analysis.md#6-使用辅助脚本分析高级事件)
  - [7. 使用 PCL 进行低级分析](tracepoint-analysis.md#7-使用-pcl-进行低级分析)
- [ftrace —— 函数跟踪器](ftrace.md)
  - [Introduction](https://docs.kernel.org/trace/ftrace.html#introduction)
  - [Implementation Details](https://docs.kernel.org/trace/ftrace.html#implementation-details)
  - [The File System](https://docs.kernel.org/trace/ftrace.html#the-file-system)
  - [The Tracers](https://docs.kernel.org/trace/ftrace.html#the-tracers)
  - [Error conditions](https://docs.kernel.org/trace/ftrace.html#error-conditions)
  - [Examples of using the tracer](https://docs.kernel.org/trace/ftrace.html#examples-of-using-the-tracer)
  - [Output format:](https://docs.kernel.org/trace/ftrace.html#output-format)
  - [Latency trace format](https://docs.kernel.org/trace/ftrace.html#latency-trace-format)
  - [trace_options](https://docs.kernel.org/trace/ftrace.html#trace-options)
  - [irqsoff](https://docs.kernel.org/trace/ftrace.html#irqsoff)
  - [preemptoff](https://docs.kernel.org/trace/ftrace.html#preemptoff)
  - [preemptirqsoff](https://docs.kernel.org/trace/ftrace.html#preemptirqsoff)
  - [wakeup](https://docs.kernel.org/trace/ftrace.html#wakeup)
  - [wakeup_rt](https://docs.kernel.org/trace/ftrace.html#wakeup-rt)
  - [Latency tracing and events](https://docs.kernel.org/trace/ftrace.html#latency-tracing-and-events)
  - [Hardware Latency Detector](https://docs.kernel.org/trace/ftrace.html#hardware-latency-detector)
  - [function](https://docs.kernel.org/trace/ftrace.html#function)
  - [Single thread tracing](https://docs.kernel.org/trace/ftrace.html#single-thread-tracing)
  - [function graph tracer](https://docs.kernel.org/trace/ftrace.html#function-graph-tracer)
  - [dynamic ftrace](https://docs.kernel.org/trace/ftrace.html#dynamic-ftrace)
  - [Selecting function filters via index](https://docs.kernel.org/trace/ftrace.html#selecting-function-filters-via-index)
  - [Dynamic ftrace with the function graph tracer](https://docs.kernel.org/trace/ftrace.html#dynamic-ftrace-with-the-function-graph-tracer)
  - [ftrace_enabled](https://docs.kernel.org/trace/ftrace.html#ftrace-enabled)
  - [Filter commands](https://docs.kernel.org/trace/ftrace.html#filter-commands)
  - [trace_pipe](https://docs.kernel.org/trace/ftrace.html#trace-pipe)
  - [trace entries](https://docs.kernel.org/trace/ftrace.html#trace-entries)
  - [Snapshot](https://docs.kernel.org/trace/ftrace.html#snapshot)
  - [Instances](https://docs.kernel.org/trace/ftrace.html#instances)
  - [Stack trace](https://docs.kernel.org/trace/ftrace.html#stack-trace)
  - [More](https://docs.kernel.org/trace/ftrace.html#more)
- Using ftrace to hook to functions
  - [Introduction](https://docs.kernel.org/trace/ftrace-uses.html#introduction)
  - [The ftrace context](https://docs.kernel.org/trace/ftrace-uses.html#the-ftrace-context)
  - [The ftrace_ops structure](https://docs.kernel.org/trace/ftrace-uses.html#the-ftrace-ops-structure)
  - [The callback function](https://docs.kernel.org/trace/ftrace-uses.html#the-callback-function)
  - [Protect your callback](https://docs.kernel.org/trace/ftrace-uses.html#protect-your-callback)
  - [The ftrace FLAGS](https://docs.kernel.org/trace/ftrace-uses.html#the-ftrace-flags)
  - [Filtering which functions to trace](https://docs.kernel.org/trace/ftrace-uses.html#filtering-which-functions-to-trace)
- Fprobe - Function entry/exit probe
  - [Introduction](https://docs.kernel.org/trace/fprobe.html#introduction)
  - [The usage of fprobe](https://docs.kernel.org/trace/fprobe.html#the-usage-of-fprobe)
  - [The fprobe entry/exit handler](https://docs.kernel.org/trace/fprobe.html#the-fprobe-entry-exit-handler)
  - [Share the callbacks with kprobes](https://docs.kernel.org/trace/fprobe.html#share-the-callbacks-with-kprobes)
  - [The missed counter](https://docs.kernel.org/trace/fprobe.html#the-missed-counter)
  - [Functions and structures](https://docs.kernel.org/trace/fprobe.html#functions-and-structures)
- Kernel Probes (Kprobes)
  - [Concepts: Kprobes and Return Probes](https://docs.kernel.org/trace/kprobes.html#concepts-kprobes-and-return-probes)
  - [Architectures Supported](https://docs.kernel.org/trace/kprobes.html#architectures-supported)
  - [Configuring Kprobes](https://docs.kernel.org/trace/kprobes.html#configuring-kprobes)
  - [API Reference](https://docs.kernel.org/trace/kprobes.html#api-reference)
  - [Kprobes Features and Limitations](https://docs.kernel.org/trace/kprobes.html#kprobes-features-and-limitations)
  - [Probe Overhead](https://docs.kernel.org/trace/kprobes.html#probe-overhead)
  - [TODO](https://docs.kernel.org/trace/kprobes.html#todo)
  - [Kprobes Example](https://docs.kernel.org/trace/kprobes.html#kprobes-example)
  - [Kretprobes Example](https://docs.kernel.org/trace/kprobes.html#kretprobes-example)
  - [Deprecated Features](https://docs.kernel.org/trace/kprobes.html#deprecated-features)
  - [The kprobes debugfs interface](https://docs.kernel.org/trace/kprobes.html#the-kprobes-debugfs-interface)
  - [The kprobes sysctl interface](https://docs.kernel.org/trace/kprobes.html#the-kprobes-sysctl-interface)
  - [References](https://docs.kernel.org/trace/kprobes.html#references)
- Kprobe-based Event Tracing
  - [Overview](https://docs.kernel.org/trace/kprobetrace.html#overview)
  - [Synopsis of kprobe_events](https://docs.kernel.org/trace/kprobetrace.html#synopsis-of-kprobe-events)
  - [Types](https://docs.kernel.org/trace/kprobetrace.html#types)
  - [User Memory Access](https://docs.kernel.org/trace/kprobetrace.html#user-memory-access)
  - [Per-Probe Event Filtering](https://docs.kernel.org/trace/kprobetrace.html#per-probe-event-filtering)
  - [Event Profiling](https://docs.kernel.org/trace/kprobetrace.html#event-profiling)
  - [Kernel Boot Parameter](https://docs.kernel.org/trace/kprobetrace.html#kernel-boot-parameter)
  - [Usage examples](https://docs.kernel.org/trace/kprobetrace.html#usage-examples)
- Uprobe-tracer: Uprobe-based Event Tracing
  - [Overview](https://docs.kernel.org/trace/uprobetracer.html#overview)
  - [Synopsis of uprobe_tracer](https://docs.kernel.org/trace/uprobetracer.html#synopsis-of-uprobe-tracer)
  - [Types](https://docs.kernel.org/trace/uprobetracer.html#types)
  - [Event Profiling](https://docs.kernel.org/trace/uprobetracer.html#event-profiling)
  - [Usage examples](https://docs.kernel.org/trace/uprobetracer.html#usage-examples)
- Fprobe-based Event Tracing
  - [Overview](https://docs.kernel.org/trace/fprobetrace.html#overview)
  - [Synopsis of fprobe-events](https://docs.kernel.org/trace/fprobetrace.html#synopsis-of-fprobe-events)
  - [BTF arguments](https://docs.kernel.org/trace/fprobetrace.html#btf-arguments)
  - [Usage examples](https://docs.kernel.org/trace/fprobetrace.html#usage-examples)
- [使用 Linux 内核跟踪点](tracepoints.md)
  - [跟踪点的用途](tracepoints.md#跟踪点的用途)
  - [使用](tracepoints.md#使用)
- Event Tracing
  - [1. Introduction](https://docs.kernel.org/trace/events.html#introduction)
  - [2. Using Event Tracing](https://docs.kernel.org/trace/events.html#using-event-tracing)
  - [3. Defining an event-enabled tracepoint](https://docs.kernel.org/trace/events.html#defining-an-event-enabled-tracepoint)
  - [4. Event formats](https://docs.kernel.org/trace/events.html#event-formats)
  - [5. Event filtering](https://docs.kernel.org/trace/events.html#event-filtering)
  - [6. Event triggers](https://docs.kernel.org/trace/events.html#event-triggers)
  - [7. In-kernel trace event API](https://docs.kernel.org/trace/events.html#in-kernel-trace-event-api)
- Subsystem Trace Points: kmem
  - [1. Slab allocation of small objects of unknown type](https://docs.kernel.org/trace/events-kmem.html#slab-allocation-of-small-objects-of-unknown-type)
  - [2. Slab allocation of small objects of known type](https://docs.kernel.org/trace/events-kmem.html#slab-allocation-of-small-objects-of-known-type)
  - [3. Page allocation](https://docs.kernel.org/trace/events-kmem.html#page-allocation)
  - [4. Per-CPU Allocator Activity](https://docs.kernel.org/trace/events-kmem.html#per-cpu-allocator-activity)
  - [5. External Fragmentation](https://docs.kernel.org/trace/events-kmem.html#external-fragmentation)
- Subsystem Trace Points: power
  - [1. Power state switch events](https://docs.kernel.org/trace/events-power.html#power-state-switch-events)
  - [2. Clocks events](https://docs.kernel.org/trace/events-power.html#clocks-events)
  - [3. Power domains events](https://docs.kernel.org/trace/events-power.html#power-domains-events)
  - [4. PM QoS events](https://docs.kernel.org/trace/events-power.html#pm-qos-events)
- NMI Trace Events
  - [nmi_handler](https://docs.kernel.org/trace/events-nmi.html#nmi-handler)
- [MSR Trace Events](https://docs.kernel.org/trace/events-msr.html)
- In-kernel memory-mapped I/O tracing
  - [Preparation](https://docs.kernel.org/trace/mmiotrace.html#preparation)
  - [Usage Quick Reference](https://docs.kernel.org/trace/mmiotrace.html#usage-quick-reference)
  - [Usage](https://docs.kernel.org/trace/mmiotrace.html#usage)
  - [How Mmiotrace Works](https://docs.kernel.org/trace/mmiotrace.html#how-mmiotrace-works)
  - [Trace Log Format](https://docs.kernel.org/trace/mmiotrace.html#trace-log-format)
  - [Explanation Keyword Space-separated arguments](https://docs.kernel.org/trace/mmiotrace.html#explanation-keyword-space-separated-arguments)
  - [Tools for Developers](https://docs.kernel.org/trace/mmiotrace.html#tools-for-developers)
- Event Histograms
  - [1. Introduction](https://docs.kernel.org/trace/histogram.html#introduction)
  - [2. Histogram Trigger Command](https://docs.kernel.org/trace/histogram.html#histogram-trigger-command)
- Histogram Design Notes
  - ['hist_debug' trace event files](https://docs.kernel.org/trace/histogram-design.html#hist-debug-trace-event-files)
  - [Basic histograms](https://docs.kernel.org/trace/histogram-design.html#basic-histograms)
  - [Variables](https://docs.kernel.org/trace/histogram-design.html#variables)
  - [Actions and Handlers](https://docs.kernel.org/trace/histogram-design.html#actions-and-handlers)
  - [A couple special cases](https://docs.kernel.org/trace/histogram-design.html#a-couple-special-cases)
- Boot-time tracing
  - [Overview](https://docs.kernel.org/trace/boottime-trace.html#overview)
  - [Options in the Boot Config](https://docs.kernel.org/trace/boottime-trace.html#options-in-the-boot-config)
  - [When to Start](https://docs.kernel.org/trace/boottime-trace.html#when-to-start)
  - [Examples](https://docs.kernel.org/trace/boottime-trace.html#examples)
- Hardware Latency Detector
  - [Introduction](https://docs.kernel.org/trace/hwlat_detector.html#introduction)
  - [Usage](https://docs.kernel.org/trace/hwlat_detector.html#usage)
- OSNOISE Tracer
  - [Usage](https://docs.kernel.org/trace/osnoise-tracer.html#usage)
  - [Tracer Configuration](https://docs.kernel.org/trace/osnoise-tracer.html#tracer-configuration)
  - [Tracer Options](https://docs.kernel.org/trace/osnoise-tracer.html#tracer-options)
  - [Additional Tracing](https://docs.kernel.org/trace/osnoise-tracer.html#additional-tracing)
  - [Running osnoise tracer without workload](https://docs.kernel.org/trace/osnoise-tracer.html#running-osnoise-tracer-without-workload)
- Timerlat tracer
  - [Usage](https://docs.kernel.org/trace/timerlat-tracer.html#usage)
  - [Tracer options](https://docs.kernel.org/trace/timerlat-tracer.html#tracer-options)
  - [timerlat and osnoise](https://docs.kernel.org/trace/timerlat-tracer.html#timerlat-and-osnoise)
  - [IRQ stacktrace](https://docs.kernel.org/trace/timerlat-tracer.html#irq-stacktrace)
  - [User-space interface](https://docs.kernel.org/trace/timerlat-tracer.html#user-space-interface)
- Intel(R) Trace Hub (TH)
  - [Overview](https://docs.kernel.org/trace/intel_th.html#overview)
  - [Bus and Subdevices](https://docs.kernel.org/trace/intel_th.html#bus-and-subdevices)
  - [Quick example](https://docs.kernel.org/trace/intel_th.html#quick-example)
  - [Host Debugger Mode](https://docs.kernel.org/trace/intel_th.html#host-debugger-mode)
  - [Software Sinks](https://docs.kernel.org/trace/intel_th.html#software-sinks)
- Lockless Ring Buffer Design
  - [Terminology used in this Document](https://docs.kernel.org/trace/ring-buffer-design.html#terminology-used-in-this-document)
  - [The Generic Ring Buffer](https://docs.kernel.org/trace/ring-buffer-design.html#the-generic-ring-buffer)
  - [Making the Ring Buffer Lockless:](https://docs.kernel.org/trace/ring-buffer-design.html#making-the-ring-buffer-lockless)
  - [Nested writes](https://docs.kernel.org/trace/ring-buffer-design.html#nested-writes)
- System Trace Module
  - [stm_source](https://docs.kernel.org/trace/stm.html#stm-source)
  - [stm_console](https://docs.kernel.org/trace/stm.html#stm-console)
  - [stm_ftrace](https://docs.kernel.org/trace/stm.html#stm-ftrace)
- [MIPI SyS-T over STP](https://docs.kernel.org/trace/sys-t.html)
- CoreSight - ARM Hardware Trace
  - [Coresight - HW Assisted Tracing on ARM](https://docs.kernel.org/trace/coresight/coresight.html)
  - [CoreSight System Configuration Manager](https://docs.kernel.org/trace/coresight/coresight-config.html)
  - [Coresight CPU Debug Module](https://docs.kernel.org/trace/coresight/coresight-cpu-debug.html)
  - [Coresight Dummy Trace Module](https://docs.kernel.org/trace/coresight/coresight-dummy.html)
  - [CoreSight Embedded Cross Trigger (CTI & CTM).](https://docs.kernel.org/trace/coresight/coresight-ect.html)
  - [ETMv4 sysfs linux driver programming reference.](https://docs.kernel.org/trace/coresight/coresight-etm4x-reference.html)
  - [CoreSight - Perf](https://docs.kernel.org/trace/coresight/coresight-perf.html)
  - [The trace performance monitoring and diagnostics aggregator(TPDA)](https://docs.kernel.org/trace/coresight/coresight-tpda.html)
  - [Trace performance monitoring and diagnostics monitor(TPDM)](https://docs.kernel.org/trace/coresight/coresight-tpdm.html)
  - [Trace Buffer Extension (TRBE).](https://docs.kernel.org/trace/coresight/coresight-trbe.html)
  - [UltraSoc - HW Assisted Tracing on SoC](https://docs.kernel.org/trace/coresight/ultrasoc-smb.html)
- user_events: User-based Event Tracing
  - [Overview](https://docs.kernel.org/trace/user_events.html#overview)
  - [Registering](https://docs.kernel.org/trace/user_events.html#registering)
  - [Deleting](https://docs.kernel.org/trace/user_events.html#deleting)
  - [Unregistering](https://docs.kernel.org/trace/user_events.html#unregistering)
  - [Status](https://docs.kernel.org/trace/user_events.html#status)
  - [Writing Data](https://docs.kernel.org/trace/user_events.html#writing-data)
  - [Example Code](https://docs.kernel.org/trace/user_events.html#example-code)
- Runtime Verification
  - [Runtime Verification](https://docs.kernel.org/trace/rv/runtime-verification.html)
  - [Deterministic Automata](https://docs.kernel.org/trace/rv/deterministic_automata.html)
  - [Deterministic Automata Monitor Synthesis](https://docs.kernel.org/trace/rv/da_monitor_synthesis.html)
  - [Deterministic Automata Instrumentation](https://docs.kernel.org/trace/rv/da_monitor_instrumentation.html)
  - [Monitor wip](https://docs.kernel.org/trace/rv/monitor_wip.html)
  - [Monitor wwnr](https://docs.kernel.org/trace/rv/monitor_wwnr.html)
- HiSilicon PCIe Tune and Trace device
  - [Introduction](https://docs.kernel.org/trace/hisi-ptt.html#introduction)
  - [Tune](https://docs.kernel.org/trace/hisi-ptt.html#tune)
  - [Trace](https://docs.kernel.org/trace/hisi-ptt.html#trace)