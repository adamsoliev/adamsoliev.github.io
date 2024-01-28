---
layout: post
title:  "Distributed Operating Systems"
date:   2024-01-27 19:14:45 -0600
categories: jekyll update
---

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

This system should provide 
  * fine-grained and dynamic unit of parallelism
  * abstraction of distrubuted memory
  * (choice of) failure transparency 

on computation side
  general-purpose programming interface that extends RPC
  `actor model` for computation
  single function call as the smallest unit of parallelism (`task`)
on memory side
  `distributed futures` enables pass-by-reference semantics for RPC





