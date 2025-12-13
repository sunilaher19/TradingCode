# TradingCode

# 📘 Technical Design Document

## Real-Time Trading Communication Framework

---

## 1. Overview

This system provides a **low-latency trading communication framework** consisting of:

* A **Front-End client** connecting via **WebSocket**
* A **Common Framework** that handles:

  * Order Creation (OC)
  * Price dissemination
* A **Matching Engine** responsible for order matching
* **Aeron** as the core messaging layer for internal communication

The design ensures:

* Clear separation of concerns
* Shared business logic
* High throughput and low latency
* Scalability for market data distribution

---

## 2. High-Level Architecture

```
+----------------+
|   Front End   |
| (WebSocket)   |
+--------+------+
         |
         | WebSocket
         v
+---------------------------+
|     Common Framework      |
|                           |
|  - Validation             |
|  - Normalization          |
|  - Order ID Generation    |
|                           |
|  Uses Aeron for internal  |
|  messaging                |
+-----------+---------------+
            |
            | Aeron (IPC / UDP Unicast)
            v
+---------------------------+
|     Order Creator (OC)    |
+-----------+---------------+
            |
            | Aeron (IPC / UDP Unicast)
            v
+---------------------------+
|     Matching Engine       |
+-----------+---------------+
            |
            | Aeron UDP Multicast
            v
+---------------------------+
|     Price Distribution    |
+---------------------------+
            |
            | Aeron UDP Multicast
            v
+---------------------------+
|     Common Framework      |
|                           |
|  - Consumes price feed    |
|  - Publishes to WebSocket |
+---------------------------+
```

---

## 3. Component Responsibilities

### 3.1 Front End

* Web-based or native client
* Connects via **WebSocket**
* Sends:

  * New order requests
* Receives:

  * Price updates
  * Order status updates

---

### 3.2 Common Framework

The **Common Framework** is the core shared layer.

#### Responsibilities

* Accept WebSocket messages
* Parse incoming data (JSON / Protobuf)
* Validate business rules:

  * Valid security ID
  * Price and quantity constraints
* Generate unique order IDs
* Forward validated orders to **Order Creator (OC)** using Aeron
* Consume price data from Aeron multicast
* Publish price updates to WebSocket clients

#### Key Design Principles

* Transport-agnostic business logic
* No matching logic
* Stateless where possible

---

### 3.3 Order Creator (OC)

#### Responsibilities

* Receive normalized orders from Common Framework
* Perform order enrichment if required
* Forward orders to Matching Engine using Aeron
* Handle order lifecycle events

#### Communication

* Aeron IPC (same host) or UDP unicast (cross-host)

---

### 3.4 Matching Engine

#### Responsibilities

* Maintain order books per security
* Match BUY and SELL orders
* Generate:

  * Trades
  * Order updates
  * Price updates

#### Output Channels

* **Order Updates:** Aeron unicast
* **Price Feed:** Aeron UDP multicast

---

### 3.5 Price Distribution

#### Responsibilities

* Publish real-time price updates via Aeron multicast
* Support one-to-many subscribers
* Ensure low-latency fan-out

---

## 4. Communication Technologies

| Layer                | Technology          | Reason                   |
| -------------------- | ------------------- | ------------------------ |
| Client ↔ Framework   | WebSocket           | Browser compatibility    |
| Framework ↔ OC       | Aeron IPC / UDP     | Low latency              |
| OC ↔ Matching Engine | Aeron IPC / UDP     | Deterministic messaging  |
| Matching → Prices    | Aeron UDP Multicast | One-to-many distribution |

---

## 5. Message Flow

### 5.1 Order Flow

```
Client
  ↓ (WebSocket)
Common Framework
  ↓ (Aeron Unicast)
Order Creator (OC)
  ↓ (Aeron Unicast)
Matching Engine
```

---

### 5.2 Price Flow

```
Matching Engine
  ↓ (Aeron Multicast)
Price Distribution
  ↓ (Aeron Multicast)
Common Framework
  ↓ (WebSocket)
Client
```

---

## 6. Data Formats

* **External (Client ↔ Framework):**

  * JSON (initial phase)
  * Protobuf (recommended for production)

* **Internal (Aeron):**

  * Protobuf or FlatBuffers
  * Fixed-size messages where possible

---

## 7. Non-Functional Requirements

### Performance

* Microsecond-level internal latency
* Zero-copy IPC where possible
* Lock-free queues

### Reliability

* Aeron retransmission for UDP
* Back-pressure handling

### Scalability

* Multiple WebSocket instances
* Multiple Aeron subscribers
* Horizontal scaling of price consumers

---

## 8. Deployment Model

### Same Machine

* WebSocket + Common Framework + OC + Matching Engine
* Aeron IPC

### Cross Machine

* Aeron UDP unicast (orders)
* Aeron multicast (prices)

---

## 9. Future Enhancements

* Risk management module
* Persistence layer (Kafka / DB)
* Snapshot + replay for price feed
* TLS termination at WebSocket gateway
* Rate limiting per client

---

## 10. Summary

This architecture:

* Uses **WebSocket only at the edge**
* Uses **Aeron for all low-latency internal communication**
* Keeps **business logic centralized**
* Is suitable for **exchange-grade systems**

---

If you want, next we can:

1. Add **sequence diagrams**
2. Define **exact Aeron channels & URIs**
3. Design **matching engine internals**
4. Write **sample Aeron + WebSocket code**

Just tell me the next step 👍
