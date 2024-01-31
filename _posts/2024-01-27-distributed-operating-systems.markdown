---
layout: post
title:  "Towards a Distributed OS for Data-Intensive Cloud Applications"
date:   2024-01-27 19:14:45 -0600
categories: jekyll update
---

<h3>
  <a href="https://www2.eecs.berkeley.edu/Pubs/TechRpts/2024/EECS-2024-3.html" target=_blank>
    Towards a Distributed OS for Data-Intensive Cloud Applications
  </a> by Stephanie Wang 
</h3>

> The following notes represents my interpretation of the above work. 
Anything discussed belong to the original author(s) unless explicitly noted
otherwise. Any errors or misinterpretations are my own.

* Problem: growing demands for data-intensive applications
* Solution:
  * HW: horizontal scaling and hw accelerators
  * SW: specilized distributed execution frameworks (e.g., data analytics, ml)
* Issues with current frameworks:
  * each implements distrubuted execution (duplicated work)
  * each does it in its unique way (hard to evolve)
  * applications with heterogeneous workloads need to stich together different
  specialized frameworks

Increasingly, there exists a class of heterogeneous applications (e.g., ML)
that require scaling with distributed execution. Such apps have historically utilized
domain-specific frameworks to handle their workloads. As they continue to be scaled,
it is clear that there should exist a common execution layer that provides distributed 
execution services, much like what a traditional OS does for applications sharing 
the same physical resoures. 

other such systems: MPI

Requirements 
  * a flexible unit of parallel computation
  * an abstraction for distributed memory
  * developer choice over recovery strategy

on computation side
  general-purpose programming interface that extends RPC
  `actor model` for computation
  single function call as the smallest unit of parallelism (`task`)
on memory side
  `distributed futures` enables pass-by-reference semantics for RPC

Landscape of data-intensive applications

Data analytics vs ML

  * on parallel computation, data analytics applications can continue to scale
  using data parallelism while ML workloads require model parallelism in addition
  to data parallelism. Hence, we need a flexible unit of parallel computation. 

  * on distributed memory side, different frameworks or units of computation need
  to efficiently exchange data. Hence, we need an abstraction for distributed
  memory.

  * on recovery strategy, in data analytics challenges of fault tolerance stem from
  the problem of producing consistent snapshots with low overhead while in ML, it
  stems from the tightly coupled and fate-sharing nature of ML clusters. Hence,
  we need a choice over recovery strategy. 


Primary execution model for data analytics: 
  * batch processing (MapReduce, Spark)
    * processes fixed-size batch of data at a time
    * typically offline, needs large input and high throughput
      * input is partitioned and each is executed as a stage of parallel tasks
    * no need for feedback-loops
    * needs central scheduler
    * fault tolerance provided through transparent re-execution of lost tasks

  * stream processing
    * uses an asynchronous and stateful model
    * each data transform is instantiated by operator(s) that share input
      stream; data partitioning and task scheduling are done on the fly, 
      as each operator independently reads from its input stream once 
      it is ready to process another record
    * fault tolerance is provided through global checkpointing and rollback

While batch and stream processing models can in principle handle DL workloads, it comes
with heavy toll on performance. Fundamentally, this stems from different core
computational patterns of DL: stochastic gradient descent (SGD) and accelerator-based
computation. 

Scaling SGD with data parallelism is possible (multiple copies of a model
executing over a disjoint subset of training dataset) but requires 
1) frequent communication (thus, poor fit for batch processing)
2) synchronization (thus, poor fit for stream processing) between workers. 

In addition, these per-layer operations require accelerators (e.g., GPUs), 
which create challenges in fault tolerance and resource management. 
For former, GPU workers must synchronize frequently, which means they fate-share 
as opposed to being able to fail/recover independently. For latter, sharing memory 
across CPU and GPU threads is expensive as it requires copying; multiplexing 
GPU tasks ('kernels') is difficult; coordinating access to shared memory 
adds overhead; GPUs use specialized communication links (NVIDIA's NVLink) 
that are separate from commodity networks.

Finally, fully connected layers in DL models require all model weights to be in
memory for acceptable performance. If that exceeds GPU memory capacity, it adds
challenge of `model parallelism' or scaleout of a single model copy across
multiple GPUs. Model parallelism strategies can be further broken down
into tensor vs. pipeline parallelism. Scalability of both strategies is fundamentally
limited by increasing communication and/or memory cost.

Thus, adding native/efficient GPU support for existing data-parallel frameworks
is complex. It is also insufficient to build GPU-based frameworks since many
workloads require both CPU and GPU execution. 


Alternative Solutions
  * Monolithic frameworks (Distributed TensorFlow, Apache MXNet, and PyTorch Distributed)
    * re-implement a dataflow interface using low-level message-passing interfaces
    * expose a specialized DAG-based programming interface
    * fault tolerance is commonly supported through checkpointing
    * they aren't general enough (hyper-parameter search, RL, CPU-based preprocessing that overlaps with GPU computations)
    * so users have two options: 
      * add more workloads to these frameworks
        * very high incremental complexity 
        * can't still handle inference (low latency for dynamic requests in inference, vs. high-throughput
on static inputs for training)
      * create 'glue systems'
        * burden falls on the application developers, as specialized frameworks
        typically do not take on the necessary responsibilities of managing resources, 
        data movement, and recovery across frameworks


Propositions
  * A general-purpose distributed programming interface based on distributed futures, actors, and tasks
    * distributed futures extend rpc with a shared and immutable address space
  * Architecture for this interface 
    * Lineage stash
      * provides a strong guarantee of exactly-once semantics for distributed futures and actors
      * resolves the tradeoff between checkpointing- vs lineage-based recovery approaches
    * Ownership
      * provides RPC-like latency and scalability, but extends RPC with a fault-tolerant distributed memory layer
      * provides minimal but essential recovery guarantees
      * key applications of this system include applications that can implement checkpointing at the application level
    * Exoflow
      * builds upon the ownership system to expand the recovery guarantees and support more applications
      * unlike others, it decouples the unit of recovery from the unit of execution


<h3>
  Distributed futures and RPC
</h3>
A key abstraction here is distrubuted futures, which enables a necessary feature
for data-intensive RPC applications: a shared but immutable address space. This
feature is essential for memory efficiency (less copying) in these
types of applications. 

`Distributed futures` is built on top of RPC as an extension. Specifically, it is a
reference, to a possibly remote value, returned from a RPC call
(an implementation can choose to return by value, reference, or both). Additionally, 
these 'extended' asynch RPC functions (`task`) take references as arguments. If that
reference is a regular one (Ref), the system dereferences it and sends value to the 
executor. If it is a shared one (SharedRef), the executor receives the reference (which 
it also can pass around as reference) instead of value. Lastly, value and its
reference are 'deleted' by the language binginds when a Ref goes out of scope. 

Providing shared address space have been tried before. Some have addressed it at
the application level, where actual values are stored in a common data store
and references to them are exchanged in RPC calls. While this works, it forces 
application developers to manage memory manually. Frameworks like Apache Spark 
(big data processing) and Distributed TensorFlow (ML) also provide a shared address space, 
building on top of RPC. The issue with these domain-specific frameworks is
interoperability; in other words, applications cross-using these tools are
redundantly copy inefficient, where some components communicate via RPC and others do so via 
framework-specific protocols.

Given all of that, it might be better to extend RPC itself (API shown below) and use that abstraction to
build a common layer that manages memory automatically, including reclamation, movement, 
and memory pressure. Since the system has full-control over reference
creation/deletion (reference is first-class), it can provide distributed
garbage collection and can decide 'when' to move the data (and provide relevant
optimizations). First-class reference combined with a memory-aware scheduler can
also coordinate memory usage of concurrent requests, easing memory pressure.
Ideally, this layer provides a common foundation for specialized frameworks, which then can be 
used like libraries (in mix or individually) to build data-intensive applications. 

![APIs]({{site.baseurl}}/assets/2024-01-27-distributed-operating-systems_APIs.png)

Related abstractions for distributed memory
  * mutable shared state
  * distributed message queues
  * distributed data structures
  * primitives for a single “object”

<h3>
  Lineage Stash
</h3>

When designing fault tolerance for distrubuted fine-grained data processing applications, 
tradeoffs should be made to meet the following requirements as closely as possible
  * good performance during normal operation (high throughput and low latency)
  * high sensitivity to recovery time (should be short)
  * recovery correctness given many types of computations performed (some are deterministic and
  some are nondeterministic)

There are two general techniques for guaranteeing global consistency after a
failure: global checkpointing and logging. With global checkpointing, the
system takes periodic application checkpoints, and in the event of failure, reruns the
job from the latest checkpoint to a consistent but possibly different state (due to
nondeterministic execution). In general, checkpointing incurs high recovery
overhead (because of global rollback) and low run-time overhead (because other
than checkpointing itself, nondeterministic events aren't recorded). With logging, 
the system durably logs events during execution, and in the event of failure, 
exactly replays the events to recover lost state, without needing to rollback 
any process that did not fail. Logging-based techniques (lineage reconstruction) 
generally incur high run-time overhead (because all nondeterministic events 
must be recorded) and low recovery overhead (because of very local rollback). 

Lineage stash is a lineage-based system (low recovery time overhead) that tries to
reduce run-time overhead of previous such systems. Instead of recording a task's lineage 
before its execution (as previous ones do), each worker receives the lineage info,
updates its local stash (contains all tasks that worker has seen recently), forwards 
the most recent unsaved lineage and flushes its stash to a remote store. Since flushing 
is asynchronous, it has negligible impact on application latency during normal operation.

The main idea here is that instead of storing the lineage in a reliable store at
run-time, one can forward the full lineage along with every task call. 
A key to make this practical is to identify a mininum set of nondeterministic events 
to synchronously log to a local, in-memory stash and asynchronously flush that stash 
to a remote store such that global consistency can be recovered in a case of failure 
while also guaranteeing low runtime overhead. 

Lineage is recorded at the granularity of a batch (as opposed to e.g.,
partition or a single message) and is tracked through data dependencies (specified through task args) 
and stateful dependencies (created between tasks that execute consecutively on the same process).

![Lineage stash architecture]({{site.baseurl}}/assets/2024-01-27-distributed-operating-systems_lineage_stash_arch.png)

Ray system model
  * a distributed scheduler dispatches tasks to local worker processes
  based on their data and stateful dependencies
  * each Ray node can host multiple worker processes, which may be stateful
  (known as actors). They share an in-memory object store
  * system metadata, such as task descriptions and object locations, 
  is stored in a logically centralized global store

  * challenges in applying lineage reconstruction to above setting
    * the granularity at which lineage is recorded is much finer than 
    in previous lineage-based systems (latency and throughput)
    * the lineage is not only larger, it must also be updated at 
    runtime to guarantee exactly-once record processing (asynch record
    processing introduces nondeterministic events when an operator 
    processes data from multiple sources)
    * these problems motivate an asynchronous logging approach, which presents
    its own challenge, maintaining the decentralized state, and complicates
    both run-time (local state has to be flushed somehow) and recovery
    (failed operator's lineage isn't in the central location). The final 
    challenge is thus in designing simple protocols for flushing local 
    state and recovering after a failure. 
    * final challenges are:
    (1) removing the cost of recording lineage from the critical path of task execution
    (2) efficiently recording nondeterministic events
    (3) designing simple, scalable protocols for flushing and recovering lineage


What info to log?
  * since we can't log every message exchanged (each might be arbitrarily large and
  computation is performed at a batch level) and know that message content is usually
  computed deterministically (easy to recompute)
  * we can record the lineage (of object, to be precise) instead of the raw data. 
  Lineage of object consists of the task that created it and 
  the lineage of each of the task’s arguments (a process’s local state 
  and one or more objects). Object here is application data and task is 
  description of the computation.
  * if there is nondeterminism in execution order, that task execution order 
  must also be recorded in the lineage
  * if a process executes nondeterministic events during a task, application 
  can record such events as part of the task description

How to log it?
  * instead of storing the lineage reliably before the task is executed, forward 
  the lineage of each of the task’s inputs with the task invocation (so node
  executing the task has all info to reconstruct the task's inputs)
  * in deterministic applications, the lineage need not be forwarded during
  normal operation and each process only needs to remember the tasks 
  that it has submitted, by storing them in its local lineage stash
  * in nondeterministic applications, each process must also forward the lineage that
  it has seen so far

  > this doesn't scale as the lineage can become very large and forwarding it
  can be prohibitive, so timely flushing of the local stash is critical

  * asynchronously flush local stash to a global store with clever garbage
  collection. In this scheme, in the global store each task has
  a unique identifier and any process may read or append to any task. This
  means that only a single copy of each task is reliably stored, and 
  garbage collection of the stable storage can be handled by a single 
  background process, which erases tasks previous to the last global
  checkpoint.

How to recover the logs?
  * upon a failure, each remaining process simply flushes its local stash to the global store
  and replies to the recovering process once all writes have been acknowledged. The
  recovering process can then retrieve its lineage by walking the task dependencies in
  the global store, starting from the tasks resubmitted by the other processes
  * deterministic processes just need get lineage of tasks they had received from other processes
  * nondeterministic processes need initial execution order, as well as any nondeterministic events 
  that occured during task's execution

<hr>
TYPO

* p19 - conscious of data movement => unconscious of data movement
* p20 - single-progress program => single-process program
* p25 - shared adddress space => shared address space



