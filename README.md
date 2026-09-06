# DakPay — Offline Mesh Payment Prototype

> **A Spring Boot–based prototype for demonstrating encrypted, offline-first payment routing through a device-to-device mesh network.**

DakPay explores a simple question:

**What if a payment could be created when the sender has no internet, travel through nearby devices, and reach the backend when one of those devices eventually reconnects?**

The project simulates that idea on a single computer. A payment is encrypted, placed into a virtual mesh, forwarded between simulated devices, and finally uploaded by an internet-connected bridge node. The backend validates, deduplicates, decrypts, and settles the payment.

> **Important:** DakPay is a learning/portfolio prototype. It is **not a real UPI implementation**, does not connect to NPCI or a bank, and should not be used for real-money transactions.

---

## ✨ Highlights

DakPay demonstrates four important engineering concepts:

- 🔐 **Hybrid encryption** using RSA-OAEP and AES-256-GCM.
- 🔁 **Mesh forwarding** through simulated offline devices.
- 🛡️ **Idempotent settlement** so duplicate deliveries do not debit the sender multiple times.
- ⏱️ **Replay/freshness protection** using timestamps and nonces.
- 💳 **Atomic settlement** using a transactional debit/credit operation.
- 🧪 **Concurrency testing** for simultaneous duplicate packet delivery.
- 📊 **Interactive dashboard** for observing devices, balances, transactions, and system activity.

---

## 🧩 What DakPay Demonstrates

The complete flow is:

```text
Sender
  │
  │  Create payment
  ▼
Encrypt payment
  │
  │  RSA-OAEP + AES-256-GCM
  ▼
Offline Mesh
  │
  ├── Virtual Device
  ├── Virtual Device
  └── Bridge Device
             │
             │ Internet available
             ▼
       Spring Boot Backend
             │
             ├── SHA-256 packet hash
             ├── Idempotency check
             ├── Decryption
             ├── Freshness validation
             └── Transactional settlement
                    │
                    ├── Debit sender
                    ├── Credit receiver
                    └── Write ledger
```

The dashboard presents this process visually so the complete concept can be demonstrated without Bluetooth hardware or multiple phones.

---

# 👥 Demo Users

DakPay uses the following demo identities in the dashboard:

| Name | VPA |
|---|---|
| **Pratap** | `pratap@demo` |
| **Singh** | `singh@demo` |

The dashboard has been updated to remove the previous **Alice/Bob** naming and use **Pratap/Singh** instead.

### Backend rename requirement

The supplied dashboard now sends:

```text
pratap@demo
singh@demo
```

Therefore, the backend seed data must use the same VPAs.

If your current `DemoService.java` still creates accounts such as:

```text
alice@demo
bob@demo
```

change those seed records to:

```text
pratap@demo
singh@demo
```

Also update the initial simulated sender device from:

```text
phone-alice
```

to:

```text
phone-pratap
```

The REST endpoints and business logic do not need to change merely because the demo identities were renamed.

---

# 🛠️ Technology Stack

| Technology | Purpose |
|---|---|
| **Java 17+** | Application language |
| **Spring Boot** | Backend application and REST APIs |
| **Spring Data JPA** | Persistence layer |
| **H2** | In-memory demonstration database |
| **Maven Wrapper** | Build and dependency management |
| **Thymeleaf** | Dashboard serving |
| **HTML / CSS / JavaScript** | Interactive DakPay dashboard |
| **RSA-OAEP** | Encryption of the AES session key |
| **AES-256-GCM** | Authenticated payload encryption |
| **SHA-256** | Packet fingerprint / idempotency key |
| **JUnit** | Automated tests |

---

# 🚀 Getting Started

## Prerequisites

You only need:

- JDK **17 or newer**
- A terminal
- An internet connection for the first dependency download

Verify Java:

```bash
java -version
```

No separately installed Maven is required because the repository contains the Maven Wrapper.

---

## ▶️ Run on Windows

Open PowerShell or Command Prompt in the project directory:

```cmd
mvnw.cmd spring-boot:run
```

If PowerShell does not recognize the wrapper, use:

```powershell
.\mvnw.cmd spring-boot:run
```

---

## ▶️ Run on macOS / Linux

```bash
./mvnw spring-boot:run
```

---

## 🌐 Open DakPay

After Spring Boot starts successfully, open:

```text
http://localhost:8080
```

You should see the **DakPay Offline Mesh** dashboard.

The dashboard automatically refreshes the mesh state, account balances, transaction ledger, and activity log.

---

# 🎬 How to Demonstrate DakPay

## Step 1 — Create a payment

Select:

```text
Sender   → Pratap
Receiver → Singh
Amount   → ₹500
PIN      → 1234
```

Then click:

**Send to Mesh**

The backend creates a payment instruction containing information such as:

- Sender VPA
- Receiver VPA
- Amount
- PIN-related data
- Unique nonce
- Timestamp

The payment payload is encrypted before it is placed into the virtual mesh.

---

## Step 2 — Run a gossip round

Click:

**Run Gossip Round**

The mesh simulator forwards packets between virtual devices.

Each hop represents a device-to-device transfer.

The packet's TTL is reduced as it moves through the network.

This simulates the idea of nearby devices carrying a payment until one of them obtains connectivity.

---

## Step 3 — Upload from the bridge

Click:

**Bridge Upload**

The simulated bridge device represents a phone that has moved into an area with internet connectivity.

It sends the encrypted packet to:

```text
POST /api/bridge/ingest
```

The backend then performs the validation and settlement pipeline.

---

## Step 4 — Observe settlement

After successful ingestion, check:

### Account Balances

The sender is debited and the receiver is credited.

### Transaction Ledger

The transaction appears with information such as:

- Transaction ID
- Sender
- Receiver
- Amount
- Status
- Bridge node
- Hop count
- Settlement time

---

# 🔐 Security Design

## 1. Hybrid Encryption

DakPay uses:

```text
RSA-OAEP
   +
AES-256-GCM
```

RSA is used to protect the randomly generated AES key, while AES-GCM encrypts the actual payment payload.

Conceptually:

```text
Payment JSON
     │
     ▼
AES-256-GCM
     │
     ▼
Encrypted Payload

AES Key
   │
   ▼
RSA-OAEP using server public key
   │
   ▼
Encrypted AES Key
```

The final encrypted packet contains the protected AES key, IV, and encrypted payload.

### Why AES-GCM?

AES-GCM provides:

- Confidentiality
- Integrity
- Authentication

If an intermediary modifies the encrypted payload, GCM authentication fails during decryption.

The mesh therefore carries an opaque encrypted packet instead of readable payment information.

---

# 🔁 Idempotency

Offline networks can deliver the same packet more than once.

For example:

```text
Bridge A ──┐
           ├── Same packet ──> Backend
Bridge B ──┤
Bridge C ──┘
```

Without idempotency:

```text
₹500 × 3 deliveries = ₹1500 debit
```

That would be incorrect.

DakPay calculates a SHA-256 hash of the ciphertext and attempts to claim it atomically.

Conceptually:

```java
Instant previous = seen.putIfAbsent(packetHash, now);
return previous == null;
```

Only the first delivery is allowed to continue.

Other deliveries become:

```text
DUPLICATE_DROPPED
```

This protects the sender from being charged multiple times for the same packet.

---

# ⏱️ Replay Protection

DakPay uses two complementary mechanisms.

### Freshness timestamp

The encrypted payment includes a timestamp.

The backend rejects packets that are outside the accepted freshness window.

### Unique nonce

Every payment instruction receives a unique nonce.

Therefore:

```text
Pratap → Singh ₹500
Nonce A

Pratap → Singh ₹500
Nonce B
```

These represent two different payment instructions.

A replay of the exact same packet can then be identified through the same ciphertext/hash.

---

# 💳 Transaction Settlement

The settlement layer performs the following operations together:

```text
Debit sender
      │
      ▼
Credit receiver
      │
      ▼
Write transaction ledger
```

The operation is wrapped in a database transaction.

The `Account` entity also uses optimistic locking as an additional protection against concurrent updates.

---

# 🏗️ Project Architecture

```text
DakPay
│
├── src/main/java/com/demo/upimesh/
│   │
│   ├── model/
│   │   ├── Account.java
│   │   ├── AccountRepository.java
│   │   ├── Transaction.java
│   │   ├── TransactionRepository.java
│   │   ├── MeshPacket.java
│   │   └── PaymentInstruction.java
│   │
│   ├── service/
│   │   ├── BridgeIngestionService.java
│   │   ├── DemoService.java
│   │   ├── IdempotencyService.java
│   │   ├── MeshSimulatorService.java
│   │   ├── SettlementService.java
│   │   └── VirtualDevice.java
│   │
│   └── UpiMeshApplication.java
│
├── src/main/resources/
│   ├── templates/
│   │   └── dashboard.html
│   └── application.properties
│
├── src/test/
│   └── IdempotencyConcurrencyTest.java
│
├── pom.xml
├── mvnw
├── mvnw.cmd
└── README.md
```

> The current Java package/class names are retained from the original implementation. The **product/project name** presented to users is now **DakPay**.

---

# 📁 Important Components

## `DemoService.java`

Responsible for demonstration setup and seeded accounts.

It also simulates the sender-side packet creation.

---

## `MeshSimulatorService.java`

Simulates the offline device mesh and packet gossip.

It models how a packet can move between virtual devices.

---

## `VirtualDevice.java`

Represents an individual simulated phone/device.

A device may:

- Hold packets
- Forward packets
- Act as a bridge when internet is available

---

## `BridgeIngestionService.java`

This is the central backend ingestion pipeline:

```text
Receive packet
     ↓
SHA-256 hash
     ↓
Idempotency claim
     ↓
Decrypt
     ↓
Freshness validation
     ↓
Settlement
```

---

## `IdempotencyService.java`

Prevents the same encrypted payment from being processed more than once.

The demo implementation uses an in-memory `ConcurrentHashMap`.

---

## `SettlementService.java`

Performs the transactional:

```text
Debit
Credit
Ledger insert
```

operation.

---

## `HybridCryptoService.java`

Handles:

```text
RSA-OAEP
AES-256-GCM
SHA-256 ciphertext hashing
```

---

# 🌐 REST API

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/` | DakPay dashboard |
| `GET` | `/api/server-key` | Server RSA public key |
| `GET` | `/api/accounts` | Account balances |
| `GET` | `/api/transactions` | Recent transactions |
| `GET` | `/api/mesh/state` | Current virtual mesh state |
| `POST` | `/api/demo/send` | Create and inject a demo payment |
| `POST` | `/api/mesh/gossip` | Execute a mesh gossip round |
| `POST` | `/api/mesh/flush` | Upload packets held by bridge nodes |
| `POST` | `/api/mesh/reset` | Reset mesh and idempotency state |
| `POST` | `/api/bridge/ingest` | Receive an encrypted bridge packet |
| `GET` | `/h2-console` | H2 database console |

---

# 📦 Bridge Ingestion Request

Example:

```http
POST /api/bridge/ingest
Content-Type: application/json

X-Bridge-Node-Id: phone-bridge-42
X-Hop-Count: 3
```

```json
{
  "packetId": "550e8400-e29b-41d4-a716-446655440000",
  "ttl": 2,
  "createdAt": 1730000000000,
  "ciphertext": "base64-encoded-encrypted-payload"
}
```

Possible outcomes include:

```text
SETTLED
DUPLICATE_DROPPED
INVALID
```

---

# 🧪 Testing

Run the complete test suite:

### Windows

```cmd
mvnw.cmd test
```

### macOS / Linux

```bash
./mvnw test
```

The project includes tests covering:

### Encryption round trip

Verifies that encrypted data can be correctly decrypted.

### Tampered ciphertext

Modifies encrypted data and verifies that the ingestion pipeline rejects it.

### Concurrent duplicate delivery

Simulates three bridge nodes delivering the same packet simultaneously.

The expected behavior is:

```text
1 × SETTLED
2 × DUPLICATE_DROPPED
```

The sender should be debited only once.

---

# 🧪 H2 Database

The prototype uses an in-memory H2 database.

Typical development configuration:

```text
JDBC URL: jdbc:h2:mem:upimesh
Username: sa
Password: <empty>
```

The H2 console is intended for development/demo inspection only and should not be exposed in a production deployment.

---

# ⚠️ What Is Simulated?

DakPay intentionally simulates several real-world components.

| Prototype | Production equivalent |
|---|---|
| H2 in-memory database | PostgreSQL / MySQL |
| `ConcurrentHashMap` idempotency | Redis / distributed idempotency store |
| Generated RSA keypair | HSM / KMS / secure key management |
| Virtual device mesh | Bluetooth Low Energy / Wi-Fi Direct |
| Demo accounts | Real authenticated/KYC accounts |
| Local settlement service | Bank/payment-network integration |
| Simple bridge endpoint | Authenticated bridge infrastructure |
| Local dashboard | Production monitoring/admin system |

---

# 🚧 Limitations

DakPay should be presented honestly as a **mesh-routed deferred-settlement prototype**, not as production-ready offline UPI.

### 1. Offline balance verification

Without a trusted offline wallet, the receiver cannot independently prove that the sender has sufficient funds at the moment the payment is created.

### 2. Offline double spending

A malicious sender could create multiple payment instructions before reconnecting.

The backend can reject later conflicting settlements, but that does not provide the same guarantees as a properly funded offline wallet.

### 3. Real Bluetooth networking

The current mesh is software-simulated.

A production implementation would require reliable device discovery, pairing, packet exchange, storage, retry handling, battery management, and platform-specific BLE behavior.

### 4. Authentication

The prototype does not represent the complete authentication, device identity, KYC, fraud detection, and authorization infrastructure required by a real payment system.

### 5. Regulatory and banking integration

DakPay does not connect to:

- NPCI
- UPI
- Banks
- Real payment accounts
- Real-money settlement systems

---

# 🔮 Future Enhancements

Potential next steps include:

- Android client using Kotlin
- Real BLE mesh communication
- Signed device identities
- Mutual TLS for bridge nodes
- Redis-based distributed idempotency
- PostgreSQL persistence
- Hardware-backed key storage
- Offline wallet / pre-funded balance model
- Stronger device authentication
- Rate limiting and fraud detection
- Packet expiry and retry policies
- Observability with structured logs and metrics
- Docker deployment
- Cloud-hosted backend
- Production-grade security audit

---

# 🎯 Why This Project Is Interesting

DakPay is designed to demonstrate more than a basic CRUD application.

It combines:

```text
Spring Boot
   +
REST APIs
   +
JPA
   +
Database Transactions
   +
Cryptography
   +
Concurrency
   +
Idempotency
   +
Distributed-system concepts
   +
Network simulation
```

The most important engineering idea is that **an unreliable delivery path does not have to result in duplicate financial settlement**.

That makes idempotency, authenticated encryption, replay protection, and transactional consistency central parts of the design.

---

# 🖥️ Dashboard

The DakPay dashboard provides:

- Payment creation controls
- Sender/receiver selection
- Mesh device status
- Bridge connectivity status
- Account balances
- Transaction ledger
- Packet IDs
- Hop counts
- Idempotency cache information
- Real-time activity logs

The dashboard refreshes automatically while the application is running.

---

# 🔧 Troubleshooting

## Java is not recognized

Verify:

```bash
java -version
```

Install JDK 17 or newer and configure `JAVA_HOME` if necessary.

---

## Port 8080 is already in use

Change the server port in:

```text
src/main/resources/application.properties
```

For example:

```properties
server.port=8081
```

Then open:

```text
http://localhost:8081
```

---

## Maven Wrapper is not recognized in PowerShell

Use:

```powershell
.\mvnw.cmd spring-boot:run
```

---

## Dashboard loads but payment fails

The updated dashboard sends:

```text
pratap@demo
singh@demo
```

Make sure the backend seed accounts use exactly the same VPAs.

Also verify that the simulated starting device exists with:

```text
phone-pratap
```

If your Java code still contains the old demo identities, update those seed values before testing the new dashboard.

---

# 📜 License

This project is currently intended for educational, demonstration, and portfolio purposes.

No separate open-source license is currently specified.

---

## 👨‍💻 Project

**DakPay — Offline Mesh Payment Prototype**

**Focus:** Offline payment routing • Cryptography • Idempotency • Distributed systems • Spring Boot

> Built as a technical demonstration of how encrypted payment instructions could be transported through an offline device mesh and safely settled after connectivity is restored.
