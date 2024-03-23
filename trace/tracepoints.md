> 原文：[Using the Linux Kernel Tracepoints  — The Linux Kernel documentation](https://docs.kernel.org/trace/tracepoints.html)
>
> 译者：jklincn \<jklincn@outlook.com\>
>
> 日期：2024.03.23
>
> 修订：

# 使用 Linux 内核跟踪点

本文档介绍了 Linux 内核跟踪点及其使用。它提供了如何在内核中插入跟踪点并将探针函数连接到这些跟踪点的示例，并提供了一些探针函数的示例。

## 跟踪点的用途

放置在代码中的跟踪点提供了一个钩子来调用您在运行时提供的函数（探针）。一个跟踪点可以是“开”（连接了一个探针）或“关”（没有附加探针）。当跟踪点是“关”的时候，它除了添加少量的时间惩罚（检查分支的条件）和空间惩罚（在插入指令的函数末尾为函数调用添加几个字节，并在单独的部分中添加数据结构）外，没有任何效果。当跟踪点是“开”的时候，每次执行跟踪点时，都会在调用方的执行上下文中调用您提供的函数。当提供的函数结束执行时，它将返回给调用方（从跟踪点站点继续）。

您可以将跟踪点放在代码中的重要位置。它们是轻量级钩子，可以传递任意数量的参数，原型在头文件中的跟踪点声明中进行描述。

它们可以用于跟踪和性能统计。

## 使用

跟踪点需要两个元素：

- 一个跟踪点定义，放在头文件中。
- 跟踪点语句，位于 C 代码中。

为了使用跟踪点，您应该包括 linux/tracepoint.h 头文件。

在 include/trace/events/subsys.h 中：

```c
#undef TRACE_SYSTEM
#define TRACE_SYSTEM subsys

#if !defined(_TRACE_SUBSYS_H) || defined(TRACE_HEADER_MULTI_READ)
#define _TRACE_SUBSYS_H

#include <linux/tracepoint.h>

DECLARE_TRACE(subsys_eventname,
        TP_PROTO(int firstarg, struct task_struct *p),
        TP_ARGS(firstarg, p));

#endif /* _TRACE_SUBSYS_H */

/* This part must be outside protection */
#include <trace/define_trace.h>
```

在 subsys/file.c 中（其中必须添加跟踪语句）：

```c
#include <trace/events/subsys.h>

#define CREATE_TRACE_POINTS
DEFINE_TRACE(subsys_eventname);

void somefct(void)
{
        ...
        trace_subsys_eventname(arg, task);
        ...
}
```

此处：

- subsys_eventname 是事件的唯一标识符
  - subsys 是子系统的名称。
  - eventname 是要跟踪的事件的名称。
- *TP_PROTO(int firstarg，struct task_struct*p)* 是此跟踪点调用的函数原型。
- *TP_ARGS(firstarg，p)* 是参数名称，与原型中的名称相同。
- 如果在多个源文件中使用头文件，*#define CREATE_TRACE_POINTS* 应仅显示在一个源文件内。

将函数（探针）连接到跟踪点是通过 register_trace_subsys_eventname() 为特定跟踪点提供一个探针（要调用的函数）来完成的。删除探针是通过 unregister_trace_subsys_eventname() 完成的；它将移除探针。

在模块退出函数结束前，必须调用 tracepoint_synchronize_unregister() 以确保没有调用者仍在使用探针。这一点，以及在探针调用周围禁用抢占的事实，确保了探针移除和模块卸载是安全的。

跟踪点机制支持插入同一个跟踪点的多个实例，，但必须在所有内核上由给定的跟踪点名称进行单个定义，以确保不会发生类型冲突。跟踪点的名称通过使用原型进行变形处理，以确保类型正确。在注册地点由编译器完成探针类型的正确性验证。跟踪点可以放在内联函数、内联静态函数、展开循环以及常规函数中。

这里建议使用 "subsys_event" 命名方案作为限制冲突的约定。跟踪点名称对内核是全局的：无论它们是在核心内核映像中还是在模块中，它们都被认为是相同的。

如果跟踪点必须在内核模块中使用，可以使用 EXPORT_TRACEPOINT_SYMBOL_GPL() 或 EXPORT_TRACEPOINT_SYMBOL() 来导出定义的跟踪点。

如果您需要为跟踪点参数做一些工作，并且该工作仅用于跟踪点，则可以将该工作封装在 if 语句中，其中包含以下内容：

```c
if (trace_foo_bar_enabled()) {
        int i;
        int tot = 0;

        for (i = 0; i < count; i++)
                tot += calculate_nuggets();

        trace_foo_bar(tot);
}
```

所有 trace\_\<tracepoint\>() 调用都有一个匹配的 trace\_\<tracepoint\>\_enabled() 函数定义，该函数在 tracepoint 启用时返回 true，否则返回 false。trace\_\<tracepoint\>() 调用应该始终位于 if (trace\_\<tracepoint\>\_enabled()) 块中，以避免在启用 tracepoint 和检查被看到之间发生竞态条件。

使用 trace\_\<tracepoint\>\_enabled() 的优势在于它使用 tracepoint 的 static_key 来允许 if 语句使用跳转标签来实现，并避免条件分支。

> [!NOTE]
>
> 宏 TRACE_EVENT 提供了定义 tracepoints 的另一种方式。请查看 http://lwn.net/Articles/379903， http://lwn.net/Articles/381064 和 http://lwn.net/Articles/383362，这是一系列包含更多细节的文章。

如果需要从头文件中调用跟踪点，则不建议直接调用跟踪点或使用 trace\_\<tracepoint\>\_enabled() 函数调用，因为如果在设置了 CREATE_TRACE_POINTS 的文件中包含头，则头文件中的跟踪点可能会产生副作用，而且 trace\_\<tracepoint\>() 不是那么小的内联函数，如果其他内联函数使用，则会使内核膨胀。相反，请包含 tracepoint-defs.h 并使用 tracepoint_enabled() 。

在 C 文件中：

```c
void do_trace_foo_bar_wrapper(args)
{
        trace_foo_bar(args);
}
```

在头文件中：

```c
DECLARE_TRACEPOINT(foo_bar);

static inline void some_inline_function()
{
        [..]
        if (tracepoint_enabled(foo_bar))
                do_trace_foo_bar_wrapper(args);
        [..]
}
```
