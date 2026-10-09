# MongoDB Profiler: Working, Configuration, and Troubleshooting

The **MongoDB Database Profiler** is a diagnostic tool that records information about database operations, helping administrators identify slow queries, investigate performance problems, and understand how MongoDB executes requests.

For a MongoDB DBA working on Linux, the profiler is particularly useful when investigating:

* Slow-running queries.
* Queries that perform collection scans instead of using indexes.
* Queries examining too many documents.
* Inefficient sorting and aggregation operations.
* High database response times.
* Performance degradation after application deployments.
* Differences in query performance between development and production.
* Operations that consume excessive resources.

We will explore the profiler with practical examples, MongoDB commands, sample output, and troubleshooting scenarios.

---

# 1. What Is the MongoDB Profiler?

MongoDB provides a built-in database profiler that records information about database operations.

When an operation executes, the profiler can capture details such as:

* The command that was executed.
* The namespace involved, such as `sales.orders`.
* The execution duration.
* The number of documents examined.
* The number of documents returned.
* The number of index keys examined.
* The query plan summary.
* Whether the operation used an index.
* The timestamp of the operation.

These details help a DBA understand what happened during a database operation.

## 1.1 Example scenario

Suppose an application executes the following query:

```javascript
db.orders.find({
    customerId: 5001
})
```

The application reports that the query takes several seconds.

As a DBA, you need to determine why.

Possible causes include:

1. MongoDB is scanning the entire collection.
2. The query does not have an appropriate index.
3. The collection contains millions of documents.
4. The query returns too many documents.
5. The server is experiencing resource contention.
6. The query is waiting on locks or other internal resources.

The profiler helps you investigate the operation by recording execution-related information.

For example, you might find a profiler record similar to this:

```javascript
{
    op: "query",
    ns: "sales.orders",
    millis: 2450,
    planSummary: "COLLSCAN",
    docsExamined: 1500000,
    keysExamined: 0,
    nreturned: 10
}
```

**Interpretation:**

| Field          |          Value | Meaning                                        |
| -------------- | -------------: | ---------------------------------------------- |
| `op`           |        `query` | The operation was a query in this example.     |
| `ns`           | `sales.orders` | The database and collection involved.          |
| `millis`       |           2450 | The operation took approximately 2.45 seconds. |
| `planSummary`  |     `COLLSCAN` | MongoDB scanned the collection.                |
| `docsExamined` |      1,500,000 | MongoDB examined 1.5 million documents.        |
| `keysExamined` |              0 | No index keys were examined.                   |
| `nreturned`    |             10 | Ten documents were returned.                   |

This record suggests that MongoDB examined 1.5 million documents to return only ten.

That is a strong indication that the query deserves further investigation.

**Important:** This is illustrative output. The exact fields and values in real profiler records depend on the MongoDB version and operation.

---

# 2. How Does the MongoDB Profiler Work?

MongoDB provides three profiling levels.

| Profiling level | Description                                                          | Typical use                                     |
| --------------- | -------------------------------------------------------------------- | ----------------------------------------------- |
| `0`             | Profiling disabled                                                   | Normal operation when profiling is unnecessary. |
| `1`             | Records operations that meet the configured slow-operation criteria. | Targeted performance investigation.             |
| `2`             | Records all eligible database operations.                            | Short, controlled diagnostic investigations.    |

Let's understand each level.

## 2.1 Level 0: Profiling disabled

This is the default profiling level in many MongoDB deployments.

```javascript
db.setProfilingLevel(0)
```

At this level, the database profiler does not record operations in the `system.profile` collection.

However, this does **not** mean MongoDB stops monitoring slow operations.

MongoDB's diagnostic logging can still report slow operations according to its configured thresholds.

For example, a slow operation might appear in the MongoDB log even when database profiling is disabled.

Therefore, remember the distinction:

* **Database profiler:** Stores profiling records in `system.profile`.
* **Diagnostic logging:** Records operational events, including slow operations, according to logging configuration.

Both are useful troubleshooting mechanisms.

## 2.2 Level 1: Profile slow operations

This is generally the most useful level for targeted performance troubleshooting.

```javascript
db.setProfilingLevel(1)
```

At level 1, MongoDB records operations that meet the applicable slow-operation criteria.

For example, suppose the slow-operation threshold is 100 milliseconds.

An operation taking 25 milliseconds may not be recorded, while one taking 800 milliseconds may be recorded.

However, the exact behavior depends on the effective profiling configuration, including the slow-operation threshold and any configured sampling settings.

We will examine these settings shortly.

## 2.3 Level 2: Profile all eligible operations

```javascript
db.setProfilingLevel(2)
```

This level records all eligible operations rather than limiting records to slow operations.

It can be useful when investigating a problem that affects both fast and slow operations, or when you need a short diagnostic trace.

However, it can generate a large volume of profiling data.

For example, if an application executes 2,000 operations per second, level 2 may generate a substantial number of records.

This can cause:

* Increased diagnostic overhead.
* Rapid growth of profiling data.
* Additional disk activity.
* Increased difficulty finding relevant operations.

**Production recommendation:** Prefer level 1 for targeted investigations. Use level 2 only for a limited, controlled period when the additional detail is necessary.

---

# 3. Where Does MongoDB Store Profiler Information?

MongoDB stores profiler records in a special collection called:

```javascript
system.profile
```

This collection exists in the database for which profiling is enabled.

For example, if you enable profiling in the `sales` database, the records are stored in:

```text
sales.system.profile
```

If you enable profiling in the `admin` database, the records are stored in:

```text
admin.system.profile
```

This is important because profiling is configured at the database level.

Enabling profiling in one application database does not automatically enable it in every other database.

## 3.1 Inspect the current profiling configuration

Switch to the database you want to investigate.

For example:

```javascript
use sales
```

Execute:

```javascript
db.getProfilingStatus()
```

Example output:

```javascript
{
    was: 0,
    slowms: 100,
    sampleRate: 1
}
```

Interpretation:

| Field        | Meaning                                                                           |
| ------------ | --------------------------------------------------------------------------------- |
| `was`        | The current profiling level.                                                      |
| `slowms`     | The configured slow-operation threshold in milliseconds.                          |
| `sampleRate` | The sampling rate for operations subject to profiling criteria, where applicable. |

In this example:

* Profiling is disabled because `was` is `0`.
* The slow-operation threshold is 100 milliseconds.
* The sample rate is 1, meaning 100% of operations meeting the applicable criteria are sampled.

The exact fields returned can vary by MongoDB version and configuration.

## 3.2 Check whether the profiler collection exists

Execute:

```javascript
db.getCollectionNames()
```

Look for:

```text
system.profile
```

Alternatively, inspect the collection directly:

```javascript
db.system.profile.findOne()
```

If profiling has not been enabled and no profiler collection exists, the command may return `null` or otherwise indicate that there are no records to display.

Do not confuse the absence of profiler records with the absence of slow operations. Profiling may simply be disabled, the operations may not meet the recording criteria, or the collection may not yet contain records.

---

# 4. Configuring the MongoDB Profiler Step by Step

Let's configure the profiler in a sample environment.

Assume:

* Database: `sales`
* Collection: `orders`
* Slow-operation threshold: 100 milliseconds

## Step 1: Connect to MongoDB

From a Linux terminal, connect using `mongosh`.

```bash
mongosh
```

If authentication is enabled, provide the appropriate authentication credentials and authentication database.

For example:

```bash
mongosh "mongodb://localhost:27017/sales" \
  --username profilerUser \
  --authenticationDatabase admin
```

You will be prompted for the password if one is not supplied in the connection string.

The account must have the privileges needed to configure profiling and inspect the profiler collection. Use an appropriately authorized DBA account for these exercises.

## Step 2: Select the database

```javascript
use sales
```

## Step 3: Check the existing configuration

```javascript
db.getProfilingStatus()
```

Suppose the result is:

```javascript
{
    was: 0,
    slowms: 100,
    sampleRate: 1
}
```

Profiling is disabled.

## Step 4: Enable profiling for slow operations

Execute:

```javascript
db.setProfilingLevel(1, {
    slowms: 100
})
```

This enables profiling at level 1 and sets the slow-operation threshold to 100 milliseconds for the database.

**Important:** MongoDB also has a server-wide slow-operation threshold used by diagnostic logging. Database profiling configuration and server-wide logging configuration are related but distinct. A database-level profiler setting should not be assumed to change the global diagnostic logging threshold.

## Step 5: Verify the configuration

```javascript
db.getProfilingStatus()
```

Example:

```javascript
{
    was: 1,
    slowms: 100,
    sampleRate: 1
}
```

Profiling is now enabled for qualifying slow operations in the `sales` database.

## Step 6: Execute a query

Let's create sample data if you do not already have an appropriate collection.

```javascript
db.orders.insertMany([
    {
        orderId: 101,
        customerId: 5001,
        amount: 1500
    },
    {
        orderId: 102,
        customerId: 5002,
        amount: 2200
    },
    {
        orderId: 103,
        customerId: 5001,
        amount: 1800
    }
])
```

Run a query:

```javascript
db.orders.find({
    customerId: 5001
})
```

This small query will probably execute too quickly to be captured with a 100-millisecond threshold.

That is expected.

For a useful test, create a sufficiently large test dataset and investigate queries that genuinely take longer than the configured threshold.

Avoid adding artificial delays or running expensive operations against production data merely to generate profiler records.

## Step 7: Read the profiler records

```javascript
db.system.profile.find().sort({
    ts: -1
}).limit(10)
```

This retrieves up to ten records, newest first.

For a more readable output:

```javascript
db.system.profile.find(
    {},
    {
        ts: 1,
        ns: 1,
        millis: 1,
        planSummary: 1,
        docsExamined: 1,
        keysExamined: 1,
        nreturned: 1,
        command: 1
    }
).sort({
    ts: -1
}).limit(10).pretty()
```

This is one of the most useful commands for an initial profiler investigation.

It shows when an operation occurred, which collection it accessed, how long it took, and whether the execution plan used an index or performed a collection scan.

---

# 5. Understanding Important Profiler Fields

To troubleshoot effectively, you need to understand what the fields mean.

Consider this illustrative record:

```javascript
{
    op: "query",
    ns: "sales.orders",
    command: {
        find: "orders",
        filter: {
            customerId: 5001
        }
    },
    millis: 1850,
    planSummary: "COLLSCAN",
    keysExamined: 0,
    docsExamined: 1200000,
    nreturned: 5,
    ts: ISODate("2026-10-09T08:30:00Z")
}
```

## 5.1 `millis`

Example:

```javascript
millis: 1850
```

This indicates that the operation took approximately 1.85 seconds.

Use it to identify slow operations.

However, do not interpret every high-duration operation as an indexing problem. The duration may reflect execution work, resource contention, storage latency, or other factors.

## 5.2 `ns`

Example:

```javascript
ns: "sales.orders"
```

This identifies the database and collection involved.

Use it to narrow the investigation to a specific collection.

## 5.3 `planSummary`

Example:

```javascript
planSummary: "COLLSCAN"
```

This indicates a collection scan.

Another possible value is:

```javascript
planSummary: "IXSCAN { customerId: 1 }"
```

This indicates an index scan involving the `customerId` index.

The exact representation varies by operation and MongoDB version.

**Important:** An `IXSCAN` does not automatically mean a query is efficient. An index scan can still examine a large number of keys or documents.

## 5.4 `docsExamined`

Example:

```javascript
docsExamined: 1200000
```

MongoDB examined 1.2 million documents.

This is often one of the most valuable clues when troubleshooting slow queries.

If MongoDB examines many documents to return only a few, investigate:

* Missing indexes.
* Poorly selective indexes.
* Index field order.
* Query predicates.
* Large result sets or residual filtering.
* Whether the query plan is appropriate for the workload.

## 5.5 `keysExamined`

Example:

```javascript
keysExamined: 0
```

No index keys were examined in this example.

That is consistent with a collection scan.

If the value is very high, the query may be examining too many index entries.

## 5.6 `nreturned`

Example:

```javascript
nreturned: 5
```

Five documents were returned.

Compare this with `docsExamined`.

For example:

| Metric             |     Value |
| ------------------ | --------: |
| Documents examined | 1,200,000 |
| Documents returned |         5 |

This discrepancy suggests that MongoDB is doing a large amount of work relative to the number of results returned.

It is a strong reason to investigate the query plan and indexing strategy.

However, it is not sufficient by itself to prove that an index is missing. Some queries legitimately need to examine many documents.

## 5.7 `ts`

Example:

```javascript
ts: ISODate("2026-10-09T08:30:00Z")
```

This records the operation timestamp.

Use it to correlate profiler events with:

* Application logs.
* Deployment timestamps.
* CPU spikes.
* Disk I/O spikes.
* Connection spikes.
* Replication events.

Be aware that the displayed timestamp may use UTC. Convert timestamps when correlating them with local-time application logs.

## 5.8 `command`

The `command` field contains details of the operation, such as the query filter, aggregation pipeline, or update specification.

For example:

```javascript
command: {
    find: "orders",
    filter: {
        customerId: 5001
    }
}
```

This allows you to identify the operation responsible for the performance problem.

When examining profiler data, be careful with sensitive information. Query filters can contain personal data, identifiers, or application-supplied values.

---

# 6. How to Find the Slowest Queries

Suppose the `sales` database is experiencing performance issues and you want to identify its slowest operations.

## 6.1 Retrieve the 10 slowest recorded operations

```javascript
db.system.profile.find({
    millis: {
        $exists: true
    }
}).sort({
    millis: -1
}).limit(10)
```

This sorts recorded operations by duration, highest first.

A more focused version is:

```javascript
db.system.profile.find(
    {},
    {
        ts: 1,
        ns: 1,
        millis: 1,
        planSummary: 1,
        docsExamined: 1,
        keysExamined: 1,
        nreturned: 1,
        command: 1
    }
).sort({
    millis: -1
}).limit(10).pretty()
```

Use the results to identify operations that deserve further investigation.

### What should you look for?

| Observation                                   | Possible explanation                                  | Next step                                               |
| --------------------------------------------- | ----------------------------------------------------- | ------------------------------------------------------- |
| High `millis` and `COLLSCAN`                  | Collection scan may be expensive.                     | Inspect the query and indexes.                          |
| High `millis` and high `docsExamined`         | Excessive document examination.                       | Review selectivity and the execution plan.              |
| High `keysExamined`                           | Many index entries are being scanned.                 | Examine index selectivity and query predicates.         |
| High `millis` with modest examination counts  | The bottleneck may not be query scanning.             | Investigate contention, I/O, and other execution costs. |
| Large `nreturned`                             | The application may be requesting too many documents. | Consider pagination and projection.                     |
| Repeated slow operations with the same filter | A frequently executed query may be inefficient.       | Investigate the query shape and workload.               |

These are diagnostic clues, not definitive conclusions. Confirm the cause before making changes.

---

# 7. Troubleshooting Scenario 1: Slow Query Caused by a Collection Scan

This is one of the most common scenarios in MongoDB performance investigations.

## Problem

An application executes:

```javascript
db.orders.find({
    customerId: 5001
})
```

The query takes approximately two seconds.

The profiler contains a record similar to:

```javascript
{
    ns: "sales.orders",
    millis: 2100,
    planSummary: "COLLSCAN",
    docsExamined: 1500000,
    keysExamined: 0,
    nreturned: 12
}
```

## Step 1: Interpret the profiler record

MongoDB:

1. Scanned 1.5 million documents.
2. Found 12 matching documents.
3. Returned those 12 documents.
4. Took approximately 2.1 seconds.

The likely problem is that the query needs to examine far more documents than necessary.

## Step 2: Check existing indexes

Execute:

```javascript
db.orders.getIndexes()
```

Suppose the output shows only:

```javascript
[
    {
        v: 2,
        key: {
            _id: 1
        },
        name: "_id_"
    }
]
```

There is no index on `customerId`.

## Step 3: Create an appropriate index

For this query, a single-field index may be appropriate:

```javascript
db.orders.createIndex({
    customerId: 1
})
```

This creates an ascending index on `customerId`.

MongoDB can now use that index to locate documents matching the filter.

**Before creating an index in production**, consider its storage requirements, write overhead, workload benefit, and whether an equivalent index already exists.

## Step 4: Examine the execution plan

Execute:

```javascript
db.orders.find({
    customerId: 5001
}).explain("executionStats")
```

Look for a winning plan involving `IXSCAN`, and examine:

```javascript
executionStats.totalDocsExamined
executionStats.totalKeysExamined
executionStats.nReturned
```

A more efficient plan might resemble:

```javascript
{
    executionStats: {
        nReturned: 12,
        totalKeysExamined: 12,
        totalDocsExamined: 12
    }
}
```

These values are illustrative; actual counts depend on the data and execution plan.

## Step 5: Compare the results

| Metric              | Before indexing | Possible result after indexing |
| ------------------- | --------------: | -----------------------------: |
| Documents examined  |       1,500,000 |                             12 |
| Index keys examined |               0 |                             12 |
| Documents returned  |              12 |                             12 |
| Execution plan      |      `COLLSCAN` |                       `IXSCAN` |
| Expected work       |       Very high |                     Much lower |

The index reduces the number of documents MongoDB needs to examine.

The actual execution time depends on data distribution, cache state, storage performance, and other factors.

### Important distinction: Profiler versus explain

The profiler tells you what happened during a recorded operation.

`explain("executionStats")` helps you investigate how MongoDB executes the query and how many documents or index keys it examines.

**Use the profiler to identify the problem, and use `explain()` to investigate the execution plan.**

---

# 8. Troubleshooting Scenario 2: The Query Uses an Index but Is Still Slow

A common mistake is to assume that every query using an index must be fast.

That is not true.

Consider this profiler record:

```javascript
{
    ns: "sales.orders",
    millis: 1800,
    planSummary: "IXSCAN { status: 1 }",
    keysExamined: 800000,
    docsExamined: 800000,
    nreturned: 500
}
```

The query uses an index, but it still examines 800,000 index entries and documents to return 500 results.

## What could be happening?

The indexed field may have low selectivity.

For example, suppose the query is:

```javascript
db.orders.find({
    status: "COMPLETED"
})
```

If 80% of the collection has the status `COMPLETED`, an index on `status` may still require MongoDB to process a large number of matching entries and documents.

Whether the index is beneficial depends on the query, the data distribution, and the requested results.

## Step 1: Inspect the query plan

```javascript
db.orders.find({
    status: "COMPLETED"
}).explain("executionStats")
```

Examine:

```javascript
executionStats.totalKeysExamined
executionStats.totalDocsExamined
executionStats.nReturned
```

Also inspect the winning plan.

## Step 2: Investigate the query requirements

Suppose the actual query is:

```javascript
db.orders.find({
    status: "COMPLETED",
    customerId: 5001
})
```

If this query is frequent, an index on both fields may be worth evaluating.

For example:

```javascript
db.orders.createIndex({
    customerId: 1,
    status: 1
})
```

The appropriate field order depends on the workload, query predicates, sorting requirements, and other index-use considerations.

Do not create compound indexes solely because multiple fields appear in a query. Confirm the benefit with representative execution plans and workload testing.

## Step 3: Compare before and after

Compare:

* `totalKeysExamined`
* `totalDocsExamined`
* `nReturned`
* Execution duration
* Index storage and write overhead

A good index should improve the overall workload, not just one isolated query.

---

# 9. Troubleshooting Scenario 3: Slow Aggregation Pipelines

The profiler is also useful when diagnosing slow aggregation operations.

Consider:

```javascript
db.orders.aggregate([
    {
        $match: {
            status: "COMPLETED"
        }
    },
    {
        $group: {
            _id: "$customerId",
            totalAmount: {
                $sum: "$amount"
            }
        }
    },
    {
        $sort: {
            totalAmount: -1
        }
    }
])
```

Suppose the profiler shows a long-running aggregation command.

Potential causes include:

* Too many input documents.
* Expensive grouping.
* An expensive sort.
* Insufficient memory for intermediate processing.
* Disk spilling during eligible stages.
* A pipeline that filters documents later than necessary.
* An inefficient query plan for the initial filtering stage.

## Step 1: Inspect the profiler record

```javascript
db.system.profile.find({
    "command.aggregate": "orders"
}).sort({
    ts: -1
}).limit(10).pretty()
```

This filters for records whose `command.aggregate` field is `orders`.

The exact command representation depends on the aggregation operation.

## Step 2: Examine the aggregation execution plan

Run:

```javascript
db.orders.explain("executionStats").aggregate([
    {
        $match: {
            status: "COMPLETED"
        }
    },
    {
        $group: {
            _id: "$customerId",
            totalAmount: {
                $sum: "$amount"
            }
        }
    },
    {
        $sort: {
            totalAmount: -1
        }
    }
])
```

Inspect the explain output for execution statistics, index usage where applicable, and aggregation-stage details.

## Step 3: Filter early

An aggregation pipeline should generally filter unnecessary documents as early as possible.

For example, `$match` near the beginning can reduce the number of documents processed by later stages.

However, MongoDB can optimize aggregation pipelines automatically, so verify the actual execution plan rather than assuming the written order directly determines every execution step.

## Step 4: Investigate expensive stages

Pay particular attention to:

* `$group`
* `$sort`
* `$lookup`
* `$unwind`

Large grouping operations and sorts can be expensive.

A sort on a computed field, such as `totalAmount` in the example, generally cannot be satisfied directly by an index on the original collection because that field is computed during aggregation.

If a stage spills to disk, investigate why it is processing so much data and whether the pipeline can be improved.

**Remember:** The profiler helps identify a slow aggregation. The aggregation explain output helps you investigate its execution.

---

# 10. Troubleshooting Scenario 4: Slow Queries Without a Collection Scan

Suppose the profiler shows:

```javascript
{
    ns: "sales.orders",
    millis: 2400,
    planSummary: "IXSCAN { customerId: 1 }",
    docsExamined: 10,
    keysExamined: 10,
    nreturned: 10
}
```

Only ten documents and ten index keys were examined.

This is unlikely to be a classic excessive-document-scan problem.

You should investigate other possibilities.

### Possible causes

1. Storage latency.
2. Resource contention.
3. A query that spends significant time in another execution phase.
4. Lock or internal resource waits.
5. Network or client-side delays that are not fully represented by the server's operation duration.
6. Other environmental or workload-related factors.

Do not immediately create another index.

Instead, investigate the server during the period in which the slow operation occurred.

## On Ubuntu, inspect CPU and memory

```bash
top
```

Look for:

* High CPU utilization.
* Memory pressure.
* Swap activity.
* Other processes consuming significant resources.

## Inspect disk I/O

```bash
iostat -xz 1
```

If `iostat` is unavailable, install the appropriate system monitoring package, such as `sysstat`, if permitted.

Look for:

* High device utilization.
* Elevated I/O wait.
* Increased storage latency.
* Unexpectedly high read/write activity.

The meaning of individual disk metrics depends on the storage device and environment.

## Inspect MongoDB logs

```bash
sudo journalctl -u mongod -n 100 --no-pager
```

Search the logs for relevant events around the time of the slow query.

You can also inspect the current server status:

```javascript
db.serverStatus()
```

For a focused look at connections:

```javascript
db.serverStatus().connections
```

And WiredTiger cache statistics:

```javascript
db.serverStatus().wiredTiger.cache
```

These commands provide additional context that the profiler alone may not supply.

---

# 11. How to Find Repeatedly Slow Queries

Sometimes a single slow query is not the main problem.

Instead, an application may execute the same moderately expensive query thousands of times.

For example, a query might take 50 milliseconds, which is acceptable in isolation.

But if it runs 1,000 times per second, it can create substantial aggregate workload.

## Step 1: Find recent operations for a specific collection

```javascript
db.system.profile.find({
    ns: "sales.orders",
    millis: {
        $gte: 100
    }
}).sort({
    ts: -1
}).limit(50)
```

This returns up to 50 recorded operations on `sales.orders` taking at least 100 milliseconds.

The profiler can help you recognize repeated patterns in query filters and commands.

## Step 2: Inspect the repeated query shape

Look for:

* Repeated filters on the same fields.
* Similar aggregation pipelines.
* Repeated updates to the same collection.
* Excessive sorting.
* Repeated scans of the same collection.
* Similar queries with different parameter values.

## Step 3: Investigate the workload

Determine whether the application is:

* Executing the same query more often than expected.
* Performing unnecessary repeated reads.
* Loading more data than it needs.
* Using inefficient pagination.
* Making unnecessary database round trips.

Possible remedies include appropriate indexing, better query design, caching where appropriate, batching, and reducing unnecessary requests.

The correct remedy depends on the workload.

---

# 12. How to Query Profiler Data Effectively

You can use ordinary MongoDB query operators to investigate profiler records.

The following examples assume you are using the `sales` database.

## 12.1 Find operations slower than one second

```javascript
db.system.profile.find({
    millis: {
        $gt: 1000
    }
}).sort({
    ts: -1
})
```

## 12.2 Find collection scans

```javascript
db.system.profile.find({
    planSummary: "COLLSCAN"
}).sort({
    ts: -1
})
```

This can identify recorded operations with that plan summary.

However, not every `COLLSCAN` is a performance problem. Small collections and queries that legitimately need many documents can make collection scans appropriate.

## 12.3 Find operations examining more than 100,000 documents

```javascript
db.system.profile.find({
    docsExamined: {
        $gt: 100000
    }
}).sort({
    docsExamined: -1
})
```

This is useful for identifying operations that examine many documents.

## 12.4 Find operations with a high examination-to-result ratio

The following aggregation calculates a ratio for profiler records that return at least one document:

```javascript
db.system.profile.aggregate([
    {
        $match: {
            nreturned: {
                $gt: 0
            },
            docsExamined: {
                $gt: 0
            }
        }
    },
    {
        $project: {
            ns: 1,
            ts: 1,
            millis: 1,
            docsExamined: 1,
            nreturned: 1,
            planSummary: 1,
            examinationRatio: {
                $divide: [
                    "$docsExamined",
                    "$nreturned"
                ]
            }
        }
    },
    {
        $sort: {
            examinationRatio: -1
        }
    },
    {
        $limit: 20
    }
])
```

Example interpretation:

| Documents examined | Documents returned |  Ratio |
| -----------------: | -----------------: | -----: |
|            100,000 |                 10 | 10,000 |
|              5,000 |                100 |     50 |
|                200 |                100 |      2 |

A high ratio can indicate that MongoDB examines many documents for relatively few results.

However, it is not a universal performance metric. It can be misleading for aggregations, commands, and operations where the relationship between examined documents and returned documents is not straightforward.

Also, an operation returning zero documents is excluded by this example even though such an operation may be expensive.

---

# 13. Configuring the Slow-Operation Threshold

One of the most important aspects of profiling is selecting an appropriate threshold.

Consider:

```javascript
db.setProfilingLevel(1, {
    slowms: 200
})
```

This enables level 1 profiling with a 200-millisecond threshold.

An operation that takes 500 milliseconds may qualify for profiling, while one that takes 50 milliseconds may not.

The threshold should reflect your investigation goals.

| Threshold | Possible use                                                      |
| --------- | ----------------------------------------------------------------- |
| 50 ms     | Investigating latency-sensitive workloads.                        |
| 100 ms    | General performance investigation.                                |
| 500 ms    | Focusing on longer-running operations.                            |
| 1,000 ms  | Identifying operations taking more than approximately one second. |

These are illustrative thresholds, not universal recommendations.

The appropriate value depends on your application's latency requirements and normal workload.

For example, a 100-millisecond database operation might be acceptable for a background reporting task but unacceptable for a latency-sensitive user request.

### Important: Sampling and thresholds

The slow-operation threshold and sampling settings determine which operations are recorded.

For example:

```javascript
db.setProfilingLevel(1, {
    slowms: 100,
    sampleRate: 0.5
})
```

This configures a 100-millisecond threshold and a sampling rate of 0.5 for eligible profiling operations.

A sampling rate of 0.5 means that approximately half of the eligible operations are sampled over time; it does not guarantee that exactly half will be recorded in every small time window.

**For a controlled investigation, consider the impact of the threshold and sampling configuration before interpreting the absence of a profiler record as proof that no slow operation occurred.**

---

# 14. How Long Should You Keep Profiling Enabled?

For production troubleshooting, enable profiling only as long as necessary.

A practical investigation might look like this:

1. Check the existing profiling configuration.
2. Enable level 1 profiling.
3. Reproduce or observe the performance problem.
4. Collect relevant profiler records.
5. Analyze the query plans.
6. Make and validate a targeted change.
7. Disable profiling if it is no longer needed.

For example:

```javascript
use sales

db.setProfilingLevel(1, {
    slowms: 100
})
```

After collecting sufficient diagnostic information:

```javascript
db.setProfilingLevel(0)
```

This disables database profiling for `sales`.

However, disabling profiling does not automatically remove previously collected profiler records.

The profiler collection is a diagnostic data store, not a temporary display that disappears when profiling is turned off.

Also, avoid assuming that profiling overhead is negligible. Its impact depends on workload volume, sampling, and deployment characteristics.

---

# 15. How to Clear Old Profiler Records

Sometimes the profiler collection contains old records that are no longer relevant to the current investigation.

Before removing any data, confirm that you do not need the records for an ongoing incident or audit.

To inspect the collection:

```javascript
db.system.profile.find().sort({
    ts: -1
}).limit(20)
```

If you intentionally need to remove profiler records in a suitable environment, use an appropriately authorized maintenance procedure.

For example, after verifying the target database and understanding the impact, an administrator may remove records using:

```javascript
db.system.profile.deleteMany({})
```

**Use this command cautiously.** It deletes the profiler records in the current database's `system.profile` collection. It does not delete application data from `orders`, but it permanently removes the selected diagnostic records.

For ongoing management, consider the collection's capped nature and retention behavior rather than routinely deleting records.

---

# 16. MongoDB Profiler Versus Diagnostic Logs Versus Explain

These tools complement one another, but they serve different purposes.

| Feature                                    | Database profiler                        | Diagnostic logs                                  | `explain()`                                                              |
| ------------------------------------------ | ---------------------------------------- | ------------------------------------------------ | ------------------------------------------------------------------------ |
| Primary purpose                            | Record database operations for analysis. | Record operational and diagnostic events.        | Examine query execution plans and statistics.                            |
| Stores historical operation records        | Yes, in `system.profile`.                | Yes, subject to log configuration and retention. | Not as a historical operation log.                                       |
| Helps identify slow operations             | Yes                                      | Yes                                              | Helps evaluate a query when explicitly run.                              |
| Shows the actual recorded operation        | Yes                                      | Often, depending on log details                  | Evaluates the query or command you execute.                              |
| Shows execution plan information           | Often includes a plan summary            | May include slow-operation details               | Provides detailed execution-plan information.                            |
| Shows documents examined                   | Often                                    | Depends on the log entry and configuration       | Provides execution statistics when supported.                            |
| Useful for investigating a past incident   | Yes, if records are retained             | Yes, if logs are retained                        | Only indirectly; the original query must be reconstructed or reproduced. |
| Suitable for routine production monitoring | Use selectively                          | Yes, with appropriate configuration              | Better for targeted investigation and testing.                           |

### Which should you use first?

A useful troubleshooting sequence is:

**Profiler → Identify the slow operation → Explain → Investigate server metrics → Validate the fix.**

For example:

1. Use the profiler to identify a query taking 2 seconds.
2. Discover that it uses `COLLSCAN`.
3. Run `explain("executionStats")` to inspect the execution plan.
4. Evaluate whether an index can reduce document examination.
5. Check CPU, memory, and disk metrics if the query remains slow.
6. Re-run the query and compare the results.

This process is more reliable than creating indexes based only on assumptions.

---

# 17. Production Troubleshooting Workflow on Ubuntu

Suppose your MongoDB application is running slowly on Ubuntu.

Here is a practical investigation procedure.

## Step 1: Confirm the database and profiler configuration

Connect to MongoDB:

```bash
mongosh
```

Select the relevant database:

```javascript
use sales
```

Check profiling:

```javascript
db.getProfilingStatus()
```

## Step 2: Enable targeted profiling

For example:

```javascript
db.setProfilingLevel(1, {
    slowms: 100
})
```

Use an appropriate threshold for the application.

## Step 3: Collect slow operations

```javascript
db.system.profile.find({
    millis: {
        $gte: 100
    }
}).sort({
    ts: -1
}).limit(50).pretty()
```

## Step 4: Identify the most expensive query

Look for:

* High execution duration.
* `COLLSCAN`.
* High `docsExamined`.
* High `keysExamined`.
* Large numbers of returned documents.
* Repeated occurrences of the same query shape.

## Step 5: Examine the query plan

Reconstruct the relevant query and run:

```javascript
db.orders.find({
    customerId: 5001
}).explain("executionStats")
```

Review the execution plan and the examination counts.

For aggregation operations, use the corresponding aggregation explain command.

## Step 6: Check server resources

On Ubuntu:

```bash
top
```

```bash
free -h
```

```bash
iostat -xz 1
```

```bash
vmstat 1
```

These commands help identify CPU pressure, memory pressure, disk activity, and other host-level issues.

## Step 7: Check MongoDB logs

```bash
sudo journalctl -u mongod -n 100 --no-pager
```

If MongoDB writes logs to a file instead of the system journal, inspect the configured log file.

Look for errors and warnings near the incident timestamp.

## Step 8: Apply a targeted change

Depending on the evidence, you might:

* Create an appropriate index.
* Adjust a compound index.
* Rewrite a query.
* Reduce the number of returned documents.
* Improve pagination.
* Optimize an aggregation pipeline.
* Address storage or resource contention.
* Reduce unnecessary application requests.

Do not make multiple unrelated changes at once if you need to determine which change solved the problem.

## Step 9: Validate the result

Re-run the query and compare its execution statistics.

For example:

```javascript
db.orders.find({
    customerId: 5001
}).explain("executionStats")
```

Compare the results with the original investigation.

If you need to measure actual operation duration, use a controlled and representative test rather than relying solely on explain output.

## Step 10: Disable profiling when finished

```javascript
db.setProfilingLevel(0)
```

Do this if you no longer need profiling for the investigation.

Remember that profiling is configured per database. Check whether profiling was enabled in any other databases you investigated.

---

# 18. Common Mistakes When Using the MongoDB Profiler

| Mistake                                         | Why it is a problem                                                                                                   | Better approach                                                    |
| ----------------------------------------------- | --------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| Enabling level 2 indefinitely                   | Can generate a large volume of diagnostic records and additional overhead.                                            | Use level 1 for most targeted investigations.                      |
| Assuming no profiler record means no slow query | Profiling may be disabled, sampling may exclude the operation, or the threshold may not be met.                       | Check the profiling configuration and diagnostic logs.             |
| Creating an index whenever a query is slow      | The bottleneck may be elsewhere, and unnecessary indexes add storage and write costs.                                 | Inspect the execution plan first.                                  |
| Treating every `COLLSCAN` as a problem          | Collection scans can be appropriate for small collections or queries that need many documents.                        | Evaluate the workload and execution statistics.                    |
| Assuming `IXSCAN` means the query is efficient  | An index scan may still examine a large number of keys and documents.                                                 | Inspect `totalKeysExamined`, `totalDocsExamined`, and `nReturned`. |
| Ignoring repeated moderately slow queries       | High-frequency operations can create substantial aggregate load.                                                      | Investigate query frequency and repeated query shapes.             |
| Investigating only the query plan               | Server-side resource contention can also affect performance.                                                          | Correlate profiler records with system metrics and logs.           |
| Forgetting to disable profiling                 | Unnecessary diagnostic overhead may continue.                                                                         | Restore the intended profiling level after the investigation.      |
| Assuming profiling is cluster-wide              | Profiling configuration is database-specific and is not automatically applied to every database or deployment member. | Check each relevant database and deployment component.             |

---

# 19. Important Considerations for Replica Sets and Sharded Clusters

If you administer a MongoDB replica set or sharded cluster, the troubleshooting process requires additional care.

## Replica sets

A replica set contains a primary and one or more secondary members.

An operation executed on the primary may not have the same performance characteristics as a read served by a secondary.

For example:

* A write operation is normally directed to the primary.
* A read operation may be directed to a secondary, depending on the client's read preference.
* Replication activity can contribute to resource contention.
* Each member has its own server-side diagnostic information.

When investigating a slow query, identify which member processed it.

If necessary, inspect the relevant member's profiler data, logs, and server metrics.

## Sharded clusters

A sharded query may involve multiple shards.

For example, if a query cannot target a specific shard, MongoDB may need to contact multiple shards.

The profiler on an individual shard shows operations observed by that shard; it does not necessarily provide a complete end-to-end picture of the application's request.

When troubleshooting a sharded query:

1. Identify the query issued by the application.
2. Examine its execution plan and shard targeting.
3. Investigate relevant shard-level operations.
4. Check whether the query is being scattered across multiple shards.
5. Correlate router, shard, and application logs.

Use the profiler as one part of the investigation rather than assuming a single profiler record captures the entire distributed operation.

---

# 20. Quick Reference: MongoDB Profiler Commands

The following commands are useful to keep handy.

| Task                                        | Command                                                        |
| ------------------------------------------- | -------------------------------------------------------------- |
| Check profiling status                      | `db.getProfilingStatus()`                                      |
| Disable profiling                           | `db.setProfilingLevel(0)`                                      |
| Profile slow operations                     | `db.setProfilingLevel(1, { slowms: 100 })`                     |
| Profile all eligible operations temporarily | `db.setProfilingLevel(2)`                                      |
| Read recent profiler records                | `db.system.profile.find().sort({ts: -1}).limit(10)`            |
| Find operations slower than one second      | `db.system.profile.find({millis: {$gt: 1000}})`                |
| Find collection scans                       | `db.system.profile.find({planSummary: "COLLSCAN"})`            |
| Find operations examining many documents    | `db.system.profile.find({docsExamined: {$gt: 100000}})`        |
| Inspect collection indexes                  | `db.orders.getIndexes()`                                       |
| Examine a query plan                        | `db.orders.find({customerId: 5001}).explain("executionStats")` |
| Check server connections                    | `db.serverStatus().connections`                                |
| Inspect WiredTiger cache                    | `db.serverStatus().wiredTiger.cache`                           |
| Inspect MongoDB service logs                | `sudo journalctl -u mongod -n 100 --no-pager`                  |
| Check CPU and memory                        | `top`                                                          |
| Check disk I/O                              | `iostat -xz 1`                                                 |

*Thresholds in these examples are illustrative. Adapt them to your workload and MongoDB configuration.*

---

# 21. The Most Important Concept to Remember

When troubleshooting MongoDB performance, **do not use the profiler merely to collect slow-query records. Use it to build a chain of evidence.**

Consider this example:

```text
Application reports slow response
              |
              v
Enable targeted profiling
              |
              v
Identify slow operation
              |
              v
Inspect millis, docsExamined,
keysExamined and planSummary
              |
              v
Investigate the query using explain()
              |
              v
Check indexes, query design,
CPU, memory and disk I/O
              |
              v
Apply a targeted fix
              |
              v
Measure performance again
              |
              v
Disable profiling if no longer needed
```

The profiler tells you **which operations deserve attention and what MongoDB recorded about them**.

The execution plan helps explain **how MongoDB processed the query**.

The server logs and Linux monitoring tools help determine whether the problem originates in query execution, storage, resource contention, or another part of the system.

Using these tools together gives you a much stronger troubleshooting methodology than relying on any one metric.

**Recommended practice:** Start with level 1 profiling, identify the slowest relevant operations, investigate their execution plans, correlate the results with system metrics, and validate each corrective change.
