# UPI Offline Mesh

> Send money when there's no internet. Encrypted payment packets hop from phone to phone via Bluetooth until one reaches 4G — then silently settles on the backend.

---

## The Problem

UPI breaks the moment internet is gone — basements, rural areas, crowded events, disasters. This project asks: what if the payment could travel through strangers' phones and settle later, without any of them being able to read or tamper with it?

---

## How It Works

```
[Alice's Phone]
      │  encrypts payment with server's RSA public key
      │  hands packet to nearby phone via Bluetooth
      ▼
[Stranger 1] → [Stranger 2] → [Bridge Phone - has 4G]
                                        │
                                        ▼ HTTPS POST
                               [Spring Boot Backend]
                                        │
                              hash → deduplicate → decrypt → settle
```

1. Alice's phone encrypts the payment and wraps it in a `MeshPacket`
2. The packet gossips device to device, TTL decrementing each hop
3. The first device that gets internet uploads it to the backend
4. Backend decrypts, checks freshness, debits Alice, credits Bob

Intermediate phones carry an encrypted blob — they see nothing.

---

## The Three Hard Problems

### 1. Untrusted Intermediates
A stranger's phone is carrying your transaction. How do you prevent them from reading or modifying it?

**Hybrid RSA-OAEP + AES-256-GCM encryption.**

RSA alone can't encrypt a full JSON payload (~245 byte limit on 2048-bit keys). So:
- Generate a fresh AES-256 key per packet
- Encrypt the payload with AES-GCM (fast + authenticated)
- Encrypt the AES key with RSA-OAEP
- Wire format: `[256B encrypted AES key][12B IV][ciphertext + 16B GCM tag]`

AES-GCM is authenticated encryption — a single flipped bit causes decryption to throw. Intermediates cannot tamper with the payload without detection.

### 2. Three Bridges Deliver Simultaneously
Multiple bridge nodes hold the same packet. They all get 4G at the same instant. Without deduplication, Alice gets debited three times.

**Atomic compare-and-set on the ciphertext hash.**

```java
// ConcurrentHashMap.putIfAbsent = JVM-local Redis SETNX
// Exactly one thread wins, all others get DUPLICATE_DROPPED
Instant prev = seen.putIfAbsent(packetHash, now);
return prev == null;
```

The idempotency key is `SHA-256(ciphertext)` — not `packetId` (which intermediates can rewrite). Two deliveries of the same packet always have identical ciphertexts, hence identical hashes.

Proven with a concurrency test — three threads, one packet, simultaneous delivery:
```
settled    = 1  ✓
duplicates = 2  ✓
alice balance changed exactly once  ✓
```

### 3. Replay Attacks
An attacker captures a valid ciphertext and replays it later.

Every `PaymentInstruction` includes a `signedAt` timestamp and a UUID `nonce`. The server rejects packets older than 24 hours. Since `signedAt` is inside the encrypted blob, it can't be altered without breaking the GCM tag.

---

## Tech Stack

| | |
|---|---|
| Backend | Java 17, Spring Boot |
| Encryption | RSA-2048/OAEP-SHA256 + AES-256-GCM |
| Database | H2 in-memory (zero setup) |
| ORM | Spring Data JPA / Hibernate |
| Frontend | HTML/CSS/JS |
| Build | Maven Wrapper |
| Tests | JUnit 5 + Spring Boot Test |

---

## Running Locally

Only requirement: **JDK 17 or 21**. No Maven, no database, no Redis needed.

```bash
# Mac/Linux
./mvnw spring-boot:run

# Windows
mvnw.cmd spring-boot:run
```

First run downloads dependencies (~2 min). Then open `http://localhost:8080`

Demo accounts seeded automatically:

| VPA | Balance |
|-----|---------|
| alice@demo | ₹5,000 |
| bob@demo | ₹1,000 |
| carol@demo | ₹2,500 |
| dave@demo | ₹500 |

---

## Demo Flow

**Step 1** — Fill the Send Payment form and click **Encrypt and inject into mesh**
The packet is encrypted and handed to Alice's device.

**Step 2** — Click **Run round**
The packet spreads to all 5 simulated devices including the bridge node.

**Step 3** — Click **Flush bridges**
Bridge uploads to backend. Server decrypts, verifies, settles. Balances update live.

**Step 4** — Click **Flush bridges again**
Same packet, second delivery → `DUPLICATE_DROPPED`. Alice's balance doesn't change.

---

## Project Structure

```
src/main/java/com/demo/upimesh/
├── config/       AppConfig (scheduling)
├── controller/   REST API + dashboard route
├── crypto/       RSA keypair + hybrid encryption
├── model/        JPA entities + repositories
└── service/
    ├── BridgeIngestionService   Main pipeline: hash → claim → decrypt → settle
    ├── IdempotencyService       ConcurrentHashMap-based dedup cache
    ├── SettlementService        @Transactional debit/credit + ledger
    ├── MeshSimulatorService     Gossip protocol across virtual devices
    ├── VirtualDevice            Simulated phone
    └── DemoService              Seeds accounts, simulates sender phone
```

---

## API Reference

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/server-key` | RSA public key for sender devices |
| POST | `/api/demo/send` | Simulate sender phone creating a packet |
| GET | `/api/mesh/state` | Current device and packet state |
| POST | `/api/mesh/gossip` | Run one gossip round |
| POST | `/api/mesh/flush` | Bridge nodes upload to backend (parallel) |
| POST | `/api/mesh/reset` | Clear mesh and idempotency cache |
| POST | `/api/bridge/ingest` | Production endpoint for real bridge nodes |
| GET | `/api/accounts` | All account balances |
| GET | `/api/transactions` | Last 20 transactions |

---

## Tests

```bash
./mvnw test
```

- `encryptDecryptRoundTrip` — verifies hybrid encryption correctness
- `tamperedCiphertextIsRejected` — flips a byte, verifies INVALID outcome
- `singlePacketDeliveredByThreeBridgesSettlesExactlyOnce` — the main concurrency test

---

## Honest Limitations

- **Idempotency is single-node** — production would use Redis `SET NX EX`
- **No sender authentication** — real system needs sender to sign with their private key
- **PIN has no salt** — real UPI uses HSM-backed PIN blocks via NPCI
- **Receiver has no guarantee of funds** — if the sender's balance is empty when the packet arrives, it's rejected. Real offline UPI (UPI Lite) solves this with pre-funded hardware wallets
- **Bluetooth simulation only** — real mesh would use Android BLE GATT

---

## H2 Console

`http://localhost:8080/h2-console`
```
JDBC URL:  jdbc:h2:mem:upimesh
Username:  sa
Password:  (blank)
```

---
