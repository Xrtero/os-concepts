# Deadlocks

In a multiprogramming environment, several threads may compete for a finite number of resources. A thread requests resources; if the resources are not available at that time, the thread enters a waiting state. Sometimes, a waiting thread can never again change state, because the resources it has requested are held by other waiting threads. This situation is called a **deadlock**. We discussed this issue briefly in Chapter 6 as a form of liveness failure. There, we defined deadlock as a situation in which every process in a set of processes is waiting for an event that can be caused only by another process in the set.

Perhaps the best illustration of a deadlock can be drawn from a law passed by the Kansas legislature early in the 20th century. It said, in part: “When two trains approach each other at a crossing, both shall come to a full stop and neither shall start up again until the other has gone.”

In this chapter, we describe methods that application developers as well as operating-system programmers can use to prevent or deal with dead- locks. Although some applications can identify programs that may dead- lock, operating systems typically do not provide deadlock-prevention facil- ities, and it remains the responsibility of programmers to ensure that they design deadlock-free programs. Deadlock problems — as well as other liveness failures—are becoming more challenging as demand continues for increased concurrency and parallelism on multicore systems.

> CHAPTER OBJECTIVES
>
> • Illustrate how deadlock can occur when mutex locks are used.  
> • Define the four necessary conditions that characterize deadlock. • Identify a deadlock situation in a resource allocation graph.  
> • Evaluate the four different approaches for preventing deadlocks. • Apply the banker’s algorithm for deadlock avoidance.  
> • Apply the deadlock detection algorithm.  
> • Evaluate approaches for recovering from deadlock.

## 8.1 System Model

A system consists of a finite number of resources to be distributed among a number of competing threads. The resources may be partitioned into several types (or classes), each consisting of some number of identical instances. CPU cycles, files, and I/O devices (such as network interfaces and DVD drives) are examples of resource types. If a system has four CPUs, then the resource type $CPU$ has four instances. Similarly, the resource type _network_ may have two instances. If a thread requests an instance of a resource type, the allocation of _any_ instance of the type should satisfy the request. If it does not, then the instances are not identical, and the resource type classes have not been defined properly.

The various synchronization tools discussed in Chapter 6, such as mutex locks and semaphores, are also system resources; and on contemporary computer systems, they are the most common sources of deadlock. However, definition is not a problem here. A lock is typically associated with a specific data structure--that is, one lock may be used to protect access to a queue, another to protect access to a linked list, and so forth. For that reason, each instance of a lock is typically assigned its own resource class.

Note that throughout this chapter we discuss kernel resources, but threads may use resources from other processes (for example, via interprocess communication), and those resource uses can also result in deadlock. Such deadlocks are not the concern of the kernel and thus not described here.

A thread must request a resource before using it and must release the resource after using it. A thread may request as many resources as it requires to carry out its designated task. Obviously, the number of resources requested may not exceed the total number of resources available in the system. In other words, a thread cannot request two network interfaces if the system has only one.

Under the normal mode of operation, a thread may utilize a resource in only the following sequence:

1. **Request**. The thread requests the resource. If the request cannot be granted immediately (for example, if a mutex lock is currently held by another thread), then the requesting thread must wait until it can acquire the resource.
2. **Use**. The thread can operate on the resource (for example, if the resource is a mutex lock, the thread can access its critical section).
3. **Release**. The thread releases the resource.

The request and release of resources may be system calls, as explained in Chapter 2. Examples are the request() and release() of a device, open() and close() of a file, and allocate() and free() memory system calls. Similarly, as we saw in Chapter 6, request and release can be accomplished through the wait() and signal() operations on semaphores and through acquire() and release() of a mutex lock. For each use of a kernel-managed resource by a thread, the operating system checks to make sure that the thread has requested and has been allocated the resource. A system table records whether each resource is free or allocated. For each resource that is allocated,the table also records the thread to which it is allocated. If a thread requests a resource that is currently allocated to another thread, it can be added to a queue of threads waiting for this resource.

A set of threads is in a deadlocked state when every thread in the set is wait- ing for an event that can be caused only by another thread in the set. The events with which we are mainly concerned here are resource acquisition and release. The resources are typically logical (for example, mutex locks, semaphores, and files); however, other types of events may result in deadlocks, including read- ing from a network interface or the IPC (interprocess communication) facilities discussed in Chapter 3.

To illustrate a deadlocked state, we refer back to the dining-philosophers problem from Section 7.1.3. In this situation, resources are represented by chopsticks. If all the philosophers get hungry at the same time, and each philosopher grabs the chopstick on her left, there are no longer any available chopsticks. Each philosopher is then blocked waiting for her right chopstick to become available.

Developers of multithreaded applications must remain aware of the pos- sibility of deadlocks. The locking tools presented in Chapter 6 are designed to avoid race conditions. However, in using these tools, developers must pay careful attention to how locks are acquired and released. Otherwise, deadlock can occur, as described next.

## 8.2 Deadlock in Multithreaded Applications

Prior to examining how deadlock issues can be identified and man- aged, we first illustrate how deadlock can occur in a multithreaded Pthread program using POSIX mutex locks. The pthread mutex init() function initializes an unlocked mutex. Mutex locks are acquired and released using pthread mutex lock() and pthread mutex unlock(), respectively. If a thread attempts to acquire a locked mutex, the call to pthread mutex lock() blocks the thread until the owner of the mutex lock invokes pthread mutex unlock().

Two mutex locks are created and initialized in the following code example:
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202311080112130.png)
Next, two threads—thread one and thread two—are created, and both these threads have access to both mutex locks. thread one and thread two run in the functions do work one() and do work two(), respectively, as shown in Figure 8.1.

In this example, thread one attempts to acquire the mutex locks in the order (1) first mutex, (2) second mutex. At the same time, thread two attempts to acquire the mutex locks in the order (1) second mutex, (2) first mutex. Deadlock is possible if thread one acquires first mutex while thread two acquires second mutex.

![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202311080112150.png)

Note that, even though deadlock is possible, it will not occur if thread_one can acquire and release the mutex locks for first_mutex and second_mutex before thread_two attempts to acquire the locks. And, of course, the order in which the threads run depends on how they are scheduled by the CPU scheduler. This example illustrates a problem with handling deadlocks: it is difficult to identify and test for deadlocks that may occur only under certain scheduling circumstances.

### 8.2.1 Livelock

**Livelock** is another form of liveness failure. It is similar to deadlock; both prevent two or more threads from proceeding, but the threads are unable to proceed for different reasons. Whereas deadlock occurs when every thread in a set is blocked waiting for an event that can be caused only by another thread in the set, livelock occurs when a thread continuously attempts an action that fails. Livelock is similar to what sometimes happens when two people attempt to pass in a hallway: One moves to his right, the other to her left, still obstructing each other's progress. Then he moves to his left, and she moves to her right, and so forth. They aren't blocked, but they aren't making any progress.

Livelock can be illustrated with the Pthreads pthread_mutex_trylock() function, which attempts to acquire a mutex lock without blocking. The code example in Figure 8.2 rewrites the example from Figure 8.1 so that it now uses pthread_mutex_trylock(). This situation can lead to livelock if thread_one acquires first_mutex, followed by thread_two acquiring second_mutex. Each thread then invokes pthread_mutex_trylock(), which fails, releases their respective locks, and repeats the same actions indefinitely.

Livelock typically occurs when threads retry failing operations at the same time. It thus can generally be avoided by having each thread retry the failing operation at random times. This is precisely the approach taken by Ethernet networks when a network collision occurs. Rather than trying to retransmit a packet immediately after a collision occurs, a host involved in a collision will _backoff_ a random period of time before attempting to transmit again.

Livelock is less common than deadlock but nonetheless is a challenging issue in designing concurrent applications, and like deadlock, it may only occur under specific scheduling circumstances.

## 8.3 Deadlock Characterization

In the previous section we illustrated how deadlock could occur in multi-threaded programming using mutex locks. We now look more closely at conditions that characterize deadlock.

### 8.3.1 Necessary Conditions

A deadlock situation can arise if the following four conditions hold simultaneously in a system:

1. **Mutual exclusion**. At least one resource must be held in a nonsharable mode; that is, only one thread at a time can use the resource. If another thread requests that resource, the requesting thread must be delayed until the resource has been released.
2. **Hold and wait**. A thread must be holding at least one resource and waiting to acquire additional resources that are currently being held by other threads.
3. **No preemption**. Resources cannot be preempted; that is, a resource can be released only voluntarily by the thread holding it, after that thread has completed its task.
4. **Circular wait**. A set $\{T_{0}$, $T_{1}$,..., $T_{n}\}$ of waiting threads must exist such that $T_{0}$ is waiting for a resource held by $T_{1}$, $T_{1}$ is waiting for a resource held by $T_{2}$,..., $T_{n-1}$ is waiting for a resource held by $T_{n}$, and $T_{n}$ is waiting for a resource held by $T_{0}$.

We emphasize that all four conditions must hold for a deadlock to occur. The circular-wait condition implies the hold-and-wait condition, so the four
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202311080114166.png)

![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202311080114642.png)
conditions are not completely independent. We shall see in Section 8.5, however, that it is useful to consider each condition separately.

### 8.3.2 Resource-Allocation Graph

Deadlocks can be described more precisely in terms of a directed graph called a **system resource-allocation graph**. This graph consists of a set of vertices V and a set of edges E. The set of vertices V is partitioned into two different types of nodes: $T=\left\{T_{1}, T_{2}, \ldots, T_{n}\right\}$, the set consisting of all the active threads in the system, and $R=\left\{R_{1}, R_{2}, \ldots, R_{m}\right\}$, the set consisting of all resource types in the system.

A directed edge from thread $T_{i}$ to resource type $R_{j}$ is denoted by $T_{i} \rightarrow R_{j}$; it signifies that thread $T_{i}$ has requested an instance of resource type $R_{j}$ and is currently waiting for that resource. A directed edge from resource type $R_{j}$ to thread $T_{i}$ is denoted by $R_{j} \rightarrow T_{i}$; it signifies that an instance of resource type $R_{j}$ has been allocated to thread $T_{i}$. A directed edge $T_{i} \rightarrow R_{j}$ is called a **request edge**; a directed edge $R_{j} \rightarrow T_{i}$ is called an **assignment edge**.

Pictorially, we represent each thread $T_{i}$ as a circle and each resource type $R_{j}$ as a rectangle. As a simple example, the resource allocation graph shown in Figure 8.3 illustrates the deadlock situation from the program in Figure 8.1. Since resource type $R_{j}$ may have more than one instance, we represent each such instance as a dot within the rectangle. Note that a request edge points only to the rectangle $R_{j}$, whereas an assignment edge must also designate one of the dots in the rectangle.

When thread $T_{i}$ requests an instance of resource type $R_{j}$, a request edge is inserted in the resource-allocation graph. When this request can be fulfilled, the request edge is instantaneously transformed to an assignment edge. When the thread no longer needs access to the resource, it releases the resource. As a result, the assignment edge is deleted.

The resource-allocation graph shown in Figure 8.4 depicts the following situation.
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202311080120091.png)

- The sets $T, R,$ and $E$ :
  - circ $T=\left\{T_{1}, T_{2}, T_{3}\right\}$
  - $R=\left\{R_{1}, R_{2}, R_{3}, R_{4}\right\}$
  - $E = \{T_{1}\to R_{1},\,T_{2}\to R_{3},\,R_{1}\to\,T_{2},\,R_{2}\to\,T_{2},\,R_{2}\to\,T_{1}, \,R_{3}\to\,T_{3}\}$

- Resource instances:
  - One instance of resource type $R_{1}$
  - Two instances of resource type $R_{2}$
  - One instance of resource type $R_{3}$
  - Three instances of resource type $R_{4}$

- Thread states:
  - Thread $T_{1}$ is holding an instance of resource type $R_{2}$ and is waiting for an instance of resource type $R_{1}$.
  - Thread $T_{2}$ is holding an instance of $R_{1}$ and an instance of $R_{2}$ and is waiting for an instance of $R_{3}$.
  - Thread $T_{3}$ is holding an instance of $R_{3}$.

Given the definition of a resource-allocation graph, it can be shown that, if the graph contains no cycles, then no thread in the system is deadlocked. If the graph does contain a cycle, then a deadlock may exist.

If each resource type has exactly one instance, then a cycle implies that a deadlock has occurred. If the cycle involves only a set of resource types, each of which has only a single instance, then a deadlock has occurred. Each thread involved in the cycle is deadlocked. In this case, a cycle in the graph is both a necessary and a sufficient condition for the existence of deadlock.

If each resource type has several instances, then a cycle does not necessarily imply that a deadlock has occurred. In this case, a cycle in the graph is a necessary but not a sufficient condition for the existence of deadlock.

To illustrate this concept, we return to the resource-allocation graph depicted in Figure 8.4. Suppose that thread $T_{3}$ requests an instance of resource type $R_{2}$. Since no resource instance is currently available, we add a request

Figure 8.4: Resource-allocation graph.
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202311080121198.png)

edge $T_{3}\to R_{2}$ to the graph (Figure 8.5). At this point, two minimal cycles exist in the system:
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202311080121499.png)
Threads $T_{1}, T_{2}$, and $T_{3}$ are deadlocked. Thread $T_{2}$ is waiting for the resource $R_{3}$, which is held by thread $T_{3}$. Thread $T_{3}$ is waiting for either thread $T_{1}$ or thread $T_{2}$ to release resource $R_{2}$. In addition, thread $T_{1}$ is waiting for thread $T_{2}$ to release resource $R_{1}$.

Now consider the resource-allocation graph in Figure 8.6. In this example, we also have a cycle:
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202311080122094.png)
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202311080122866.png)
However, there is no deadlock. Observe that thread $T_{4}$ may release its instance of resource type $R_{2}$. That resource can then be allocated to $T_{3}$, breaking the cycle.

In summary, if a resource-allocation graph does not have a cycle, then the system is _not_ in a deadlocked state. If there is a cycle, then the system _may_ or _may_ not be in a deadlocked state. This observation is important when we deal with the deadlock problem.

## 8.4 Methods for Handling Deadlocks

Generally speaking, we can deal with the deadlock problem in one of three ways:

- We can ignore the problem altogether and pretend that deadlocks never occur in the system.
- We can use a protocol to prevent or avoid deadlocks, ensuring that the system will _never_ enter a deadlocked state.
- We can allow the system to enter a deadlocked state, detect it, and recover.

The first solution is the one used by most operating systems, including Linux and Windows. It is then up to kernel and application developers to write programs that handle deadlocks, typically using approaches outlined in the second solution. Some systems--such as databases--adopt the third solution, allowing deadlocks to occur and then managing the recovery.

Next, we elaborate briefly on the three methods for handling deadlocks. Then, in Section 8.5 through Section 8.8, we present detailed algorithms. Before proceeding, we should mention that some researchers have argued that none of the basic approaches alone is appropriate for the entire spectrum of resource-allocation problems in operating systems. The basic approaches can be combined, however, allowing us to select an optimal approach for each class of resources in a system.

To ensure that deadlocks never occur, the system can use either a deadlock-prevention or a deadlock-avoidance scheme. **Deadlock prevention** provides a set of methods to ensure that at least one of the necessary conditions (Section 8.3.1) cannot hold. These methods prevent deadlocks by constraining how requests for resources can be made. We discuss these methods in Section 8.5.

**Deadlock avoidance** requires that the operating system be given additional information in advance concerning which resources a thread will request and use during its lifetime. With this additional knowledge, the operating system can decide for each request whether or not the thread should wait. To decide whether the current request can be satisfied or must be delayed, the system must consider the resources currently available, the resources currently allocated to each thread, and the future requests and releases of each thread. We discuss these schemes in Section 8.6.

If a system does not employ either a deadlock-prevention or a deadlock-avoidance algorithm, then a deadlock situation may arise. In this environment, the system can provide an algorithm that examines the state of the system to determine whether a deadlock has occurred and an algorithm to recover from the deadlock (if a deadlock has indeed occurred). We discuss these issues in Section 8.7 and Section 8.8.

In the absence of algorithms to detect and recover from deadlocks, we may arrive at a situation in which the system is in a deadlocked state yet has no way of recognizing what has happened. In this case, the undetected deadlock will cause the system's performance to deteriorate, because resources are being held by threads that cannot run and because more and more threads, as they make requests for resources, will enter a deadlocked state. Eventually, the system will stop functioning and will need to be restarted manually.

Although this method may not seem to be a viable approach to the deadlock problem, it is nevertheless used in most operating systems, as mentioned earlier. Expense is one important consideration. Ignoring the possibility of deadlocks is cheaper than the other approaches. Since in many systems, deadlocks occur infrequently (say, once per month), the extra expense of the other methods may not seem worthwhile.

In addition, methods used to recover from other liveness conditions, such as livelock, may be used to recover from deadlock. In some circumstances, a system is suffering from a liveness failure but is not in a deadlocked state. We see this situation, for example, with a real-time thread running at the highest priority (or any thread running on a nonpreemptive scheduler) and never returning control to the operating system. The system must have manual recovery methods for such conditions and may simply use those techniques for deadlock recovery.

## 8.5 Deadlock Prevention

As we noted in Section 8.3.1, for a deadlock to occur, each of the four necessary conditions must hold. By ensuring that at least one of these conditions cannot hold, we can _prevent_ the occurrence of a deadlock. We elaborate on this approach by examining each of the four necessary conditions separately.

### 8.5.1 Mutual Exclusion

The mutual-exclusion condition must hold. That is, at least one resource must be nonsharable. Sharable resources do not require mutually exclusive access and thus cannot be involved in a deadlock. Read-only files are a good example of a sharable resource. If several threads attempt to open a read-only file at the same time, they can be granted simultaneous access to the file. A thread never needs to wait for a sharable resource. In general, however, we cannot prevent deadlocks by denying the mutual-exclusion condition, because some resources are intrinsically nonsharable. For example, a mutex lock cannot be simultaneously shared by several threads.

### 8.5.2 Hold and Wait

To ensure that the hold-and-wait condition never occurs in the system, we must guarantee that, whenever a thread requests a resource, it does not hold any other resources. One protocol that we can use requires each thread to request and be allocated all its resources before it begins execution. This is, of course,impractical for most applications due to the dynamic nature of requesting resources.

An alternative protocol allows a thread to request resources only when it has none. A thread may request some resources and use them. Before it can request any additional resources, it must release all the resources that it is currently allocated.

Both these protocols have two main disadvantages. First, resource utilization may be low, since resources may be allocated but unused for a long period. For example, a thread may be allocated a mutex lock for its entire execution, yet only require it for a short duration. Second, starvation is possible. A thread that needs several popular resources may have to wait indefinitely, because at least one of the resources that it needs is always allocated to some other thread.

### 8.5.3 No Preemption

The third necessary condition for deadlocks is that there be no preemption of resources that have already been allocated. To ensure that this condition does not hold, we can use the following protocol. If a thread is holding some resources and requests another resource that cannot be immediately allocated to it (that is, the thread must wait), then all resources the thread is currently holding are preempted. In other words, these resources are implicitly released. The preempted resources are added to the list of resources for which the thread is waiting. The thread will be restarted only when it can regain its old resources, as well as the new ones that it is requesting.

Alternatively, if a thread requests some resources, we first check whether they are available. If they are, we allocate them. If they are not, we check whether they are allocated to some other thread that is waiting for additional resources. If so, we preempt the desired resources from the waiting thread and allocate them to the requesting thread. If the resources are neither available nor held by a waiting thread, the requesting thread must wait. While it is waiting, some of its resources may be preempted, but only if another thread requests them. A thread can be restarted only when it is allocated the new resources it is requesting and recovers any resources that were preempted while it was waiting.

This protocol is often applied to resources whose state can be easily saved and restored later, such as CPU registers and database transactions. It cannot generally be applied to such resources as mutex locks and semaphores, precisely the type of resources where deadlock occurs most commonly.

### 8.5.4 Circular Wait

The three options presented thus far for deadlock prevention are generally impractical in most situations. However, the fourth and final condition for deadlocks -- the circular-wait condition -- presents an opportunity for a practical solution by invalidating one of the necessary conditions. One way to ensure that this condition never holds is to impose a total ordering of all resource types and to require that each thread requests resources in an increasing order of enumeration.

To illustrate, we let $R$ = {$R_{1}$, $R_{2}$,..., $R_{m}$} be the set of resource types. We assign to each resource type a unique integer number, which allows us to

### Deadlock Prevention

Compare two resources and to determine whether one precedes another in our ordering. Formally, we define a one-to-one function $F$: $R\to N$, where $N$ is the set of natural numbers. We can accomplish this scheme in an application program by developing an ordering among all synchronization objects in the system. For example, the lock ordering in the Pthread program shown in Figure 8.1 could be
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202311080125480.png)
We can now consider the following protocol to prevent deadlocks: Each thread can request resources only in an increasing order of enumeration. That is, a thread can initially request an instance of a resource--say, $R_{i}$. After that, the thread can request an instance of resource $R_{j}$ if and only if $F(R_{j})>F(R_{i})$. For example, using the function defined above, a thread that wants to use both first_mutex and second_mutex at the same time must first request first_mutex and then second_mutex. Alternatively, we can require that a thread requesting an instance of resource $R_{j}$ must have released any resources $R_{i}$ such that $F(R_{j})\geq F(R_{j})$. Note also that if several instances of the same resource type are needed, a _single_ request for all of them must be issued.

If these two protocols are used, then the circular-wait condition cannot hold. We can demonstrate this fact by assuming that a circular wait exists (proof by contradiction). Let the set of threads involved in the circular wait be $\{T_{0}$, $T_{1}$,..., $T_{n}\}$, where $T_{i}$ is waiting for a resource $R_{i}$, which is held by thread $T_{i+1}$. (Modulo arithmetic is used on the indexes, so that $T_{n}$ is waiting for a resource $R_{n}$ held by $T_{0}$.) Then, since thread $T_{i+1}$ is holding resource $R_{i}$ while requesting resource $R_{i+1}$, we must have $F(R_{i})<F(R_{i+1})$ for all $i$. But this condition means that $F(R_{0})<F(R_{1})<...<F(R_{n})<F(R_{0})$. By transitivity, $F(R_{0})<F(R_{0})$, which is impossible. Therefore, there can be no circular wait.

Keep in mind that developing an ordering, or hierarchy, does not in itself prevent deadlock. It is up to application developers to write programs that follow the ordering. However, establishing a lock ordering can be difficult, especially on a system with hundreds--or thousands--of locks. To address this challenge, many Java developers have adopted the strategy of using the method System.identityHashCode(Object) (which returns the default hash code value of the Object parameter it has been passed) as the function for ordering lock acquisition.

It is also important to note that imposing a lock ordering does not guarantee deadlock prevention if locks can be acquired dynamically. For example, assume we have a function that transfers funds between two accounts. To prevent a race condition, each account has an associated mutex lock that is obtained from a get_lock() function such as that shown in Figure 8.7. Deadlock is possible if two threads simultaneously invoke the transaction() function, transposing different accounts. That is, one thread might invoke
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202311080126691.png)
and another might invoke
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202311080126834.png)

![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202311080126956.png)

## 8.6 Deadlock Avoidance

Deadlock-prevention algorithms, as discussed in Section 8.5, prevent dead- locks by limiting how requests can be made. The limits ensure that at least one of the necessary conditions for deadlock cannot occur. Possible side effects of preventing deadlocks by this method, however, are low device utilization and reduced system throughput.

An alternative method for avoiding deadlocks is to require additional information about how resources are to be requested. For example, in a system with resources R1 and R2, the system might need to know that thread P will request first R1 and then R2 before releasing both resources, whereas thread Q will request R2 and then R1. With this knowledge of the complete sequence of requests and releases for each thread, the system can decide for each request whether or not the thread should wait in order to avoid a possible future deadlock. Each request requires that in making this decision the system consider the resources currently available, the resources currently allocated to each thread, and the future requests and releases of each thread.

The various algorithms that use this approach differ in the amount and type of information required. The simplest and most useful model requires that each thread declare the maximum number of resources of each type that it may need. Given this a priori information, it is possible to construct an algorithm that ensures that the system will never enter a deadlocked state. A deadlock- avoidance algorithm dynamically examines the resource-allocation state to ensure that a circular-wait condition can never exist. The resource-allocation state is defined by the number of available and allocated resources and the maximum demands of the threads. In the following sections, we explore two deadlock-avoidance algorithms.

> LINUX LOCKDEP TOOL
>
> Although ensuring that resources are acquired in the proper order is the responsibility of kernel and application developers, certain software can be used to verify that locks are acquired in the proper order. To detect possible deadlocks, Linux provides lockdep, a tool with rich functionality that can be used to verify locking order in the kernel. lockdep is designed to be enabled on a running kernel as it monitors usage patterns of lock acquisitions and releases against a set of rules for acquiring and releasing locks. Two examples follow, but note that lockdep provides significantly more functionality than what is described here:
>
> 1. Theorderinwhichlocksareacquiredisdynamicallymaintainedbythe system. If lockdep detects locks being acquired out of order, it reports a possible deadlock condition.
>
> 2. In Linux, spinlocks can be used in interrupt handlers. A possible source of deadlock occurs when the kernel acquires a spinlock that is also used in an interrupt handler. If the interrupt occurs while the lock is being held, the interrupt handler preempts the kernel code currently holding the lock and then spins while attempting to acquire the lock, resulting in deadlock. The general strategy for avoiding this situation is to disable interrupts on the current processor before acquiring a spinlock that is also used in an interrupt handler. If lockdep detects that interrupts are enabled while kernel code acquires a lock that is also used in an interrupt handler, it will report a possible deadlock scenario.
>lockdep was developed to be used as a tool in developing or modifying code in the kernel and not to be used on production systems, as it can significantly slow down a system. Its purpose is to test whether software such as a new device driver or kernel module provides a possible source of deadlock. The designers of lockdep have reported that within a few years of its development in 2006, the number of deadlocks from system reports had been reduced by an order of magnitude.âŁž Although lockdep was originally designed only for use in the kernel, recent revisions of this tool can now be used for detecting deadlocks in user applications using Pthreads mutex locks. Further details on the lockdep tool can be found at <https://www.kernel.org/doc/Documentation/locking/lockdep-design.txt>.

### 8.6.1 Safe State

A state is _safe_ if the system can allocate resources to each thread (up to its maximum) in some order and still avoid a deadlock. More formally, a system is in a safe state only if there exists a **safe sequence**. A sequence of threads <$T_{1}$, $T_{2}$,..., $T_{n}$> is a safe sequence for the current allocation state if, for each $T_{i}$, the resource requests that $T_{i}$ can still make can be satisfied by the currently available resources plus the resources held by all $T_{j}$, with $j<i$. In this situation, if the resources that $T_{i}$ needs are not immediately available, then $T_{i}$ can wait until all $T_{j}$ have finished. When they have finished, $T_{i}$ can obtain all of itsneeded resources, complete its designated task, return its allocated resources, and terminate. When $T_{i}$ terminates, $T_{i+1}$ can obtain its needed resources, and so on. If no such sequence exists, then the system state is said to be _unsafe_.
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202311080129784.png)

A safe state is not a deadlocked state. Conversely, a deadlocked state is an unsafe state. Not all unsafe states are deadlocks, however (Figure 8.8). An unsafe state _may_ lead to a deadlock. As long as the state is safe, the operating system can avoid unsafe (and deadlocked) states. In an unsafe state, the operating system cannot prevent threads from requesting resources in such a way that a deadlock occurs. The behavior of the threads controls unsafe states.

To illustrate, consider a system with twelve resources and three threads: $T_{0}$, $T_{1}$, and $T_{2}$. Thread $T_{0}$ requires ten resources, thread $T_{1}$ may need as many as four, and thread $T_{2}$ may need up to nine resources. Suppose that, at time $t_{0}$, thread $T_{0}$ is holding five resources, thread $T_{1}$ is holding two resources, and thread $T_{2}$ is holding two resources. (Thus, there are three free resources.)

![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202311080129303.png)

At time $t_{0}$, the system is in a safe state. The sequence <$T_{1}$, $T_{0}$, $T_{2}$> satisfies the safety condition. Thread $T_{1}$ can immediately be allocated all its resources and then return them (the system will then have five available resources); then thread $T_{0}$ can get all its resources and return them (the system will then have ten available resources); and finally thread $T_{2}$ can get all its resources and return them (the system will then have all twelve resources available).

A system can go from a safe state to an unsafe state. Suppose that, at time $t_{1}$, thread $T_{2}$ requests and is allocated one more resource. The system is no longer in a safe state. At this point, only thread $T_{1}$ can be allocated all its resources. When it returns them, the system will have only four available resources. Since thread $T_{0}$ is allocated five resources but has a maximum of ten, it may request five more resources. If it does so, it will have to wait, because they are unavailable. Similarly, thread $T_{2}$ may request six additional resources and have to wait, resulting in a deadlock. Our mistake was in granting the request from thread $T_{2}$ for one more resource. If we had made $T_{2}$ wait until either of the other threads had finished and released its resources, then we could have avoided the deadlock.

Given the concept of a safe state, we can define avoidance algorithms that ensure that the system will never deadlock. The idea is simply to ensure that the system will always remain in a safe state. Initially, the system is in a safe state. Whenever a thread requests a resource that is currently available, the system must decide whether the resource can be allocated immediately or the thread must wait. The request is granted only if the allocation leaves the system in a safe state.

In this scheme, if a thread requests a resource that is currently available, it may still have to wait. Thus, resource utilization may be lower than it would otherwise be.

### 8.6.2 Resource-Allocation-Graph Algorithm

If we have a resource-allocation system with only one instance of each resource type, we can use a variant of the resource-allocation graph defined in Section 8.3.2 for deadlock avoidance. In addition to the request and assignment edges already described, we introduce a new type of edge, called a **claim edge**. A claim edge $T_{i}\to R_{j}$ indicates that thread $T_{i}$ may request resource $R_{j}$ at some time in the future. This edge resembles a request edge in direction but is represented in the graph by a dashed line. When thread $T_{i}$ requests resource $R_{j}$, the claim edge $T_{i}\to R_{j}$ is converted to a request edge. Similarly, when a resource $R_{j}$ is released by $T_{i}$, the assignment edge $R_{j}\to T_{i}$ is reconverted to a claim edge $T_{i}\to R_{j}$.

Note that the resources must be claimed a priori in the system. That is, before thread $T_{i}$ starts executing, all its claim edges must already appear in the resource-allocation graph. We can relax this condition by allowing a claim edge $T_{i}\to R_{j}$ to be added to the graph only if all the edges associated with thread $T_{i}$ are claim edges.

Now suppose that thread $T_{i}$ requests resource $R_{j}$. The request can be granted only if converting the request edge $T_{i}\to R_{j}$ to an assignment edge $R_{j}\to T_{i}$ does not result in the formation of a cycle in the resource-allocation graph. We check for safety by using a cycle-detection algorithm. An algorithm for detecting a cycle in this graph requires an order of $n^{2}$ operations, where $n$ is the number of threads in the system.

If no cycle exists, then the allocation of the resource will leave the system in a safe state. If a cycle is found, then the allocation will put the system in an unsafe state. In that case, thread $T_{i}$ will have to wait for its requests to be satisfied.

To illustrate this algorithm, we consider the resource-allocation graph of Figure 8.9. Suppose that $T_{2}$ requests $R_{2}$. Although $R_{2}$ is currently free, we cannot allocate it to $T_{2}$, since this action will create a cycle in the graph (Figure 8.10). A cycle, as mentioned, indicates that the system is in an unsafe state. If $T_{1}$ requests $R_{2}$, and $T_{2}$ requests $R_{1}$, then a deadlock will occur.

### 8.6.3 Banker's Algorithm

The resource-allocation-graph algorithm is not applicable to a resource-allocation system with multiple instances of each resource type. The deadlock-avoidance algorithm that we describe next is applicable to such a system but is less efficient than the resource-allocation graph scheme. This algorithm is commonly known as the **banker's algorithm**. The name was chosen because the algorithm could be used in a banking system to ensure that the bank never allocated its available cash in such a way that it could no longer satisfy the needs of all its customers.
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202311080130757.png)
When a new thread enters the system, it must declare the maximum number of instances of each resource type that it may need. This number may not exceed the total number of resources in the system. When a user requests a set of resources, the system must determine whether the allocation of these resources will leave the system in a safe state. If it will, the resources are allocated; otherwise, the thread must wait until some other thread releases enough resources.

Several data structures must be maintained to implement the banker's algorithm. These data structures encode the state of the resource-allocation system. We need the following data structures, where $n$ is the number of threads in the system and $m$ is the number of resource types:

- **Available**. A vector of length $m$ indicates the number of available resources of each type. If $\textit{Available}[j]$ equals $k$, then $k$ instances of resource type $R_{j}$ are available.
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202311080130030.png)
- **Max**. An $n\times m$ matrix defines the maximum demand of each thread. If $Max[i][j]$ equals $k$, then thread $T_{i}$ may request at most $k$ instances of resource type $R_{j}$.
- **Allocation**. An $n\times m$ matrix defines the number of resources of each type currently allocated to each thread. If $Allocation[i][j]$ equals $k$, then thread $T_{i}$ is currently allocated $k$ instances of resource type $R_{j}$.
- **Need**. An $n\times m$ matrix indicates the remaining resource need of each thread. If $Need[i][j]$ equals $k$, then thread $T_{i}$ may need $k$ more instances of resource type $R_{j}$ to complete its task. Note that $Need[i][j]$ equals $Max[i][j]$$- Allocation[i][j]$.

These data structures vary over time in both size and value.

To simplify the presentation of the banker's algorithm, we next establish some notation. Let $X$ and $Y$ be vectors of length $n$. We say that $X\leq Y$ if and only if $X[i]\leq Y[i]$ for all $i=1,2,...,n$. For example, if $X=$ (1,7,3,2) and $Y=$ (0,3,2,1), then $Y\leq X$. In addition, $Y<X$ if $Y\leq X$ and $Y\neq X$.

We can treat each row in the matrices $Allocation$ and $Need$ as vectors and refer to them as $Allocation_{i}$ and $Need_{i}$. The vector $Allocation_{i}$ specifies the resources currently allocated to thread $T_{i}$; the vector $Need_{i}$ specifies the additional resources that thread $T_{i}$ may still request to complete its task.

#### 8.6.3.1 Safety Algorithm

We can now present the algorithm for finding out whether or not a system is in a safe state. This algorithm can be described as follows:

1. Let $Work$ and $Finish$ be vectors of length $m$ and $n$, respectively. Initialize $Work=Available$ and $Finish[i]=false$ for $i=0,1,...,n-1$.
2. Find an index $i$ such that both 1. $Finish[i]==false$ 2. $Need_{i}\leq Work$ If no such $i$ exists, go to step 4.
3. $Work=Work+Allocation_{i}$ $Finish[i]=true$ Go to step 2.
4. If $Finish[i]==true$ for all $i$, then the system is in a safe state.

This algorithm may require an order of $m\times n^{2}$ operations to determine whether a state is safe.

#### 8.6.3.2 Resource-Request Algorithm

Next, we describe the algorithm for determining whether requests can be safely granted. Let $Request_{i}$ be the request vector for thread $T_{i}$. If $Request_{i}$$[j]==k$, then thread $T_{i}$ wants $k$ instances of resource type $R_{j}$. When a request for resources is made by thread $T_{i}$, the following actions are taken:

1. If $\boldsymbol{Request}_{i}\leq\boldsymbol{Need}_{i}$, go to step 2. Otherwise, raise an error condition, since the thread has exceeded its maximum claim.
2. If $\boldsymbol{Request}_{i}\leq\boldsymbol{Available}$, go to step 3. Otherwise, $T_{i}$ must wait, since the resources are not available.
3. Have the system pretend to have allocated the requested resources to thread $T_{i}$ by modifying the state as follows:
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202311080132386.png)
If the resulting resource-allocation state is safe, the transaction is completed, and thread $T_{i}$ is allocated its resources. However, if the new state is unsafe, then $T_{i}$ must wait for $\boldsymbol{Request}_{i}$, and the old resource-allocation state is restored.

#### 8.6.3 An Illustrative Example

To illustrate the use of the banker's algorithm, consider a system with five threads $T_{0}$ through $T_{4}$ and three resource types $A$, $B$, and $C$. Resource type $A$ has ten instances, resource type $B$ has five instances, and resource type $C$ has seven instances. Suppose that the following snapshot represents the current state of the system:
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202311080132987.png)
The content of the matrix $\boldsymbol{Need}$ is defined to be $\boldsymbol{Max}-\boldsymbol{Allocation}$ and is as follows:
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202311080132374.png)
We claim that the system is currently in a safe state. Indeed, the sequence <$T_{1}$, $T_{3}$, $T_{4}$, $T_{2}$, $T_{0}$> satisfies the safety criteria. Suppose now that thread $T_{1}$ requests one additional instance of resource type $A$ and two instances of resource type $C$, so $\boldsymbol{Request}_{1}$ = (1,0,2). To decide whether this request can be immediately granted, we first check that $\boldsymbol{Request}_{1}\leq\boldsymbol{Available}$--that is, that(1,0,2) $\leq$ (3,3,2), which is true. We then pretend that this request has been fulfilled, and we arrive at the following new state:
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202311080133164.png)

We must determine whether this new system state is safe. To do so, we execute our safety algorithm and find that the sequence <$T_{1}$, $T_{3}$, $T_{4}$, $T_{0}$, $T_{2}$> satisfies the safety requirement. Hence, we can immediately grant the request of thread $T_{1}$.

You should be able to see, however, that when the system is in this state, a request for (3,3,0) by $T_{4}$ cannot be granted, since the resources are not available. Furthermore, a request for (0,2,0) by $T_{0}$ cannot be granted, even though the resources are available, since the resulting state is unsafe.

We leave it as a programming exercise for students to implement the banker's algorithm.

## 8.7 Deadlock Detection

If a system does not employ either a deadlock-prevention or a deadlock-avoidance algorithm, then a deadlock situation may occur. In this environment, the system may provide:

- An algorithm that examines the state of the system to determine whether a deadlock has occurred
- An algorithm to recover from the deadlock

Next, we discuss these two requirements as they pertain to systems with only a single instance of each resource type, as well as to systems with several instances of each resource type. At this point, however, we note that a detection-and-recovery scheme requires overhead that includes not only the run-time costs of maintaining the necessary information and executing the detection algorithm but also the potential losses inherent in recovering from a deadlock.

### 8.7.1 Single Instance of Each Resource Type

If all resources have only a single instance, then we can define a deadlock-detection algorithm that uses a variant of the resource-allocation graph, called a **wait-for** graph. We obtain this graph from the resource-allocation graph by removing the resource nodes and collapsing the appropriate edges.

More precisely, an edge from $T_{i}$ to $T_{j}$ in a wait-for graph implies that thread $T_{i}$ is waiting for thread $T_{j}$ to release a resource that $T_{i}$ needs. An edge exists in a wait-for graph if and only if the corresponding resource-allocation graph contains two edges $T_{i}\to R_{q}$ and $R_{q}\to T_{j}$ for some resource $R_{q}$. In Figure 8.11, we present a resource-allocation graph and the corresponding wait-for graph.

![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202311080133487.png)

As before, a deadlock exists in the system if and only if the wait-for graph contains a cycle. To detect deadlocks, the system needs to _maintain_ the wait-for graph and periodically _inovoke an algorithm_ that searches for a cycle in the graph. An algorithm to detect a cycle in a graph requires $O(n^{2})$ operations, where $n$ is the number of vertices in the graph.

The BCC toolkit described in Section 2.10.4 provides a tool that can detect potential deadlocks with Pthreads mutex locks in a user process running on a Linux system. The BCC tool deadlock_detector operates by inserting probes which trace calls to the pthread_mutex_lock() and pthread_mutex_unlock() functions. When the specified process makes a call to either function, deadlock_detector constructs a wait-for graph of mutex locks in that process, and reports the possibility of deadlock if it detects a cycle in the graph.

### 8.7.2 Several Instances of a Resource Type

The wait-for graph scheme is not applicable to a resource-allocation system with multiple instances of each resource type. We turn now to a deadlock-detection algorithm that is applicable to such a system. The algorithm employs several time-varying data structures that are similar to those used in the banker's algorithm (Section 8.6.3):

- **Available**. A vector of length $m$ indicates the number of available resources of each type.

> **DEADLOCK DETECTION USING JAVA THREAD DUMPS**
>
> Although Java does not provide explicit support for deadlock detection, a thread dump can be used to analyze a running program to determine if there is a deadlock. A thread dump is a useful debugging tool that displays a snapshot of the states of all threads in a Java application. Java thread dumps also show locking information, including which locks a blocked thread is waiting to acquire. When a thread dump is generated, the JVM searches the wait-for graph to detect cycles, reporting any deadlocks it detects. To generate a thread dump of a running application, from the command line enter:
> `Ctrl-L (UNIX, Linux, or macOS)`
> `Ctrl-Break (Windows)`
> In the source-code download for this text, we provide a Java example of the program shown in Figure 8.1 and describe how to generate a thread dump that reports the deadlocked Java threads.

- **Allocation**. An $n\times m$ matrix defines the number of resources of each type currently allocated to each thread.
- **Request**. An $n\times m$ matrix indicates the current request of each thread. If $\textit{Request}[i][j]$ equals $k$, then thread $T_{i}$ is requesting $k$ more instances of resource type $R_{j}$.

The $\leq$ relation between two vectors is defined as in Section 8.6.3. To simplify notation, we again treat the rows in the matrices $\textit{Allocation}$ and $\textit{Request}$ as vectors; we refer to them as $\textit{Allocation}_{i}$ and $\textit{Request}_{i}$. The detection algorithm described here simply investigates every possible allocation sequence for the threads that remain to be completed. Compare this algorithm with the banker's algorithm of Section 8.6.3.

1. Let $\textit{Work}$ and $\textit{Finish}$ be vectors of length $m$ and $n$, respectively. Initialize $\textit{Work}=\textit{Available}$. For $i=0$, $1$,..., $n$-$1$, if $\textit{Allocation}_{i}\neq 0$, then $\textit{Finish}[i]=\textit{false}$. Otherwise, $\textit{Finish}[i]=\textit{true}$.
2. Find an index $i$ such that both
  a. $\textit{Finish}[i]==\textit{false}$
  b. $\textit{Request}_{i}\leq\textit{Work}$
 If no such $i$ exists, go to step 4.
3. $\textit{Work}=\textit{Work}+\textit{Allocation}_{i}$ $\textit{Finish}[i]=\textit{true}$ Go to step 2.
4. If $\textit{Finish}[i]==\textit{false}$ for some $i$, $0\leq i<n$, then the system is in a deadlocked state. Moreover, if $\textit{Finish}[i]==\textit{false}$, then thread $T_{i}$ is deadlocked.

This algorithm requires an order of $m\times n^{2}$ operations to detect whether the system is in a deadlocked state.

You may wonder why we reclaim the resources of thread $T_{i}$ (in step 3) as soon as we determine that $\textbf{{Request}}_{i}\leq\textbf{{Work}}$ (in step 2b). We know that $T_{i}$ is currently _not_ involved in a deadlock (since $\textbf{{Request}}_{i}\leq\textbf{{Work}}$). Thus, we take an optimistic attitude and assume that $T_{i}$ will require no more resources to complete its task; it will thus soon return all currently allocated resources to the system. If our assumption is incorrect, a deadlock may occur later. That deadlock will be detected the next time the deadlock-detection algorithm is invoked.

To illustrate this algorithm, we consider a system with five threads $T_{0}$ through $T_{4}$ and three resource types $A$, $B$, and $C$. Resource type $A$ has seven instances, resource type $B$ has two instances, and resource type $C$ has six instances. The following snapshot represents the current state of the system:
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202311080136277.png)
We claim that the system is not in a deadlocked state. Indeed, if we execute our algorithm, we will find that the sequence $<$$T_{0}$, $T_{2}$, $T_{3}$, $T_{1}$, $T_{4}$$>$ results in $\textbf{{Finish}}[i]$ == _true_ for all $i$.

Suppose now that thread $T_{2}$ makes one additional request for an instance of type $C$. The $\textbf{{Request}}$ matrix is modified as follows:
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202311080136872.png)
We claim that the system is now deadlocked. Although we can reclaim the resources held by thread $T_{0}$, the number of available resources is not sufficient to fulfill the requests of the other threads. Thus, a deadlock exists, consisting of threads $T_{1}$, $T_{2}$, $T_{3}$, and $T_{4}$.

### 8.7.3 Detection-Algorithm Usage

When should we invoke the detection algorithm? The answer depends on two factors:

1. How **_often_** is a deadlock likely to occur?
2. How **_many_** threads will be affected by deadlock when it happens?

> **MANAGING DEADLOCK IN DATABASES**
>
> Database systems provide a useful illustration of how both open-source and commercial software manage deadlock. Updates to a database may be performed as transactions, and to ensure data integrity, locks are typically used. A transaction may involve several locks, so it comes as no surprise that deadlocks are possible in a database with multiple concurrent transac- tions. To manage deadlock, most transactional database systems include a deadlock detection and recovery mechanism. The database server will peri- odically search for cycles in the wait-for graph to detect deadlock among a set of transactions. When deadlock is detected, a victim is selected and the transaction is aborted and rolled back, releasing the locks held by the victim transaction and freeing the remaining transactions from deadlock. Once the remaining transactions have resumed, the aborted transaction is reissued. Choice of a victim transaction depends on the database system; for instance, MySQL attempts to select transactions that minimize the number of rows being inserted, updated, or deleted.

If deadlocks occur frequently, then the detection algorithm should be invoked frequently. Resources allocated to deadlocked threads will be idle until the deadlock can be broken. In addition, the number of threads involved in the deadlock cycle may grow.

Deadlocks occur only when some thread makes a request that cannot be granted immediately. This request may be the final request that completes a chain of waiting threads. In the extreme, then, we can invoke the deadlock- detection algorithm every time a request for allocation cannot be granted immediately. In this case, we can identify not only the deadlocked set of threads but also the specific thread that “caused” the deadlock. (In reality, each of the deadlocked threads is a link in the cycle in the resource graph, so all of them, jointly, caused the deadlock.) If there are many different resource types, one request may create many cycles in the resource graph, each cycle completed by the most recent request and “caused” by the one identifiable thread.

Of course, invoking the deadlock-detection algorithm for every resource request will incur considerable overhead in computation time. A less expen- sive alternative is simply to invoke the algorithm at defined intervals—for example, once per hour or whenever CPU utilization drops below 40 percent. (A deadlock eventually cripples system throughput and causes CPU utilization to drop.) If the detection algorithm is invoked at arbitrary points in time, the resource graph may contain many cycles. In this case, we generally cannot tell which of the many deadlocked threads “caused” the deadlock.

## 8.8 Recovery from Deadlock

When a detection algorithm determines that a deadlock exists, several alter- natives are available. One possibility is to inform the operator that a deadlock has occurred and to let the operator deal with the deadlock manually. Another possibility is to let the system recover from the deadlock automatically. There are two options for breaking a deadlock. One is simply to abort one or more threads to break the circular wait. The other is to preempt some resources from one or more of the deadlocked threads.

Data-drivenare two options for breaking a deadlock. One is simply to abort one or more threads to break the circular wait. The other is to preempt some resources from one or more of the deadlocked threads.

### 8.8.1 Process and Thread Termination

To eliminate deadlocks by aborting a process or thread, we use one of two methods. In both methods, the system reclaims all resources allocated to the terminated processes.

- **Abort all deadlocked processes**. This method clearly will break the deadlock cycle, but at great expense. The deadlocked processes may have computed for a long time, and the results of these partial computations must be discarded and probably will have to be recomputed later.
- **Abort one process at a time until the deadlock cycle is eliminated**. This method incurs considerable overhead, since after each process is aborted, a deadlock-detection algorithm must be invoked to determine whether any processes are still deadlocked.

Aborting a process may not be easy. If the process was in the midst of updating a file, terminating it may leave that file in an incorrect state. Similarly, if the process was in the midst of updating shared data while holding a mutex lock, the system must restore the status of the lock as being available, although no guarantees can be made regarding the integrity of the shared data.

If the partial termination method is used, then we must determine which deadlocked process (or processes) should be terminated. This determination is a policy decision, similar to CPU-scheduling decisions. The question is basically an economic one; we should abort those processes whose termination will incur the minimum cost. Unfortunately, the term _minimum cost_ is not a precise one. Many factors may affect which process is chosen, including:

1. What the priority of the process is
2. How long the process has computed and how much longer the process will compute before completing its designated task
3. How many and what types of resources the process has used (for example, whether the resources are simple to preempt)
4. How many more resources the process needs in order to complete
5. How many processes will need to be terminated

### 8.8.2 Resource Preemption

To eliminate deadlocks using resource preemption, we successively preempt some resources from processes and give these resources to other processes until the deadlock cycle is broken.

If preemption is required to deal with deadlocks, then three issues need to be addressed:

1. **Selecting a victim**. Which resources and which processes are to be pre-empted? As in process termination, we must determine the order of preemption to minimize cost. Cost factors may include such parameters as the number of resources a deadlocked process is holding and the amount of time the process has thus far consumed.
2. **Rollback**. If we preempt a resource from a process, what should be done with that process? Clearly, it cannot continue with its normal execution; it is missing some needed resource. We must roll back the process to some safe state and restart it from that state. Since, in general, it is difficult to determine what a safe state is, the simplest solution is a total rollback: abort the process and then restart it. Although it is more effective to roll back the process only as far as necessary to break the deadlock, this method requires the system to keep more information about the state of all running processes.
3. **Starvation**. How do we ensure that starvation will not occur? That is, how can we guarantee that resources will not always be preempted from the same process? In a system where victim selection is based primarily on cost factors, it may happen that the same process is always picked as a victim. As a result, this process never completes its designated task, a starvation situation any practical system must address. Clearly, we must ensure that a process can be picked as a victim only a (small) finite number of times. The most common solution is to include the number of rollbacks in the cost factor.

## 8.9 Summary

- Deadlock occurs in a set of processes when every process in the set is waiting for an event that can only be caused by another process in the set.
- There are four necessary conditions for deadlock: (1) mutual exclusion, (2) hold and wait, (3) no preemption, and (4) circular wait. Deadlock is only possible when all four conditions are present.
- Deadlocks can be modeled with resource-allocation graphs, where a cycle indicates deadlock.
- Deadlocks can be prevented by ensuring that one of the four necessary conditions for deadlock cannot occur. Of the four necessary conditions, eliminating the circular wait is the only practical approach.
- Deadlock can be avoided by using the banker's algorithm, which does not grant resources if doing so would lead the system into an unsafe state where deadlock would be possible.
- A deadlock-detection algorithm can evaluate processes and resources on a running system to determine if a set of processes is in a deadlocked state.
- If deadlock does occur, a system can attempt to recover from the deadlock by either aborting one of the processes in the circular wait or preempting resources that have been assigned to a deadlocked process.

8.1 List three examples of deadlocks that are not related to a computer-system environment.

8.2 Suppose that a system is in an unsafe state. Show that it is possible for the threads to complete their execution without entering a deadlocked state.

8.3 Consider the following snapshot of a system:
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202311080141812.png)

Answer the following questions using the banker's algorithm:

- What is the content of the matrix _Need_?
- Is the system in a safe state?
- If a request from thread $T_{1}$ arrives for (0,4,2,0), can the request be granted immediately?

8.4 A possible method for preventing deadlocks is to have a single, higher-order resource that must be requested before any other resource. For example, if multiple threads attempt to access the synchronization objects $A\cdots E$, deadlock is possible. (Such synchronization objects may include mutexes, semaphores, condition variables, and the like.) We can prevent deadlock by adding a sixth object $F$. Whenever a thread wants to acquire the synchronization lock for any object $A\cdots E$, it must first acquire the lock for object $F$. This solution is known as containment: the locks for objects $A\cdots E$ are contained within the lock for object $F$. Compare this scheme with the circular-wait scheme of Section 8.5.4.

8.5 Prove that the safety algorithm presented in Section 8.6.3 requires an order of $m\times n^{2}$ operations.

8.6 Consider a computer system that runs 5,000 jobs per month and has no deadlock-prevention or deadlock-avoidance scheme. Deadlocks occur about twice per month, and the operator must terminate and rerun about ten jobs per deadlock. Each job is worth about two dollars (in CPU time), and the jobs terminated tend to be about half done when they are aborted.

A systems programmer has estimated that a deadlock-avoidance algorithm (like the banker's algorithm) could be installed in the system with an increase of about 10 percent in the average execution time perjob. Since the machine currently has 30 percent idle time, all 5,000 jobs per month could still be run, although turnaround time would increase by about 20 percent on average.

- What are the arguments for installing the deadlock-avoidance algorithm?
- What are the arguments against installing the deadlock-avoidance algorithm?

8.7 Can a system detect that some of its threads are starving? If you answer "yes," explain how it can. If you answer "no," explain how the system can deal with the starvation problem.

8.8 Consider the following resource-allocation policy. Requests for and releases of resources are allowed at any time. If a request for resources cannot be satisfied because the resources are not available, then we check any threads that are blocked waiting for resources. If a blocked thread has the desired resources, then these resources are taken away from it and are given to the requesting thread. The vector of resources for which the blocked thread is waiting is increased to include the resources that were taken away.

For example, a system has three resource types, and the vector $\boldsymbol{Available}$ is initialized to (4,2,2). If thread $T_{0}$ asks for (2,2,1), it gets them. If $T_{1}$ asks for (1,0,1), it gets them. Then, if $T_{0}$ asks for (0,0,1), it is blocked (resource not available). If $T_{2}$ now asks for (2,0,0), it gets the available one (1,0,0), as well as one that was allocated to $T_{0}$ (since $T_{0}$ is blocked). $T_{0}$'s $\boldsymbol{Allocation}$ vector goes down to (1,2,1), and its $\boldsymbol{Need}$ vector goes up to (1,0,1).

- Can deadlock occur? If you answer "yes," give an example. If you answer "no," specify which necessary condition cannot occur.
- Can indefinite blocking occur? Explain your answer.

8.9 Consider the following snapshot of a system:
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202311080142578.png)
Using the banker's algorithm, determine whether or not each of the following states is unsafe. If the state is safe, illustrate the order in which the threads may complete. Otherwise, illustrate why the state is unsafe.

- $\boldsymbol{Available}=(0,3,0,1)$
- $\boldsymbol{Available}=(1,0,0,2)$

8.10 Suppose that you have coded the deadlock-avoidance safety algorithm that determines if a system is in a safe state or not, and now have been asked to implement the deadlock-detection algorithm. Can you do so by simply using the safety algorithm code and redefining $\textit{Max}_{i}=\textit{Waiting}_{i}$ + $\textit{Allocation}_{i}$, where $\textit{Waiting}_{i}$ is a vector specifying the resources for which thread $i$ is waiting and $\textit{Allocation}_{i}$ is as defined in Section 8.6? Explain your answer.

**8.11**: Is it possible to have a deadlock involving only one single-threaded process? Explain your answer.

## Further Reading

Most research involving deadlock was conducted many years ago. [Dijkstra(1965)] was one of the first and most influential contributors in the deadlock area.

Details of how the MySQL database manages deadlock can be found at [http://dev.mysql.com/](http://dev.mysql.com/).

Details on the lockdep tool can be found at [https://www.kernel.org/doc/Documentation/locking/lockdep-design.txt](https://www.kernel.org/doc/Documentation/locking/lockdep-design.txt).

## Bibliography

**[11]** **[12]** **E. W. Dijkstra, "Cooperating Sequential Processes", Technical report, Technological University, Eindhoven, the Netherlands (1965).**

## Chapter 8 Exercises

- 8.12 Consider the traffic deadlock depicted in Figure 8.12.
- Show that the four necessary conditions for deadlock hold in this example.
- State a simple rule for avoiding deadlocks in this system.
- 8.13 Draw the resource-allocation graph that illustrates deadlock from the program example shown in Figure 8.1 in Section 8.2.
- 8.14 In Section 6.8.1, we described a potential deadlock scenario involving processes $P_{0}$ and $P_{1}$ and semaphores $\mathsf{S}$ and $\mathsf{Q}$. Draw the resource-allocation graph that illustrates deadlock under the scenario presented in that section.
- 8.15 Assume that a multithreaded application uses only reader$-$writer locks for synchronization. Applying the four necessary conditions for deadlock, is deadlock still possible if multiple reader$-$writer locks are used?
- 8.16 The program example shown in Figure 8.1 doesn't always lead to deadlock. Describe what role the CPU scheduler plays and how it can contribute to deadlock in this program.
- 8.17 In Section 8.5.4, we described a situation in which we prevent deadlock by ensuring that all locks are acquired in a certain order. However, we also point out that deadlock is possible in this situation if two threads simultaneously invoke the transaction() function. Fix the transaction() function to prevent deadlocks.
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202311080143305.png)

8.18 Which of the six resource-allocation graphs shown in Figure 8.12 illustrate deadlock? For those situations that are deadlocked, provide the cycle of threads and resources. Where there is not a deadlock situation, illustrate the order in which the threads may complete execution.

8.19 Compare the circular-wait scheme with the various deadlock-avoidance schemes (like the banker's algorithm) with respect to the following issues:

- Runtime overhead
- System throughput
- In a real computer system, neither the resources available nor the demands of threads for resources are consistent over long periods (months). Resources break or are replaced, new processes and threads come and go, and new resources are bought and added to the system. If deadlock is controlled by the banker's algorithm, which of the
*![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202311080144348.png)

following changes can be made safely (without introducing the possibility of deadlock), and under what circumstances?

1. Increase _Available_ (new resources added).
2. Decrease _Available_ (resource permanently removed from system).
3. Increase _Max_ for one thread (the thread needs or wants more resources than allowed).
4. Decrease _Max_ for one thread (the thread decides it does not need that many resources).
5. Increase the number of threads.
6. Decrease the number of threads.

8.21 Consider the following snapshot of a system:
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202311080145965.png)

What are the contents of the _Need_ matrix?

8.22 Consider a system consisting of four resources of the same type that are shared by three threads, each of which needs at most two resources. Show that the system is deadlock free.

8.23 Consider a system consisting of $m$ resources of the same type being shared by $n$ threads. A thread can request or release only one resource at a time. Show that the system is deadlock free if the following two conditions hold:

1. The maximum need of each thread is between one resource and $m$ resources.
2. The sum of all maximum needs is less than $m+n$.

8.24 Consider the version of the dining-philosophers problem in which the chopsticks are placed at the center of the table and any two of them can be used by a philosopher. Assume that requests for chopsticks are made one at a time. Describe a simple rule for determining whether a particular request can be satisfied without causing deadlock given the current allocation of chopsticks to philosophers.

8.25 Consider again the setting in the preceding exercise. Assume now that each philosopher requires three chopsticks to eat. Resource requests are still issued one at a time. Describe some simple rules for determining whether a particular request can be satisfied without causing deadlock given the current allocation of chopsticks to philosophers.

8.26 We can obtain the banker's algorithm for a single resource type from the general banker's algorithm simply by reducing the dimensionality of the various arrays by 1.
Show through an example that we cannot implement the multiple-resource-type banker's scheme by applying the single-resource-type scheme to each resource type individually.

8.27 Consider the following snapshot of a system:
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202311080147532.png)

 Using the banker’s algorithm, determine whether or not each of the following states is unsafe. If the state is safe, illustrate the order in which the threads may complete. Otherwise, illustrate why the state is unsafe.
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202311080148007.png)
8.28  Consider the following snapshot of a system:
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202311080148930.png)
Answer the following questions using the banker’s algorithm:
    a. Illustrate that the system is in a safe state by demonstrating an order in which the threads may complete.
    b. If a request from thread $T_4$ arrives for (2, 2, 2, 4), can the request be granted immediately?
    c. If a request from thread $T_2$ arrives for (0, 1, 1, 0), can the request be granted immediately?
    d. If a request from thread $T_3$ arrives for (2, 2, 1, 2), can the request be granted immediately?

What is the optimistic assumption made in the deadlock-detection algorithm? How can this assumption be violated?

- A single-lane bridge connects the two Vermont villages of North Tunbridge and South Tunbridge. Farmers in the two villages use this bridge to deliver their produce to the neighboring town. The bridge can become deadlocked if a northbound and a southbound farmer get on the bridge at the same time. (Vermont farmers are stubborn and are unable to back up.) Using semaphores and/or mutex locks, design an algorithm in pseudocode that prevents deadlock. Initially, do not be concerned about starvation (the situation in which northbound farmers prevent southbound farmers from using the bridge, or vice versa).
- Modify your solution to Exercise 8.30 so that it is starvation-free.

8.29  What is the optimistic assumption made in the deadlock-detection algorithm? How can this assumption be violated?  

8.30  A single-lane bridge connects the two Vermont villages of North Tun- bridge and South Tunbridge. Farmers in the two villages use this bridge to deliver their produce to the neighboring town. The bridge can become deadlocked if a northbound and a southbound farmer get on the bridge at the same time. (Vermont farmers are stubborn and are unable to back up.) Using semaphores and/or mutex locks, design an algorithm in pseudocode that prevents deadlock. Initially, do not be concerned about starvation (the situation in which northbound farmers prevent southbound farmers from using the bridge, or vice versa).

8.31  Modify your solution to Exercise 8.30 so that it is starvation-free.

8.32 Implement your solution to Exercise 8.30 using POSIX synchronization. In particular, represent northbound and southbound farmers as separate threads. Once a farmer is on the bridge, the associated thread will sleep for a random period of time, representing traveling across the bridge. Design your program so that you can create several threads representing the northbound and southbound farmers.

8.33 In Figure 8.7, we illustrate a transaction() function that dynamically acquires locks. In the text, we describe how this function presents difficulties for acquiring locks in a way that avoids deadlock. Using the Java implementation of transaction() that is provided in the source-code download for this text, modify it using the Sys-tem.identityHashCode() method so that the locks are acquired in order.

## Programming Projects

### Banker’s Algorithm

For this project, you will write a program that implements the banker’s algo- rithm discussed in Section 8.6.3. Customers request and release resources from the bank. The banker will grant a request only if it leaves the system in a safe state. A request that leaves the system in an unsafe state will be denied. Although the code examples that describe this project are illustrated in C, you may also develop a solution using Java.

### The Banker

The banker will consider requests from n customers for m resources types, as outlined in Section 8.6.3. The banker will keep track of the resources using the following data structures:
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202311080152874.png)
The banker will grant a request if it satisfies the safety algorithm outlined in Section 8.6.3.1. If a request does not leave the system in a safe state, the banker will deny it. Function prototypes for requesting and releasing resources are as follows:
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202311080152315.png)
The request resources() function should return 0 if successful and –1 if unsuccessful.

### Testing Your Implementation

Design a program that allows the user to interactively enter a request for resources, to release resources, or to output the values of the different data structures (available, maximum, allocation, and need) used with the banker's algorithm.

You should invoke your program by passing the number of resources of each type on the command line. For example, if there were four resource types, with ten instances of the first type, five of the second type, seven of the third type, and eight of the fourth type, you would invoke your program as follows:
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202311080153740.png)
The available array would be initialized to these values.

Your program will initially read in a file containing the maximum number of requests for each customer. For example, if there are five customers and four resources, the input file would appear as follows:
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202311080153409.png)
where each line in the input file represents the maximum request of each resource type for each customer. Your program will initialize the maximum array to these values.

Your program will then have the user enter commands responding to a request of resources, a release of resources, or the current values of the different data structures. Use the command 'RQ' for requesting resources, 'RL' for releasing resources, and '*' to output the values of the different data structures. For example, if customer 0 were to request the resources $(3,1,2,1)$, the following command would be entered:
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202311080154857.png)
Your program would then output whether the request would be satisfied or denied using the safety algorithm outlined in Section 8.6.3.1.

Similarly, if customer 4 were to release the resources $(1,2,3,1)$, the user would enter the following command:
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202311080155935.png)
Finally, if the command '$\star$' is entered, your program would output the values of the available, maximum, allocation, and need arrays.
