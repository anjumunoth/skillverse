# MongoDB Upgrade Paths, FCV Management, Rolling Upgrades, and Linux Patch Management

This guide focuses on **MongoDB 8.x on Ubuntu/Linux**, with practical commands, upgrade procedures, compatibility considerations, and methods for minimizing downtime in production environments.

The topics are closely related, but they solve different problems:

* **Upgrade paths:** Which MongoDB version can you upgrade from and to?
* **Feature Compatibility Version (FCV):** Which version's compatibility rules and features is MongoDB using?
* **Rolling upgrades:** How do you upgrade a replica set or sharded cluster while keeping the service available?
* **Linux patch management:** How do you patch the operating system and MongoDB packages while minimizing disruption?

---

## 1. Understanding MongoDB versioning

MongoDB version numbers identify the software release installed on the server.

For example:

MongoDB's version numbering and supported upgrade paths depend on the release series. Do not assume that every version can be upgraded directly to every newer version.

### Three important version concepts

| Concept                          | Meaning                                                         | Example                                 |
| -------------------------------- | --------------------------------------------------------------- | --------------------------------------- |
| Binary version                   | Version of the installed MongoDB server executable              | `mongod` 8.0.x                          |
| FCV                              | Compatibility mode governing certain data and feature behaviors | FCV `8.0`                               |
| Operating system/package version | Ubuntu release and installed MongoDB package build              | Ubuntu 24.04 with a MongoDB 8.0 package |

The binary version and FCV are related, but they are **not the same thing**.

A MongoDB server can run newer binaries while retaining an older FCV during a supported upgrade transition. This provides a compatibility window before committing to the newer version's features and data formats.

## 2. Check the current version and FCV

Run these commands on Ubuntu.

### Step 1: Check the installed binary version

Example output:

To check the running server, connect using `mongosh`:

Then execute:

### Step 2: Check the FCV

Example output:

### Step 3: Check the server process and package

These checks establish the starting point before any upgrade.

---

## 3. MongoDB upgrade paths and compatibility rules

### 3.1 Major-version upgrades

A major-version upgrade changes the major release series, such as 6.0 to 7.0 or 7.0 to 8.0.

For MongoDB 8.0, the supported major-version path is:

**MongoDB 6.0**

**MongoDB 7.0**

**MongoDB 8.0**

Each major upgrade requires its own preparation, compatibility checks, and upgrade procedure.

If you are upgrading from MongoDB 6.0 to 8.0, upgrade to 7.0 first. You cannot skip the required intermediate major release. ([mongodb.com][1])

### 3.2 Minor-version and patch-version upgrades

Consider these examples:

* `8.0.1 → 8.0.4`: patch release upgrade.
* `8.0 → 8.2`: minor release upgrade.
* `7.0 → 8.0`: major release upgrade.

The exact permitted path depends on the release and deployment type. MongoDB 8.2 and later also have additional minor-release upgrade considerations for on-premises deployments. Always consult the upgrade guide for the exact target release. ([mongodb.com][2], [mongodb.com][3])

### 3.3 Pre-upgrade checklist

Before upgrading, check the following:

| Check          | What to verify                                              |
| -------------- | ----------------------------------------------------------- |
| Source version | The current release is supported as a source                |
| FCV            | The deployment has the required FCV                         |
| Drivers        | Application drivers support the target release              |
| Compatibility  | Deprecated features and breaking changes have been reviewed |
| Replication    | All required members are healthy and caught up              |
| Backup         | A verified backup and recovery procedure exist              |
| Resources      | Disk space, memory, and CPU are sufficient                  |
| Testing        | The application has passed tests on the target release      |

For a production deployment, test the upgrade against a representative staging environment before touching production.

---

## 4. Feature Compatibility Version (FCV) management

FCV controls the compatibility behavior of certain MongoDB features. It is especially important when the new server binary supports features that are incompatible with the previous release.

### 4.1 How FCV works

Suppose you have MongoDB 7.0 and want to upgrade to 8.0.

**Before upgrade**

## Binary 7.0 · FCV 7.0

**Upgrade the binaries**

## Binary 8.0 · FCV 7.0

New binaries, old compatibility behavior

**After validation and burn-in**

## Binary 8.0 · FCV 8.0

New-version compatibility features enabled

The intermediate stage lets you validate the new binaries before enabling the newer compatibility behavior. This can make recovery planning easier, but it does **not** guarantee that a binary downgrade will be supported.

### 4.2 Check FCV

Connect to the appropriate MongoDB instance using `mongosh`.

Example:

### 4.3 Change FCV

After completing a supported upgrade from 7.0 to 8.0, validating the deployment, and confirming the target release's requirements, run the following on the replica set primary or, for a sharded cluster, through `mongos`:

The `confirm: true` parameter is required by the relevant modern upgrade procedures. The operation may perform internal writes and wait for the required replication acknowledgements. ([mongodb.com][1], [mongodb.com][3])

**Important:** Do not change FCV simply because the newer binary has been installed. Complete the prescribed binary upgrade first.

### 4.4 When should you change FCV?

A good production practice is to separate the binary upgrade from the FCV change.

1. Upgrade the binaries on all required members.
2. Verify the cluster is healthy.
3. Run application smoke tests and monitor errors, latency, replication lag, and logs.
4. Allow a suitable burn-in period.
5. Change FCV when the deployment is stable and rollback requirements have been reviewed.

### 4.5 FCV and rollback

FCV is not a substitute for a backup or rollback plan.

| Situation                             | Recommended approach                                                                  |
| ------------------------------------- | ------------------------------------------------------------------------------------- |
| New binaries installed, FCV unchanged | Validate the exact release's supported recovery options                               |
| FCV changed to the new version        | Treat the upgrade as more committed; assess backward-incompatible data and features   |
| Downgrade required                    | Follow the specific downgrade guide; verify binary, FCV, and data-format restrictions |
| Upgrade failure                       | Preserve logs and data files; do not delete or recreate database directories          |

MongoDB's downgrade rules vary by release and edition. For example, MongoDB 8.2 has restrictions on binary downgrades and FCV changes. Never assume that reinstalling an older package will safely reverse an upgrade. ([mongodb.com][2])

---

## 5. Rolling upgrades for replica sets

A **rolling upgrade** upgrades one replica set member at a time, allowing the remaining members to continue serving requests.

A typical three-member replica set looks like this:

**Node 1**

(PRIMARY)

Current writer

**Node 2**

(SECONDARY)

Upgrade first

**Node 3**

(SECONDARY)

Upgrade next

---

Upgrade one secondary, wait for it to recover, upgrade the other secondary, then perform a controlled primary stepdown and upgrade the former primary.

### 5.1 Pre-upgrade health checks

Connect to the replica set:

Check the member states:

Check replication lag and member health:

Check the replica set configuration:

Before proceeding, ensure:

* The primary is healthy.
* All required data-bearing members are healthy.
* Secondaries are sufficiently caught up.
* A majority of voting members can remain available during each restart.
* A recent backup has been verified.

### 5.2 Upgrade the first secondary

On the first secondary, stop the service:

Upgrade the MongoDB package using the approved repository and package version for the target release. Restart the service:

Verify that it has recovered:

From `mongosh`:

Proceed only after the member returns to `SECONDARY` and replication is healthy.

Repeat the procedure for the other secondary.

> **Production caution:** Stopping a secondary temporarily reduces redundancy. Never upgrade multiple members simultaneously unless the exact procedure explicitly permits it.

### 5.3 Upgrade the primary

Once all secondaries are upgraded and healthy, connect to the current primary and initiate a controlled stepdown:

This asks the primary to step down for approximately 60 seconds, allowing an election to occur. Clients may briefly encounter interrupted writes or transient errors while the new primary is elected.

Check the new primary:

Once a different member is `PRIMARY`, upgrade the former primary using the same stop, package-upgrade, and restart procedure.

Finally, verify all members are healthy and the set has a primary.

### 5.4 Final validation

Also verify application reads and writes, connection pools, monitoring, and replication lag.

The exact sequence above follows the general rolling-upgrade pattern in MongoDB's version-specific replica set upgrade guides. ([mongodb.com][3], [mongodb.com][4])

---

## 6. Rolling upgrades for sharded clusters

A sharded cluster has more components than a replica set. Each component must be upgraded in the correct order.

**Application**

**`mongos` routers**

Config server replica set (CSRS)

Config 1

Config 2

Config 3

Shard 1

Replica set

Shard 2

Replica set

### 6.1 Correct upgrade order

For a MongoDB 7.0-to-8.0 sharded-cluster upgrade, the prescribed order is:

1. Upgrade the config server replica set.
2. Upgrade the shard replica sets, one shard at a time.
3. Upgrade all `mongos` routers.
4. Re-enable the balancer if it was disabled.
5. Validate the cluster and change FCV when ready.

This order is important because the routers and data-bearing components must remain compatible during the transition. ([mongodb.com][1])

### 6.2 Step 1: Disable the balancer

Connect to `mongos`:

Run:

Check the state:

Expected:

This prevents new balancing activity during the upgrade. In-progress migrations may need to finish before the balancer stops.

### 6.3 Step 2: Upgrade config servers

If the config server replica set has three members:

1. Upgrade one secondary.
2. Wait for it to return to `SECONDARY`.
3. Upgrade the other secondary.
4. Step down the primary.
5. Upgrade the former primary.

Keep the config server replica set healthy throughout.

### 6.4 Step 3: Upgrade each shard

For each shard replica set:

1. Upgrade its secondaries individually.
2. Confirm replication and member health.
3. Step down its primary.
4. Upgrade the former primary.
5. Confirm that the shard is healthy before moving to the next shard.

Do not upgrade all shards simultaneously.

### 6.5 Step 4: Upgrade `mongos`

Upgrade and restart each `mongos` router individually. Keep compatible routers available to serve application traffic while the others are restarted.

### 6.6 Step 5: Restore balancing and change FCV

After the cluster is healthy:

Verify the state:

Once the entire cluster has passed validation and the burn-in period, set FCV through `mongos`:

Use the FCV command only when the source and target release instructions permit it. ([mongodb.com][1])

---

## 7. Upgrading a standalone MongoDB server

A standalone MongoDB server has no replica-set members to take over its workload. Therefore, **restarting it for an in-place upgrade causes database unavailability**.

**Important distinction**

* Standalone, in-place upgrade: downtime is expected.
* Replica set, rolling upgrade: downtime can often be avoided at the database-service level.
* Standalone with a high-availability migration plan: downtime can be minimized by introducing redundancy before upgrading.

### 7.1 In-place upgrade procedure

**Step 1 — Check the version and FCV**

**Step 2 — Take a verified backup**

For example, using `mongodump`:

This is a logical backup. For large databases, evaluate physical backups or filesystem snapshots as appropriate, and verify your restore procedure.

**Step 3 — Upgrade the package**

Use the approved MongoDB repository and exact target package. For example, on a system using the MongoDB 8.0 repository:

Review the candidate version before installing it.

**Step 4 — Restart and validate**

Then:

Review the logs:

**Step 5 — Validate FCV and application behavior**

Check the FCV, collections, indexes, application connectivity, and representative read/write operations. Change FCV only when the target release's upgrade guide says it is appropriate.

### 7.2 How can you avoid downtime for a standalone?

If availability is a strict requirement, plan a migration to a replica set or another suitable high-availability architecture before the upgrade. This introduces additional infrastructure and requires careful data synchronization and application cutover planning.

Simply upgrading a standalone package in place cannot provide zero downtime.

---

## 8. Linux patch management without downtime

Linux patch management covers two distinct activities:

* **Operating-system patching:** Kernel, OpenSSL, system libraries, security fixes, and other Ubuntu packages.
* **MongoDB patching:** Updating the `mongodb-org` packages and related binaries.

They must be coordinated because a Linux reboot or MongoDB restart can interrupt database availability.

### 8.1 Inspect available updates

On Ubuntu:

List upgradeable packages:

Check MongoDB package versions:

Review the running kernel:

Check whether a reboot is indicated:

This flag is a useful indicator, not a complete guarantee that a reboot is unnecessary.

### 8.2 Why patching every server together is dangerous

Suppose a replica set has three voting members.

| Action                  | Possible consequence                           |
| ----------------------- | ---------------------------------------------- |
| Reboot one secondary    | Reduced redundancy while it is unavailable     |
| Reboot both secondaries | Primary may lose voting majority and step down |
| Reboot the primary      | Election and brief write interruption          |
| Reboot all three nodes  | Complete database outage                       |

A rolling patch strategy handles one member at a time and verifies recovery before proceeding.

### 8.3 Recommended patching sequence

1. **Pre-check:** Confirm backups, replication health, sufficient disk space, and a healthy majority of voting members.
2. **Patch one secondary:** Apply approved OS updates and reboot if required.
3. **Validate recovery:** Confirm the MongoDB service is running, the node is `SECONDARY`, and replication has caught up.
4. **Patch the next secondary:** Repeat only after the previous node is healthy.
5. **Step down the primary:** Perform a controlled election after the patched secondaries are healthy.
6. **Patch the former primary:** Apply updates and reboot if necessary.
7. **Final validation:** Confirm the primary, member health, replication lag, application connectivity, and monitoring alerts.

### 8.4 Ubuntu commands for patching a secondary

First inspect available updates:

For an OS-only maintenance window, avoid unintentionally changing MongoDB versions. Review and explicitly select the packages you intend to update.

Apply the approved OS package updates, for example:

If a reboot is required:

After the server returns:

Check the MongoDB log:

Check replica-set status from `mongosh`:

Do not patch the next member until the current member is healthy and sufficiently caught up.

**Note:** `apt-get upgrade` behavior depends on package dependencies and the system's package configuration. Inspect the proposed changes before confirming. For a controlled maintenance workflow, use an approved patch list rather than blindly upgrading every package.

---

## 9. MongoDB patching versus OS patching

| Aspect               | MongoDB package update                                                                | Ubuntu OS update                                                      |
| -------------------- | ------------------------------------------------------------------------------------- | --------------------------------------------------------------------- |
| Example              | MongoDB 8.0.4 to a later 8.0 patch                                                    | OpenSSL or system-library update                                      |
| Typical impact       | Requires MongoDB process restart                                                      | May require service restarts or a system reboot                       |
| Main risk            | Version incompatibility, changed behavior, startup issues                             | Kernel, library, driver, or dependency incompatibility                |
| FCV impact           | FCV usually remains unchanged for a patch update, but verify the release requirements | Normally unchanged                                                    |
| Recommended approach | Test release and driver compatibility; roll through members                           | Stage updates and roll through hosts                                  |
| Validation           | Binary version, FCV, replica-set health, application tests                            | Kernel version, service health, replica-set health, application tests |

A MongoDB package update does not automatically imply that FCV should change.

### 9.1 Prevent accidental MongoDB upgrades during OS maintenance

Before applying OS updates, inspect whether MongoDB packages are included in the proposed transaction.

If MongoDB must remain at a particular version temporarily, you can hold its packages:

Check holds:

Release a hold when the approved MongoDB upgrade is ready:

Package names vary by installation and release. Confirm which packages are installed before applying these commands. Holding packages is a temporary change-control measure, not a substitute for security updates.

---

## 10. Achieving minimal downtime: important limitations

A rolling upgrade reduces downtime, but it does not guarantee that every application operation will complete uninterrupted.

Potential causes of interruption include:

* Primary elections.
* Client connection pools that do not recover automatically.
* Long-running operations or transactions interrupted by a stepdown.
* Insufficient replica-set voting majority.
* Replication lag or an unhealthy secondary.
* Driver incompatibility with the new release.
* Operating-system changes that require all services to restart.

For applications requiring high availability, configure replica-set-aware connection strings, appropriate retry behavior, health checks, and suitable write concerns. Test failure and election scenarios before production maintenance.

## 11. Production upgrade checklist

## Maintenance readiness

/ complete

[Reset checklist] [Copy checklist]

## 12. Official MongoDB documentation

* [MongoDB release notes and versioning](https://www.mongodb.com/docs/manual/release-notes/) — determine the exact supported upgrade path.
* [Upgrade a replica set to MongoDB 8.0](https://www.mongodb.com/docs/v8.0/release-notes/8.0-upgrade-replica-set/).
* [Upgrade a sharded cluster to MongoDB 8.0](https://www.mongodb.com/docs/v8.0/release-notes/8.0-upgrade-sharded-cluster/).
* [FCV command reference](https://www.mongodb.com/docs/manual/reference/command/setFeatureCompatibilityVersion/).

**The main principle:** upgrade binaries first using the release-specific procedure, validate the deployment, and change FCV only when ready. For high availability, patch and upgrade one replica-set member at a time, maintaining a healthy voting majority throughout.

[1]: https://www.mongodb.com/docs/v8.0/release-notes/8.0-upgrade-sharded-cluster/ "Database Manual v8.0 - MongoDB Docs"
[2]: https://www.mongodb.com/docs/manual/release-notes/8.2/ "Database Manual - MongoDB Docs"
[3]: https://www.mongodb.com/docs/manual/release-notes/8.3-upgrade-from-8.0-replica-set/ "Database Manual - MongoDB Docs"
[4]: https://www.mongodb.com/docs/manual/release-notes/8.3-upgrade-replica-set/ "Database Manual - MongoDB Docs"
