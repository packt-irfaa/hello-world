# Chapter 3: Baseline application & Domain Modeling

## Introduction

Just as you cannot learn to be a detective in a town with zero crime, you cannot learn to diagnose problems in a system that never breaks. Because simple applications lack the latency and concurrency issues found in production, they fail to demonstrate the true utility of observability. To bridge this gap, this chapter introduces OpenTel E-Commerce, a complex microservices application designed to validate why observability is not just useful, but necessary.

We did not build this application to be easy, we built it to be real. It serves as a baseline of chaos, simulating how distributed services inevitably grow apart and fail independently. You will face an environment where stock reservations time out, queries degrade under load, and errors cascade. It is only by operating a system that actually breaks that we can understand why observability is essential.

---

## Architecture Overview

OpenTel E-Commerce follows a standard microservices architecture, emphasizing realistic separation of concerns rather than simplicity.

```
<<Placeholder for Architecture image>>
```

The backend implementation utilizes Rust and the Axum web framework, structured around a central gateway and three core services. By relying on HTTP REST APIs for communication, the system replicates the isolation found in modern production environments. This architecture leverages Rust’s type safety for performance while retaining the complexities of distributed coordination creating a system that is robust locally but challenging globally.

*   **OtelMart Service (Gateway)**: Acts as the entry point and implements the **Backend-for-Frontend (BFF)** pattern. Rather than exposing raw backend APIs to the client, the BFF aggregates, filters, and shapes data specifically for the frontend's needs. It also handles authentication and routes requests to the appropriate backend services via HTTP.
*   **Products Service**: Manages the catalog, requiring support for complex search logic due to its read-heavy nature.
*   **Inventory Service**: Serves as the source of truth for stock, handling high-concurrency write operations (reservations) and ensuring strict consistency.
*   **Orders Service**: Acts as the orchestrator, managing the checkout lifecycle and coordinating data across products, inventory, and payments.

All services utilize a single PostgreSQL instance but remain strictly isolated through dedicated database schemas. While some production architectures employ physically separate databases, this logical isolation provides a comparable boundary. Each service operates as a completely independent process within its own container, listening on a distinct port and communicating exclusively via HTTP REST APIs.

This schema separation covering products, inventory, orders, and users is enforced at the connection level. For instance, the products service (port 3001) configures its pool with SET `search_path` to products, while the orders service (port 3003) is similarly restricted to its own domain. This strict configuration prevents services from circumventing architectural boundaries via direct table joins. Consequently, if the Orders service requires product data, it must call the Products service API's. This design creates the necessary infrastructure to demonstrate distributed tracing and highlights the fault isolation challenges inherent in distributed systems.

## Containerized Local Development

To run the full application locally in a production-like setup, we use Docker Compose to orchestrate all services on a single machine. This allows each service to run in its own container with well-defined network boundaries and isolation, closely matching how the system behaves in production while remaining practical for local development.

Our `docker-compose.yml` file defines the complete application stack:

**Database Service (PostgreSQL)**
- Runs PostgreSQL 17 with automatic schema initialization
- On startup, it executes SQL scripts that create separate schemas for `products`, `inventory`, `orders`, and `users`
- Persists data in a Docker volume (`postgres_data`) so the database survives container restarts
- Exposes port 5433 locally (mapped from internal port 5432)
- Includes health checks to ensure it's ready before services connect

**Application Services (Products, Inventory, Orders)**
- Each service builds from its own `Dockerfile`, creating independent container images
- Products Service listens on port 3001
- Inventory Service listens on port 3002  
- Orders Service listens on port 3003
- All receive the same database URL but, due to their schema configuration, can only access their own schema
- All run on a shared Docker network (`app-network`), allowing them to communicate via hostnames (`http://products:3001`)
- Each service depends on PostgreSQL being healthy before starting
- Automatic restart ensures services recover from crashes

**Data Ingestion Service**
- Runs after services start to populate initial test data
- Executes SQL scripts to load products, inventory, and sample orders

**Network Isolation**
- Services communicate via an internal Docker network, not the host network
- This replicates how microservices communicate in production without exposing every service port globally
- Services use container hostnames (e.g., `http://orders:3003`) rather than `localhost`, reinforcing that they are separate systems

**Building and Running Locally**
To run the entire system:
```bash
docker-compose up --build
```

To stop the system:
```bash
docker-compose down
```

This approach enables developers to experience the same distributed challenges that operators face in production: network boundaries, service startup ordering, partial failures, and schema isolation. When a service fails, you must diagnose the problem across multiple containers. When a query times out, you must determine whether the issue is in that service or a downstream dependency. This is precisely the kind of environment where observability becomes invaluable.

## Domain Modeling: From Database to Rust

A well designed domain model bridges the gap between your database schema and your application code. In OpenTel E-Commerce, we balance the strict correctness required in a financial system with the flexibility needed for a modern product catalog.

### Identities and Types

One of the first challenges in a distributed system is identity. We employ a **dual identifier strategy**:

*   **Internal ID (`i32`)**: Used strictly for database joins and foreign keys, ensuring fast and index-friendly operations. However, this numeric ID can be duplicated across different databases.
*   **External ID (`UUID`)**: Exposed to the frontend and other services, preventing enumeration attacks and allowing ID generation client-side or in upstream services without database round-trips. Unlike numeric IDs, UUIDs are guaranteed to be unique across all services and databases.

In our Rust code, this duality is represented as follows:

```rust
pub struct Product {
    // Internal use only
    pub id: i32,
    
    // Public identifier
    pub uuid: Uuid,
    
    // ...
}
```

This UUID based approach is critical for distributed systems. When the Orders Service requests product details, it passes the product UUID to the Products Service. When that request is logged or traced across both services, the same UUID appears in both contexts, creating a natural link between them. When debugging a failure, you can search logs across *all* services for a single UUID and reconstruct the entire request flow a single identifier that spans service boundaries. Even when services use completely different databases, the UUID remains the consistent identifier. This simplifies debugging and logging without exposing internal database IDs or creating security vulnerabilities.

We also strictly avoid floating-point math for monetary values. While `f64` is suitable for graphics, it can introduce unacceptable rounding errors in pricing. Instead, we use `bigdecimal::BigDecimal` (mapping to PostgreSQL's `DECIMAL`) to ensure that $10.00 is precisely $10.00, avoiding discrepancies like $10.00000001.

### The Services

#### Products Service
The catalog is hierarchical, utilizing a **materialized path pattern** (e.g., path `1/23/45`) for efficient retrieval of entire category subtrees in a single query. Products leverage PostgreSQL's `GIN` indexes to support full-text search across names and descriptions, providing search engine like capabilities directly within the primary database.

#### Inventory Service
The Inventory service prioritizes consistency. The `available_quantity` is not merely a field we update; it is a **generated column** in PostgreSQL (`stock_quantity - reserved_quantity`). A generated column is a database-computed field: PostgreSQL *automatically* recalculates `available_quantity` every time `stock_quantity` or `reserved_quantity` changes. This guarantees that our "available" stock is mathematically tied to our physical stock and reservations—a formula enforced by the database itself, not by application code. We can never drift out of sync due to a buggy update statement or forgotten code path.

#### Orders Service
The Orders service represents the true complexity of a real-world transaction system. An "Order" is far more than a simple database row, it is a point-in-time snapshot of a binding agreement between the customer and the business.

The design decision here reveals a crucial principle that separates theoretical systems from production ones **denormalization for immutability**. When an order is created, we do not simply store a `product_id` reference. Instead, we copy the `product_name`, `sku`, and `unit_price` directly into the `order_items` table. At first glance, this seems wasteful. Why duplicate data?

Consider what happens when a product's price changes, an everyday occurrence in retail. A month later, a customer reviews their purchase history. If we had only stored the `product_id`, the system would reconstruct the price from the *current* product record. The customer sees they paid $19.99, when in reality they paid $29.99 at the time of purchase. When they contact support asking, "Why does the order say $29.99 but the product page shows $19.99?", you have created an unnecessary support case and eroded customer trust.

In production systems, this denormalized snapshot is non-negotiable for three critical reasons:

- **Auditability**: Every order is immutable proof of what was sold and at what price. The data cannot change retroactively.
- **Compliance**: Jurisdictions worldwide require this for tax records, consumer protection regulations, and dispute resolution.
- **Customer Service**: When a customer disputes a charge, you must know *exactly* what they received and what they paid—not reconstruct it from tables that have since been modified.

The cost of duplicating a few fields pales in comparison to the operational cost of debugging a customer complaint that spans months of data mutations, or worse, defending yourself against a regulatory inquiry.

---

## Inter-Service Communication: The Distributed Transaction

Microservices must communicate effectively to fulfill complex business workflows. In OpenTel E-Commerce, the most critical flow is **order creation**. This process is not a simple insert. It constitutes a distributed transaction that spans three services and multiple failure domains.

### The "Happy Path"

When a user clicks "Buy", the **Orders Service** takes on the role of orchestrator:

1.  **Validate**: It queries the **Products Service** for details (price, name) to ensure the user is not purchasing a non-existent item.
2.  **Reserve**: It instructs the **Inventory Service** to hold the stock: "Reserve 1 unit of Item X".
3.  **Commit**: Only after the stock is reserved does it insert the order into its own database.
4.  **Confirm**: Finally, it instructs Inventory to permanently deduct the stock.

### The "Unhappy Path"

In a monolithic architecture, this entire sequence would be encapsulated within a single database transaction (`BEGIN...COMMIT`). If any part of the process failed, the database would automatically roll back all changes.

In a distributed system, however, **there is no global rollback**.

If the Orders Service reserves stock but fails to save the order (due to a full database disk or a code bug), we end up with an orphaned stock reservation. The Inventory Service believes the item is sold, but no corresponding order exists.

To manage this, we implement a **Saga Pattern**. A Saga is a pattern for managing distributed transactions: instead of relying on a single database rollback, each service compensates for failures by calling a "compensating operation" on dependent services. When a step fails, we manually trigger rollback requests to undo previous steps. Here is the critical logic in our `create_order` handler:

```rust
// 1. Reserve stock
let stock_response = match reserve_stock(&state, product.eid, item.quantity).await {
    Ok(resp) => resp,
    Err(e) => {
        // If reservation fails, we must manually undo any PRIOR reservations
        release_all_reserved_stock(&state, &reserved_items).await;
        return e;
    }
};

// ...

// 2. Try to save order to DB
if let Err(e) = tx.commit().await {
    // CRITICAL: The DB commit failed, but we already told Inventory to reserve!
    // We must manually call the Inventory API to undo the reservation.
    release_all_reserved_stock(&state, &reserved_items).await;
    return internal_error(e);
}
```

This `release_all_reserved_stock` function creates a network call to the Inventory Service. **But what if that network call fails?**

If the network is down when we try to release stock, the system enters an inconsistent state. We have "leaked" inventory, items are marked as reserved in the Inventory Service, but no corresponding order exists in the Orders Service. From the customer's perspective, the checkout failed, so they try again. But the inventory system still sees the first reservation, potentially double booking the item. This is a classic distributed system problem, and it's exactly the kind of *silent* failure that creates support nightmares—customer complaints about "phantom reservations" inventory mismatches, and lost sales. Without observability, you have no way to detect or diagnose these leaks.

## The Current State of Logging

Currently, we try to manage this chaos with `println!`.

```rust
println!("Creating order for customer: {}", request.customer_email);
// ...
eprintln!("Failed to fetch product {}: {}", product_uuid, e);
```

When things work, this is fine. When they fail, it's a disaster.

### The "Black Box" Problem

Imagine an order fails. You see this in the Orders Service logs:
`[ERROR] Failed to reserve stock for product a7f3c5...`

But *why*? Was the Inventory Service down? Was it slow? Did it return a 400 or a 500? To find out, you have to SSH into the Inventory Service, guess the approximate timestamp (hoping clocks are synced), and grep for that UUID.

If the request touched 5 services, you are grepping 5 log files.

### Structured Logging (A Half Step)

We can improve this with **structured logging** using the `tracing` crate. Instead of just text, we log JSON objects with context.

```rust
info!(
    customer_email = %request.customer_email,
    item_count = request.items.len(),
    "Received order creation request"
);
```

This gives us searchable fields (e.g., `item_count > 5`), but it still lacks **causality**. It doesn't tell us that *this* log in the Inventory Service was caused by *that* request in the Orders Service.

To solve this, we need **Correlation IDs**—a unique ID passed in HTTP headers (`X-Correlation-ID`) from service to service.

```rust
// When Gateway receives a request, create a correlation ID
let correlation_id = Uuid::new_v4();

// Pass it to Orders Service
let orders_response = state.http_client()
    .post(&orders_url)
    .header("X-Correlation-ID", correlation_id.to_string())
    .send()
    .await?;

// Orders Service receives it and passes it to Inventory
let inventory_response = state.http_client()
    .post(&inventory_url)
    .header("X-Correlation-ID", correlation_id.to_string())
    .send()
    .await?;
```

With correlation IDs, a single request can be traced across all services. When you search logs for a specific correlation ID, you'll find entries in Gateway, Orders, and Inventory—the entire request flow in one place. Correlation IDs are the first step towards observability. They allow us to stitch logs together. But even with correlation IDs, we are missing the **time dimension**. We know *what* happened, but we don't know *where the time went*. Which service was slow? Did the delay happen during network I/O or database queries? Logs tell us *events*, but not *durations*.

## Why Observability is Non-Negotiable

In OpenTel E-Commerce, a slow checkout could be caused by:
1.  Slow SQL query in Orders.
2.  Network latency between Orders and Products.
3.  Lock contention in Inventory's database.
4.  Garbage collection pause in the Gateway.

Logs—even structured ones—are discrete points in time. They tell you *what* happened at specific moments, but they don't show the continuous timeline between those moments. They don't show the *gaps*. With correlation IDs, you know all the services involved, but not which one caused the delay.

This is why we need **Distributed Tracing**. Tracing turns those discrete log points into a continuous timeline. Instead of just events, traces record durations: how long did the request spend in each service? How long was the network call? Tracing visualizes the request flow and automatically highlights bottlenecks. You see that the 2-second delay happened *during* the network call to Inventory, not during the database insert. You see that Inventory spent 1.8 seconds waiting for a lock, while Products only took 0.1 seconds.

## Summary

We have established our baseline:
*   **A Realistic Architecture**: Independent services with strict boundaries.
*   **Real Complexity**: Partial failures, distributed transactions, and network dependencies.
*   **The Observability Gap**: We have logs, but we lack the ability to see the system as a whole.

The "chaos" is baked in. Now we need the tools to tame it. Correlation IDs help link requests across services, but they require manual implementation at every service boundary. Structured logging helps search for events, but it doesn't capture timing. In the next chapter, **Chapter 4: Telemetry Skeleton & Trace Taxonomy**, we will introduce **OpenTelemetry**, which automates trace propagation and timing information. OpenTelemetry eliminates the manual work of passing correlation IDs by hand and measuring latencies with stop watches. Instead, it instruments your code to automatically capture the complete request flow—which service called which, how long each step took, and when failures occurred—all without changing your business logic.
