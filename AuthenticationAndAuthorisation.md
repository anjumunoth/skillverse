**MongoDB security** is best understood as a **layered security model**. MongoDB doesn't rely on just usernames and passwords; it provides multiple mechanisms to protect the database, data, communication, and administrative operations.

A useful way to visualize it is:

```text
                    ┌─────────────────────────────┐
                    │       MongoDB Security      │
                    └──────────────┬──────────────┘
                                   │
          ┌────────────────────────┼────────────────────────┐
          │                        │                        │
          ▼                        ▼                        ▼
   Authentication             Authorization             Encryption
   "Who are you?"             "What can you do?"        "Can others read it?"
          │                        │                        │
          │                        │              ┌─────────┴─────────┐
          │                        │              │                   │
          ▼                        ▼              ▼                   ▼
     SCRAM / X.509             RBAC           At Rest            In Transit
     LDAP / OIDC              Roles           Encryption           TLS
                                                │
                                                ▼
                                          Encryption Keys
```

There are also additional security areas such as **auditing, network security, client-side field-level encryption, security hardening, and database activity monitoring**.

---

# 1. The major areas of MongoDB security

| Security Area                            | Main Question                            | MongoDB Mechanism                    |
| ---------------------------------------- | ---------------------------------------- | ------------------------------------ |
| **Authentication**                       | Who are you?                             | SCRAM, X.509, LDAP, OIDC             |
| **Authorization**                        | What are you allowed to do?              | RBAC                                 |
| **Encryption in Transit**                | Can someone read network traffic?        | TLS/SSL                              |
| **Encryption at Rest**                   | Can someone read database files?         | Encrypted storage engine             |
| **Client-Side / Field-Level Encryption** | Can MongoDB itself see sensitive fields? | CSFLE / Queryable Encryption         |
| **Auditing**                             | Who did what?                            | MongoDB Auditing                     |
| **Network Security**                     | Who can reach MongoDB?                   | Bind IP, firewall, network isolation |
| **Key Management**                       | Where are encryption keys stored?        | KMIP / KMS / key management systems  |
| **Operational Security**                 | Is MongoDB securely configured?          | Hardening, least privilege, patching |

**Authentication = proving the identity of a client, user, or MongoDB server.**

MongoDB's major authentication mechanisms are:

| Mechanism               | What it uses                       | Typical use                                                |
| ----------------------- | ---------------------------------- | ---------------------------------------------------------- |
| **SCRAM-SHA-256**       | Username + password                | Most common application/user authentication                |
| **SCRAM-SHA-1**         | Username + password                | Older/legacy compatibility                                 |
| **X.509**               | Digital certificates               | Strong certificate-based authentication                    |
| **LDAP**                | Enterprise LDAP directory          | Centralized corporate identity                             |
| **OIDC**                | OpenID Connect / identity provider | Modern enterprise/cloud SSO                                |
| **AWS IAM**             | AWS identity credentials           | MongoDB deployments integrated with AWS                    |
| **Keyfile**             | Shared secret                      | MongoDB internal cluster authentication                    |
| **X.509 internal auth** | Certificates                       | Strong internal replica-set/sharded-cluster authentication |

There are two broad categories worth keeping separate:

```text
                    MongoDB Authentication
                           │
              ┌────────────┴────────────┐
              │                         │
              ▼                         ▼
       Client/User Auth          Internal Auth
              │                         │
       ┌──────┼──────┐             ┌─────┴─────┐
       │      │      │             │           │
     SCRAM   X.509  LDAP          Keyfile    X.509
                     │
                    OIDC
```

Let's go through them.

---

# 1. SCRAM Authentication

**SCRAM** stands for:

**Salted Challenge Response Authentication Mechanism**

This is the most common MongoDB username/password authentication mechanism.

MongoDB supports:

* **SCRAM-SHA-256**
* **SCRAM-SHA-1**

For modern deployments, **SCRAM-SHA-256** is generally the preferred choice.

---

## How SCRAM works

Suppose:

```text
Application
     |
     | username = appuser
     | password = ********
     |
     ▼
 MongoDB
```

A common misconception is that MongoDB simply receives:

```text
username + password
```

and compares the password directly.

That's not the basic idea of SCRAM.

Instead, there is a **challenge-response exchange**.

Simplified:

```text
Client                              MongoDB
  │                                    │
  │── Username ───────────────────────►│
  │                                    │
  │◄── Challenge + Salt ──────────────│
  │                                    │
  │── Client Proof ───────────────────►│
  │                                    │
  │                         Verify proof
  │                                    │
  │◄──────── Authentication OK ────────│
```

The server maintains credential material derived from the password rather than simply needing the plaintext password for authentication.

---

## Example

Create a user:

```javascript
use shop

db.createUser({
    user: "appuser",
    pwd: "StrongPassword123!",
    roles: [
        {
            role: "readWrite",
            db: "shop"
        }
    ]
})
```

Connect:

```bash
mongosh \
  --host localhost \
  --port 27017 \
  -u appuser \
  -p \
  --authenticationDatabase shop
```

You'll be prompted for the password.

---

# 2. SCRAM-SHA-256 vs SCRAM-SHA-1

| Feature              | SCRAM-SHA-1                    | SCRAM-SHA-256             |
| -------------------- | ------------------------------ | ------------------------- |
| Hash function        | SHA-1                          | SHA-256                   |
| Security             | Older                          | Stronger                  |
| Modern deployments   | Generally avoid where possible | Preferred                 |
| Legacy compatibility | Better                         | Less legacy compatibility |
| MongoDB support      | Yes                            | Yes                       |

For a new MongoDB 8.x deployment, you'd normally prefer:

```text
SCRAM-SHA-256
```

---

# 3. X.509 Authentication

X.509 is fundamentally different.

Instead of:

```text
username + password
```

you use:

```text
Digital Certificate
```

Conceptually:

```text
                Certificate Authority
                        │
              ┌─────────┴─────────┐
              │                   │
              ▼                   ▼
       Client Certificate   MongoDB Certificate
              │                   │
              ▼                   ▼
          Application           MongoDB
```

The certificate proves the identity of the connecting party.

---

# 4. How X.509 authentication works

Suppose an application has:

```text
client.pem
```

and MongoDB has:

```text
server.pem
```

A trusted CA signs the certificates.

```text
             CA
             │
       ┌─────┴─────┐
       │           │
       ▼           ▼
    Client       MongoDB
  Certificate   Certificate
```

During the TLS handshake, the certificates are exchanged/validated according to the configured TLS mode.

With **mutual TLS**, MongoDB can authenticate the client certificate as well.

```text
Application                         MongoDB
     │                                  │
     │──── Client Certificate ─────────►│
     │                                  │
     │◄─── MongoDB Certificate ────────│
     │                                  │
     │       Certificate validation     │
     │                                  │
     │◄────── Authentication OK ────────│
```

---

# 5. Why use X.509?

It is particularly useful when you don't want applications/services to manage passwords.

For example:

```text
Microservice A
     │
     │ Certificate
     ▼
MongoDB
```

You can revoke or rotate certificates through your PKI infrastructure.

This is very common in enterprise environments.

---

# 6. X.509 also has an important second use

X.509 isn't only for client authentication.

MongoDB can use X.509 certificates for **internal authentication between MongoDB members**.

For example:

```text
Replica Set

       PRIMARY
          │
    X.509 authentication
          │
     ┌────┴────┐
     ▼         ▼
 Secondary  Secondary
```

This is particularly attractive for highly secured production deployments.

---

# 7. LDAP Authentication

LDAP is useful when an organization already has a centralized identity directory.

For example:

```text
                    Corporate LDAP
                         │
               ┌─────────┼─────────┐
               │         │         │
               ▼         ▼         ▼
             Alice      Bob       Anju
               │         │         │
               └─────────┼─────────┘
                         │
                         ▼
                      MongoDB
```

Instead of maintaining separate MongoDB credentials for every user, authentication can be integrated with the organization's LDAP infrastructure.

For example, the organization might already have:

```text
username: anju
password: corporate-password
```

and the same identity can be used for MongoDB authentication.

---

# 8. Why LDAP is useful

Imagine an organization with:

```text
20,000 employees
```

You don't want DBAs creating:

```text
20,000 MongoDB users
```

Instead:

```text
Corporate Identity
       │
       ▼
     LDAP
       │
       ▼
    MongoDB
```

Centralized identity management becomes possible.

When an employee leaves:

```text
Employee disabled in corporate directory
              ↓
       MongoDB access affected
```

This is much easier to manage than manually changing database credentials everywhere.

---

# 9. LDAP vs SCRAM

|                                 | SCRAM   | LDAP                |
| ------------------------------- | ------- | ------------------- |
| Credentials stored/managed      | MongoDB | External directory  |
| Centralized corporate identity  | ❌       | ✅                   |
| Simple setup                    | ✅       | More complex        |
| Good for small deployments      | ✅       | Usually unnecessary |
| Enterprise identity integration | Limited | Excellent           |

---

# 10. OIDC Authentication

**OIDC = OpenID Connect**

OIDC is based on modern identity and authentication infrastructure.

Instead of MongoDB directly managing passwords, authentication can be delegated to an **Identity Provider (IdP)**.

Examples of identity providers include enterprise identity platforms such as:

* Microsoft Entra ID
* Okta
* other OIDC-compliant providers

Conceptually:

```text
                         Identity Provider
                                │
                       User authenticates
                                │
                                ▼
                         Identity Token
                                │
                                ▼
                              MongoDB
```

---

# 11. OIDC authentication flow

Simplified:

```text
User
 │
 │ Login
 ▼
Identity Provider
 │
 │ Authentication
 │
 ▼
Token
 │
 ▼
MongoDB
 │
 │ Validate identity
 ▼
Authenticated
```

This is particularly useful in environments where organizations already have centralized SSO.

---

# 12. OIDC vs LDAP

They solve somewhat similar enterprise problems but use different technologies.

|                       | LDAP                               | OIDC                             |
| --------------------- | ---------------------------------- | -------------------------------- |
| Technology            | Directory protocol                 | Identity/authentication protocol |
| Typical environment   | Traditional enterprise directories | Modern cloud/SSO                 |
| Tokens                | Not the core model                 | Yes                              |
| SSO integration       | Possible                           | Strong                           |
| Modern cloud identity | Less natural                       | Excellent                        |

---

# 13. AWS IAM authentication

For MongoDB deployments integrated with AWS, **AWS IAM authentication** can be used in supported MongoDB environments.

Instead of maintaining a MongoDB password:

```text
Application
     │
     │ AWS identity/credentials
     ▼
   MongoDB
```

The authentication decision can leverage AWS identity mechanisms.

Conceptually:

```text
AWS IAM
   │
   ├── IAM User
   ├── IAM Role
   └── IAM Policy
          │
          ▼
       MongoDB
```

This is especially useful for applications running in AWS.

For example:

```text
EC2 / ECS / EKS
       │
       │ IAM Role
       ▼
    MongoDB
```

This can reduce the need to embed database passwords in application configuration.

---

# 14. Keyfile authentication

Now we come to something slightly different.

**Keyfile authentication is primarily for internal authentication between MongoDB members.**

Suppose we have:

```text
Primary
   │
   ├────────► Secondary 1
   │
   └────────► Secondary 2
```

How does Secondary 1 know that the incoming connection really belongs to a trusted MongoDB member?

A shared keyfile can be used.

```text
             Shared Key
                  │
       ┌──────────┼──────────┐
       ▼          ▼          ▼
    Primary    Secondary1  Secondary2
```

Each MongoDB member has access to the same secret.

---

# 15. Example keyfile configuration

On Ubuntu, you might have:

```text
/etc/mongodb-keyfile
```

with appropriate restrictive permissions.

For example:

```bash
sudo chmod 400 /etc/mongodb-keyfile
sudo chown mongodb:mongodb /etc/mongodb-keyfile
```

Then MongoDB configuration can reference it:

```yaml
security:
  keyFile: /etc/mongodb-keyfile
```

The exact configuration should be designed consistently across all members.

---

# 16. Keyfile vs X.509 internal authentication

Both can authenticate MongoDB members.

|                                     | Keyfile       | X.509           |
| ----------------------------------- | ------------- | --------------- |
| Authentication                      | Shared secret | Certificates    |
| Setup complexity                    | Simple        | Higher          |
| Certificate infrastructure required | No            | Yes             |
| Enterprise PKI integration          | ❌             | ✅               |
| Common/simple deployments           | ✅             |                 |
| High-security environments          | Possible      | Often preferred |

Think of it as:

```text
Keyfile

Primary ───── Shared Secret ───── Secondary


X.509

Primary ───── Certificates ───── Secondary
       \                       /
        └────── Trusted CA ───┘
```

---

# 17. Authentication vs TLS — VERY important

This distinction often causes confusion.

**Authentication and TLS are not the same thing.**

Suppose you use:

```text
SCRAM
```

That answers:

> Who is the user?

TLS answers:

> Is the communication channel protected from interception?

So a secure connection may look like:

```text
                 TLS
Application ═══════════════════ MongoDB
      │                              │
      │      SCRAM authentication    │
      └──────────────────────────────┘
```

They work together.

---

# 18. Authentication vs Authorization

Another important distinction:

### Authentication

```text
"Who are you?"
```

Example:

```text
anju
```

### Authorization

```text
"What can Anju do?"
```

Example:

```text
anju
  |
  └── readWrite on shop
```

Therefore:

```text
Authentication
       ↓
Identify user
       ↓
Authorization
       ↓
Determine privileges
```

---

# 19. A complete MongoDB security flow

Imagine an application connecting to MongoDB:

```text
                    Application
                         │
                         │
                         ▼
                  ┌─────────────┐
                  │     TLS     │
                  └──────┬──────┘
                         │
                         ▼
                  Authentication
                         │
              ┌──────────┼──────────┐
              │          │          │
            SCRAM       X.509      OIDC
              │          │          │
              └──────────┼──────────┘
                         │
                         ▼
                    Authenticated
                         │
                         ▼
                    Authorization
                         │
                         ▼
                       RBAC
                         │
                         ▼
                Allowed Operation
```

---

# 20. Which mechanism should you choose?

A practical decision matrix:

| Scenario                             | Recommended mechanism            |
| ------------------------------------ | -------------------------------- |
| Local development                    | SCRAM-SHA-256                    |
| Small production deployment          | SCRAM-SHA-256 + TLS              |
| Standard application authentication  | SCRAM-SHA-256 + TLS              |
| Enterprise PKI                       | X.509                            |
| Corporate LDAP                       | LDAP                             |
| Enterprise SSO / cloud identity      | OIDC                             |
| AWS-integrated environment           | AWS IAM where supported          |
| Replica-set internal authentication  | Keyfile or X.509                 |
| Highly secured enterprise deployment | X.509 + TLS + RBAC               |
| Highly sensitive fields              | Add Queryable Encryption / CSFLE |

---

# 21. The most important distinction

For your MongoDB DBA training, I would organize the topic like this:

```text
                MONGODB AUTHENTICATION
                         │
       ┌─────────────────┴─────────────────┐
       │                                   │
       ▼                                   ▼
 CLIENT / USER AUTH                 INTERNAL AUTH
       │                                   │
       ├── SCRAM                           ├── Keyfile
       │                                   │
       ├── X.509                           └── X.509
       │
       ├── LDAP
       │
       ├── OIDC
       │
       └── AWS IAM
```

Then separately:

```text
Authentication
      ↓
"Who are you?"

Authorization
      ↓
"What can you do?"

TLS
      ↓
"Can someone intercept the communication?"

Encryption at Rest
      ↓
"Can someone read the database files?"

Auditing
      ↓
"What did the user actually do?"
```

Authorization

Authorization answers:

> **"What are you allowed to do?"**

Suppose we have:

```text
Alice
   ↓
MongoDB
   ↓
Authenticated
```

MongoDB then asks:

```text
What permissions does Alice have?
```

For example:

```text
Alice
 ├── read customer data
 ├── insert orders
 └── cannot drop databases
```

MongoDB primarily implements authorization through **Role-Based Access Control (RBAC)**.

---

# 5. MongoDB RBAC

Instead of giving permissions directly to every user, we assign **roles**.

```text
User
  |
  ▼
Role
  |
  ▼
Privileges
```

For example:

```text
anju
   |
   └── readWrite
          |
          ├── find
          ├── insert
          ├── update
          └── remove
```

MongoDB has built-in roles such as:

### Database-level roles

```text
read
readWrite
dbAdmin
userAdmin
```

### Cluster-level roles

```text
clusterMonitor
clusterManager
clusterAdmin
hostManager
```

### High-privilege role

```text
root
```

---
MongoDB security layers — the easiest way to remember

Remember MongoDB security as **7 layers**:

```text
              ┌──────────────────────┐
              │  7. Security Audit   │
              ├──────────────────────┤
              │  6. Key Management   │
              ├──────────────────────┤
              │  5. Encryption       │
              ├──────────────────────┤
              │  4. Network Security │
              ├──────────────────────┤
              │  3. Authorization    │
              ├──────────────────────┤
              │  2. Authentication   │
              ├──────────────────────┤
              │  1. Infrastructure  │
              └──────────────────────┘
```

Or, in question form:

| Question                                        | Security mechanism                 |
| ----------------------------------------------- | ---------------------------------- |
| **Who are you?**                                | Authentication                     |
| **What can you do?**                            | Authorization / RBAC               |
| **Can someone intercept traffic?**              | TLS                                |
| **Can someone steal database files?**           | Encryption at Rest                 |
| **Can DB administrators see sensitive fields?** | Client-Side / Queryable Encryption |
| **How are encryption keys protected?**          | KMS / KMIP                         |
| **Who did what?**                               | Auditing                           |
| **Who can connect to MongoDB?**                 | Network security / firewall        |
| **Can an unauthorized MongoDB node join?**      | Internal authentication            |

---

## 24. A practical Ubuntu security setup

Since you're working with **MongoDB 8.x on Ubuntu**, a typical hardened configuration would look conceptually like:

```text
Ubuntu Server
│
├── Firewall
│      └── Only trusted application servers → 27017
│
├── MongoDB
│      ├── Authentication enabled
│      ├── Authorization enabled
│      ├── TLS enabled
│      ├── Internal authentication enabled
│      ├── Auditing enabled
│      └── Encryption at rest
│
├── /var/lib/mongodb
│      └── Encrypted database files
│
└── Key Management
       └── External KMS / KMIP
```

A very important security principle is:

> **Never treat authentication as the entire security solution.**

A MongoDB deployment can have a strong password policy and still be insecure if:

* port 27017 is exposed publicly,
* TLS is disabled,
* excessive privileges are granted,
* database files aren't protected,
* encryption keys are poorly stored,
* auditing isn't available,
* replica-set internal authentication isn't configured.

### The complete security picture is therefore:

**Authentication + Authorization + Network Security + TLS + Encryption at Rest + Field-Level Encryption + Key Management + Auditing + Secure Configuration.**
