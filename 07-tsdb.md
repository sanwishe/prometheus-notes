# Writing a Time Series Database from Scratch

原文地址：Writing a Time Series Database from Scratch

原文是prometheus贡献者之一Fabian Reinartz的一篇关于prometheus存储和tsdb相关的论文。

## Current problem

### Time series data

We have a system that collects data points over time.

> identifier -> (t0, v0), (t1, v1), (t2, v2), (t3, v3), ....

**Each data point is a tuple of a timestamp and a value**. For the purpose of monitoring, the timestamp is an integer and the value any number. 

A sequence of data points with strictly monotonically increasing timestamps is a series, which is addressed by an identifier. Our identifier is a metric name with a dictionary of label dimensions. Label dimensions partition the measurement space of a single metric. **Each metric name plus a unique set of labels is its own time series that has a value stream associated with it**.

This is a typical set of series identifiers that are part of metric counting requests:

```
{__name__="requests_total", path="/status", method="GET", instance=”10.0.0.1:80”}
{__name__="requests_total", path="/status", method="POST", instance=”10.0.0.3:80”}
{__name__="requests_total", path="/", method="GET", instance=”10.0.0.2:80”}
```

### Vertical and Horizontal

```
series
  ^   
  │   . . . . . . . . . . . . . . . . .   . . . . .   {__name__="request_total", method="GET"}
  │     . . . . . . . . . . . . . . . . . . . . . .   {__name__="request_total", method="POST"}
  │         . . . . . . .
  │       . . .     . . . . . . . . . . . . . . . .                  ... 
  │     . . . . . . . . . . . . . . . . .   . . . .   
  │     . . . . . . . . . .   . . . . . . . . . . .   {__name__="errors_total", method="POST"}
  │           . . .   . . . . . . . . .   . . . . .   {__name__="errors_total", method="GET"}
  │         . . . . . . . . .       . . . . .
  │       . . .     . . . . . . . . . . . . . . . .                  ... 
  │     . . . . . . . . . . . . . . . .   . . . . 
  v
    <-------------------- time --------------------->
 
```

At the scale of collecting millions of data points per second, batching writes is a non-negotiable performance requirement. Thus, we want to write larger chunks of data in sequence.

#### Current solution

Time to take a look at how Prometheus’s current storage, let’s call it “V2”, we create one file per time series that contains all of its samples in sequential order. As appending single samples to all those files every few seconds is expensive, we batch up 1KiB chunks of samples for a series in memory and append those chunks to the individual files, once they are full.

Facebook’s paper on their Gorilla TSDB describes a similar chunk-based approach and introduces a compression format that reduces 16 byte samples to an average of 1.37 bytes. The V2 storage uses various compression formats including a variation of Gorilla’s.

While the chunk-based approach is great, keeping a separate file for each series is troubling the V2 storage for various reasons:

- Need a lot more files than the number of time series we currently collected
- Requires thousands of individual disk writes every second
- It’s infeasible to keep all files open for reads and writes
- Deletions are actually write intensive operations that often takes hours
- Chunks that are currently accumulating are only held in memory. If the application crashes, data will be lost

The key take away from the existing design is the concept of chunks, which we most certainly want to keep. The most recent chunks always being held in memory is also generally good. After all, the most recent data is queried the most by a large margin.
Having one file per time series is a concept we would like to find an alternative to.

### Series Churn

We use the term series churn to describe that a set of time series becomes inactive. i.e. receives no more data points, and a new set of active series appears instead.

```
series
  ^
  │   . . . . . .
  │   . . . . . .
  │   . . . . . .
  │               . . . . . . .
  │               . . . . . . .
  │               . . . . . . .
  │                             . . . . . .
  │                             . . . . . .
  │                                         . . . . .
  │                                         . . . . .
  │                                         . . . . .
  v
    <-------------------- time --------------------->
```

#### Current solution

The current V2 storage of Prometheus has an index based on LevelDB for all series that are currently stored.The current V2 storage of Prometheus has an index based on LevelDB for all series that are currently stored

### Resource consumption

The problem(resources consumption) is rather its relative unpredictability and instability in face of changes. By its architecture the V2 storage slowly builds up chunks of sample data, which causes the memory consumption to ramp up over time. As chunks get completed, they are written to disk and can be evicted from memory. Eventually, Prometheus’s memory usage reaches a steady state. That is until the monitored environment changes — series churn increases the usage of memory, CPU, and disk IO every time we scale an application or do a rolling update.

## Starting over

By now we have a good idea of our problem domain, how the V2 storage solves it, and where its design has issues. We also saw some great concepts that we want to adapt more or less seamlessly. To keep things fun, the auther decided to take a stab at writing an entire time series database — from scratch, i.e. writing bytes to the file system.

We have to find the right set of algorithms and disk layout for our data to implement a well-performing storage layer.

### V3-Macro design

```bash
$ tree ./data
./data
├── b-000001
│   ├── chunks
│   │   ├── 000001
│   │   ├── 000002
│   │   └── 000003
│   ├── index
│   └── meta.json
├── b-000004
│   ├── chunks
│   │   └── 000001
│   ├── index
│   └── meta.json
├── b-000005
│   ├── chunks
│   │   └── 000001
│   ├── index
│   └── meta.json
└── b-000006
    ├── meta.json
    └── wal
        ├── 000001
        ├── 000002
        └── 000003
```

Obviously, there is no longer a single file per series but instead a handful of files holds chunks for many of them. The “index” file contains a lot of black magic allowing us to find labels, their possible values, entire time series and the chunks holding their data points. But why are there several directories containing the layout of index and chunk files? And why does the last one contain a “wal” directory instead? Understanding those two questions, solves about 90% of our problems.

#### Many Little Databases

We partition horizontal dimension into non-overlapping blocks. Each block acts as a fully independent database containing all time series data for its time window. Hence, it has its own index and set of chunk files.

```
t0            t1             t2             t3             now
 ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐
 │           │  │           │  │           │  │           │                 ┌────────────┐
 │           │  │           │  │           │  │  mutable  │ <─── write ──── ┤ Prometheus │
 │           │  │           │  │           │  │           │                 └────────────┘
 └───────────┘  └───────────┘  └───────────┘  └───────────┘                        ^
       └──────────────┴───────┬──────┴──────────────┘                              │
                              │                                                  query
                              │                                                    │
                            merge ─────────────────────────────────────────────────┘
```

Every block of data is immutable, but must be able to add new series and samples to the most recent block as we collect new data. For this block, all new data is written to an in-memory database that provides the same lookup properties as our persistent blocks. The in-memory data structures can be updated efficiently. To prevent data loss, all incoming data is also written to a temporary write ahead log, which is the set of files in our “wal” directory, from which we can re-populate the in-memory database on restart.

This horizontal partitioning adds a few great capabilities:

- When querying a time range, we can easily ignore all data blocks outside of this range. 
- When completing a block, we can persist the data from our in-memory database by sequentially writing just a handful of larger files. We avoid any write-amplification and serve SSDs and HDDs equally well.
- We keep the good property of V2 that recent chunks, which are queried most, are always hot in memory.
- We can pick any size that makes the most sense for the individual data points and chosen compression format.
- Deleting old data becomes extremely cheap and instantaneous. 

A meta.json file holds human-readable information about the block to easily understand the state of our storage and the data it contains.

#### mmap

Moving from millions of small files to a handful of larger allows us to keep all files open with little overhead. This unblocks the usage of mmap, a system call that allows us to transparently back a virtual memory region by file contents(like swap space).

This means we can treat all contents of our database as if they were in memory without occupying any physical RAM. Only if we access certain byte ranges in our database files, the operating system lazily loads pages from disk. This puts the operating system in charge of all memory management related to our persisted data. Generally, it is more qualified to make such decisions, as it has the full view on the entire machine and all its processes. Queried data can be rather aggressively cached in memory, yet under memory pressure the pages will be evicted. If the machine has unused memory, Prometheus will now happily cache the entire database, yet will immediately return it once another application needs it.
Therefore, queries can longer easily OOM our process by querying more persisted data than fits into RAM. The memory cache size becomes fully adaptive and data is only loaded once the query actually needs it.

### Compaction

The storage has to periodically “cut” a new block and write the previous one, which is now completed, onto disk. Only after the block was successfully persisted, the wal files are deleted.
We are interested in keeping each block reasonably short to avoid accumulating too much data in memory. When querying multiple blocks, we have to merge their results into an overall result. 

To achieve both, we introduce compaction. Compaction describes the process of taking one or more blocks of data and writing them into one block.

```
t0             t1            t2             t3             t4             now
 ┌────────────┐  ┌──────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐
 │ 1          │  │ 2        │  │ 3         │  │ 4         │  │ 5 mutable │    before
 └────────────┘  └──────────┘  └───────────┘  └───────────┘  └───────────┘
 ┌─────────────────────────────────────────┐  ┌───────────┐  ┌───────────┐
 │ 1              compacted                │  │ 4         │  │ 5 mutable │    after (option A)
 └─────────────────────────────────────────┘  └───────────┘  └───────────┘
 ┌──────────────────────────┐  ┌──────────────────────────┐  ┌───────────┐
 │ 1       compacted        │  │ 3      compacted         │  │ 5 mutable │    after (option B)
 └──────────────────────────┘  └──────────────────────────┘  └───────────┘
```

In this example we have the sequential blocks [1, 2, 3, 4]. Blocks 1, 2, and 3 can be compacted together and the new layout is [1, 4]. Alternatively, compact them in pairs of two into [1, 3]. All time series data still exist but now in fewer blocks overall. This significantly reduces the merging cost at query time as fewer partial query results have to be merged.

### Retention

How can we drop old data in our block based design? Quite simply, by just deleting the directory of a block that has no data within our configured retention window. In the example below, block 1 can safely be deleted, whereas 2 has to stick around until it falls fully behind the boundary.

```                      |
 ┌────────────┐  ┌────┼─────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐
 │ 1          │  │ 2  |     │  │ 3         │  │ 4         │  │ 5         │   . . .
 └────────────┘  └────┼─────┘  └───────────┘  └───────────┘  └───────────┘
                      |
                      |
             retention boundary
```

An upper limit has to be applied so blocks don’t grow to span the entire database and thus diminish the original benefits of our design.

## The Index

An inverted index provides a fast lookup of data items based on a subset of their contents. Simply put, I can look up all series that have a label app=”nginx" without having to walk through every single series and check whether it contains that label.

For that, each series is assigned a unique ID by which it can be retrieved in constant time, i.e. O(1). In this case the ID is our forward index.

Example: If the series with IDs 10, 29, and 9 contain the label app="nginx", the inverted index for the label “nginx” is the simple list [10, 29, 9], which can be used to quickly retrieve all series containing the label. Even if there were 20 billion further series, it would not affect the speed of this lookup.

If n is our total number of series, and m is the result size for a given query, the complexity of our query using the index is now O(m). Queries scaling along the amount of data they retrieve (m) instead of the data body being searched (n) is a great property as m is generally significantly smaller.

For brevity, let’s assume we can retrieve the inverted index list itself in constant time.

The keen observer will have noticed, that in the worst case, a label exists in all series and thus m is, again, in O(n). This is expected and perfectly fine. If you query all data, it naturally takes longer. Things become problematic once we get involved with more complex queries.

### Combining Labels

To find all series satisfying both label selectors, we take the inverted index list for each and intersect them. The resulting set will typically be orders of magnitude smaller than each input list individually. As each input list has the worst case size O(n), the brute force solution of nested iteration over both lists, has a runtime of O(n^2). When adding further label selectors to the query, the exponent increases for each to O(n^3), O(n^4), O(n^5), … O(n^k). A lot of tricks can be played to minimize the effective runtime by changing the execution order. The more sophisticated, the more knowledge about the shape of the data and the relationships between labels is needed. This introduces a lot of complexity, yet does not decrease our algorithmic worst case runtime.

What happens if we assume that the IDs in our inverted indices are sorted?

Suppose this example of lists for our initial query:

```
__name__="requests_total"   ->   [ 9999, 1000, 1001, 2000000, 2000001, 2000002, 2000003 ]
     app="foo"              ->   [ 1, 3, 10, 11, 12, 100, 311, 320, 1000, 1001, 10002 ]

             intersection   =>   [ 1000, 1001 ]
```

We can find it by setting a cursor at the beginning of each list and always advancing the one at the smaller number. When both numbers are equal, we add the number to our result and advance both cursors. Overall, we scan both lists in this zig-zag pattern and thus have a total cost of O(2n) as we only ever move forward in either list. The procedure for more than two lists of different set operations works similarly. So the number of k set operations merely modifies the factor (O(k*n)) instead of the exponent (O(n^k)) of our worst-case lookup runtime. A great improvement.

What I described here is a simplified version of the canonical search index used by practically any full text search engine out there. Every series descriptor is treated as a short “document”, and every label (name + fixed value) as a “word” inside of it. We can ignore a lot of additional data typically encountered in search engine indices, such as word position and frequency data.

## BenchMarking

Our tool allows us to declaratively define a benchmarking scenario, which is then deployed to a Kubernetes cluster on AWS. While this is not the best environment for all-out benchmarking, it certainly reflects our user base better than dedicated bare metal servers with 64 cores and 128GB of memory.
We deploy two Prometheus 1.5.2 servers (V2 storage) and two Prometheus servers from the 2.0 development branch (V3 storage). Each Prometheus server runs on a dedicated machine with an SSD. A horizontally scaled application exposing typical microservice metrics is deployed to worker nodes. Additionally, the Kubernetes cluster itself and the nodes are being monitored. The whole setup is supervised by yet another Meta-Prometheus, monitoring each Prometheus server for health and performance.
To simulate series churn, the microservice is periodically scaled up and down to remove old pods and spawn new pods, exposing new series. Query load is simulated by a selection of “typical” queries, run against one server of each Prometheus version.

Overall the scaling and querying load as well as the sampling frequency significantly exceed today’s production deployments of Prometheus. For instance, we swap out 60% of our microservice instances every 15 minutes to produce series churn. This would likely only happen 1-5 times a day in a modern infrastructure. This ensures that our V3 design is capable of handling the workloads of the years ahead. As a result, the performance differences between Prometheus 1.5.2 and 2.0 are larger than in a more moderate environment.
In total, we are collecting about 110,000 samples per second from 850 targets exposing half a million series at a time.

After leaving this setup running for a while, we can take a look at the numbers. We evaluate several metrics over the first 12 hours within both versiones reached a steady state.

Obviously, the queried servers are consuming more memory, which can largely be attributed to overhead of the query engine, which will be subject to future optimizations.

## Conclusion

Well, it seems to work pretty well. Prometheus 2.0 or later releases with the new V3 storage is available.

It’s surprisingly agnostic to Prometheus itself and could be widely useful for a wider range of applications looking for an efficient local storage time series database.
