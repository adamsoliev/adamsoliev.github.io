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



