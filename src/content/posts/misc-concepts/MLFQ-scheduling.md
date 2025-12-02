---
title: Multi-Level Feedback Queue Scheduler
published: 2025-12-01
description: 'Implementation of the MLFQ Job Scheduler'
image: ''
tags: [operating-systems]
category: 'programming'
draft: false
lang: 'en'
---

 Project 2 - Threading Library

## Implementation Overview
The multithreading library controls the scheduling, creation, yielding, 
blocking, and termination of threads. To keep track of scheduling a common 
queue was implemented. Thread creation was handled by ```uthread.c```, yielding
comes from a thread being preempted, and blocking comes from a semaphore not
having enough resources.

## Queue
Our queue is used in two places inside this library. One is for a global
"ready_queue" that stores threads waiting to be executed, and a "blocked_queue"
that stores threads that are blocked by the sempahore. To allow for easy 
testing and versatility of ```queue.c``` data is stored in a ```void*```.

In our queue implementation there are two structs, ```item``` and ```queue```.
```c
typedef struct item {
    void* data;
    struct item* prev; // previous item that entered the queue
    struct item* next; // next item in the queue
} item;

typedef struct queue {
    item* back;
    item* front;
    int length;
} queue;
```
The item struct has a previous and next pointer to allow for easy traversal 
of the queue. The queue struct has 2 item pointers (front and back) to allow 
for quick ```queue_dequeue()```'s and ```queue_enqueue```'s.

## Uthread
The Uthread API is used to create and schedule threads. Threads conceptually
need to store a State ```{READY, RUNNING, BLOCKED, EXITED}```, a PC, CPU 
registers, and SP. In ```context.c``` three of these variables were 
abstracted by using the ```uthread_ctx_t``` data member.
```c
typedef struct uthread_tcb {
    // includes PC, registers, stack pointer
    uthread_ctx_t ctx;

} uthread_tcb;
```
In our implementation you may notice there is no state, this is because threads
that that are ```EXITED``` do not need to be tracked, and global variables 
store the remaining states. In this library there are three global variables:
```c
// Thread used to call uthread_run
static uthread_tcb* idle_tcb;
// Currently executing thread
static uthread_tcb* current_tcb;
// Queue of threads to be scheduled
static queue_t ready_queue;
```
```current_tcb``` stores the ```RUNNING``` thread, and the ```ready_queue```
stores the ```READY``` threads. *Note: In semaphores, queues store* ```BLOCKED``` 
*threads.* ```idle_tcb``` represents the process that called the multithreading
library. It's stored after context switching to the first thread specified
in ```uthread_run()```.

The Uthread Library has other functionalities, like ```uthread_create(func, arg)``` 
which allocates space for a new thread with the specified function and 
argument, and adds it to the ```ready_queue```. ```uthread_yield()``` switches 
what thread is being run. So, the ```current_tcb``` becomes the thread that was 
dequeued from the ```ready_queue```. The thread being switched from is then 
added to the end of the ```ready_queue```. ```uthread_exit()``` is called when 
a thread has finished its execution. The running thread will be deleted and if 
there is a thread in the ```ready_queue```, it runs. If there isn't, we go back 
to executing the process that called the multithreading library ```idle_tcb``` 
(as there is nothing left to schedule). Lastly, ```uthread_block()``` deletes 
the current running thread and starts running the next thread. 
```uthread_unblock(thread)``` simply adds a thread to the ```ready_queue```.

## Semaphores
Semaphores keep track of the number of resources that are allocated towards 
threads. Each running thread uses a resource. If there are no available 
resources in the semaphore then the currently executing thread must be 
```BLOCKED``` and this blocked thread needs to be stored. In our 
implementation this is done through a data member in the semaphore struct:
 ```c
typedef struct semaphore {
    size_t resource_count;
    queue_t blocked_queue;
} semaphore;
```
which keeps tracks of all the potentially blocked threads. ```sem.c``` hosts 
some library functions; other than ```sem_create()``` and ```sem_destory()``` 
it has:

```sem_down()```: Decrements the ```resource_count```, unless its 0. This means 
that there are no resources to run the current process, so we block its 
execution through ```uthread_block()``` and add that thread to that semaphore's 
```blocked_queue```.

```sem_up()```: Adds to the ```resource_count```. If there are thread in the
```blocked_queue```, they now have the opportunity to run. Unblock the thread
using ```uthread_unblock()``` and decrement the ```resource_count``` since a
resource was just used.

## Preempt 
In order to avoid a thread taking up all the CPU's resources for an extended 
period, we need a preemption protocol that periodically switches the context 
from one thread to another thread that's waiting in the ```ready_queue```.

```preempt_start(bool)```: This function will initialize the preempt operations
by using the signals.h library. We created a timer that would raise a signal 
every 10 ms. Each signal causes the current running thread to yield.
```c 
void preempt_start(bool preempt)
{
	if(!preempt)  { return; }
	//create an interval of 10 ms
	next.it_interval.tv_sec = 0;
	next.it_interval.tv_usec = HZ * 100;

	next.it_value.tv_sec = 0;
	next.it_value.tv_usec = HZ * 100;
	//clear all signals
	sigemptyset(&sig);
	//add SIGVTALRM to sig
	sigaddset(&sig, SIGVTALRM);

	sig_action.sa_handler = signalHandler;
	sigemptyset(&sig_action.sa_mask);

	sigaction(SIGVTALRM, &sig_action, &prev_action);
	setitimer(ITIMER_VIRTUAL, &next, &prev);
}
```
Before we start the timer process, we want to ensure that any signal that was
there previously gets cleared by ```sigemptyset(sigaction)```. Once this is 
done, we will add the SIGVTALRM signal to the signal set and make sure that we
assign the sig_action with the a unique ```signal_handler()``` function that
will simply call ```uthread_yield()``` which switches the context of the 
current thread to the next thread that's waiting in line. We also set the 
sa_mask property as empty because we do not want to ignore any signals that are
raised when the virtual timer is set off. The ```sigaction()``` function will 
then keep a look out for any SIGVTALRM signal and will call the 
```signal_handler()``` that we set up earlier. 


```preempt_stop(void)```: This function will revert the changes maked by
```preempt_start(bool)```, where it will restore the previous signal action 
that was saved before ```preempt_start()``` was called. 

```preempt_enable()``` and ```preempt_disable()```: These two functions use
```sigprocmask()``` in order to add or remove the virtual timer to the blocked
list, or a list of signals that are ignored. When either are called, we are
simply blocking from reading the signal or unblocking from the signal being
read. They are used to protect the alteration of global variables found in
```uthread.c``` and ```sem.c```.

## Preempt Tester
In order to test preemption, we have two threads that never stop running when 
started. This is done through the use of a ```while(1)``` loop. Each thread
prints its id (2 or 3) infinitely, so if there is an alternating sequence of 2 
or 3's in the terminal it means the threads are being yielded by preempt.c.
