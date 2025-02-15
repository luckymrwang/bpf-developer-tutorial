# eBPF Beginner's Development Tutorial Ten: Capturing Interrupt Events in eBPF Using hardirqs or softirqs

eBPF (Extended Berkeley Packet Filter) is a powerful network and performance analysis tool on the Linux kernel. It allows developers to dynamically load, update, and run user-defined code at runtime in the kernel.

This article is the tenth part of the eBPF Beginner's Development Tutorial, focusing on capturing interrupt events using hardirqs or softirqs in eBPF.
hardirqs and softirqs are two different types of interrupt handlers in the Linux kernel. They are used to handle interrupt requests generated by hardware devices, as well as asynchronous events in the kernel. In eBPF, we can use the eBPF tools hardirqs and softirqs to capture and analyze information related to interrupt handling in the kernel.

## What are hardirqs and softirqs?

hardirqs are hardware interrupt handlers. When a hardware device generates an interrupt request, the kernel maps it to a specific interrupt vector and executes the associated hardware interrupt handler. Hardware interrupt handlers are commonly used to handle events in device drivers, such as completion of device data transfer or device errors.

softirqs are software interrupt handlers. They are a low-level asynchronous event handling mechanism in the kernel, used for handling high-priority tasks in the kernel. softirqs are commonly used to handle events in the network protocol stack, disk subsystem, and other kernel components. Compared to hardware interrupt handlers, software interrupt handlers have more flexibility and configurability.

## Implementation Details

In eBPF, we can capture and analyze hardirqs and softirqs by attaching specific kprobes or tracepoints. To capture hardirqs and softirqs, eBPF programs need to be placed on relevant kernel functions. These functions include:

- For hardirqs: irq_handler_entry and irq_handler_exit.
- For softirqs: softirq_entry and softirq_exit.

When the kernel processes hardirqs or softirqs, these eBPF programs are executed to collect relevant information such as interrupt vectors, execution time of interrupt handlers, etc. The collected information can be used for analyzing performance issues and other interrupt handling related problems in the kernel.

To capture hardirqs and softirqs, the following steps can be followed:

1. Define data structures and maps in eBPF programs for storing interrupt information.
2. Write eBPF programs and attach them to the corresponding kernel functions to capture hardirqs or softirqs.
3. In eBPF programs, collect relevant information about interrupt handlers and store this information in the maps.
4. In user space applications, read the data from the maps to analyze and display the interrupt handling information.

By following the above approach, we can use hardirqs and softirqs in eBPF to capture and analyze interrupt events in the kernel, identifying potential performance issues and interrupt handling related problems.

## Implementation of hardirqs Code

The main purpose of the hardirqs program is to obtain the name, execution count, and execution time of interrupt handlers and display the distribution of execution time in the form of a histogram. Let's analyze this code step by step.

```c
// SPDX-License-Identifier: GPL-2.0
// Copyright (c) 2020 Wenbo Zhang
#include <vmlinux.h>
#include <bpf/bpf_core_read.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_tracing.h>
#include "hardirqs.h"
#include "bits.bpf.h"
#include "maps.bpf.h"

#define MAX_ENTRIES 256

const volatile bool filter_cg = false;
const volatile bool targ_dist = false;
const volatile bool targ_ns = false;
const volatile bool do_count = false;

struct {
 __uint(type, BPF_MAP_TYPE_CGROUP_ARRAY);
 __type(key, u32);
 __type(value, u32);
 __uint(max_entries, 1);
} cgroup_map SEC(".maps");

struct {
 __uint(type, BPF_MAP_TYPE_PERCPU_ARRAY);
 __uint(max_entries, 1);
 __type(key, u32);
 __type(value, u64);
``````c
} start SEC(".maps");

struct {
 __uint(type, BPF_MAP_TYPE_HASH);
 __uint(max_entries, MAX_ENTRIES);
 __type(key, struct irq_key);
 __type(value, struct info);
} infos SEC(".maps");

static struct info zero;

static int handle_entry(int irq, struct irqaction *action)
{
 if (filter_cg && !bpf_current_task_under_cgroup(&cgroup_map, 0))
  return 0;

 if (do_count) {
  struct irq_key key = {};
  struct info *info;

  bpf_probe_read_kernel_str(&key.name, sizeof(key.name), BPF_CORE_READ(action, name));
  info = bpf_map_lookup_or_try_init(&infos, &key, &zero);
  if (!info)
   return 0;
  info->count += 1;
  return 0;
 } else {
  u64 ts = bpf_ktime_get_ns();
  u32 key = 0;

  if (filter_cg && !bpf_current_task_under_cgroup(&cgroup_map, 0))
   return 0;

  bpf_map_update_elem(&start, &key, &ts, BPF_ANY);
  return 0;
 }
}

static int handle_exit(int irq, struct irqaction *action)
{
 struct irq_key ikey = {};
 struct info *info;
 u32 key = 0;
 u64 delta;
 u64 *tsp;

 if (filter_cg && !bpf_current_task_under_cgroup(&cgroup_map, 0))
  return 0;

 tsp = bpf_map_lookup_elem(&start, &key);
 if (!tsp)
  return 0;

 delta = bpf_ktime_get_ns() - *tsp;
 if (!targ_ns)
  delta /= 1000U;

 bpf_probe_read_kernel_str(&ikey.name, sizeof(ikey.name), BPF_CORE_READ(action, name));
 info = bpf_map_lookup_or_try_init(&infos, &ikey, &zero);
 if (!info)
  return 0;

 if (!targ_dist) {
  info->count += delta;
 } else {
  u64 slot;

  slot = log2(delta);
  if (slot >= MAX_SLOTS)
   slot = MAX_SLOTS - 1;
  info->slots[slot]++;
 }

 return 0;
}

SEC("tp_btf/irq_handler_entry")
int BPF_PROG(irq_handler_entry_btf, int irq, struct irqaction *action)
{
 return handle_entry(irq, action);
}

SEC("tp_btf/irq_handler_exit")
int BPF_PROG(irq_handler_exit_btf, int irq, struct irqaction *action)
{
 return handle_exit(irq, action);
}

SEC("raw_tp/irq_handler_entry")
int BPF_PROG(irq_handler_entry, int irq, struct irqaction *action)
{
 return handle_entry(irq, action);
}

SEC("raw_tp/irq_handler_exit")
```int BPF_PROG(irq_handler_exit, int irq, struct irqaction *action)
{
 return handle_exit(irq, action);
}

char LICENSE[] SEC("license") = "GPL";
```

This code is an eBPF program used to capture and analyze the execution information of hardware interrupt handlers (hardirqs) in the kernel. The main purpose of the program is to obtain the name, execution count, and execution time of the interrupt handler, and display the distribution of execution time in the form of a histogram. Let's analyze this code step by step.

1. Include necessary header files and define data structures:

    ```c
    #include <vmlinux.h>
    #include <bpf/bpf_core_read.h>
    #include <bpf/bpf_helpers.h>
    #include <bpf/bpf_tracing.h>
    #include "hardirqs.h"
    #include "bits.bpf.h"
    #include "maps.bpf.h"
    ```

    This program includes the standard header files required for eBPF development, as well as custom header files for defining data structures and maps.

2. Define global variables and maps:

    ```c
    #define MAX_ENTRIES 256

    const volatile bool filter_cg = false;
    const volatile bool targ_dist = false;
    const volatile bool targ_ns = false;
    const volatile bool do_count = false;

    ...
    ```

    This program defines some global variables that are used to configure the behavior of the program. For example, `filter_cg` controls whether to filter cgroups, `targ_dist` controls whether to display the distribution of execution time, etc. Additionally, the program defines three maps for storing cgroup information, start timestamps, and interrupt handler information.

3. Define two helper functions `handle_entry` and `handle_exit`:

    These two functions are called at the entry and exit points of the interrupt handler. `handle_entry` records the start timestamp or updates the interrupt count, while `handle_exit` calculates the execution time of the interrupt handler and stores the result in the corresponding information map.

4. Define the entry points of the eBPF program:

    ```c
    SEC("tp_btf/irq_handler_entry")
    int BPF_PROG(irq_handler_entry_btf, int irq, struct irqaction *action)
    {
    return handle_entry(irq, action);
    }

    SEC("tp_btf/irq_handler_exit")
    int BPF_PROG(irq_handler_exit_btf, int irq, struct irqaction *action)
    {
    return handle_exit(irq, action);
    }

    SEC("raw_tp/irq_handler_entry")
    int BPF_PROG(irq_handler_entry, int irq, struct irqaction *action)
    {
    return handle_entry(irq, action);
    }

    SEC("raw_tp/irq_handler_exit")
    int BPF_PROG(irq_handler_exit, int irq, struct irqaction *action)
    {
    return handle_exit(irq, action);
    }
    ```

    Here, four entry points of the eBPF program are defined, which are used to capture the entry and exit events of the interrupt handler. `tp_btf` and `raw_tp` represent capturing events using BPF Type Format (BTF) and raw tracepoints, respectively. This ensures that the program can be ported and run on different kernel versions.

The code for Softirq is similar, and I won't elaborate on it here.

## Run code.Translated content:

"eunomia-bpf is an open-source eBPF dynamic loading runtime and development toolchain that combines Wasm. Its purpose is to simplify the development, building, distribution, and execution of eBPF programs. You can refer to <https://github.com/eunomia-bpf/eunomia-bpf> to download and install the ecc compilation toolchain and ecli runtime. We use eunomia-bpf to compile and run this example.

To compile this program, use the ecc tool:

```console
$ ecc hardirqs.bpf.c
Compiling bpf object...
Packing ebpf object and config into package.json...
```

Then run:

```console
sudo ecli run ./package.json
```

## Summary

In this chapter (eBPF Getting Started Tutorial Ten: Capturing Interrupt Events in eBPF with Hardirqs or Softirqs), we learned how to capture and analyze the execution information of hardware interrupt handlers (hardirqs) in the kernel using eBPF programs. We explained the example code in detail, including how to define data structures, mappings, eBPF program entry points, and how to call helper functions to record execution information at the entry and exit points of interrupt handlers.

By studying the content of this chapter, you should have mastered the methods of capturing interrupt events with hardirqs or softirqs in eBPF, as well as how to analyze these events to identify performance issues and other problems related to interrupt handling in the kernel. These skills are crucial for analyzing and optimizing the performance of the Linux kernel.

To better understand and practice eBPF programming, we recommend reading the official documentation of eunomia-bpf: <https://github.com/eunomia-bpf/eunomia-bpf>. In addition, we provide a complete tutorial and source code for you to view and learn from at <https://github.com/eunomia-bpf/bpf-developer-tutorial>. We hope this tutorial can help you get started with eBPF development smoothly and provide useful references for your further learning and practice."