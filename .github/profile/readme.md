# ShopX-ecom with Microservices architecture 

### Migrating monolith ([Shopx-E_commerce](https://github.com/Sameer377/Shopx_E-commerce)) to event driven microservices 

> Built by **Sameer Bilal Shaikh**  
> Converting Monolith application into microservices based application from scratch.
>  A distributed, event-driven e-commerce backend implementing enterprise-grade patterns including Saga, Circuit Breaker, gRPC, Kafka, Idempotency, and JWT Role-Based Auth. 

---

## Table of Contents

- [Overview](#overview)
- [Architecture Diagram](#architecture-diagram)
- [Services](#services)
- [Tech Stack](#tech-stack)
- [Key Patterns & Concepts](#key-patterns--concepts)
- [Service Communication Map](#service-communication-map)
- [API Gateway & Security](#api-gateway--security)
- [Event Flow — Order Placement](#event-flow--order-placement)
- [Saga Pattern — Failure Handling](#saga-pattern--failure-handling)
- [Running Locally](#running-locally)
- [Environment Variables](#environment-variables)

---

## Overview

ShopX is a fully distributed e-commerce backend built with a microservices architecture. Each service is independently deployable, owns its own database, and communicates via REST, gRPC, or Kafka depending on the use case.

The system is designed to handle real-world production concerns:
- **Zero message loss** via Kafka with replication
- **No double payments** via Idempotency
- **No cascading failures** via Circuit Breaker
- **Distributed tracing** via Zipkin
- **Eventual consistency** via Saga Pattern
- **Role-based access** via JWT + Spring Security

---

## Architecture Diagram

```
                        ┌─────────────────────┐
                        │    Client Request    │
                        │  (Mobile / Web App)  │
                        └──────────┬──────────┘
                                   │ HTTPS
                                   ▼
                    ┌──────────────────────────────┐
                    │     Spring Cloud Gateway      │
                    │                              │
                    │  → JWT Validation            │
                    │  → Rate Limiting (Redis)     │
                    │  → Request Routing           │
                    │  → Load Balancing            │
                    └──────────────┬───────────────┘
                                   │
                                   ▼
                    ┌──────────────────────────────┐
                    │        Eureka Server          │
                    │     (Service Discovery)       │
                    └──────────────┬───────────────┘
                                   │
          ┌──────────────┬─────────┴──────────┬──────────────┐
          │              │                    │              │
          ▼              ▼                    ▼              ▼
   ┌────────────┐ ┌────────────┐    ┌──────────────┐ ┌────────────┐
   │    Auth    │ │    User    │    │   Product /  │ │   Order    │
   │  Service   │ │  Service   │    │  Inventory   │ │  Service   │
   │            │ │            │    │   Service    │ │            │
   │ JWT, BCrypt│ │ Cache(Redis│    │ gRPC Server  │ │ Saga + Kfk │
   └────────────┘ │ gRPC Srvr) │    │ Stock Mgmt   │ │ Idempotent │
                  └────────────┘    └──────────────┘ └─────┬──────┘
                                                           │
                                    ┌──────────────────────┤
                                    │         KAFKA        │
                                    │   (Message Broker)   │
                                    └──┬───────────────┬───┘
                                       │               │
                              ┌────────┴───┐    ┌──────┴───────┐
                              │  Billing   │    │ Notification │
                              │  Service   │    │   Service    │
                              │ Idempotent │    │ Kafka Cnsmer │
                              │ Saga Step  │    │ Email/SMS    │
                              └────────────┘    └──────────────┘
```

---

## Services

### 1. Auth Service
**Port:** `8081`  
**Responsibility:** Authentication only. Issues and validates JWT tokens.

| Feature | Details |
|---|---|
| Registration | BCrypt password encoding (strength 10) |
| Login | JWT Access Token (15 min) + Refresh Token (7 days) |
| Token Refresh | Refresh token stored in DB, validated on each refresh |
| Logout | Refresh token deleted from DB |
| Security | Stateless — no sessions |

**Endpoints:**
```
POST /api/auth/register
POST /api/auth/login
POST /api/auth/refresh
POST /api/auth/logout
```

---

### 2. User Service
**Port:** `8082`  
**Responsibility:** User profile management, addresses, wallet, loyalty points.

| Feature | Details |
|---|---|
| Profile Management | CustomerDetail entity extending User (JOINED inheritance) |
| Address Management | Multiple addresses, default address support |
| Wallet | Credit/Debit with balance validation |
| Loyalty Points | Earn on orders, redeem on checkout |
| Cache | Redis — user profile cached for 5 mins |
| gRPC Server | Exposes user data to other services via gRPC |

**Database:** PostgreSQL (own schema)  
**Cache:** Redis  
**Communication:** gRPC server (consumed by Billing Service)

---

### 3. Product / Inventory Service
**Port:** `8083`  
**Responsibility:** Product catalog management and real-time stock control.

| Feature | Details |
|---|---|
| Product CRUD | Full product lifecycle management |
| Stock Management | Reserve, release, and confirm stock |
| Cache | Redis — product details cached for 10 mins, evicted on update |
| gRPC Server | Exposes stock check + reservation to Order Service |
| Kafka Consumer | Listens to `payment.failed` → releases reserved stock |

**Database:** PostgreSQL (own schema)  
**Cache:** Redis  
**Communication:** gRPC server (consumed by Order Service)

**gRPC Methods:**
```protobuf
service InventoryService {
    rpc CheckStock(StockRequest) returns (StockResponse);
    rpc ReserveStock(ReserveRequest) returns (ReserveResponse);
    rpc ReleaseStock(ReleaseRequest) returns (ReleaseResponse);
}
```

---

### 4. Order Service
**Port:** `8084`  
**Responsibility:** Core order lifecycle. The most complex service — orchestrates the entire Saga.

| Feature | Details |
|---|---|
| Place Order | Idempotent — X-Idempotency-Key header required |
| Stock Check | gRPC call to Inventory Service |
| Saga Orchestration | Publishes Kafka events, listens to compensating events |
| Circuit Breaker | Wraps all gRPC/REST calls to other services |
| Order Status | PENDING → CONFIRMED → SHIPPED → DELIVERED / CANCELLED |
| Kafka Publisher | Publishes `order.created`, `order.shipped`, `order.delivered` |
| Kafka Consumer | Listens to `payment.failed` → cancels order |

**Database:** PostgreSQL (own schema)  
**Communication:** gRPC client (Inventory), REST client (User), Kafka publisher/consumer  
**Resilience:** Circuit Breaker (Resilience4j) + Retry

---

### 5. Billing Service
**Port:** `8085`  
**Responsibility:** Payment processing. Strictly idempotent — never charges twice.

| Feature | Details |
|---|---|
| Payment Processing | Idempotent via Redis-backed idempotency store |
| Saga Step | Listens to `stock.reserved` → charges customer |
| Success Flow | Publishes `payment.success` |
| Failure Flow | Publishes `payment.failed` → triggers compensation |
| Refund | Triggered by compensating transaction |
| Circuit Breaker | Wraps payment gateway calls |
| gRPC Client | Fetches billing details from User Service |

**Database:** PostgreSQL (own schema)  
**Cache:** Redis (idempotency keys — 24hr TTL)  
**Communication:** Kafka consumer/publisher, gRPC client

---

### 6. Notification Service
**Port:** `8086`  
**Responsibility:** Pure Kafka consumer. Sends emails, SMS, push notifications. Stateless.

| Kafka Topic | Trigger | Action |
|---|---|---|
| `order.created` | Order placed | Send order confirmation |
| `payment.success` | Payment done | Send payment receipt |
| `payment.failed` | Payment failed | Send failure alert |
| `order.shipped` | Order shipped | Send tracking info |
| `order.delivered` | Order delivered | Send delivery confirmation |
| `stock.low` | Stock below threshold | Alert admin |

**Database:** None  
**Communication:** Kafka consumer only  
**Note:** Completely decoupled — adding new notification types requires zero changes to other services.

---

### 7. Spring Cloud Gateway
**Port:** `8080`  
**Responsibility:** Single entry point for all client requests.

| Feature | Details |
|---|---|
| JWT Validation | Validates token before forwarding any request |
| Rate Limiting | Redis-backed, per-user rate limiting |
| Routing | Routes to services via Eureka service discovery |
| Load Balancing | Automatic via `lb://SERVICE-NAME` |
| CORS | Configured globally for frontend origins |

**Rate Limits:**
```yaml
Order Service:   10 req/sec (burst: 20)
Product Service: 50 req/sec (burst: 100)
Auth Service:    5 req/sec  (burst: 10)
```

---

### 8. Eureka Server
**Port:** `8761`  
**Responsibility:** Service registry and discovery.

Every service registers on startup with its name, IP, and port. When Order Service needs to call User Service, it asks Eureka — no hardcoded URLs anywhere.

---

## Tech Stack

| Category | Technology |
|---|---|
| Language | Java 17 |
| Framework | Spring Boot 3.x |
| Service Discovery | Spring Cloud Netflix Eureka |
| API Gateway | Spring Cloud Gateway |
| Async Messaging | Apache Kafka |
| Sync Communication | gRPC (Protocol Buffers) |
| REST | Spring Web (RestTemplate / WebClient) |
| Security | Spring Security + JWT (JJWT) |
| Database | PostgreSQL (per service) |
| Cache | Redis |
| Distributed Tracing | Zipkin + Spring Cloud Sleuth |
| Resilience | Resilience4j (Circuit Breaker + Retry) |
| ORM | Spring Data JPA / Hibernate |
| Containerization | Docker + Docker Compose |
| CI/CD | GitHub Actions + AWS EC2 |
| Cloud | AWS (EC2, S3) |
| Build Tool | Maven |
| Testing | JUnit 5, Mockito |
| Documentation | Swagger / OpenAPI |

---

## Key Patterns & Concepts

### 1. Saga Pattern (Choreography)
Manages distributed transactions across services without a central coordinator.

```
ORDER_CREATED
    → Inventory reserves stock → STOCK_RESERVED
    → Billing charges customer → PAYMENT_SUCCESS
    → Order confirmed → ORDER_CONFIRMED
    → Notification sent ✅

FAILURE PATH:
    → Billing fails → PAYMENT_FAILED
    → Inventory releases stock (compensating transaction)
    → Order cancelled
    → Notification sent (failure alert) ✅
```

### 2. Circuit Breaker (Resilience4j)
Prevents cascading failures when a service is down.

```
CLOSED  → Normal operation, calls go through
OPEN    → Service failing, calls blocked, fallback used
HALF-OPEN → Testing recovery after timeout

Config:
  failure-rate-threshold: 50%
  wait-duration-in-open-state: 10s
  sliding-window-size: 10 calls
```

### 3. Idempotency
Prevents duplicate payments and duplicate orders.

```
Client sends: POST /api/orders
Header: X-Idempotency-Key: uuid-here

First call  → processes order → stores result in Redis
Second call → returns same result → NO duplicate order

Payment idempotency:
First charge  → charges card → stores in Redis (24hr)
Second charge → returns cached response → NO double charge
```

### 4. gRPC Communication
Used for high-frequency, performance-critical inter-service calls.

```
Order Service → Inventory Service: stock check/reservation
Order Service → Billing Service:   payment initiation
Billing Service → User Service:    fetch billing details

Why gRPC over REST here:
→ Binary protocol (faster than JSON)
→ Strongly typed contracts (proto files)
→ Streaming support
→ Lower latency on high-frequency calls
```

### 5. Distributed Tracing (Zipkin)
Every request gets a trace ID that follows it across all services.

```
Client request → Gateway → Order Service → Inventory (gRPC)
                                        → Kafka → Billing
                                                → Notification

One trace ID ties all of this together in Zipkin UI.
Find bottlenecks. Debug failures. Track latency per service.
```

### 6. Redis Cache Strategy
```
Product details:    TTL 10 min  (read-heavy, rarely changes)
User profile:       TTL 5 min   (fetched on every order)
Rate limit counters: TTL 1 min  (sliding window)
Idempotency keys:   TTL 24 hrs  (payment dedup)
JWT blacklist:      TTL = token expiry (logout invalidation)
```

---

## Service Communication Map

```
┌─────────────────────────────────────────────────────────┐
│                    COMMUNICATION MAP                     │
│                                                         │
│  Gateway      → All Services       : REST (routing)    │
│  Order        → Inventory          : gRPC              │
│  Order        → User               : REST + CB         │
│  Billing      → User               : gRPC              │
│  Order        → Kafka              : Publisher         │
│  Billing      → Kafka              : Publisher/Consumer │
│  Inventory    → Kafka              : Consumer          │
│  Notification → Kafka              : Consumer only     │
│  Auth         → Redis              : JWT blacklist     │
│  Gateway      → Redis              : Rate limiting     │
│  Order        → Redis              : Idempotency       │
│  Billing      → Redis              : Idempotency       │
│  Product      → Redis              : Cache             │
│  User         → Redis              : Cache             │
└─────────────────────────────────────────────────────────┘
```

---

## API Gateway & Security

### JWT Flow
```
1. Client logs in → Auth Service issues:
   - Access Token (15 min)
   - Refresh Token (7 days, stored in DB)

2. Client sends: Authorization: Bearer <access_token>

3. Gateway validates token before forwarding

4. Controller uses @AuthenticationPrincipal
   to get current user without DB call

5. Token expires → client sends refresh token
   → new access token issued
   → same refresh token returned

6. Logout → refresh token deleted from DB
   → access token expires naturally
```

### Role-Based Access Control
```
ROLE_USER:
  → Browse products
  → Place orders
  → View own orders
  → Manage own profile

ROLE_SELLER:
  → Create/update own products
  → View own product orders

ROLE_ADMIN:
  → Everything
  → Delete any product
  → View all orders
  → Manage users
```

---

## Event Flow — Order Placement

```
Step 1: Client → POST /api/orders (with Idempotency-Key)
Step 2: Gateway validates JWT
Step 3: Order Service checks idempotency key in Redis
Step 4: Order Service calls Inventory via gRPC (CheckStock)
Step 5: If stock available → Reserve via gRPC
Step 6: Order saved with status PENDING
Step 7: Order Service publishes order.created to Kafka
Step 8: Returns response to client immediately (async)

--- Background (Kafka) ---
Step 9:  Billing Service consumes order.created
Step 10: Billing checks idempotency key
Step 11: Charges customer
Step 12: Publishes payment.success

Step 13: Order Service consumes payment.success
Step 14: Updates order status → CONFIRMED
Step 15: Publishes order.confirmed

Step 16: Notification Service consumes order.confirmed
Step 17: Sends confirmation email/SMS to customer ✅
```

---

## Saga Pattern — Failure Handling

```
--- Payment Failure Scenario ---

Step 1-7: Same as above (order created, stock reserved)
Step 8:   Billing Service fails to charge customer
Step 9:   Billing publishes payment.failed

Step 10: Inventory Service consumes payment.failed
         → Releases reserved stock (compensating transaction)

Step 11: Order Service consumes payment.failed
         → Updates order status → CANCELLED

Step 12: Notification Service consumes payment.failed
         → Sends "Order cancelled, payment failed" to customer

Result:
→ No money charged ✅
→ Stock restored ✅
→ Order cancelled ✅
→ Customer notified ✅
→ Data consistent across all services ✅
```

---

## Running Locally

### Prerequisites
```bash
Java 17+
Maven 3.8+
Docker + Docker Compose
```

### Start Infrastructure
```bash
# Start Kafka, Zookeeper, Redis, PostgreSQL, Zipkin
docker-compose up -d
```

### Start Services (in order)
```bash
# 1. Eureka Server
cd eureka-server && mvn spring-boot:run

# 2. Auth Service
cd auth-service && mvn spring-boot:run

# 3. User Service
cd user-service && mvn spring-boot:run

# 4. Product/Inventory Service
cd inventory-service && mvn spring-boot:run

# 5. Order Service
cd order-service && mvn spring-boot:run

# 6. Billing Service
cd billing-service && mvn spring-boot:run

# 7. Notification Service
cd notification-service && mvn spring-boot:run

# 8. API Gateway (last)
cd api-gateway && mvn spring-boot:run
```

### Service URLs
```
API Gateway:    http://localhost:8080
Eureka:         http://localhost:8761
Zipkin:         http://localhost:9411
Kafka UI:       http://localhost:9000
```

---

## Environment Variables

```properties
# JWT
JWT_SECRET=your-256-bit-secret
JWT_ACCESS_EXPIRATION=900000       # 15 minutes
JWT_REFRESH_EXPIRATION=604800000   # 7 days

# Database (per service)
DB_HOST=localhost
DB_PORT=5432
DB_NAME=shopx_orders     # change per service
DB_USERNAME=postgres
DB_PASSWORD=password

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379

# Kafka
KAFKA_BOOTSTRAP_SERVERS=localhost:9092

# Zipkin
ZIPKIN_BASE_URL=http://localhost:9411

# Eureka
EUREKA_SERVER_URL=http://localhost:8761/eureka/
```

---

## Project Structure

```
shopx-ecom/
├── api-gateway/
├── eureka-server/
├── auth-service/
├── user-service/
├── inventory-service/
├── order-service/
├── billing-service/
├── notification-service/
├── common/                  ← shared DTOs, events, proto files
│   ├── events/
│   │   ├── OrderCreatedEvent.java
│   │   ├── PaymentSuccessEvent.java
│   │   └── PaymentFailedEvent.java
│   └── proto/
│       ├── inventory.proto
│       └── user.proto
├── docker-compose.yml
└── README.md
```

---

## Author

**Sameer Bilal Shaikh**  
Backend Engineer | Distributed Systems  
[LinkedIn](#) | [GitHub](#) | [Blog](#)

> *"Built this to go beyond CRUD — real distributed systems, real failure handling, real production patterns. Every technology here solves a specific problem. Nothing is added for the sake of a resume."*
