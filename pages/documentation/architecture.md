---
title: Architecture
hide_sidebar: false
keywords: apex architecture
permalink: architecture.html
topnav: topnav
folder: documentation
toc: false
---


# Apache Apex Platform Overview {#overview}
## Streaming Computational Model {#StreamingComputeModel}
In this chapter, we describe the the basics of the real-time streaming platform and its computational model.

The platform is designed to enable completely asynchronous real time computations done in as unblocked a way as possible with minimal overhead .

Applications running in the platform are represented by a Directed Acyclic Graph (DAG) made up of  operators and streams. All computations are done in memory on arrival of the input data, with an option to save the output to disk (HDFS) in a non-blocking way. The data that flows between operators consists of atomic data elements. Each data element along with its type definition (henceforth called schema) is called a tuple. An application is a design of the flow of these tuples to and from the appropriate compute units to enable the computation of the final desired results. A message queue (henceforth called  buffer server) manages tuples streaming between compute units in different processes.This server keeps track of all consumers, publishers, partitions, and enables replay. More information is given in later section.

The streaming application is monitored by a decision making entity called STRAM (streaming application manager). STRAM is designed to be a light weight controller that has minimal but sufficient interaction with the application. This is done via periodic heartbeats. The STRAM does the initial launch and periodically analyzes the system metrics to decide if any run time action needs to be taken.

A fundamental building block for the streaming platform is the concept of breaking up a stream into equal finite time slices called streaming windows. Each window contains the ordered set of tuples in that time slice. A typical duration of a window is 500 ms, but can be configured per application (the Yahoo! Finance application configures this value in the properties.xml file to be 1000ms = 1s). Each window is preceded by a begin_window event and is terminated by an end_window event, and is assigned a unique window ID. Even though the platform performs computations at the tuple level, bookkeeping is done at the window boundary, making the computations within a window an atomic event in the platform.  We can think of each window as an atomic micro-batch of tuples, to be processed together as one atomic operation (See Figure 2).

This atomic batching allows the platform to avoid the very steep per tuple bookkeeping cost and instead has a manageable per batch bookkeeping cost. This translates to higher throughput, low recovery time, and higher scalability. Later in this document we illustrate how the atomic micro-batch concept allows more efficient optimization algorithms.

The platform also has in-built support for application windows.  An application window is part of the application specification, and can be a small or large multiple of the streaming window.  An example from our Yahoo! Finance test application is the moving average, calculated over a sliding application window of 5 minutes which equates to 300 (= 5 * 60) streaming windows.

Note that these two window concepts are distinct.  A streaming window is an abstraction of many tuples into a higher atomic event for easier management.  An application window is a group of consecutive streaming windows used for data aggregation (e.g. sum, average, maximum, minimum) on a per operator level.



Alongside the platform, a set of predefined, benchmarked standard library operator templates is provided for ease of use and rapid development of application. These operators are open sourced to Apache Software Foundation under the project name “Malhar” as part of our efforts to foster community innovation. These operators can be used in a DAG as is, while others have properties that can be set to specify the desired computation. Those interested in details, should refer to Apex-Malhar operator library.

The platform is a Hadoop YARN native application. It runs in a Hadoop cluster just like any other YARN application (MapReduce etc.) and is designed to seamlessly integrate with rest of Hadoop technology stack. It leverages Hadoop as much as possible and relies on it as its distributed operating system. Hadoop dependencies include resource management, compute/memory/network allocation, HDFS, security, fault tolerance, monitoring, metrics, multi-tenancy, logging etc. Hadoop classes/concepts are reused as much as possible. The aim is to enable enterprises to leverage their existing Hadoop infrastructure for real time streaming applications. The platform is designed to scale with big data applications and scale with Hadoop.

A streaming application is an asynchronous execution of computations across distributed nodes. All computations are done in parallel on a distributed cluster. The computation model is designed to do as many parallel computations as possible in a non-blocking fashion. The task of monitoring of the entire application is done on (streaming) window boundaries with a streaming window as an atomic entity. A window completion is a quantum of work done. There is no assumption that an operator can be interrupted at precisely a particular tuple or window.

An operator itself also cannot assume or predict the exact time a tuple that it emitted would get consumed by downstream operators. The operator processes the tuples it gets and simply emits new tuples based on its business logic. The only guarantee it has is that the upstream operators are processing either the current or some later window, and the downstream operator is processing either the current or some earlier window. The completion of a window (i.e. propagation of the end_window event through an operator) in any operator guarantees that all upstream operators have finished processing this window. Thus, the end_window event is blocking on an operator with multiple outputs, and is a synchronization point in the DAG. The begin_window event does not have any such restriction, a single begin_window event from any upstream operator triggers the operator to start processing tuples.
