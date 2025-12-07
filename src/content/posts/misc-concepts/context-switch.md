---
title: Context Switching Internals
published: 2025-12-03
description: 'Go over what Context Switching is, and how it functions under the hood'
image: ''
tags: ['operating-systems']
category: 'Programming'
draft: true 
lang: 'en'
---

# Purpose

The operating system is just one big process scheduler on your computer. All your applications, all
your keypresses, and the videos that are rendered on your monitor is all scheduled on the CPU by the
operating system's scheduler. As you may know, there is a limited amount of CPU cores that are able
to process these programs at the same time. Naturally, this would mean that the scheduler needs to
keep track of what processes are currently running, and which of them need to be processed at a
certain time. This indicates that the operating system has the pivotal task of virtualizing the CPU
and allow for all the processes to share the limited amount of cores in the CPU. We need to make it
seem as though a process believes that it is the sole job running on this CPU without needing to
know that it's actually sharing the resources with other processes waiting for it's turn with the
CPU. This leads us to the idea of the Context Switch, the main focus of this blog.

# Context Switching

When a process is scheduled to run, it is most likely in a ```READY``` state and simply awaits for
the scheduler to determine that the currently running process should give up it's stake on the CPU
for another process.

:::note
For simplicity, all these examples will assume a single-core machine, because a multicore model has
concurrency considerations to take into account as well. For now, our focus is on the mechanisms
behind context switching.
:::

There are two ways that you're operating system could take control of the CPU in order to make this
scheduling decision.

## Cooperative Multitasking

In a cooperative model, there is the assumption by the operating system that the user program will
make a system call which would naturally give up the CPU to the operating system in order to perform
the system call, which requires privileges. This model works only if the operating system can fully
trust that the user program will not hog the CPU through either no system calls, or a bug where
infinite loops may occur, causing the machine to freeze. In this situation, there is nothing that
can be done other than to reboot the machine. Therefore, this approach is less utilized in the
practice.

:::note
After writing this section, I was curious and looked up some real world applications of where
Cooperative Multitasking would show up if it relied too much on the trust of the programmer. After
one query to Gemini 3.0, I learned that in fact I was correct to assume that this particular model
of multitasking was bad for a system like a personal computer. But, for web servers such as Node.js
that typically handle tens of thousands of users concurrently, this would mean that preemptive
switching between all these user's and their jobs would take up most of the execution time, and
would be ineffecient. And they get over the hump of the trust in the program, because the server
developers themselves write the programs, so they trust themselves not to break their own systems.
I guess the stakes are high if you're working on one of these core programs in a server. One bug
could shutdown a [shutdown a large chunk of your servers.](https://www.bbc.com/news/articles/cev1en9077ro).
:::

## Preemptive Multitasking

In a preemptive model, this is a more cynical approach of the two. It can not be assumed that a user
program would be bug free or not have malicious intent. Therefore, it is the operating system's job
to ensure that no matter what, periodically, the operating systemd would gain control of the CPU.
How does it do that? Through a periodic interrupt. 
