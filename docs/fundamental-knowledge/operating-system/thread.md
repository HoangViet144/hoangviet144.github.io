---
layout: default
title: Thread
parent: Operating system
nav_order: 5
---

# Thread
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
   {:toc}

---

## Overview
A thread is a basic unit of CPU utilization; it comprises a thread ID, a program
counter (PC), a register set, and a stack. It shares with other threads belonging
to the same process its code section, data section, and other operating-system
resources, such as open files and signals. 

![img.png](/asset/image/operating-system/multi-thread-vs-single-thread.png)

Benefits of multi-threading are:
- Responsiveness: the application continues to response to user while let another thread handles heavy task
- Resource sharing: share the same data and code and heap memory
- Economy: faster context switch, creation time
- Scalability: can run on multiple core

## Concurrency vs parallelism
Concurrency is the ability to run multiple tasks by rapidly switching between processes, allowing each process to make progress
![img.png](/asset/image/operating-system/concurrency.png)

Parallelism is the ability to run mulple tasks simultaneously by letting processes run on different cores
![img.png](/asset/image/operating-system/parallelism.png)

A system with a single core only supports concurrency. 

## Amdahl’S law
Amdahl’s Law is a formula that identifies potential performance gains from adding additional computing cores to an application that has both serial
(nonparallel) and parallel components. If S is the portion of the application that must be performed serially on a system with N processing cores, the formula appears as follows:

$$
speedup \leq \frac{1}{S + \frac{1-S}{N}}
$$

As N approaches infinity, the speedup converges to 1∕S.

## Multithreading model
When a process is running, it runs in the userspace, where all resources are restricted and limited (ie. it cannot access memory of other process, etc)
When a process needs to interact with the hardware (IO, network, etc), it needs to make a system call to kernel space. 
Kernel space is the place to manage all resources and the kernel space also has kernel thread as well.

There are 3 types of relationship between user threads and kernel threads:
- Many-to-one: this model appears in old systems where a system call may block entire kernel thread 
![img.png](/asset/image/operating-system/thread-many-to-one.png)
- One-to-one: Linux and window use this kind of model. The drawback is that when a user thread is created, a kernel thread is spawn, which may burden the performance of the sytem
![img.png](/asset/image/operating-system/thread-one-to-one.png)
- Many-to-many
![img.png](/asset/image/operating-system/thread-many-to-many.png)