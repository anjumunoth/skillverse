**monitoring and performance tuning** is really about answering four questions:

1. **Is MongoDB CPU-bound, memory-bound, disk-bound, or network-bound?**
2. **Are clients waiting for connections or database operations?**
3. **Is WiredTiger cache healthy, or is eviction struggling?**
4. **Are slow operations caused by queries, indexes, locking/concurrency, or the underlying Linux host?**

Below is a practical explanation, with **MongoDB 8.x + Ubuntu/Linux** in mind.

---

# 1. MongoDB Performance Monitoring — Big Picture

A useful monitoring architecture looks like this:

```text
                    MongoDB Application
                           │
                           │
                    Connection Pool
                           │
                           ▼
                    ┌──────────────┐
                    │   mongod     │
                    └──────────────┘
                       │    │    │
          ┌────────────┘    │    └──────────────┐
          ▼                 ▼                   ▼
     Connections        WiredTiger          Operations
          │              Cache                 │
          │                 │                  │
          ▼                 ▼                  ▼
     serverStatus()     serverStatus()     profiler/logs
     mongostat          mongostat          mongotop
          │
          └────────────────────┐
                               ▼
                        Linux OS metrics
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
             CPU             Memory             Disk
             top             free              iostat
             vmstat          vmstat             df
                                                │
                                                ▼
                                             Network
                                             ss/netstat
```

The important point is:

> **MongoDB performance cannot be diagnosed only from MongoDB metrics.**

For example:

```text
MongoDB query is slow
        │
        ├── Bad query?
        ├── Missing index?
        ├── Too many connections?
        ├── WiredTiger cache pressure?
        ├── CPU saturation?
        ├── Disk I/O latency?
        └── Network problem?
```

You need to correlate MongoDB metrics with Linux metrics.

---

# 2. Key Metrics to Monitor

The most important categories are:

| Category          | Important metrics                  | What it tells you          |
| ----------------- | ---------------------------------- | -------------------------- |
| Connections       | current, available, created        | Client connection pressure |
| CPU               | CPU %, load average                | CPU saturation             |
| Memory            | RSS, virtual memory, available RAM | Memory pressure            |
| WiredTiger        | cache used, dirty bytes, eviction  | Storage engine health      |
| Disk              | read/write latency, IOPS           | Storage bottleneck         |
| Operations        | queries, inserts, updates, deletes | Workload                   |
| Queues            | read/write queues                  | Operations waiting         |
| Locks/concurrency | tickets, waits                     | Internal contention        |
| Network           | bytes in/out                       | Network workload           |
| Cursors           | open cursors                       | Query/application behavior |
| Replication       | lag, oplog window                  | Replica-set health         |

---

# 3. Connections

MongoDB maintains client connections between applications and `mongod`.

You should monitor:

```text
current connections
available connections
connections created
connections rejected
```

A basic command:

```javascript
db.serverStatus().connections
```

Example:

```javascript
{
    current: 245,
    available: 510,
    totalCreated: 128456
}
```

### Meaning

```text
current
   ↓
Currently open client connections

available
   ↓
Connections still available to MongoDB

totalCreated
   ↓
Total connections created since mongod started
```

### Important distinction

`totalCreated` is **not the number of currently active connections**.

For example:

```text
totalCreated = 1,000,000
current      = 500
```

doesn't mean one million connections currently exist.

It means MongoDB has created one million connections over its lifetime, while only 500 currently remain open.

---

# 4. Connection Pressure

Suppose:

```text
current = 950
available = 50
```

This deserves investigation.

Possible causes:

```text
Application
    │
    ├── Connection pool too large
    ├── Connection leak
    ├── Too many application instances
    └── Poor connection lifecycle management
```

MongoDB itself may be healthy, but the application could be creating excessive connections.

---

# 5. MongoDB `serverStatus()`

This is one of the most important DBA commands.

```javascript
db.serverStatus()
```

It returns a large amount of operational information.

For example:

```javascript
db.serverStatus().connections
```

```javascript
db.serverStatus().mem
```

```javascript
db.serverStatus().wiredTiger
```

```javascript
db.serverStatus().opcounters
```

```javascript
db.serverStatus().network
```

```javascript
db.serverStatus().globalLock
```

---

# 6. Useful `serverStatus()` Sections

A practical monitoring script might start with:

```javascript
db.serverStatus({
    connections: 1,
    mem: 1,
    network: 1,
    opcounters: 1,
    wiredTiger: 1
})
```

Depending on MongoDB version and command behavior, selecting fields can help reduce the amount of output you're looking at.

---

# 7. Operation Counters

Run:

```javascript
db.serverStatus().opcounters
```

Example:

```javascript
{
    insert: 12500,
    query: 875000,
    update: 35000,
    delete: 5000,
    getmore: 120000,
    command: 450000
}
```

This gives you a picture of workload.

For example:

```text
Queries       ████████████████████
Commands      ██████████
getMore       █████
Updates       ██
Inserts       █
Deletes       ▏
```

A very high query rate isn't automatically a problem.

The question is:

> **How much work is MongoDB doing for each operation?**

A million efficient indexed queries may be fine.

A few thousand collection scans can be disastrous.

---

# 8. `mongostat`

`mongostat` is one of the simplest ways to get a **live operational view**.

Run on Ubuntu:

```bash
mongostat
```

Or specify the server:

```bash
mongostat --host localhost:27017
```

You can refresh every 2 seconds:

```bash
mongostat --host localhost:27017 --rowcount 0 2
```

Conceptually, you might see:

```text
insert query update delete getmore command dirty used flushes vsize res
    10   850     20      2      50     400      5    60      0  3.2G 1.8G
```

### Important columns

| Column  | Meaning                   |
| ------- | ------------------------- |
| insert  | Inserts/sec               |
| query   | Queries/sec               |
| update  | Updates/sec               |
| delete  | Deletes/sec               |
| getmore | Cursor getMore operations |
| command | Commands/sec              |
| dirty   | Dirty cache percentage    |
| used    | WiredTiger cache usage    |
| flushes | Disk flush activity       |
| vsize   | Virtual memory            |
| res     | Resident memory           |

---

# 9. When `mongostat` Is Useful

Imagine users report:

> "The application suddenly became slow."

Run:

```bash
mongostat 2
```

You might discover:

```text
query    update    dirty    used
5000     100       85%      98%
```

That immediately suggests:

```text
Very high query workload
        +
WiredTiger cache almost full
        +
High dirty percentage
```

Then you investigate deeper.

---

# 10. `mongotop`

`mongotop` answers a different question:

> **Which collections are consuming the most read/write activity?**

Run:

```bash
mongotop
```

Or:

```bash
mongotop 2
```

Example:

```text
                     ns       total     read     write
sales.orders                   800ms     700ms    100ms
sales.customers                150ms     120ms     30ms
inventory.products               80ms      60ms     20ms
```

This can immediately identify a hot collection.

---

# 11. `mongostat` vs `mongotop`

| Tool             | Primary purpose           |
| ---------------- | ------------------------- |
| `mongostat`      | Overall MongoDB activity  |
| `mongotop`       | Collection-level activity |
| `serverStatus()` | Detailed internal metrics |

Think:

```text
mongostat
   ↓
"Is MongoDB busy?"

mongotop
   ↓
"Which collection is busy?"

serverStatus()
   ↓
"Why is MongoDB busy?"
```

---

# 12. Linux `top`

MongoDB performance analysis should always include Linux.

Run:

```bash
top
```

For a more convenient interface:

```bash
htop
```

Look for:

```text
CPU %
MEM %
load average
mongod process
```

Example:

```text
PID     USER      %CPU     %MEM
1234    mongodb   380%     18.5
```

On a multi-core machine, CPU can exceed 100%.

For example:

```text
100% = approximately one CPU core fully utilized

400% = approximately four cores fully utilized
```

---

# 13. CPU Diagnosis

Suppose:

```text
mongod CPU = 390%
load average = 8.5
```

and the machine has:

```text
8 CPUs
```

MongoDB could be heavily CPU-bound.

But don't immediately increase hardware.

First determine:

```text
CPU high
   │
   ├── Expensive queries?
   ├── Missing indexes?
   ├── Aggregation workload?
   ├── Large sorts?
   ├── Too many concurrent operations?
   └── Compression/encryption overhead?
```

---

# 14. Linux Memory

Use:

```bash
free -h
```

Example:

```text
               total    used    free
Mem:             32G     25G     2G
Swap:             4G      0G     4G
```

More important than simply looking at `free` is understanding:

```text
available
```

Linux deliberately uses unused memory for filesystem cache.

So:

```text
free memory = low
```

does **not automatically mean**

```text
memory shortage
```

---

# 15. `vmstat`

`vmstat` is extremely useful for identifying memory and CPU pressure.

Run:

```bash
vmstat 2
```

Example:

```text
r  b   swpd   free   buff   cache   si   so   us   sy   id   wa
8  2      0  2000M  500M    10G     0    0   70   10   15    5
```

Important columns:

| Column | Meaning            |
| ------ | ------------------ |
| `r`    | Runnable processes |
| `b`    | Processes blocked  |
| `si`   | Swap in            |
| `so`   | Swap out           |
| `us`   | User CPU           |
| `sy`   | System CPU         |
| `id`   | Idle CPU           |
| `wa`   | I/O wait           |

---

# 16. Why `wa` Matters

Suppose:

```text
us = 20
sy = 10
id = 10
wa = 60
```

That tells you:

> The CPUs are spending significant time waiting for I/O.

This strongly suggests investigating storage.

---

# 17. `iostat`

Install if necessary:

```bash
sudo apt install sysstat
```

Then:

```bash
iostat -xz 2
```

Important metrics include:

```text
%util
await
r/s
w/s
rkB/s
wkB/s
```

### `await`

Average I/O request wait time.

High `await` can indicate storage latency.

### `%util`

Indicates how busy the device is.

For example:

```text
Device      r/s    w/s    await    %util
nvme0n1     800    600     2ms       55%
```

This is very different from:

```text
nvme0n1     800    600    40ms       99%
```

The second situation deserves immediate investigation.

---

# 18. Disk Performance and MongoDB

MongoDB's WiredTiger storage engine performs significant disk activity.

The flow is approximately:

```text
MongoDB operation
       │
       ▼
WiredTiger cache
       │
       ├── clean pages
       │
       └── dirty pages
              │
              ▼
          checkpoint
              │
              ▼
          filesystem
              │
              ▼
            disk
```

If the disk is slow:

```text
WiredTiger
    ↓
eviction/checkpoint pressure
    ↓
I/O latency
    ↓
MongoDB operations slow down
```

---

# 19. Network Monitoring

Modern Linux systems usually use:

```bash
ss
```

rather than the older `netstat`.

For MongoDB:

```bash
ss -tnp | grep 27017
```

You can inspect listening ports:

```bash
ss -lntp | grep 27017
```

If `netstat` is installed:

```bash
netstat -an | grep 27017
```

You can investigate:

```text
Too many connections
TCP backlog
Connection establishment
Network traffic
```

---

# 20. Connection Pooling

This is one of the most important performance topics.

Applications normally should **not create a new MongoDB connection for every query**.

Bad design:

```text
Request
   ↓
Create MongoDB connection
   ↓
Execute query
   ↓
Close connection
```

Imagine:

```text
10,000 requests/sec
        ↓
10,000 connections/sec
```

That's extremely inefficient.

Instead:

```text
Application
     │
     ▼
Connection Pool
 ┌───┬───┬───┬───┬───┐
 │ C │ C │ C │ C │ C │
 └───┴───┴───┴───┴───┘
     │
     ▼
   MongoDB
```

Connections are reused.

---

# 21. Connection Pool Size

Suppose:

```text
20 application instances
```

and each instance has:

```text
maxPoolSize = 100
```

Potential maximum:

```text
20 × 100
     =
2000 connections
```

This is an extremely important calculation.

Don't look only at the pool size of one application instance.

Calculate:

```text
Total potential MongoDB connections
=
Number of application instances
×
maxPoolSize
```

---

# 22. Connection Pool Tuning

Typical parameters in MongoDB drivers include:

```text
maxPoolSize
minPoolSize
maxIdleTimeMS
waitQueueTimeoutMS
connectTimeoutMS
serverSelectionTimeoutMS
```

Exact configuration syntax depends on the driver.

For example, conceptually:

```text
maxPoolSize = 100
minPoolSize = 10
maxIdleTimeMS = ...
waitQueueTimeoutMS = ...
```

### Don't simply increase `maxPoolSize`

A common mistake is:

> "The application is slow. Increase the connection pool."

That can make things worse.

```text
More connections
       ↓
More concurrent operations
       ↓
More CPU contention
       ↓
More cache pressure
       ↓
More disk pressure
       ↓
Potentially worse performance
```

---

# 23. Connection Pool Queueing

Imagine:

```text
Pool size = 50

100 application requests arrive
```

Only 50 can immediately obtain connections.

The others wait.

```text
Requests
   │
   ├── 50 → MongoDB
   │
   └── 50 → waiting
```

If the wait becomes excessive:

```text
Application latency ↑
```

The correct solution may be:

```text
Optimize query
```

rather than:

```text
Increase pool to 500
```

---

# 24. WiredTiger Cache

MongoDB's WiredTiger storage engine maintains an in-memory cache.

Conceptually:

```text
                 RAM
        ┌───────────────────┐
        │   WiredTiger      │
        │      Cache        │
        │                   │
        │ Hot data          │
        │ Index pages       │
        │ Dirty pages       │
        └───────────────────┘
                 │
                 ▼
              Disk
```

The cache exists because accessing RAM is much faster than repeatedly reading from disk.

---

# 25. Why Cache Matters

Suppose an application repeatedly accesses:

```text
Customer collection
Order collection
Product indexes
```

If frequently used pages remain in memory:

```text
Query
  ↓
Cache hit
  ↓
Fast
```

If pages aren't available:

```text
Query
  ↓
Cache miss
  ↓
Disk read
  ↓
Higher latency
```

Therefore:

> A healthy cache is one of the foundations of MongoDB performance.

---

# 26. Checking WiredTiger Cache

You can inspect:

```javascript
db.serverStatus().wiredTiger.cache
```

There are many fields.

Important ones include concepts such as:

```text
bytes currently in the cache
maximum bytes configured
bytes belonging to dirty pages
pages read into cache
pages written from cache
eviction activity
```

For example:

```javascript
db.serverStatus().wiredTiger.cache
```

You might see fields such as:

```text
bytes currently in the cache
maximum bytes configured
tracked dirty bytes in the cache
pages read into cache
pages written from cache
```

---

# 27. Cache Utilization

Conceptually:

```text
Cache utilization =
current cache usage
--------------------
maximum cache size
```

For example:

```text
Current = 12 GB
Maximum = 16 GB

Utilization = 75%
```

A high cache utilization by itself isn't necessarily bad.

MongoDB is designed to **use memory aggressively**.

The important question is:

> Is the cache under pressure?

---

# 28. Cache Eviction

WiredTiger cannot allow the cache to grow indefinitely.

Therefore:

```text
Cache becomes full
       │
       ▼
Eviction starts
       │
       ▼
Less recently needed pages removed
       │
       ▼
Space becomes available
```

Conceptually:

```text
             Cache
┌───────────────────────────┐
│ Hot │ Hot │ Dirty │ Clean │
│ Hot │ Hot │ Dirty │ Clean │
└───────────────────────────┘
              │
              ▼
          Eviction
              │
              ▼
      Clean pages removed
```

Dirty pages are more complicated because they may need to be written to disk first.

---

# 29. Dirty Pages

A page becomes dirty when data in memory differs from its persisted version.

For example:

```text
Disk:
Age = 35

Update:
Age = 36

             ↓

Memory:
Age = 36
Disk:
Age = 35
```

The page is now dirty.

Eventually:

```text
Dirty page
     ↓
Written to disk
     ↓
Page becomes clean
```

---

# 30. Checkpoints and Cache

WiredTiger periodically performs checkpoints.

Conceptually:

```text
Application writes
       ↓
WiredTiger cache
       ↓
Dirty pages
       ↓
Checkpoint
       ↓
Persistent storage
```

If checkpointing and eviction cannot keep up with the workload, performance can degrade.

---

# 31. Memory Sizing

A common mistake is:

> "The server has 64 GB RAM, so give MongoDB 60 GB cache."

You shouldn't think about MongoDB memory in isolation.

The operating system also needs memory.

For example:

```text
64 GB physical RAM
│
├── MongoDB / WiredTiger
├── OS
├── filesystem cache
├── monitoring
├── backup tools
├── other services
└── safety margin
```

---

# 32. WiredTiger Cache Configuration

MongoDB allows explicit WiredTiger cache configuration.

In `mongod.conf`, conceptually:

```yaml
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 8
```

Then MongoDB can be started with the configuration.

However:

> **Do not blindly configure this value.**

Modern MongoDB versions have automatic cache sizing intended for typical deployments.

Manual configuration is useful when you have a specific reason, such as:

```text
Shared server
Multiple mongod processes
Other memory-intensive applications
Container memory limits
Special workload requirements
```

---

# 33. Example: Shared Linux Server

Suppose:

```text
RAM = 32 GB
```

And the server runs:

```text
mongod
application
monitoring agent
backup process
```

Giving MongoDB almost all RAM could cause:

```text
OS memory pressure
      ↓
filesystem cache pressure
      ↓
swap risk
      ↓
I/O latency
      ↓
MongoDB slowdown
```

The goal is not:

> **Maximum MongoDB memory**

The goal is:

> **Maximum sustainable overall system performance.**

---

# 34. Swap

For MongoDB servers, you generally want to avoid active swapping.

Check:

```bash
free -h
```

and:

```bash
vmstat 2
```

Look at:

```text
si
so
```

If you observe sustained:

```text
si > 0
so > 0
```

investigate memory pressure.

---

# 35. Putting Everything Together

Imagine the application reports:

> "MongoDB became slow."

You could troubleshoot like this:

### Step 1 — MongoDB workload

```bash
mongostat 2
```

Check:

```text
queries
updates
inserts
commands
dirty
used
```

---

### Step 2 — Hot collections

```bash
mongotop 2
```

Identify which namespace is busy.

---

### Step 3 — MongoDB internal metrics

```javascript
db.serverStatus().connections
```

```javascript
db.serverStatus().wiredTiger.cache
```

```javascript
db.serverStatus().opcounters
```

---

### Step 4 — CPU

```bash
top
```

or:

```bash
htop
```

---

### Step 5 — Memory

```bash
free -h
```

and:

```bash
vmstat 2
```

---

### Step 6 — Disk

```bash
iostat -xz 2
```

---

### Step 7 — Network

```bash
ss -tnp | grep 27017
```

---

# 36. A Practical Diagnostic Decision Tree

```text
                 MongoDB is slow
                       │
                       ▼
                  mongostat
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
      CPU?          Memory?         Disk?
        │              │              │
        ▼              ▼              ▼
       top          free/vmstat     iostat
        │              │              │
        └──────────────┼──────────────┘
                       ▼
                MongoDB metrics
                       │
             ┌─────────┼─────────┐
             ▼         ▼         ▼
        Connections   Cache    Operations
             │         │         │
             ▼         ▼         ▼
          Pooling   Eviction   Queries
                                │
                                ▼
                            Indexes
```

This is a much better approach than randomly changing MongoDB parameters.

---

# 37. Example Performance Scenario

Suppose we observe:

```text
mongostat:

query       = 8000/s
dirty       = 92%
used        = 98%
```

Linux:

```text
CPU = 40%
wa  = 35%
```

Disk:

```text
await = 50 ms
util  = 99%
```

Connections:

```text
current = 1800
```

Now we can form a hypothesis:

```text
High workload
      +
Cache pressure
      +
Heavy disk I/O
      +
Large number of concurrent connections
```

The answer is **not automatically**:

```text
Increase WiredTiger cache
```

because if disk is already saturated, simply increasing cache may not solve the fundamental problem.

We would investigate:

```text
1. Slow queries
2. Missing/inefficient indexes
3. Excessive concurrency
4. Connection pool sizing
5. Disk performance
6. Working-set size
7. Eviction/checkpoint behavior
```

---

# 38. Metrics You Should Build Into a Monitoring Dashboard

For a production MongoDB deployment, I would group the dashboard into:

### MongoDB Overview

```text
Operations/sec
Connections
Network traffic
Query latency
Errors
```

### Memory

```text
Resident memory
WiredTiger cache usage
Dirty bytes
Eviction activity
Swap
```

### CPU

```text
CPU utilization
Load average
System CPU
User CPU
I/O wait
```

### Disk

```text
Read IOPS
Write IOPS
Read latency
Write latency
Disk utilization
Filesystem usage
```

### Connections

```text
Current
Available
Created
Pool utilization
Wait queue
```

### Replication

```text
Replication lag
Oplog window
Primary/secondary state
Election events
```

---

# 39. Most Important DBA Principle

When tuning MongoDB, avoid this approach:

```text
Problem
  ↓
Change configuration
  ↓
Hope it improves
```

Instead:

```text
Measure
   ↓
Identify bottleneck
   ↓
Form hypothesis
   ↓
Change ONE thing
   ↓
Measure again
   ↓
Compare
```

For example:

```text
Slow query
   ↓
Explain()
   ↓
COLLSCAN discovered
   ↓
Create appropriate index
   ↓
Explain() again
   ↓
Compare executionStats
```

That's genuine performance tuning.

---

# 40. Quick Command Cheat Sheet

| Purpose                  | Ubuntu / MongoDB command             |
| ------------------------ | ------------------------------------ |
| MongoDB live statistics  | `mongostat 2`                        |
| Collection activity      | `mongotop 2`                         |
| MongoDB internal metrics | `db.serverStatus()`                  |
| Connections              | `db.serverStatus().connections`      |
| Operations               | `db.serverStatus().opcounters`       |
| WiredTiger cache         | `db.serverStatus().wiredTiger.cache` |
| CPU/processes            | `top` / `htop`                       |
| Memory                   | `free -h`                            |
| CPU/memory/I/O           | `vmstat 2`                           |
| Disk performance         | `iostat -xz 2`                       |
| MongoDB TCP connections  | `ss -tnp \| grep 27017`              |
| Listening port           | `ss -lntp \| grep 27017`             |
| Filesystem capacity      | `df -h`                              |

### The mental model to remember

```text
                 APPLICATION
                      │
               Connection Pool
                      │
                      ▼
                 ┌─────────┐
                 │ MongoDB │
                 └─────────┘
                  │   │   │
          ┌───────┘   │   └────────┐
          ▼           ▼            ▼
     Connections   WiredTiger   Operations
                      Cache
                        │
                        ▼
                      Disk
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
         CPU          Memory        Network
          │             │             │
          ▼             ▼             ▼
         top         vmstat/free      ss
                         │
                         ▼
                      iostat
```

**The key DBA skill is correlating these layers.** A MongoDB metric by itself rarely tells the complete story. For example, high cache usage may be perfectly normal, while high cache usage **combined with eviction pressure + disk latency + application latency** is a meaningful performance signal.
