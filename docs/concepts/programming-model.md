---
title: Dataflow Programming Model
nav-id: programming-model
nav-pos: 1
nav-title: Programming Model
nav-parent_id: concepts
---
<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

* This will be replaced by the TOC
{:toc}

## Programs and Dataflows

The basic building blocks of Flink programs are **streams** and **transformations**. (Note that the
DataSets used in Flink's batch API are also streams internally -- more about that
later.) Conceptually a *stream* is a never-ending flow of data records, and a *transformation* is an
operation that takes one or more streams as input, and produces one or more output streams as a
result.

When executed, Flink programs are mapped to **streaming dataflows**, consisting of **streams** and transformation **operators**.
Each dataflow starts with one or more **sources** and ends in one or more **sinks**. The dataflows resemble
arbitrary **directed acyclic graphs** *(DAGs)*. Although special forms of cycles are permitted via
*iteration* constructs, for the most part we will gloss over this for simplicity.

<img src="{{ site.baseurl }}/fig/program_dataflow.svg" alt="A DataStream program, and its dataflow." class="offset" width="80%" />

Often there is a one-to-one correspondence between the transformations in the programs and the operators
in the dataflow. Sometimes, however, one transformation may consist of multiple transformation operators.

{% top %}

## Parallel Dataflows

Programs in Flink are inherently parallel and distributed. This parallelism is expressed in Flink's
DataStream API with the *keyBy()* operator, which can be thought of as a declaration that the stream can
be operated on in parallel for different values of the key.

*Streams* are split into **stream partitions**, and *operators* are split into **operator
subtasks**. The operator subtasks are independent of one another, and execute in different threads
and possibly on different machines or containers.

The number of operator subtasks is the **parallelism** of that particular operator. The parallelism of a stream
is always that of its producing operator. Different operators of the same program may have different
levels of parallelism.

<img src="{{ site.baseurl }}/fig/parallel_dataflow.svg" alt="A parallel dataflow" class="offset" width="80%" />

Streams can transport data between two operators in a *one-to-one* (or *forwarding*) pattern, or in a *redistributing* pattern:

  - **One-to-one** streams (for example between the *Source* and the *map()* operators in the figure
    above) preserve the partitioning and ordering of the
    elements. That means that subtask[1] of the *map()* operator will see the same elements in the same order as they
    were produced by subtask[1] of the *Source* operator.

  - **Redistributing** streams (as between *map()* and *keyBy/window* above, as well as between
    *keyBy/window* and *Sink*) change the partitioning of streams. Each *operator subtask* sends
    data to different target subtasks, depending on the selected transformation. Examples are
    *keyBy()* (which re-partitions by hashing the key), *broadcast()*, or *rebalance()* (which
    re-partitions randomly). In a *redistributing* exchange the ordering among the elements is
    only preserved within each pair of sending and receiving subtasks (for example, subtask[1]
    of *map()* and subtask[2] of *keyBy/window*). So in this example, the ordering within each key
    is preserved, but the parallelism does introduce non-determinism regarding the order in
    which the aggregated results for different keys arrive at the sink.

{% top %}

## Windows

Aggregating events (e.g., counts, sums) works differently on streams than in batch processing.
For example, it is impossible to count all elements in a stream,
because streams are in general infinite (unbounded). Instead, aggregates on streams (counts, sums, etc),
are scoped by **windows**, such as *"count over the last 5 minutes"*, or *"sum of the last 100 elements"*.

Windows can be *time driven* (example: every 30 seconds) or *data driven* (example: every 100 elements).
One typically distinguishes different types of windows, such as *tumbling windows* (no overlap),
*sliding windows* (with overlap), and *session windows* (punctuated by a gap of inactivity).

<img src="{{ site.baseurl }}/fig/windows.svg" alt="Time- and Count Windows" class="offset" width="80%" />

More window examples can be found in this [blog post](https://flink.apache.org/news/2015/12/04/Introducing-windows.html).

{% top %}

## Time

When referring to time in a streaming program (for example to define windows), one can refer to different notions
of time:

  - **Event Time** is the time when an event was created. It is usually described by a timestamp in the events,
    for example attached by the producing sensor, or the producing service. Flink accesses event timestamps
    via [timestamp assigners]({{ site.baseurl }}/dev/event_timestamps_watermarks.html).

  - **Ingestion time** is the time when an event enters the Flink dataflow at the source operator.

  - **Processing Time** is the local time at each operator that performs a time-based operation.

<img src="{{ site.baseurl }}/fig/event_ingestion_processing_time.svg" alt="Event Time, Ingestion Time, and Processing Time" class="offset" width="80%" />

More details on how to handle time are in the [event time docs]({{ site.baseurl }}/dev/event_time.html).

{% top %}

## Stateful Operations

While many operations in a dataflow simply look at one individual *event at a time* (for example an event parser),
some operations remember information across multiple events (for example window operators).
These operations are called **stateful**.

The state of stateful operations is maintained in what can be thought of as an embedded key/value store.
The state is partitioned and distributed strictly together with the streams that are read by the
stateful operators. Hence, access to the key/value state is only possible on *keyed streams*, after a *keyBy()* function,
and is restricted to the values associated with the current event's key. Aligning the keys of streams and state
makes sure that all state updates are local operations, guaranteeing consistency without transaction overhead.
This alignment also allows Flink to redistribute the state and adjust the stream partitioning transparently.

<img src="{{ site.baseurl }}/fig/state_partitioning.svg" alt="State and Partitioning" class="offset" width="50%" />

{% top %}

## Checkpoints for Fault Tolerance

Flink implements fault tolerance using a combination of **stream replay** and **checkpointing**. A
checkpoint is related to a specific point in each of the input streams along with the corresponding state for each
of the operators. A streaming dataflow can be resumed from a checkpoint while maintaining consistency *(exactly-once
processing semantics)* by restoring the state of the operators and replaying the events from the
point of the checkpoint.

The checkpoint interval is a means of trading off the overhead of fault tolerance during execution with the recovery time (the number
of events that need to be replayed).

More details on checkpoints and fault tolerance are in the [fault tolerance docs]({{ site.baseurl }}/internals/stream_checkpointing.html).

{% top %}

## Batch on Streaming

Flink executes batch programs as a special case of streaming programs, where the streams are bounded (finite number of elements).
A *DataSet* is treated internally as a stream of data. The concepts above thus apply to batch programs in the
same way as well as they apply to streaming programs, with minor exceptions:

  - Programs in the DataSet API do not use checkpoints. Recovery happens by fully replaying the streams.
    That is possible, because inputs are bounded. This pushes the cost more towards the recovery,
    but makes the regular processing cheaper, because it avoids checkpoints.

  - Stateful operations in the DataSet API use simplified in-memory/out-of-core data structures, rather than
    key/value indexes.

  - The DataSet API introduces special synchronized (superstep-based) iterations, which are only possible on
    bounded streams. For details, check out the [iteration docs]({{ site.baseurl }}/dev/batch/iterations.html).

{% top %}

## Next Steps

Continue with the basic concepts in Flink's [Distributed Runtime]({{ site.baseurl }}/concepts/runtime).
