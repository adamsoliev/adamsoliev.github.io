---
layout: post
title:  "Distributed Operating Systems"
date:   2024-01-27 19:14:45 -0600
categories: jekyll update
---

<h3>
  <a href="https://www2.eecs.berkeley.edu/Pubs/TechRpts/2024/EECS-2024-3.html" target=_blank>
    Towards a Distributed OS for Data-Intensive Cloud Applications
  </a> by Stephanie Wang 
</h3>

> The following notes represents my interpretation of the above work. 
Anything discussed belong to the original author(s) unless explicitly noted. 
Any errors or misinterpretations are my own.

* Problem: growing demands for data-intensive applications
* Solution:
  * HW: horizontal scaling and hw accelerators
  * SW: specilized distributed execution frameworks (e.g., data analytics, ml)
* Issues:
  * each reinvents 'the distributed execution' wheel  
  * each does it in its own way, hence they aren't easy to evolve 
  * applications with heterogeneous workloads need to stich together different
  specialized frameworks

Increasingly, there exists a class of heterogeneous applications (e.g., ML)
that require scaling with distributed execution. Such apps have historically utilized
domain-specific frameworks to handle their workloads. As we continue to scale
these apps and face challenges, it is clear that there should exist an
execution layer that provides distributed execution services, much like what
a traditional OS does for applications sharing the same physical resoures. To
support different use cases, this system should allow distrubuted frameworks to
be built on top as libraries.

[picture]

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

two rollback recovery approaches for fault tolerance
* lineage reconstruction - low recovery overhead and high run-time overhead
* check-pointing - the opposite tradeoff

proposition
* lineage stash - low recovery and run-time overhead

<hr>
TYPO

* p19 - conscious of data movement => unconscious of data movement
* p20 - single-progress program => single-process program
* p25 - shared adddress space => shared address space



