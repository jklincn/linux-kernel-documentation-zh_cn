> 原文：[Notes on Analysing Behaviour Using Events and Tracepoints  — The Linux Kernel documentation](https://docs.kernel.org/trace/tracepoint-analysis.html)
>
> 译者：jklincn \<jklincn@outlook.com\>
>
> 日期：2024.03.23
>
> 修订：

# 使用事件和跟踪点分析行为的注意事项

## 1. 引言

跟踪点（见[使用 Linux 内核跟踪点](tracepoints.md)）可以在不创建自定义内核模块的情况下使用，通过事件跟踪基础结构来注册探针函数。

简单来说，跟踪点代表了一些重要事件，这些事件可以与其他跟踪点结合起来，构建出系统内部正在发生的事件的“宏图”。有很多方法可以收集和解释这些事件。鉴于目前缺乏最佳实践，本文档描述了一些可用的方法。

本文档假设 debugfs 已经挂载在 /sys/kernel/debug 上，并且内核已配置了适当的跟踪选项。假设 PCL 工具 tools/perf 已经安装并且在您的路径中。

## 2. 列出可用事件

### 2.1 标准工具

所有可能的事件都可以从 /sys/kernel/tracing/events 看到。简单地使用：

```shell
$ find /sys/kernel/tracing/events -type d
```

将给出可用活动数量的合理指示。

### 2.2 PCL（Linux 的性能计数器）

perf 工具提供了所有计数器和事件的发现和枚举，包括跟踪点。以下是获取可用事件的列表的一个简单例子：

```bash
$ perf list 2>&1 | grep Tracepoint
ext4:ext4_free_inode                     [Tracepoint event]
ext4:ext4_request_inode                  [Tracepoint event]
ext4:ext4_allocate_inode                 [Tracepoint event]
ext4:ext4_write_begin                    [Tracepoint event]
ext4:ext4_ordered_write_end              [Tracepoint event]
[ .... remaining output snipped .... ]
```

## 3. 启用事件

### 3.1 启用系统范围事件

有关如何在系统范围内启用事件的正确描述，请参阅[事件跟踪](https://docs.kernel.org/trace/events.html)。启用与页面分配相关的所有事件的简短示例如下：

```bash
$ for i in `find /sys/kernel/tracing/events -name "enable" | grep mm_`; do echo 1 > $i; done
```

### 3.2 使用 SystemTap 启用系统范围的事件

在 SystemTap 中，可以使用 kernel.trace() 函数调用来访问跟踪点。以下是一个示例，每隔 5 秒报告正在分配页面的进程。

```c
global page_allocs

probe kernel.trace("mm_page_alloc") {
      page_allocs[execname()]++
}

function print_count() {
      printf ("%-25s %-s\n", "#Pages Allocated", "Process Name")
      foreach (proc in page_allocs-)
              printf("%-25d %s\n", page_allocs[proc], proc)
      printf ("\n")
      delete page_allocs
}

probe timer.s(5) {
        print_count()
}
```

### 3.3 使用 PCL 启用系统范围的事件

通过指定 -a 开关并分析 sleep，可以检查一段时间内的系统范围事件。

```bash
$ perf stat -a \
       -e kmem:mm_page_alloc -e kmem:mm_page_free \
       -e kmem:mm_page_free_batched \
       sleep 10
Performance counter stats for 'sleep 10':

          9630  kmem:mm_page_alloc
          2143  kmem:mm_page_free
          7424  kmem:mm_page_free_batched

  10.002577764  seconds time elapsed
```

类似地，可以执行一个 shell 并在需要报告时退出。

### 3.4 启用本地事件

> 译者注：“系统范围事件”（system-wide events）和“本地事件”（local events）指的是事件数据收集的范围和侧重点的不同。
>
> - 系统范围事件指的是在整个操作系统范围内收集的事件数据。这意味着监控和分析的是所有进程、线程和核心在给定时间内的活动和性能指标。系统范围的数据收集对于理解操作系统的整体性能表现非常有用，比如 CPU 使用率、内存分配和释放、I/O 操作等。
>
> - 本地事件，有时也称为线程特定事件或进程特定事件，是针对特定线程或进程收集的事件数据。这种方式的监控和分析侧重于单个程序或服务的性能，允许开发者和性能分析师深入了解特定应用程序内部的行为，例如函数调用频率、特定代码段的执行时间、线程间的同步和通信等。

[ftrace - 函数跟踪器](ftrace.md)描述了如何使用 set_ftrace_pid 在每个线程的基础上启用事件。

### 3.5 使用 PCL 启用本地事件

通过使用 PCL，可以在本地基础上激活和跟踪一个进程的持续时间，如下所示。

```shell
$ perf stat -e kmem:mm_page_alloc -e kmem:mm_page_free \
               -e kmem:mm_page_free_batched ./hackbench 10
Time: 0.909

  Performance counter stats for './hackbench 10':

        17803  kmem:mm_page_alloc
        12398  kmem:mm_page_free
         4827  kmem:mm_page_free_batched

  0.973913387  seconds time elapsed
```

## 4. 事件过滤

[ftrace - 函数跟踪器](ftrace.md)深入介绍了如何在 ftrace 中过滤事件。显然，使用 trace_pipe 的 grep 和 awk 是一种选择，任何读取 trace_pipe 的脚本也是如此。

## 5. 使用 PCL 分析事件差异

任何工作负载都可能在运行之间出现差异，了解标准差是什么可能很重要。总的来说，这需要性能分析师手动完成。如果离散事件发生对性能分析师有用，则可以使用 perf 。

```shell
$ perf stat --repeat 5 -e kmem:mm_page_alloc -e kmem:mm_page_free
                      -e kmem:mm_page_free_batched ./hackbench 10
Time: 0.890
Time: 0.895
Time: 0.915
Time: 1.001
Time: 0.899

 Performance counter stats for './hackbench 10' (5 runs):

        16630  kmem:mm_page_alloc         ( +-   3.542% )
        11486  kmem:mm_page_free          ( +-   4.771% )
         4730  kmem:mm_page_free_batched  ( +-   2.325% )

  0.982653002  seconds time elapsed   ( +-   1.448% )
```

如果需要一些基于离散事件聚合的高级事件，则需要编写一个脚本。

使用 --repeat，也可以使用 -a 和 sleep 在系统范围内随时间查看事件如何波动。

```shell
$ perf stat -e kmem:mm_page_alloc -e kmem:mm_page_free \
              -e kmem:mm_page_free_batched \
              -a --repeat 10 \
              sleep 1
Performance counter stats for 'sleep 1' (10 runs):

         1066  kmem:mm_page_alloc         ( +-  26.148% )
          182  kmem:mm_page_free          ( +-   5.464% )
          890  kmem:mm_page_free_batched  ( +-  30.079% )

  1.002251757  seconds time elapsed   ( +-   0.005% )
```

## 6. 使用辅助脚本分析高级事件

当事件被激活时，可以以人类可读的格式（尽管也存在二进制选项）从  /sys/kernel/tracing/trace_pipe 读取正在触发的事件。通过对输出进行后处理，可以根据需要在线收集更多信息。后处理的示例可能包括：

- 从 /proc 读取触发事件的 PID 的信息
- 从一系列低级事件中派生出一个高级事件
- 计算两个事件之间的延迟

Documentation/trace/postprocess/trace-pagealloc-postprocess.pl 是一个示例脚本，可以从 STDIN 或一个跟踪的副本读取 trace_pipe 。在线使用时，可可以中断一次以生成报告而不退出，中断两次则退出。

> 译者注：这里的“在线”指在系统运行时进行数据分析或处理，而不是事后从已经收集的数据中进行分析。事件数据会被即时捕捉并处理，而不是存储下来之后再分析。可理解为“实时”。

简单来说，脚本只是读取 STDIN 并计算事件次数，但它也可以做更多事情，例如：

- 从许多低级事件中派生高级事件。如果从每 CPU 列表中有多个页面被释放到主分配器，它会将其识别为每个 CPU 的一次排空操作，即使该事件没有特定的跟踪点
- 它可以基于 PID 或单独的进程号进行聚合
- 在内存外部碎片化的事件中，它会报告碎片化事件是严重还是中等的
- 在接收到有关 PID 的事件时，它可以记录父进程是谁，以便如果大量事件来自运行时间非常短的进程，可以识别出负责创建所有辅助进程的父进程

## 7. 使用 PCL 进行低级分析

还有可能需要识别程序内部哪些函数正在在内核中生成事件。开始这类分析前，必须先记录数据。在撰写本文时，这需要 root 权限：

```shell
$ perf record -c 1 \
      -e kmem:mm_page_alloc -e kmem:mm_page_free \
      -e kmem:mm_page_free_batched \
      ./hackbench 10
Time: 0.894
[ perf record: Captured and wrote 0.733 MB perf.data (~32010 samples) ]
```

注意使用 “-c 1” 来设置事件采样周期。默认采样周期非常高，以最大限度地减少开销，但由此收集的信息可能会相当粗略。

这条记录输出了一个名为 perf.data 的文件，可以使用 perf report 进行分析。

```shell
$ perf report
# Samples: 30922
#
# Overhead    Command                     Shared Object
# ........  .........  ................................
#
    87.27%  hackbench  [vdso]
     6.85%  hackbench  /lib/i686/cmov/libc-2.9.so
     2.62%  hackbench  /lib/ld-2.9.so
     1.52%       perf  [vdso]
     1.22%  hackbench  ./hackbench
     0.48%  hackbench  [kernel]
     0.02%       perf  /lib/i686/cmov/libc-2.9.so
     0.01%       perf  /usr/bin/perf
     0.01%       perf  /lib/ld-2.9.so
     0.00%  hackbench  /lib/i686/cmov/libpthread-2.9.so
#
# (For more details, try: perf report --sort comm,dso,symbol)
#
```

据此，绝大多数事件在 VDSO 内触发。对于简单的二进制文件，通常会出现这种情况，所以我们来看一个稍微不同的例子。在撰写本文的过程中，注意到 X 正在产生大量的页面分配，所以让我们来看看它：

```shell
$ perf record -c 1 -f \
              -e kmem:mm_page_alloc -e kmem:mm_page_free \
              -e kmem:mm_page_free_batched \
              -p `pidof X`
```

几秒钟后中断，并执行

```shell
$ perf report
# Samples: 27666
#
# Overhead  Command                            Shared Object
# ........  .......  .......................................
#
    51.95%     Xorg  [vdso]
    47.95%     Xorg  /opt/gfx-test/lib/libpixman-1.so.0.13.1
     0.09%     Xorg  /lib/i686/cmov/libc-2.9.so
     0.01%     Xorg  [kernel]
#
# (For more details, try: perf report --sort comm,dso,symbol)
#
```

因此，几乎一半的事件发生在一个库中。为了弄清楚是哪个符号：

```shell
$ perf report --sort comm,dso,symbol
# Samples: 27666
#
# Overhead  Command                            Shared Object  Symbol
# ........  .......  .......................................  ......
#
    51.95%     Xorg  [vdso]                                   [.] 0x000000ffffe424
    47.93%     Xorg  /opt/gfx-test/lib/libpixman-1.so.0.13.1  [.] pixmanFillsse2
     0.09%     Xorg  /lib/i686/cmov/libc-2.9.so               [.] _int_malloc
     0.01%     Xorg  /opt/gfx-test/lib/libpixman-1.so.0.13.1  [.] pixman_region32_copy_f
     0.01%     Xorg  [kernel]                                 [k] read_hpet
     0.01%     Xorg  /opt/gfx-test/lib/libpixman-1.so.0.13.1  [.] get_fast_path
     0.00%     Xorg  [kernel]                                 [k] ftrace_trace_userstack
```

要查看 pixmanFillsse2 函数内部出现问题的位置：

```shell
$ perf annotate pixmanFillsse2
[ ... ]
  0.00 :         34eeb:       0f 18 08                prefetcht0 (%eax)
       :      }
       :
       :      extern __inline void __attribute__((__gnu_inline__, __always_inline__, _
       :      _mm_store_si128 (__m128i *__P, __m128i __B) :      {
       :        *__P = __B;
 12.40 :         34eee:       66 0f 7f 80 40 ff ff    movdqa %xmm0,-0xc0(%eax)
  0.00 :         34ef5:       ff
 12.40 :         34ef6:       66 0f 7f 80 50 ff ff    movdqa %xmm0,-0xb0(%eax)
  0.00 :         34efd:       ff
 12.39 :         34efe:       66 0f 7f 80 60 ff ff    movdqa %xmm0,-0xa0(%eax)
  0.00 :         34f05:       ff
 12.67 :         34f06:       66 0f 7f 80 70 ff ff    movdqa %xmm0,-0x90(%eax)
  0.00 :         34f0d:       ff
 12.58 :         34f0e:       66 0f 7f 40 80          movdqa %xmm0,-0x80(%eax)
 12.31 :         34f13:       66 0f 7f 40 90          movdqa %xmm0,-0x70(%eax)
 12.40 :         34f18:       66 0f 7f 40 a0          movdqa %xmm0,-0x60(%eax)
 12.31 :         34f1d:       66 0f 7f 40 b0          movdqa %xmm0,-0x50(%eax)
```

乍一看，似乎时间都花在了将 pixmap 复制到卡上。需要进一步调查为什么要如此频繁地复制 pixmap，但一个起点可能是从库路径中移除几个月前完全被遗忘的 libpixmap 的古老构建！
