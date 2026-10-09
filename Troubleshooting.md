Best way to understand troubleshooting is to correlate:

**MongoDB symptom → MongoDB metrics/logs → Linux OS metrics → root cause → corrective action**


---

# MongoDB Troubleshooting on Linux

## 1. The DBA Troubleshooting Mindset

When a MongoDB server becomes slow or unavailable, don't immediately assume MongoDB is the problem.

A useful troubleshooting flow is:

```text
                    MongoDB Problem
                          │
                          ▼
                 What is the symptom?
                          │
          ┌───────────────┼────────────────┐
          ▼               ▼                ▼
      Slow queries    Connection error   Server down
          │               │                │
          ▼               ▼                ▼
     MongoDB stats    Connections       mongod logs
          │               │                │
          └───────────────┼────────────────┘
                          ▼
                    Linux OS metrics
                          │
          ┌───────────────┼─────────────────┐
          ▼               ▼                 ▼
         CPU             RAM               Disk
          │               │                 │
          └───────────────┼─────────────────┘
                          ▼
                    Root Cause
                          │
                          ▼
                     Resolution
```

The important principle is:

> **Always correlate MongoDB evidence with Linux evidence.**

For example:

```text
MongoDB queries are slow
        ↓
serverStatus() shows high ticket utilization
        ↓
iostat shows high disk utilization
        ↓
mongod logs show checkpoint/eviction pressure
        ↓
WiredTiger cache is under pressure
        ↓
Disk I/O is the underlying bottleneck
```

---

# 2. Diagnosing High CPU on Linux

High CPU can originate from:

* MongoDB query workload
* inefficient queries
* missing indexes
* aggregation operations
* sorting
* replication
* compression/decompression
* excessive connections
* WiredTiger background activity
* another Linux process

---

## 2.1 Check CPU Usage

Start with:

```bash
top
```

or:

```bash
htop
```

Example:

```text
top - 23:45:10 up 5 days
Tasks: 220 total
%Cpu(s): 92.0 us, 4.0 sy, 0.0 id
```

Important fields:

| Field | Meaning           |
| ----- | ----------------- |
| us    | User-space CPU    |
| sy    | Kernel/system CPU |
| id    | Idle CPU          |
| wa    | I/O wait          |
| st    | Steal time        |

For MongoDB:

```text
PID     USER      %CPU
1234    mongodb   350
```

If the machine has 4 CPU cores, MongoDB using approximately 350% means it is using about 3.5 cores.

---

## 2.2 Identify the Process

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu | head
```

You might see:

```text
PID    CMD          %CPU
2451   mongod       380
3122   java         25
```

This tells us that MongoDB is actually consuming the CPU.

---

# 3. Find Which MongoDB Activity Is Causing CPU

Linux tells us **who is consuming CPU**.

MongoDB needs to tell us **why**.

Run:

```javascript
db.serverStatus()
```

Useful sections include:

```javascript
db.serverStatus().opcounters
db.serverStatus().connections
db.serverStatus().wiredTiger
```

For active operations:

```javascript
db.currentOp()
```

For example:

```javascript
db.currentOp({
    active: true
})
```

Look for:

* long-running operations
* aggregation
* collection scans
* large sorts
* blocked operations

---

## 3.1 Check Slow Queries

Enable the profiler carefully in production.

For example:

```javascript
db.setProfilingLevel(1, {
    slowms: 100
})
```

Then:

```javascript
db.system.profile.find().sort({
    ts: -1
}).limit(10)
```

A query such as:

```javascript
db.orders.find({
    customerId: 1001
})
```

may become expensive if `customerId` isn't indexed.

Check:

```javascript
db.orders.find({
    customerId: 1001
}).explain("executionStats")
```

Look at:

```text
totalDocsExamined
totalKeysExamined
executionTimeMillis
nReturned
```

A classic warning sign:

```text
nReturned: 10
totalDocsExamined: 5,000,000
```

This can produce significant CPU and I/O load.

---

# 4. Diagnosing High Memory Usage

Linux:

```bash
free -h
```

Example:

```text
              total   used   free
Mem:           32G    29G    1G
Swap:           2G     1G    1G
```

Don't automatically conclude:

> "MongoDB is consuming too much memory."

Linux uses memory aggressively for:

* filesystem cache
* MongoDB
* applications
* kernel structures

MongoDB's WiredTiger cache is only one part of memory usage.

---

## 4.1 Check MongoDB WiredTiger Cache

```javascript
db.serverStatus().wiredTiger.cache
```

Important fields include:

```text
bytes currently in the cache
maximum bytes configured
bytes dirty
tracked dirty bytes
pages evicted
```

Conceptually:

```text
RAM
│
├── MongoDB process
│   ├── WiredTiger cache
│   ├── connections
│   ├── query execution
│   └── other allocations
│
├── OS filesystem cache
│
└── Other Linux processes
```

---

# 5. Memory Pressure and Swap

Check:

```bash
free -h
```

and:

```bash
vmstat 1
```

Example:

```text
procs --------memory------- ---swap-- -----io----
 r  b   swpd   free   buff  cache   si   so
 5  2   512M   500M   1G    10G     20   40
```

Important:

```text
si = swap in
so = swap out
```

If `si` and `so` continuously increase, the system may be under memory pressure.

Also check:

```bash
swapon --show
```

---

# 6. Diagnosing High Disk I/O

This is extremely important for MongoDB.

Use:

```bash
iostat -xz 1
```

If not installed:

```bash
sudo apt install sysstat
```

Example:

```text
Device   r/s   w/s   rkB/s   wkB/s   %util
nvme0n1  500   900   8000    20000    98
```

The key field is:

```text
%util
```

If it remains close to:

```text
90–100%
```

the device may be saturated.

---

## 6.1 Check I/O Wait

In:

```bash
top
```

look at:

```text
wa
```

Example:

```text
%Cpu(s): 10 us, 5 sy, 2 id, 83 wa
```

This is a major clue.

The CPU isn't necessarily doing computation.

It is **waiting for storage I/O**.

---

# 7. MongoDB + Disk I/O Relationship

A typical chain could be:

```text
Heavy writes
      ↓
WiredTiger dirty pages
      ↓
Eviction/checkpoint activity
      ↓
Storage writes
      ↓
Disk saturation
      ↓
High I/O wait
      ↓
MongoDB latency increases
```

This is why looking only at CPU can be misleading.

---

# 8. Analyzing MongoDB Logs

On Ubuntu, the log location depends on the installation/configuration.

Common location:

```bash
/var/log/mongodb/mongod.log
```

Check:

```bash
sudo tail -100 /var/log/mongodb/mongod.log
```

Live monitoring:

```bash
sudo tail -f /var/log/mongodb/mongod.log
```

Search errors:

```bash
grep -i "error" /var/log/mongodb/mongod.log
```

Search warnings:

```bash
grep -i "warning" /var/log/mongodb/mongod.log
```

Search replication problems:

```bash
grep -Ei "replica|election|heartbeat|rollback" \
/var/log/mongodb/mongod.log
```

Search WiredTiger:

```bash
grep -i "wiredtiger" /var/log/mongodb/mongod.log
```

---

# 9. MongoDB Log Structure

Modern MongoDB logs are commonly structured JSON.

Conceptually:

```json
{
  "t": {"$date": "..."},
  "s": "I",
  "c": "NETWORK",
  "id": 12345,
  "ctx": "listener",
  "msg": "Listening on port"
}
```

Important fields:

| Field | Meaning        |
| ----- | -------------- |
| `t`   | Timestamp      |
| `s`   | Severity       |
| `c`   | Component      |
| `id`  | Message ID     |
| `ctx` | Context/thread |
| `msg` | Message        |

Severity commonly includes:

```text
F = Fatal
E = Error
W = Warning
I = Informational
D = Debug
```

---

# 10. Log Rotation

MongoDB logs should not be allowed to grow indefinitely.

Check configuration:

```bash
sudo cat /etc/mongod.conf
```

You may see:

```yaml
systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true
```

MongoDB supports log rotation.

For example:

```javascript
db.adminCommand({
    logRotate: 1
})
```

Depending on deployment and configuration, Linux `logrotate` can also be used.

Check:

```bash
ls /etc/logrotate.d/
```

Potential MongoDB configuration:

```bash
cat /etc/logrotate.d/mongodb
```

---

# 11. Why Log Growth Is Dangerous

Suppose:

```text
mongod.log = 40 GB
```

and the filesystem has only:

```text
50 GB free
```

Eventually:

```text
Disk full
   ↓
MongoDB cannot write
   ↓
Journal/checkpoint/log writes fail
   ↓
MongoDB becomes unstable
```

Therefore:

> **Disk-space monitoring is a MongoDB availability concern, not merely an OS housekeeping task.**

---

# 12. Connection Problems

Common symptoms:

```text
MongoServerSelectionError
Connection refused
Connection timed out
Too many connections
Server selection timeout
```

First determine whether the problem is:

```text
Network connectivity
       OR
MongoDB listener
       OR
Connection limit
       OR
Authentication
       OR
TLS
```

---

# 13. Check Whether MongoDB Is Listening

Ubuntu:

```bash
sudo ss -lntp | grep 27017
```

Example:

```text
LISTEN 0 4096 0.0.0.0:27017
```

Check MongoDB:

```bash
sudo systemctl status mongod
```

Check connectivity:

```bash
nc -vz localhost 27017
```

---

# 14. Connection Timeout

Suppose an application reports:

```text
Connection timed out
```

Check:

```bash
ping <server>
```

Then:

```bash
nc -vz <server> 27017
```

If:

```text
Connection timed out
```

investigate:

* firewall
* security groups
* routing
* network ACLs
* bindIp
* TLS
* MongoDB availability

Check:

```bash
sudo ufw status
```

---

# 15. Connection Refused

This is different.

```text
Connection refused
```

usually means the network path reached the host but nothing is accepting the connection on that port.

Check:

```bash
sudo ss -lntp | grep 27017
```

and:

```bash
sudo systemctl status mongod
```

---

# 16. MongoDB Connection Limits

Check:

```javascript
db.serverStatus().connections
```

Example:

```json
{
  "current": 4900,
  "available": 100
}
```

If:

```text
current → max
```

you may be approaching the connection limit.

Also inspect:

```javascript
db.serverStatus().connections
```

for:

```text
current
available
totalCreated
```

---

# 17. Why Too Many Connections Happens

Common causes:

```text
Application
   ↓
Connection created per request
   ↓
Connections not reused
   ↓
Connection pool grows
   ↓
MongoDB connection count increases
```

Correct architecture:

```text
Application
     │
     ▼
Connection Pool
 ┌───┼────┬────┐
 ▼   ▼    ▼    ▼
Conn Conn Conn Conn
        │
        ▼
     MongoDB
```

Connection pooling is critical.

Don't solve connection exhaustion simply by increasing the MongoDB connection limit.

First identify **why connections are accumulating**.

---

# 18. WiredTiger Lock Contention

MongoDB uses WiredTiger's concurrency mechanisms rather than relying on a single global database lock.

When many operations compete for resources, you can observe contention.

Check:

```javascript
db.serverStatus().locks
```

and:

```javascript
db.serverStatus().wiredTiger.concurrentTransactions
```

---

# 19. WiredTiger Concurrency Tickets

This is particularly important in MongoDB troubleshooting.

WiredTiger uses **read and write tickets** to control how many operations can concurrently enter certain storage-engine execution paths.

Conceptually:

```text
                 MongoDB Operations
                       │
              ┌────────┴────────┐
              ▼                 ▼
         Read operations   Write operations
              │                 │
              ▼                 ▼
        Read tickets       Write tickets
              │                 │
              └────────┬────────┘
                       ▼
                  WiredTiger
```

Check:

```javascript
db.serverStatus().wiredTiger.concurrentTransactions
```

You may see:

```json
{
  "read": {
    "out": 128,
    "available": 0,
    "totalTickets": 128,
    "tickets": 128
  },
  "write": {
    "out": 128,
    "available": 0,
    "totalTickets": 128,
    "tickets": 128
  }
}
```

The exact structure/values depend on the MongoDB version and configuration.

---

# 20. Ticket Exhaustion

Suppose:

```text
available read tickets = 0
```

and many operations are waiting.

That can indicate:

```text
Ticket exhaustion
       ↓
Operations waiting
       ↓
Latency increases
       ↓
Application sees slow queries/timeouts
```

But don't immediately conclude:

> "Increase tickets."

You need to determine **why tickets are being held**.

Potential causes include:

* slow storage
* long-running operations
* cache pressure
* heavy workload
* inefficient queries
* disk contention

A useful diagnostic relationship is:

```text
Ticket exhaustion
       +
High disk latency
       +
Long-running operations
       =
Potential storage/workload bottleneck
```

---

# 21. Disk Space Problems

Check:

```bash
df -h
```

Example:

```text
Filesystem      Size  Used Avail Use%
/dev/nvme0n1    500G  490G   10G  98%
```

Then:

```bash
df -i
```

This checks **inodes**.

You can have:

```text
Disk space available = YES
Inodes available = NO
```

and still be unable to create files.

---

# 22. Find What Is Consuming Disk

Use:

```bash
sudo du -xh /var | sort -h | tail -20
```

MongoDB-specific:

```bash
sudo du -sh /var/lib/mongodb
sudo du -sh /var/log/mongodb
```

Check journal:

```bash
sudo du -sh /var/lib/mongodb/journal
```

---

# 23. MongoDB Journal Growth

WiredTiger maintains a journal for durability.

Conceptually:

```text
Application
    ↓
MongoDB
    ↓
WiredTiger
    ├── Journal
    ├── Data files
    └── Checkpoints
```

Journal files should normally be managed automatically.

If you see unexpected persistent growth, investigate:

* checkpoint problems
* slow storage
* filesystem problems
* MongoDB process state
* long-running write workload
* abnormal shutdown/recovery behavior

**Do not manually delete WiredTiger journal files while MongoDB is running.**

---

# 24. Replica Set Troubleshooting

Suppose:

```text
PRIMARY
   │
   ├──── SECONDARY 1
   │
   └──── SECONDARY 2
```

and a secondary becomes unhealthy.

Start with:

```javascript
rs.status()
```

Also:

```javascript
rs.conf()
```

and:

```javascript
rs.printSecondaryReplicationInfo()
```

Depending on the troubleshooting requirement, also inspect:

```javascript
rs.printReplicationInfo()
```

---

# 25. Replica Set Log Investigation

Search:

```bash
grep -Ei \
"election|heartbeat|rollback|sync|replication|PRIMARY|SECONDARY" \
/var/log/mongodb/mongod.log
```

Typical events to look for:

```text
Heartbeat failed
Connection closed
Election started
Election succeeded
Rollback
Initial sync
Oplog
Replication lag
```

---

# 26. Election Failure Example

Suppose:

```text
PRIMARY
   X
SECONDARY
```

Logs might show:

```text
Heartbeat failed
```

Then:

```text
Election initiated
```

Then:

```text
Candidate
   ↓
Request votes
   ↓
Receive majority
   ↓
Become PRIMARY
```

If elections repeatedly occur:

```text
PRIMARY
 ↓
Election
 ↓
PRIMARY
 ↓
Election
 ↓
PRIMARY
```

investigate:

* network instability
* CPU starvation
* disk latency
* process pauses
* overloaded server
* clock/configuration problems
* connectivity between members

Repeated elections are often a **symptom**, not the root cause.

---

# 27. Replication Lag

Check:

```javascript
rs.printSecondaryReplicationInfo()
```

You might find:

```text
source: primary
syncedTo: ...
0 secs
```

versus:

```text
syncedTo: ...
3600 secs
```

One hour of lag is significant.

Investigate:

```text
Secondary
   ↓
Can it read oplog fast enough?
   ↓
Can it apply operations fast enough?
   ↓
Is disk I/O saturated?
   ↓
Is CPU saturated?
```

---

# 28. Sharding Troubleshooting

A sharded cluster looks like:

```text
                  mongos
                    │
          ┌─────────┼─────────┐
          ▼         ▼         ▼
       Shard 1   Shard 2   Shard 3
          │         │         │
       Replica   Replica   Replica
        Set       Set       Set
                    │
                    ▼
              Config Servers
```

Important components to troubleshoot:

* `mongos`
* shard replica sets
* config server replica set
* balancer
* chunk migrations
* network connectivity

---

# 29. Sharding Failure Investigation

Check:

```javascript
sh.status()
```

Look for:

```text
shards
databases
collections
chunks
balancer
```

Then inspect logs.

For example:

```bash
grep -Ei \
"migration|balancer|chunk|shard|config" \
/var/log/mongodb/mongos.log
```

Potential problems:

```text
Chunk migration failed
        ↓
Network problem
        OR
Disk problem
        OR
Shard unavailable
        OR
Lock/resource contention
```

---

# 30. Corrupted MongoDB Data Files

This is one of the most serious situations.

Symptoms may include:

```text
WiredTiger error
WT_PANIC
corruption detected
checksum error
unable to open file
Fatal assertion
```

First principle:

> **Do not immediately run `mongod --repair`.**

First determine:

**Is this a standalone server or a replica-set member?**

---

# 31. If the Server Is a Replica Set Member

This is usually much safer.

Suppose:

```text
PRIMARY
SECONDARY A ← corrupted
SECONDARY B
```

If the corrupted node can be rebuilt from another healthy replica member, a common strategy is:

```text
Stop corrupted member
        ↓
Remove/rebuild data
        ↓
Start MongoDB
        ↓
Initial sync
        ↓
Member rejoins replica set
```

This is generally preferable to attempting to repair the data files in place.

---

# 32. Why Replica Sets Help With Corruption

Replication provides another copy of the data.

```text
             PRIMARY
                │
       ┌────────┴────────┐
       ▼                 ▼
   Secondary A       Secondary B
      GOOD               GOOD
```

If:

```text
Secondary C → corrupted
```

you can potentially rebuild it from the healthy members.

This is one of the major operational advantages of replica sets.

---

# 33. `mongod --repair`

Repair should generally be treated as a **last-resort recovery mechanism**, particularly when no healthy copy or backup exists.

Conceptually:

```bash
mongod --dbpath /path/to/db --repair
```

The exact command depends on the installation and deployment.

Potential concerns:

* repair can be time-consuming
* repair may require substantial disk space
* damaged data may be lost
* repair does not replace a proper backup strategy

The preferred hierarchy is generally:

```text
Healthy replica member
        ↓
Backup
        ↓
Restore
        ↓
Repair as last resort
```

---

# 34. Never Do This

Do **not** randomly delete:

```text
WiredTiger.wt
WiredTigerLog.*
collection-*.wt
index-*.wt
```

or other MongoDB database files.

Also don't manually delete journal files to "free disk space."

You can make a recoverable situation unrecoverable.

---

# 35. `lsof` — Open Files

`lsof` means:

> **List Open Files**

MongoDB is heavily file-oriented.

Check files opened by MongoDB:

```bash
sudo lsof -p $(pidof mongod)
```

Find deleted files still held open:

```bash
sudo lsof | grep deleted
```

This is extremely useful for disk-space problems.

---

# 36. Deleted File but Disk Still Full

Suppose:

```bash
rm huge.log
```

but:

```bash
df -h
```

still says the disk is full.

Why?

Because a process may still have the file open.

```text
File
 │
 ├── Directory entry deleted
 │
 └── mongod still has file descriptor
              │
              ▼
        Disk blocks remain allocated
```

Find it:

```bash
sudo lsof | grep deleted
```

You might see:

```text
mongod  1234 mongodb  10w REG ... /var/log/mongodb/mongod.log (deleted)
```

The space is reclaimed only after the process closes the file descriptor.

---

# 37. `dmesg` — Kernel-Level Diagnosis

`dmesg` displays kernel messages.

Run:

```bash
sudo dmesg -T | tail -100
```

Look for:

```text
I/O errors
disk failures
filesystem errors
OOM killer
memory errors
network problems
```

Search specifically:

```bash
sudo dmesg -T | grep -Ei "error|fail|oom|nvme|ext4|xfs"
```

---

# 38. OOM Killer

One particularly important Linux event is:

```text
Out Of Memory
```

Linux may kill a process:

```text
Kernel
  ↓
Memory exhausted
  ↓
OOM killer
  ↓
mongod killed
```

Look for:

```bash
sudo dmesg -T | grep -i "oom"
```

or:

```bash
sudo dmesg -T | grep -i "killed process"
```

If you see:

```text
Killed process 1234 (mongod)
```

MongoDB may have disappeared because Linux killed it.

That's very different from MongoDB crashing itself.

---

# 39. `strace` — System Call Diagnosis

`strace` traces Linux system calls.

It can show MongoDB doing things such as:

```text
open()
read()
write()
fsync()
poll()
epoll_wait()
connect()
accept()
```

For a running process:

```bash
sudo strace -p <mongod-pid>
```

For example:

```bash
sudo strace -p 1234
```

You might observe:

```text
fsync(...)
write(...)
poll(...)
```

---

# 40. When Is `strace` Useful?

Use it when the problem is difficult to explain from MongoDB metrics alone.

For example:

```text
MongoDB appears stuck
       ↓
serverStatus doesn't explain it
       ↓
strace
       ↓
大量 fsync() / I/O calls
       ↓
Investigate storage
```

Or:

```text
mongod unable to connect somewhere
       ↓
strace
       ↓
connect() → ECONNREFUSED
       ↓
Investigate network/service
```

**Caution:** tracing a busy production MongoDB process can add overhead. Use it carefully and preferably during controlled troubleshooting.

---

# 41. A Very Useful Linux Diagnostic Toolkit

For MongoDB DBAs on Ubuntu:

| Tool         | Primary purpose                |
| ------------ | ------------------------------ |
| `top`        | CPU/memory                     |
| `htop`       | Interactive process monitoring |
| `free`       | Memory                         |
| `vmstat`     | CPU/memory/I/O                 |
| `iostat`     | Disk I/O                       |
| `df`         | Disk capacity                  |
| `du`         | Disk usage                     |
| `ss`         | Network sockets                |
| `lsof`       | Open files                     |
| `dmesg`      | Kernel messages                |
| `strace`     | System calls                   |
| `ps`         | Process information            |
| `systemctl`  | MongoDB service                |
| `journalctl` | systemd logs                   |
| `grep`       | Log searching                  |

---

# 42. Common MongoDB Error Patterns

Here is a useful DBA troubleshooting table.

| Error/Symptom           | Likely area                  | First checks                 |
| ----------------------- | ---------------------------- | ---------------------------- |
| Connection refused      | mongod not listening         | `systemctl`, `ss`            |
| Connection timeout      | Network/firewall             | `nc`, firewall, routing      |
| Too many connections    | Connection pool/application  | `serverStatus().connections` |
| Slow queries            | Index/query design           | profiler, `explain()`        |
| High CPU                | Query/workload               | `top`, profiler              |
| High I/O wait           | Storage                      | `iostat`, `vmstat`           |
| Ticket exhaustion       | Resource contention          | `concurrentTransactions`     |
| Replication lag         | Secondary overloaded         | `rs.status()`                |
| Election storms         | Network/resource instability | replica logs                 |
| Disk full               | Data/log/journal growth      | `df`, `du`                   |
| OOM killed mongod       | Memory pressure              | `dmesg`, `free`              |
| WT_PANIC                | Serious storage/data issue   | logs, backup/replica         |
| Checksum error          | Possible corruption          | logs/storage                 |
| TLS handshake failure   | TLS configuration            | mongod logs                  |
| Authentication failure  | Credentials/auth config      | logs, user config            |
| Permission denied       | Linux filesystem permissions | `ls -l`, service user        |
| Address already in use  | Port conflict                | `ss -lntp`                   |
| Initial sync failure    | Network/data/storage         | replica logs                 |
| Chunk migration failure | Sharding/resource/network    | `sh.status()`, logs          |

---

# 43. MongoDB + Linux Troubleshooting Decision Tree

This is a very useful operational model:

```text
                    MongoDB Problem
                          │
                          ▼
                   Is mongod running?
                    /             \
                  NO               YES
                  │                 │
           systemctl status    Is it reachable?
                                      │
                              ┌───────┴───────┐
                             NO               YES
                             │                 │
                        ss / firewall      Performance?
                                               │
                                    ┌──────────┼──────────┐
                                    ▼          ▼          ▼
                                   CPU        RAM        Disk
                                    │          │          │
                                  top        free      iostat
                                  perf       vmstat    iostat
                                    │          │          │
                                    └──────────┼──────────┘
                                               ▼
                                        MongoDB metrics
                                               │
                               ┌───────────────┼──────────────┐
                               ▼               ▼              ▼
                           Queries         WiredTiger     Replication
                               │               │              │
                           explain()       tickets/cache    rs.status()
```

---

# 44. A Realistic Troubleshooting Scenario

Imagine users report:

> "The application is very slow."

You shouldn't immediately restart MongoDB.

### Step 1 — Check MongoDB

```bash
systemctl status mongod
```

MongoDB is running.

### Step 2 — Check CPU

```bash
top
```

You see:

```text
mongod  25% CPU
```

CPU isn't the problem.

### Step 3 — Check memory

```bash
free -h
```

Memory looks healthy.

### Step 4 — Check disk

```bash
iostat -xz 1
```

You find:

```text
%util = 99%
await = 150 ms
```

Now you have a strong clue.

### Step 5 — Check MongoDB

```javascript
db.serverStatus().wiredTiger.concurrentTransactions
```

You find limited available tickets.

### Step 6 — Check logs

```bash
grep -i "checkpoint\|evict\|wiredtiger" \
/var/log/mongodb/mongod.log
```

You find evidence of storage pressure.

### Diagnosis

```text
Heavy workload
      ↓
Storage latency
      ↓
WiredTiger operations take longer
      ↓
Concurrency resources become constrained
      ↓
Operations wait
      ↓
Application latency increases
```

The correct solution isn't necessarily:

```text
Increase MongoDB tickets
```

The actual solution may involve:

* faster storage
* query optimization
* reducing workload
* proper indexes
* addressing cache pressure
* workload redesign

---

# 45. The Most Important DBA Principle

When troubleshooting MongoDB on Linux, avoid looking at one metric in isolation.

For example:

### High CPU

Don't conclude:

> MongoDB has a CPU problem.

Correlate:

```text
top
 +
mongod currentOp
 +
slow queries
 +
explain()
```

### High memory

Correlate:

```text
free
 +
vmstat
 +
WiredTiger cache
 +
OOM logs
```

### Slow MongoDB

Correlate:

```text
MongoDB latency
 +
CPU
 +
memory
 +
disk I/O
 +
WiredTiger tickets
 +
replication
 +
logs
```

---

# 46. A Practical MongoDB DBA Troubleshooting Checklist

When someone says:

> **"MongoDB is slow/down."**

Run these first:

```bash
# 1. Service
sudo systemctl status mongod

# 2. CPU/process
top
ps -ef | grep mongod

# 3. Memory
free -h
vmstat 1 5

# 4. Disk capacity
df -h
df -i

# 5. Disk usage
sudo du -sh /var/lib/mongodb
sudo du -sh /var/log/mongodb

# 6. Disk performance
iostat -xz 1 5

# 7. Network/listener
sudo ss -lntp | grep 27017

# 8. Kernel errors
sudo dmesg -T | tail -100

# 9. MongoDB logs
sudo tail -100 /var/log/mongodb/mongod.log
```

Then inside MongoDB:

```javascript
db.serverStatus()

db.serverStatus().connections

db.serverStatus().wiredTiger

db.serverStatus().wiredTiger.concurrentTransactions

db.currentOp()

rs.status()
```

For slow queries:

```javascript
db.collection.find({...}).explain("executionStats")
```

---

## 47. The Mental Model to Remember

For MongoDB troubleshooting, think in **four layers**:

```text
┌─────────────────────────────────────────────┐
│              APPLICATION                    │
│   Connection pools / queries / timeouts     │
└──────────────────────┬──────────────────────┘
                       │
┌──────────────────────▼──────────────────────┐
│                 MONGODB                     │
│ Queries / locks / tickets / replication     │
└──────────────────────┬──────────────────────┘
                       │
┌──────────────────────▼──────────────────────┐
│               WIREDTIGER                    │
│ Cache / eviction / checkpoints / journal    │
└──────────────────────┬──────────────────────┘
                       │
┌──────────────────────▼──────────────────────┐
│                  LINUX                      │
│ CPU / RAM / Disk / Network / Kernel / FS   │
└─────────────────────────────────────────────┘
```

**The real skill of a MongoDB DBA is not memorizing commands. It is connecting these four layers and finding the first point where the system starts behaving abnormally.**

The command

is used in Ubuntu/Linux to **view the latest 100 log entries for the MongoDB service**. It is especially useful when troubleshooting MongoDB startup failures, crashes, upgrade issues, and connection problems.

## 1. Understanding each part of the command

| Part         | Explanation                                                                                         |
| ------------ | --------------------------------------------------------------------------------------------------- |
| `sudo`       | Runs the command with administrator privileges, allowing access to protected system logs.           |
| `journalctl` | Displays logs collected by the Linux systemd journal.                                               |
| `-u mongod`  | Filters logs to the `mongod` systemd service.                                                       |
| `-n 100`     | Displays the latest 100 log entries.                                                                |
| `--no-pager` | Prints the output directly in the terminal instead of opening an interactive viewer such as `less`. |

## 2. Example: MongoDB fails to start

Suppose you upgraded MongoDB and then ran:

The service failed to start. You can inspect the reason using:

Example output might look like this:

Interpretation:

* `Starting mongod.service`: Linux attempted to start MongoDB.
* `Address already in use`: MongoDB could not bind to a port because another process may already be using it.
* `Main process exited`: The MongoDB process terminated.
* `Failed to start`: systemd reports that the service failed.

You would then investigate the port conflict rather than repeatedly restarting MongoDB.

## 3. Useful variations

| Command                                          | Purpose                                               |
| ------------------------------------------------ | ----------------------------------------------------- |
| `sudo journalctl -u mongod -n 50 --no-pager`     | View the latest 50 entries.                           |
| `sudo journalctl -u mongod -f`                   | Follow new MongoDB service log entries live.          |
| `sudo journalctl -u mongod --since "1 hour ago"` | View logs from the past hour.                         |
| `sudo journalctl -u mongod -p err --no-pager`    | Show error-priority entries and more severe messages. |
| `sudo journalctl -u mongod -b --no-pager`        | View MongoDB service logs from the current boot.      |

## 4. Important distinction: `journalctl` versus MongoDB's own log file

MongoDB may also write detailed application logs to a file, depending on its configuration.

For example, the default log file in many Ubuntu package installations is:

To view its latest 100 lines:

To follow that file live:

**When should you use which?**

* Use `journalctl` to investigate service startup, systemd failures, and messages captured by the system journal.
* Use `mongod.log` to investigate detailed MongoDB events, queries or operations when logged, storage-engine messages, replication, and other database-level issues.

The two sources may overlap, depending on your MongoDB logging configuration.

**Practical tip:** If MongoDB fails immediately after an upgrade, start with `sudo journalctl -u mongod -n 100 --no-pager`, identify the earliest meaningful error, and then inspect the MongoDB log file for more detail.
