
## 1. What is RBAC in MongoDB?

MongoDB RBAC follows this basic model:

```text
                 ┌─────────────────────┐
                 │       USER          │
                 │   alice / bob / dba │
                 └──────────┬──────────┘
                            │
                            │ assigned
                            ▼
                 ┌─────────────────────┐
                 │        ROLE         │
                 │ read / readWrite /  │
                 │ dbAdmin / custom    │
                 └──────────┬──────────┘
                            │
                            │ grants
                            ▼
                 ┌─────────────────────┐
                 │    PRIVILEGES       │
                 │ find                 │
                 │ insert               │
                 │ update               │
                 │ remove               │
                 │ createIndex          │
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │     RESOURCES       │
                 │ database            │
                 │ collection           │
                 └─────────────────────┘
```

The important idea is:

> **Users don't normally receive permissions directly. They receive roles, and roles contain privileges.**

---

# 2. Authentication vs Authorization

This distinction is extremely important when teaching MongoDB security.

### Authentication

Answers:

> **Who are you?**

For example:

```text
username = appuser
password = ********
```

MongoDB verifies the credentials.

### Authorization

Answers:

> **What are you allowed to do?**

For example:

```text
appuser
   |
   └── readWrite
          |
          └── salesDB
```

Therefore:

```text
Authentication → Identity
Authorization  → Permissions
RBAC           → How permissions are organized
```

---

# 3. MongoDB RBAC hierarchy

MongoDB security can be thought of as:

```text
User
  │
  ├── Role 1
  │     ├── Privilege
  │     └── Privilege
  │
  └── Role 2
        ├── Privilege
        └── Privilege
```

A role contains:

### Privileges

A privilege consists of:

```text
Action + Resource
```

For example:

```text
Action:
    find

Resource:
    salesDB.customers
```

means:

> The user can execute `find` against the `customers` collection in `salesDB`.

---

# 4. MongoDB built-in roles

MongoDB provides many predefined roles.

Some commonly used ones are:

| Role             | Purpose                                    |
| ---------------- | ------------------------------------------ |
| `read`           | Read data                                  |
| `readWrite`      | Read and modify data                       |
| `dbAdmin`        | Database administration                    |
| `userAdmin`      | Manage users and roles                     |
| `dbOwner`        | Combination of several database privileges |
| `clusterMonitor` | Monitoring                                 |
| `clusterManager` | Cluster management                         |
| `hostManager`    | Host management                            |
| `backup`         | Backup operations                          |
| `restore`        | Restore operations                         |
| `root`           | Full administrative privileges             |

---

# 5. Example: `read` role

Suppose we have:

```text
salesDB
   |
   ├── customers
   ├── orders
   └── products
```

Create:

```text
user: reportingUser
role: read
database: salesDB
```

The user can do:

```javascript
db.customers.find()
db.orders.find()
db.products.find()
```

But cannot do:

```javascript
db.customers.insertOne(...)
```

or:

```javascript
db.customers.deleteOne(...)
```

---

# 6. Example: `readWrite`

Suppose:

```text
applicationUser
       |
       └── readWrite
              |
              └── salesDB
```

The application can perform operations such as:

```javascript
find
insert
update
delete
```

but does not automatically receive every administrative capability.

This is an important security principle:

> **Give users the minimum permissions required for their job.**

This is called **Principle of Least Privilege**.

---

# 7. MongoDB RBAC setup — complete process

Let's create a practical setup on Ubuntu.

We'll use:

```text
MongoDB
    |
    └── salesDB
          |
          ├── customers
          └── orders

Users:

adminUser
    └── administrative privileges

salesApp
    └── readWrite on salesDB

reportUser
    └── read on salesDB
```

---

# Step 1 — Check MongoDB service

On Ubuntu:

```bash
sudo systemctl status mongod
```

If it isn't running:

```bash
sudo systemctl start mongod
```

Enable it at startup:

```bash
sudo systemctl enable mongod
```

---

# Step 2 — Connect to MongoDB

Initially, if authorization has not yet been enabled:

```bash
mongosh
```

You should see something like:

```text
test>
```

Check the server:

```javascript
db.version()
```

For MongoDB 8.x:

```text
8.x.x
```

---

# Step 3 — Create the first administrator

This is an important step.

Switch to the `admin` database:

```javascript
use admin
```

Create an administrative user:

```javascript
db.createUser({
    user: "adminUser",
    pwd: "Admin@12345",
    roles: [
        { role: "root", db: "admin" }
    ]
})
```

You should get:

```text
Successfully added user
```

---

# 8. Why is the user created in `admin`?

This is a common point of confusion.

When you execute:

```javascript
use admin
```

you are selecting the database where the **user definition is stored**.

The role:

```javascript
{ role: "root", db: "admin" }
```

means the user receives the `root` role defined in the `admin` database.

The `admin` database is special because it contains system-wide administration information.

---

# 9. Enable authorization

Now edit MongoDB configuration:

```bash
sudo nano /etc/mongod.conf
```

Find:

```yaml
security:
```

or add:

```yaml
security:
  authorization: enabled
```

For example:

```yaml
storage:
  dbPath: /var/lib/mongodb

systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true

security:
  authorization: enabled
```

Save the file.

---

# 10. Restart MongoDB

```bash
sudo systemctl restart mongod
```

Check:

```bash
sudo systemctl status mongod
```

Also check the log:

```bash
sudo tail -50 /var/log/mongodb/mongod.log
```

---

# 11. Connect using the administrator

Now authentication is enabled.

Connect:

```bash
mongosh
```

You will not be authenticated as the administrator simply because you are on the local machine.

Use:

```bash
mongosh -u adminUser -p --authenticationDatabase admin
```

MongoDB prompts:

```text
Enter password:
```

Enter:

```text
Admin@12345
```

Alternatively:

```bash
mongosh \
  --username adminUser \
  --password \
  --authenticationDatabase admin
```

This is preferable because it doesn't expose the password in shell history.

---

# 12. Verify authentication

Inside `mongosh`:

```javascript
db.runCommand({
    connectionStatus: 1
})
```

You should see authenticated users, something similar to:

```text
authenticatedUsers:
[
    {
        user: "adminUser",
        db: "admin"
    }
]
```

And:

```text
authenticatedUserRoles:
[
    {
        role: "root",
        db: "admin"
    }
]
```

---

# 13. Create application database

Now create:

```javascript
use salesDB
```

Insert sample data:

```javascript
db.customers.insertMany([
    {
        name: "Ravi",
        city: "Chennai"
    },
    {
        name: "Priya",
        city: "Bangalore"
    }
])
```

Create orders:

```javascript
db.orders.insertOne({
    customer: "Ravi",
    amount: 5000
})
```

---

# 14. Create an application user

We want:

```text
salesApp
     |
     └── readWrite
            |
            └── salesDB
```

Execute:

```javascript
use salesDB
```

Then:

```javascript
db.createUser({
    user: "salesApp",
    pwd: "SalesApp@12345",
    roles: [
        {
            role: "readWrite",
            db: "salesDB"
        }
    ]
})
```

---

# 15. Create a reporting user

We want:

```text
reportUser
     |
     └── read
            |
            └── salesDB
```

Execute:

```javascript
db.createUser({
    user: "reportUser",
    pwd: "Report@12345",
    roles: [
        {
            role: "read",
            db: "salesDB"
        }
    ]
})
```

---

# 16. Test `salesApp`

Exit:

```javascript
exit
```

Connect:

```bash
mongosh -u salesApp -p --authenticationDatabase salesDB
```

Then:

```javascript
use salesDB
```

Test reading:

```javascript
db.customers.find()
```

Should work.

Test inserting:

```javascript
db.customers.insertOne({
    name: "Anil",
    city: "Mumbai"
})
```

Should work.

Test updating:

```javascript
db.customers.updateOne(
    { name: "Anil" },
    { $set: { city: "Pune" } }
)
```

Should work.

Test deleting:

```javascript
db.customers.deleteOne({
    name: "Anil"
})
```

Should work.

---

# 17. Test `reportUser`

Exit:

```javascript
exit
```

Connect:

```bash
mongosh -u reportUser -p --authenticationDatabase salesDB
```

Then:

```javascript
use salesDB
```

Read:

```javascript
db.customers.find()
```

Works.

Now try:

```javascript
db.customers.insertOne({
    name: "Test",
    city: "Delhi"
})
```

You should get an authorization error similar to:

```text
MongoServerError:
not authorized on salesDB to execute command
```

This demonstrates RBAC.

---

# 18. Visualizing the setup

Your final configuration looks like this:

```text
                       MongoDB
                          │
                 ┌────────┴────────┐
                 │                 │
             admin DB          salesDB
                 │                 │
                 │          ┌──────┴──────┐
                 │          │             │
                 │      customers       orders
                 │
            adminUser
                 │
               root
                 │
          Full administration


salesApp
   │
   └── readWrite
          │
          └── salesDB


reportUser
   │
   └── read
         │
         └── salesDB
```

---

# 19. Checking users

As an administrator:

```javascript
use admin
```

You can inspect users:

```javascript
db.getUsers()
```

For a particular database:

```javascript
use salesDB
db.getUsers()
```

You might see:

```text
[
  {
    user: "salesApp",
    roles: [
      {
        role: "readWrite",
        db: "salesDB"
      }
    ]
  },
  {
    user: "reportUser",
    roles: [
      {
        role: "read",
        db: "salesDB"
      }
    ]
  }
]
```

---

# 20. Checking roles

To see the privileges associated with a role:

```javascript
use admin

db.getRole(
    "readWrite",
    {
        showPrivileges: true
    }
)
```

You can also examine inherited roles:

```javascript
db.getRole(
    "readWrite",
    {
        showPrivileges: true,
        showBuiltinRoles: true
    }
)
```

---

# 21. User can have multiple roles

A user doesn't have to have just one role.

For example:

```javascript
db.createUser({
    user: "developer",
    pwd: "Dev@12345",
    roles: [
        {
            role: "readWrite",
            db: "salesDB"
        },
        {
            role: "read",
            db: "reportingDB"
        }
    ]
})
```

Now:

```text
developer
   │
   ├── readWrite → salesDB
   │
   └── read      → reportingDB
```

So the same user has different permissions on different databases.

---

# 22. Custom roles

Built-in roles are convenient, but sometimes they provide **more permissions than necessary**.

Suppose your application should only be able to:

```text
find
insert
update
```

but should **not delete documents**.

You could create a custom role.

For example:

```javascript
use salesDB

db.createRole({
    role: "salesApplicationRole",

    privileges: [
        {
            resource: {
                db: "salesDB",
                collection: "orders"
            },
            actions: [
                "find",
                "insert",
                "update"
            ]
        }
    ],

    roles: []
})
```

Then create a user:

```javascript
db.createUser({
    user: "orderApp",
    pwd: "OrderApp@12345",
    roles: [
        {
            role: "salesApplicationRole",
            db: "salesDB"
        }
    ]
})
```

Now:

```text
orderApp
   │
   └── salesApplicationRole
             │
             ├── find
             ├── insert
             └── update
```

No:

```text
delete
```

permission.

---

# 23. Role inheritance

MongoDB roles can inherit other roles.

For example:

```text
customRole
    │
    ├── privileges
    │
    └── inherited roles
             │
             └── read
```

You can create a role that inherits another role:

```javascript
db.createRole({
    role: "salesManager",
    privileges: [
        {
            resource: {
                db: "salesDB",
                collection: "orders"
            },
            actions: [
                "find",
                "insert",
                "update"
            ]
        }
    ],
    roles: [
        {
            role: "read",
            db: "reportingDB"
        }
    ]
})
```

So the role can combine permissions from multiple sources.

---

# 24. Role → Privilege → Resource

This is one of the most important concepts for MongoDB DBAs.

Think of it as:

```text
                 ROLE
                  │
                  ▼
             PRIVILEGES
                  │
             ┌────┴────┐
             │         │
          ACTION     RESOURCE
             │         │
          find       salesDB
          insert     orders
          update
```

For example:

```text
Role:
    orderManager

Privilege:
    update

Resource:
    salesDB.orders
```

Meaning:

> `orderManager` allows the user to update documents in `salesDB.orders`.

---

# 25. Database-level vs collection-level access

This distinction is very useful.

### Database-level

```javascript
{
    role: "readWrite",
    db: "salesDB"
}
```

means access across the database's collections, according to that role's privileges.

### Collection-level

A custom role can target:

```text
salesDB.orders
```

without giving the same permission to:

```text
salesDB.customers
```

For example:

```text
salesDB
│
├── customers
│
├── orders        ← allowed
│
└── payments
```

A custom role could restrict access to:

```text
salesDB.orders
```

only.

---

# 26. Important DBA principle — don't use `root` for applications

Avoid this:

```text
Application
     │
     └── root
```

Instead:

```text
Application
     │
     └── application-specific role
                 │
                 └── minimum required privileges
```

For example:

```text
Order Service
     │
     └── orderServiceUser
             │
             └── readWrite
                    │
                    └── orderDB
```

The application shouldn't have:

```text
root
dbOwner
userAdminAnyDatabase
```

unless there is a very specific administrative requirement.

---

# 27. Recommended production architecture

For a production environment, I would typically structure users like this:

```text
                         MongoDB
                            │
       ┌────────────────────┼─────────────────────┐
       │                    │                     │
       ▼                    ▼                     ▼
 Application            Reporting             DBA
       │                    │                     │
       ▼                    ▼                     ▼
 appUser              reportUser            dbaUser
       │                    │                     │
       ▼                    ▼                     ▼
 readWrite               read                 DBA roles
       │                    │
       ▼                    ▼
 applicationDB          reportingDB
```

And avoid sharing the same account among:

```text
developers
applications
DBAs
reporting tools
monitoring tools
```

Each should have its own identity and appropriate role.

---

# 28. Complete setup flow to remember

For a MongoDB DBA, the complete process is:

```text
1. Start MongoDB
       ↓
2. Connect to MongoDB
       ↓
3. Create first administrator
       ↓
4. Enable authorization
       ↓
5. Restart mongod
       ↓
6. Authenticate as administrator
       ↓
7. Create application database
       ↓
8. Create appropriate users
       ↓
9. Assign built-in/custom roles
       ↓
10. Test permissions
       ↓
11. Verify users and roles
       ↓
12. Apply least privilege
```

### Key commands

```bash
# Check MongoDB
sudo systemctl status mongod

# Connect
mongosh

# Connect as administrator
mongosh -u adminUser -p --authenticationDatabase admin
```

MongoDB:

```javascript
// Create admin
use admin

db.createUser({
    user: "adminUser",
    pwd: "Admin@12345",
    roles: [
        { role: "root", db: "admin" }
    ]
})
```

Enable:

```yaml
security:
  authorization: enabled
```

Create application user:

```javascript
use salesDB

db.createUser({
    user: "salesApp",
    pwd: "SalesApp@12345",
    roles: [
        { role: "readWrite", db: "salesDB" }
    ]
})
```

Create reporting user:

```javascript
db.createUser({
    user: "reportUser",
    pwd: "Report@12345",
    roles: [
        { role: "read", db: "salesDB" }
    ]
})
```

Verify:

```javascript
db.getUsers()
```

and:

```javascript
db.runCommand({
    connectionStatus: 1
})
```

---

## One important security point

 **RBAC is only one layer of MongoDB security**. A hardened deployment should combine:

```text
Authentication
       +
RBAC / Authorization
       +
TLS
       +
Network restrictions / firewall
       +
Encryption at rest
       +
Auditing
       +
Secure credential management
       +
Least privilege
```
 **Role-Based Access Control (RBAC)** has several forms and supporting mechanisms.

### 1. Role-Based Access Control (RBAC) — primary mechanism

MongoDB uses RBAC to assign permissions through roles:

```text
User
  │
  ▼
Role
  │
  ▼
Privileges
  │
  ├── Actions
  │    ├── find
  │    ├── insert
  │    ├── update
  │    └── delete
  │
  └── Resources
       ├── Database
       └── Collection
```

For example:

```javascript
{
    user: "reportUser",
    roles: [
        { role: "read", db: "salesDB" }
    ]
}
```

The user can read `salesDB`, but cannot modify it.

---

## 2. Built-in roles

MongoDB provides predefined roles.

Some important ones are:

| Category                | Roles            | Purpose                                  |
| ----------------------- | ---------------- | ---------------------------------------- |
| Data access             | `read`           | Read data                                |
| Data access             | `readWrite`      | Read and modify data                     |
| Database administration | `dbAdmin`        | Database administration                  |
| User administration     | `userAdmin`      | Manage users and roles                   |
| Database ownership      | `dbOwner`        | Full administrative access to a database |
| Cluster monitoring      | `clusterMonitor` | Monitor cluster                          |
| Cluster management      | `clusterManager` | Manage cluster                           |
| Backup                  | `backup`         | Perform backup operations                |
| Restore                 | `restore`        | Restore operations                       |
| Host management         | `hostManager`    | Manage server/host                       |
| Global administration   | `root`           | Broadest built-in privileges             |

Example:

```javascript
db.createUser({
    user: "appUser",
    pwd: "StrongPassword",
    roles: [
        {
            role: "readWrite",
            db: "salesDB"
        }
    ]
})
```

---

# 3. Custom roles

When built-in roles provide too many permissions, MongoDB allows you to create **custom roles**.

For example, suppose an application should be able to:

```text
orders collection:
    find
    insert
    update

but NOT:
    delete
```

You can create:

```javascript
db.createRole({
    role: "orderApplicationRole",

    privileges: [
        {
            resource: {
                db: "salesDB",
                collection: "orders"
            },
            actions: [
                "find",
                "insert",
                "update"
            ]
        }
    ],

    roles: []
})
```

Then:

```javascript
db.createUser({
    user: "orderApp",
    pwd: "StrongPassword",
    roles: [
        {
            role: "orderApplicationRole",
            db: "salesDB"
        }
    ]
})
```

This provides much finer-grained authorization.

---

# 4. Role inheritance

MongoDB roles can inherit other roles.

For example:

```text
salesManager
     │
     ├── readWrite → salesDB
     │
     └── read → reportingDB
```

A custom role can contain other roles:

```javascript
db.createRole({
    role: "salesManager",
    privileges: [],
    roles: [
        {
            role: "readWrite",
            db: "salesDB"
        },
        {
            role: "read",
            db: "reportingDB"
        }
    ]
})
```

So authorization can be composed from multiple roles.

---

# 5. Collection-level authorization

MongoDB can restrict privileges to a particular collection using custom roles.

For example:

```text
salesDB
│
├── customers
├── orders       ← allowed
└── payments
```

A custom privilege can target:

```javascript
{
    resource: {
        db: "salesDB",
        collection: "orders"
    },
    actions: ["find", "insert", "update"]
}
```

This is more restrictive than simply giving the user `readWrite` on the whole database.

---

# 6. Action-level authorization

MongoDB authorization ultimately works with **actions**.

Examples include:

```text
find
insert
update
remove
createIndex
dropCollection
collStats
dbStats
listCollections
```

For example:

```text
Role
  │
  └── Privilege
         │
         ├── Action: find
         │
         └── Resource: salesDB.orders
```

This means:

> The user can perform `find` on `salesDB.orders`.

This is the most granular way of thinking about MongoDB authorization.

---

# 7. Database-level authorization

A role can provide permissions across a database.

Example:

```javascript
{
    role: "readWrite",
    db: "salesDB"
}
```

Conceptually:

```text
salesApp
   │
   └── readWrite
          │
          └── salesDB
                │
                ├── customers
                ├── orders
                └── products
```

---

# 8. Cluster-level authorization

Some roles provide privileges that aren't restricted to one application database.

For example:

```text
clusterMonitor
clusterManager
hostManager
backup
restore
```

These are useful for DBAs, monitoring systems, backup tools, and cluster-management operations.

For example:

```javascript
db.createUser({
    user: "monitoringUser",
    pwd: "StrongPassword",
    roles: [
        {
            role: "clusterMonitor",
            db: "admin"
        }
    ]
})
```

This is preferable to giving a monitoring application `root`.

---

# 9. User administration authorization

MongoDB also separates **data access** from **user/role administration**.

For example:

```text
userAdmin
```

allows a user to manage users and roles within its permitted scope.

There are also broader administrative roles such as:

```text
userAdminAnyDatabase
```

which is significantly more powerful.

This is why you should avoid casually giving developers or applications administrative roles.

---

# 10. LDAP authorization

MongoDB Enterprise can integrate with **LDAP**.

There are two separate concepts that are often confused:

```text
LDAP Authentication
        +
LDAP Authorization
```

LDAP can be used to authenticate users, while MongoDB can use LDAP groups/identity information to determine authorization.

Conceptually:

```text
User
 │
 ▼
LDAP / Active Directory
 │
 ▼
LDAP Group
 │
 ▼
MongoDB Role Mapping
 │
 ▼
MongoDB Privileges
```

For example:

```text
AD User
   │
   └── MongoDB-Reporting-Users
             │
             ▼
        MongoDB read role
             │
             ▼
          salesDB
```

This is particularly useful in enterprise environments where user/group management is centralized.

---

# 11. X.509 certificate-based identity + authorization

MongoDB Enterprise also supports **X.509 authentication**.

The certificate establishes the user's identity:

```text
Client
  │
  │ X.509 certificate
  ▼
MongoDB
  │
  ▼
Authenticated identity
  │
  ▼
MongoDB roles
  │
  ▼
Privileges
```

The important distinction is:

> **X.509 primarily establishes identity; MongoDB authorization still determines what that identity can do.**

---

# 12. Key point: Authentication mechanisms are not authorization mechanisms

This is especially important when studying MongoDB security.

You may have:

| Authentication         | Authorization                     |
| ---------------------- | --------------------------------- |
| SCRAM                  | MongoDB RBAC                      |
| LDAP                   | MongoDB RBAC / LDAP group mapping |
| Kerberos               | MongoDB RBAC                      |
| X.509                  | MongoDB RBAC                      |
| AWS IAM authentication | MongoDB authorization model       |

So don't think:

```text
LDAP = authorization
```

or:

```text
X.509 = authorization
```

Instead:

```text
              AUTHENTICATION
                    │
          "Who are you?"
                    │
                    ▼
               Identity
                    │
                    ▼
              AUTHORIZATION
                    │
          "What can you do?"
                    │
                    ▼
              MongoDB Roles
                    │
                    ▼
                Privileges
```

---

# 13. MongoDB's authorization model in one picture

```text
                         MongoDB Security
                               │
                 ┌─────────────┴─────────────┐
                 │                           │
           Authentication               Authorization
          "Who are you?"              "What can you do?"
                 │                           │
       ┌─────────┼─────────┐                 │
       │         │         │                 ▼
     SCRAM     LDAP      X.509             RBAC
       │         │         │                 │
       │       Kerberos    │          ┌──────┼──────┐
       │                   │          │      │      │
       └──────────┬────────┘       Built-in Custom Inherited
                  │                  roles   roles   roles
                  ▼
              Identity
                                           │
                                           ▼
                                      Privileges
                                           │
                                  ┌────────┴────────┐
                                  │                 │
                                Action           Resource
                                  │                 │
                            find/insert/etc.    DB/collection
```

## The important takeaway

MongoDB doesn't have several completely independent authorization systems in the way it has several authentication mechanisms.

**The core MongoDB authorization mechanism is RBAC**, and it can be implemented using:

1. **Built-in roles** — `read`, `readWrite`, `dbAdmin`, etc.
2. **Custom roles** — precisely defined privileges.
3. **Role inheritance** — roles can include other roles.
4. **Database-level privileges**.
5. **Collection-level privileges**.
6. **Action-level privileges**.
7. **Cluster/administrative roles**.
8. **LDAP group/role mapping** in MongoDB Enterprise deployments.

