# Chapter 5: Data Layer Instrumentation

## Introduction: Following the Request into SQL (Hop 3)

In Chapter 4, we instrumented the HTTP boundaries—the first two hops in our request journey:
- **Hop 1:** Gateway receives the checkout request (span name: `checkout: POST /checkout`)
- **Hop 2:** Gateway calls the Orders service (span name: `orders: POST /orders`)

But when we examined our first checkout trace, we encountered a familiar frustration: the `orders: POST /orders` span took 800ms, yet everything beneath it was invisible. The handler remained a black box. We knew the service was slow, but not *what inside it* was slow.

This chapter follows the request into **Hop 3: Database queries**. We'll instrument SQLx to produce first-class database spans that reveal the true structure of data access—which queries run, in what order, and how long each takes. By the end, every database operation in OpenTel E-Commerce will appear as a properly attributed span, nested under its parent request, following OpenTelemetry's semantic conventions.

**Quick recap from Chapter 4:**

In Chapter 4, we built the **telemetry skeleton** the foundational infrastructure that makes observability possible:
- `TraceLayer` middleware creates spans for incoming HTTP requests
- `#[instrument]` macro adds spans to handlers and business logic
- `traceparent` header propagates trace context across services (gateway -> orders -> products)
- Jaeger visualizes the resulting distributed traces

But our traces had a blind spot: database operations were invisible. This chapter adds the "muscles and organs"—the database operations that do the actual work.

**In this chapter, we will cover the following key topics:**

- OpenTelemetry semantic conventions for database spans
- Designing a stable, low-cardinality span schema
- Understanding SQLx and the repository layer pattern
- Instrumenting read queries (single-row, lists, not-found cases)
- Instrumenting write queries (reservations, order creation)
- Wrapping transactions as first-class spans
- Connection pool visibility
- Viewing database spans in Jaeger

**What this chapter does not cover:**

This chapter focuses on *visibility*, not optimization. Query tuning, index analysis, lock contention diagnosis, and connection pool sizing are intentionally deferred to Chapter 11 (Database Bottlenecks). Here, we establish the instrumentation foundation that makes such diagnosis possible.

---

## 5.1 OpenTelemetry Semantic Conventions for Databases

First, we need to follow a shared "vocabulary" called **semantic conventions**. These are just standard names for common pieces of information (like database names or host addresses). When every service uses the same names, tools like Jaeger can easily group and search your data, regardless of which programming language was used.

### Why Conventions Matter

Without conventions, teams invent their own schemas:

- Team A uses `query.duration` and `table_name`
- Team B uses `db.latency_ms` and `db.table`
- Team C uses `sql.time` and `collection`

Without these standards, every team might use different names, making it impossible to see the big picture. Semantic conventions ensure everyone speaks the same language.

### Core Database Attributes

The following attributes form the foundation of database span instrumentation:

| Attribute | Type | Description | Example |
|-----------|------|-------------|---------|
| `db.system.name` | string | The database management system | `postgresql` |
| `db.namespace` | string | Database name, schema, or catalog | `products` |
| `db.operation.name` | string | The operation being performed | `SELECT`, `INSERT`, `UPDATE` |
| `db.collection.name` | string | The table or collection | `products`, `inventory` |
| `db.query.text` | string | The query (sanitized) | `SELECT * FROM products WHERE uuid = $1` |
| `server.address` | string | Database host | `postgres`, `localhost` |
| `server.port` | int | Database port | `5432` |

> [!TIP]
> Avoid hardcoding strings like `db.system.name`. Instead, use the `opentelemetry-semantic-conventions` crate. This prevents typos and ensures you are using the names the industry has agreed upon
> ```rust
> // ...
> use opentelemetry_semantic_conventions as semconv;
>
> // Correct implementation
> let attributes = vec![
>     KeyValue::new(semconv::trace::DB_SYSTEM, "postgresql"),
>     KeyValue::new(semconv::trace::DB_NAME, "users_db"),
> ];
> ```

All attributes used in this chapter are from OpenTelemetry **stable** semantic conventions unless explicitly noted. You can rely on these names remaining consistent across OpenTelemetry SDK updates.

### Attribute Requirements

Not all attributes are required on every span. OpenTelemetry defines requirement levels:

- **Required**: Must always be present
- **Conditionally Required**: Must be present if a condition is met
- **Recommended**: Should be present if available
- **Opt-In**: Only include if explicitly enabled (often due to sensitivity)

For SQL databases, the specification recommends:

- `db.system.name`: Always include (we use `postgresql`)
- `db.namespace`: Include if available (our schema name)
- `db.operation.name`: Recommended if a single operation describes the call
- `db.collection.name`: Recommended if the operation targets a specific table
- `db.query.text`: Recommended, but must be sanitized

### The Cardinality Rule

One principle appears throughout semantic conventions is **low cardinality**. Span names and attribute values should not include highly variable data like UUIDs, timestamps, or user input. This is to prevent the observability backend from being overwhelmed with too many unique time series.

**Wrong:**
```
span_name: "SELECT * FROM products WHERE uuid = 'a7f3c5d2-1234-5678-9abc-def012345678'"
```

**Right:**
```
span_name: "SELECT products"
attribute db.query.text: "SELECT * FROM products WHERE uuid = $1"
```

The span name groups similar operations for aggregation. The attribute carries detail for individual analysis. This distinction is essential for backends that aggregate spans by name—high-cardinality names create millions of unique time series and degrade performance.

### Query Text Sanitization

The `db.query.text` attribute deserves special attention. You might worry that logging queries exposes sensitive data like passwords or emails. Fortunately, `sqlx` uses **parameterized queries**, which separate the SQL structure from the data.

Parameterized queries prevent sensitive data from leaking into traces. Instead of the raw query:

```sql
SELECT * FROM users WHERE email = 'customer@example.com' AND password_hash = 'abc123...'
```

Traces capture the safe, parameterized template:

```sql
SELECT * FROM users WHERE email = $1 AND password_hash = $2
```

By using SQLx's parameterized queries (both macros: `query_as!` and `query!`, and runtime forms), your query text is safe to capture by default. Traces record only the template (e.g., `WHERE email = $1`), while sensitive values are sent directly to the database driver. This provides full visibility into *what* is happening without ever leaking *who* is involved.

## 5.2 Designing a Stable Database Span Schema

With conventions understood, we now design a **telemetry schema** tailored to OpenTel E-Commerce. This schema balances standard compliance with our specific needs.

> [!INFO]
> A **telemetry schema** is a set of rules that define how telemetry data is collected, processed, and stored. It includes span naming conventions, attribute naming conventions, and other rules that define how telemetry data is collected.

### Span Naming Strategy

Span names should be low-cardinality and describe the operation's intent. We adopt the pattern:

```
{operation} {target}
```

Where `{operation}` is the SQL verb and `{target}` is the table or a descriptive noun. Span names use SQL verbs (`SELECT`, `INSERT`, `UPDATE`) combined with a logical target (table or entity), not query intent or business action names. This keeps names stable even as business logic evolves.

**Examples:**

| Operation | Span Name |
|-----------|-----------|
| Fetch single product | `SELECT products` |
| List products with filter | `SELECT products` |
| Reserve inventory | `SELECT reserve_stock` |
| Create order | `INSERT orders` |
| Create order items (batch) | `INSERT order_items` |
| Checkout transaction | `TRANSACTION checkout` |

Note that different queries on the same table (e.g., `SELECT * FROM products` vs `SELECT count(*) FROM products`) share the **same** span name: `SELECT products`. This allows you to aggregate performance metrics for the table as a whole. Differentiation comes from attributes, not the name, for example, you look at the `db.query.text` and other attributes to distinguish between them.

### Required Attributes for All Services

Every database span in OpenTel E-Commerce includes:

```rust
fields(
    db.system = "postgresql",
    server.address = "postgres",  // Docker service name
    server.port = 5432,
)
```
> [!NOTE] those attributes are not required for all services, but they are standard for database client spans.

In Rust, you typically define these attributes in two places:
1. Resource Attributes: Global info about your service (e.g., service name, version).
2. Span Attributes: Specific info about an operation (e.g., a SQL query).

```rust
// Global Resource Attributes (The "Service" Level)
let resource = Resource::new(vec![
    KeyValue::new("service.name", "my-rust-service"),
    KeyValue::new("service.version", "1.0.0"),
    KeyValue::new("deployment.environment", "production"),
]);

let provider = TracerProvider::builder()
    .with_resource(resource)
    // ... rest of setup
    .build();
```



### Service-Specific Attributes

Each service adds its schema context:

| Service | `db.namespace` | Tables |
|---------|----------------|--------|
| Products | `products` | `products`, `categories`, `ratings` |
| Inventory | `inventory` | `product_inventory`, `product_pricing` |
| Orders | `orders` | `orders`, `order_items`, `payments`, `shipping_addresses` |
| OtelMart | `users` | `users`, `user_sessions`, `user_addresses` |

### Business Context Attributes

Beyond standard conventions, we add business context using a custom namespace (`otelmart.*`):

| Attribute | Purpose | Example |
|-----------|---------|---------|
| `otelmart.product.uuid` | Link span to product entity | `a7f3c5d2-...` |
| `otelmart.order.id` | Link span to order | `ord_12345` |
| `otelmart.quantity` | Operation magnitude | `5` |
| `otelmart.customer.email_hash` | Pseudonymized customer identifier | `sha256:abc...` |

These attributes enable filtering traces by business entity without exposing PII.

### What We Explicitly Exclude

- **Raw SQL with literals**: Never include unparameterized queries
- **Customer PII**: No plain text emails, names, or addresses in spans
- **Internal database IDs**: Use UUIDs, not auto-increment integers
- **High-cardinality span names**: No UUIDs or timestamps in span names

## 5.3 SQLx Instrumentation Strategy

To build effective traces, we must first understand the baseline capabilities of `SQLx` and identify where custom instrumentation is necessary.

### SQLx Configuration in OpenTel E-Commerce

Our `Cargo.toml` configures SQLx:

```toml
sqlx = { version = "0.8.6", features = [
    "runtime-tokio-rustls",
    "postgres",
    "chrono",
    "uuid",
    "json",
    "bigdecimal",
    "migrate"
]}
```
Notice what's *not* included: the `tracing` feature. While SQLx can automatically emit spans, they are often too noisy for production. They use the full SQL statement as the span name (leading to high cardinality) and lack the business specific attributes we need.

### Why We Instrument Manually

Just as we manually instrumented HTTP handlers in Chapter 4 (rather than relying solely on automatic middleware), we choose manual instrumentation at the repository layer for several reasons:

1. **Stable, Low-Cardinality Names**: Instead of volatile SQL statements (`SELECT * FROM products ...`), we get stable names (`SELECT products`) that align with our schema design.
2. **Rich Business Context**: We can inject attributes like `otelmart.product.uuid` or `otelmart.cart.total` that the database driver simply doesn't know about.
3. **Security by Design**: We explicitly opt-in to recording data, preventing accidental PII leaks (like email addresses in raw queries).
4. **Compile-Time Guarantees**: The `#[instrument]` macro validates our syntax at build time, catching typoed attribute names before they reach production.
5. **Rust Philosophy Alignment**: Explicit over implicit

### Compile-Time Verification

One of the SQLx's strongest features is `query!` and `query_as!`. These macros connect to your development database at compile time, verifying that:
1. The SQL syntax is valid
2. All tables and columns exist
3. Parameter types match the column types (e.g., passing a Uuid to a UUID column)
4. Result types match your struct fields

This means if you rename a database column but forget to update the SQL query, your code *won't compile*. This safety net catches an entire class of runtime errors before they ever deploy.

> [!NOTE] OpenTel E-Commerce uses the runtime `query_as::<_, T>()` form rather than the `query_as!()` macro. While the macro provides compile-time checking, the runtime form offers more flexibility when queries use complex JOINs, CTEs, or dynamic `QueryBuilder` construction all common in our services. The tradeoff is intentional: we gain query composition flexibility at the cost of compile-time SQL verification.

### The Repository Layer Pattern

OpenTel E-Commerce uses a repository pattern. Handlers call repository functions, which execute SQL:

```
Handler (HTTP layer)
    ↓
Repository function (business logic + SQL)
    ↓
SQLx query execution
    ↓
PostgreSQL
```

We instrument at the repository layer because it captures **business intent**. A function named `get_product_by_uuid` carries semantic meaning, whereas the raw SQL `SELECT * ...` is just an implementation detail.

### The Database Wrapper

Each service wraps its connection pool:

```rust
// ...

impl Database {
    pub async fn new(database_url: &str, max_connections: u32) -> Result<Self> {
        // 1. Configure the connection options
        let mut connect_opts = PgConnectOptions::from_str(database_url)?
            .disable_statement_logging() // vital for avoiding duplicate logs
            .application_name("products-service"); // Set application name for identification

        // 2. Create the connection pool
        let pool = PgPoolOptions::new()
            .max_connections(max_connections)
            // 3. Configure per-connection behavior
            .after_connect(|conn, _meta| {
                Box::pin(async move {
                    sqlx::query("SET search_path TO products, public")
                        .execute(conn)
                        .await?;
                    Ok(())
                })
            })
            .connect_with(connect_opts)
            .await?;

        Ok(Self { pool })
    }
    // ...
}
```

> [!INFO]
> `SET search_path TO products, public` tells the database: "When I say `SELECT * FROM table`, look in the `products` schema first".

The `after_connect` hook sets the schema search path, ensuring each service only accesses its own tables by default. Each service configures its own path:

| Service | Search Path | Why |
|---------|-------------|-----|
| Products | `products, public` | Only needs its own schema |
| Inventory | `inventory, products, public` | Joins with products for pricing views |
| Orders | `orders, products, public` | References product data in order items |
| OtelMart | `users, public` | User management only |

This isolation is invisible to traces but convenient for development, acting as a safeguard against accidental cross-schema queries. (Note: Real database security relies on PostgreSQL roles and `GRANT`/`REVOKE` permissions, which we enforce at the user level, not just the search path.)

## 5.4 Connecting Database Spans to HTTP Spans

A complete trace requires connecting database operations to the incoming HTTP request. Extending the HTTP boundary established spans using `TraceLayer` in Chapter 4 (specifically 4.6) into the data layer allows us to see the full picture. Now we're adding database spans. How do they connect?

### Automatic Context Propagation Within a Service

The `tracing` crate maintains span context within an `async` task. As we learned in Chapter 4, the `#[instrument]` macro automatically preserves span context across `await` points using task-local storage. When you call an instrumented repository function from an instrumented handler, the spans automatically form a parent-child relationship:

```rust
// Handler span created by TraceLayer (from Chapter 4, see 4.6)
pub async fn get_product_by_id(
    State(pool): State<PgPool>,
    Path(uuid): Path<Uuid>,
) -> impl IntoResponse {
    // When we call this, the repository span becomes a CHILD of the handler span
    match db::get_product_by_uuid(&pool, uuid).await {
        Ok(Some(product)) => (StatusCode::OK, Json(product)).into_response(),
        Ok(None) => not_found_error("Product not found", /* ... */),
        Err(e) => internal_error("Failed to fetch product", e.to_string()),
    }
}
```

The `#[instrument]` macro on `get_product_by_uuid` automatically detects the current span (the handler) and creates its span as a child. No explicit propagation code is required within a single service.

### Cross-Service Propagation

For HTTP calls between services (e.g., Orders calling Inventory), propagation still uses the `traceparent` header as established in Chapter 4. The database spans in the downstream service will be children of that service's HTTP handler span, which is itself linked to the upstream trace via the propagated context.

```
Orders Service                          Inventory Service
─────────────────                       ─────────────────
POST /orders (handler span)
  │
  ├─► TRANSACTION span
  │     └─► INSERT orders span
  │
  └─► HTTP POST /reserve ──────────────► POST /reserve (handler span)
        [traceparent header]               │
                                           └─► SELECT reserve_stock span
```

The key insight: **within a service, context flows automatically through the `async` runtime. Across services, context flows through HTTP headers.**

> [!NOTE]
> The `axum-tracing-opentelemetry` (see Ch 4, #4.6) crate provides a `TraceLayer` that automatically instruments HTTP requests, responses and passes the span context to the downstream services via the `traceparent` header.

## 5.5 Instrumenting Read Queries

Let's instrument the Products service, starting with the simplest case: fetching a single product.

### Single-Row Fetch

```rust
// products/src/db/repository.rs
// ... (imports)

#[instrument(
    name = "SELECT products",
    skip(pool),
    fields(
        db.system.name = "postgresql",
        db.namespace = "products",
        db.operation.name = "SELECT",
        db.collection.name = "products",
        db.query.text = "SELECT * FROM products WHERE uuid = $1",
        otelmart.product.uuid = %uuid
    )
)]
pub async fn get_product_by_uuid(
    pool: &PgPool,
    uuid: Uuid,
) -> Result<Option<ProductDetail>, sqlx::Error> {
    // ... (implementation)
}
```

**Key decisions:**

- **`name = "SELECT products"`**: Low-cardinality span name following `{operation} {target}`
- **`skip(pool)`**: Connection pools aren't useful in traces
- **`db.*` attributes**: Standard semantic conventions
- **`otelmart.product.uuid`**: Business context linking this span to the entity

### List Queries

For queries returning multiple rows:

```rust
// products/src/db/repository.rs
// ... (imports)

#[instrument(
    name = "SELECT products",
    skip(pool, params),
    fields(
        db.system.name = "postgresql",
        db.namespace = "products",
        db.operation.name = "SELECT",
        db.collection.name = "products",
        db.query.text = "SELECT * FROM products WHERE ... ORDER BY ... LIMIT ... OFFSET ...",
        otelmart.page = page,
        otelmart.page_size = page_size,
        otelmart.category.id = ?params.category_id
    )
)]
pub async fn list_products(
    pool: &PgPool,
    params: &ProductQueryParams,
    page: i32,
    page_size: i32,
    offset: i32,
) -> Result<(Vec<ProductWithRating>, i64), sqlx::Error> {
    // ... (implementation)
}
```

Both `get_product_by_uuid` and `list_products` produce spans named `SELECT products`. The difference is visible in attributes: one has `otelmart.product.uuid`, the other has `otelmart.page`, `otelmart.page_size`, and `otelmart.category.id`.

### Handling "Not Found"

A missing product is not a database error, it's a valid query result. The repository simply returns `None`, and the handler decides how to respond:

```rust
// Repository: returns Option — no error for missing data
pub async fn get_product_by_uuid(
    pool: &PgPool,
    uuid: Uuid,
) -> Result<Option<ProductDetail>, sqlx::Error> {
    sqlx::query_as::<_, ProductDetail>(/* ... */)
        .bind(uuid)
        .fetch_optional(pool)  // Returns None if not found
        .await
}

// Handler: interprets None as 404
pub async fn get_product_by_id(
    State(pool): State<PgPool>,
    Path(uuid): Path<Uuid>,
) -> impl IntoResponse {
    match db::get_product_by_uuid(&pool, uuid).await {
        Ok(Some(product)) => (StatusCode::OK, Json(product)).into_response(),
        Ok(None) => not_found_error(
            "Product not found",
            serde_json::json!({"uuid": uuid.to_string()}),
        ),
        Err(e) => internal_error("Failed to fetch product", e.to_string()),
    }
}
```

The database span completes successfully regardless, because `fetch_optional` returned a valid result. The "not found" condition is an HTTP concern handled at the handler layer, not a database error.

## 5.6 Instrumenting Write Queries

Write operations, `INSERT`, `UPDATE`, and `DELETE`, require extra scrutiny. They mutate state, risk lock contention, and carry significant business weight upon success or failure.

### Inventory Reservation

The `reserve_stock` function is the most contention-prone operation in OpenTel E-Commerce. It delegates to a PostgreSQL stored function that atomically checks available stock and creates a reservation:

```rust
// inventory/src/db/repository.rs
// ... (imports)

#[instrument(
    name = "SELECT reserve_stock",
    skip(pool),
    fields(
        db.system.name = "postgresql",
        db.namespace = "inventory",
        db.operation.name = "SELECT",
        db.collection.name = "product_inventory",
        db.query.text = "SELECT reserve_stock($1, $2)",
        otelmart.product.uuid = %product_uuid,
        otelmart.quantity = quantity
    )
)]
pub async fn reserve_stock(
    pool: &PgPool,
    product_uuid: Uuid,
    quantity: i32,
) -> Result<bool, sqlx::Error> {
    sqlx::query_scalar::<_, bool>(r#"SELECT reserve_stock($1, $2)"#) // ... (implementation)
}
```

**Key decisions:**
- **Atomic Concurrency**: The `reserve_stock()` stored function encapsulates the check-and-update logic, preventing race conditions that application-level code might face under high load.
- **Consistent Mapping**: We set `db.operation.name = "SELECT"` to match the SQL verb used to invoke the stored function, even though it mutates state internally.
- **Business Logic as Data**: Returning `bool` allows us to handle "out of stock" as a valid business outcome. Since the query itself succeeded, we avoid marking the span as an error, keeping our technical failure metrics clean.

### Recording Values After Execution with `tracing::field::Empty`

Some span attributes aren't known until after the query executes, like how many rows were affected. The `tracing` crate handles this with `tracing::field::Empty`, you declare the field in `#[instrument]`, then record its value later using `Span::current().record()`.

The `update_stock` function demonstrates this pattern:

```rust
// inventory/src/db/repository.rs
// ... (imports)

#[instrument(
    name = "UPDATE inventory",
    skip(pool),
    fields(
        db.system.name = "postgresql",
        db.namespace = "inventory",
        db.operation.name = "UPDATE",
        db.collection.name = "product_inventory",
        db.query.text = "UPDATE product_inventory SET stock_quantity = $1 ... WHERE product_uuid = $4",
        otelmart.product.uuid = %product_uuid,
        otelmart.quantity = quantity,
        db.response.returned_rows = tracing::field::Empty  // Declared now, recorded later
    )
)]
pub async fn update_stock(
    pool: &PgPool,
    product_uuid: Uuid,
    quantity: i32,
    reorder_level: Option<i32>,
    reorder_quantity: Option<i32>,
) -> Result<Option<i32>, sqlx::Error> {
    let result = // ... (implementation)

    // Record the actual rows affected after execution
    let rows = if result.is_some() { 1 } else { 0 };
    Span::current().record("db.response.returned_rows", rows);

    Ok(result.map(|row| row.get("available_quantity")))
}
```

**Why `tracing::field::Empty`?** If you don't declare a field in the `#[instrument]` macro, you can't record it later, because `tracing` needs to know the field exists when the span is created. `Empty` reserves the slot, and `Span::current().record()` fills it in after the query returns.

This pattern is essential for `db.response.returned_rows`, because a `0` on an UPDATE means the WHERE clause didn't match (product doesn't exist), while `1` confirms success. You'll see the same pattern on `create_order` and `create_order_items` (see `orders/src/db/repository.rs`).

### Handling Database Errors

Database errors, like constraint violations or deadlocks, are distinct from application logic errors (like "insufficient stock"). We want these to appear as errors in our traces, but we also want to preserve the specific error codes for debugging.

When using `#[instrument]`, errors returned by the function are automatically recorded as `span` exceptions. For deeper diagnosis, you could add a helper that records the specific PostgreSQL error code as a `span` attribute. OpenTel E-Commerce doesn't implement this helper, as `#[instrument]`'s automatic error recording is sufficient for most cases, but the pattern is worth knowing for high contention systems.

```rust
// A pattern for enhanced error recording (not used in OpenTel E-Commerce,
// but useful when you need to query traces by specific error codes)
fn record_db_error(span: &tracing::Span, error: &sqlx::Error) {
    span.record("otel.status_code", "ERROR");
    
    if let sqlx::Error::Database(db_err) = error {
        if let Some(code) = db_err.code() {
            span.record("db.response.status_code", code.to_string());
        }
    }
}
```

Common PostgreSQL error codes you might see in traces:
- **23505**: Unique violation (e.g., duplicate email)
- **23503**: Foreign key violation (e.g., invalid product ID)
- **40001**: Serialization failure (deadlock or concurrent update)
- **40P01**: Deadlock detected

By recording these codes, you can later query Jaeger for "all spans where db.response.status_code = 40001" to identify hot spots for concurrency issues.

## 5.7 Instrumenting Transactions

Transactions group multiple operations into an atomic unit. Without explicit instrumentation, transaction boundaries are invisible, you see individual query spans but not the container that groups them.

### Why Transactions Deserve Their Own Spans

Consider the checkout flow:

1. Fetch product details
2. Reserve inventory
3. Create order record
4. Create order items

If step 4 fails, steps 2 and 3 must roll back. Without a transaction span, your trace would look like a collection of disjointed successes followed by a failure, losing the 'atomic' connection between them. A transaction span provides this crucial metadata:

- **Visibility into Atomicity**: Tracks the total duration of the entire unit of work.
- **Outcome Tracking**: Explicitly records whether the operation resulted in a `commit` or `rollback`.
- **Causal Grouping**: Automatically organizes all child query spans, revealing how individual operations interact within the transaction.

```
Trace Hierarchy (as seen in Jaeger):

orders: POST /orders                              [================] 850ms
│
└─► TRANSACTION (checkout)                        [==============] 800ms
    │   db.system.name: postgresql
    │   otelmart.transaction.name: checkout
    │   otelmart.transaction.outcome: commit
    │
    ├─► HTTP GET products/{uuid}                  [==] 45ms
    │
    ├─► HTTP POST inventory/reserve               [========] 520ms
    │
    ├─► INSERT orders                             [=] 35ms
    │
    └─► INSERT order_items                        [=] 40ms
```

### The Transaction Wrapper

```rust
// orders/src/db/transaction.rs
// ... (imports)

#[instrument(   
    name = "TRANSACTION",
    skip(pool, f),
    fields(
        db.system.name = "postgresql",
        db.operation.name = "transaction",
        otelmart.transaction.name = %name,
        otelmart.transaction.outcome = tracing::field::Empty
    )
)]
pub async fn with_transaction<F, T, E>(
    pool: &PgPool,
    name: &'static str,
    f: F,
) -> Result<T, E>
where
    F: for<'c> FnOnce(
        &'c mut Transaction<'_, Postgres>,
    ) -> Pin<Box<dyn Future<Output = Result<T, E>> + Send + 'c>>,
    E: From<sqlx::Error>,
{
    let mut tx = pool.begin().await.map_err(E::from)?;

    match f(&mut tx).await {
        Ok(result) => {
            tx.commit().await.map_err(E::from)?;
            Span::current().record("otelmart.transaction.outcome", "commit");
            Ok(result)
        }
        Err(e) => {
            // Rollback is automatic on drop, but we make it explicit for clarity
            let _ = tx.rollback().await;
            Span::current().record("otelmart.transaction.outcome", "rollback");
            Err(e)
        }
    }
}
```

> [!NOTE] *Note on Rust Syntax*: The function signature above uses advanced Rust features like Higher-Rank Trait Bounds (HRTB) and `Pin`ning to handle async (return a `Future`) closures. If the syntax feels overwhelming, focus on the usage pattern below, that's what you'll write in your daily application code.

### Using the Transaction Wrapper

The checkout handler becomes:

```rust
pub async fn create_order(
    State(state): State<AppState>,
    Json(request): Json<CreateOrderRequest>,
) -> impl IntoResponse {
    // ... validation and stock reservation via HTTP calls ...

    // Calculate totals
    let totals = OrderTotals {
        subtotal,
        tax_amount,
        shipping_amount,
        total: total.clone(),
    };

    // Execute all database operations within an instrumented transaction
    let result = with_transaction(state.pool(), "checkout", |tx| {
        // ... (validation and stock reservation via HTTP calls) ...

        Box::pin(async move {
            // Generate order number
            let order_number = db::generate_order_number(tx).await?;

            // Create order record
            let created_order = db::create_order(
                tx,
                &order_number,
                &customer_email,
                customer_phone.as_deref(),
                &totals,
            ).await?;

            // Create order items
            db::create_order_items(tx, created_order.id, &items).await?;

            // Create shipping address
            db::create_shipping_address(tx, created_order.id, &shipping_address).await?;

            // Create payment
            let payment_reference = db::generate_payment_reference(tx).await?;
            db::create_payment(tx, created_order.id, &payment, &payment_reference, &total).await?;

            // Update order status
            db::update_order_payment_status(tx, created_order.id, "paid", "processing").await?;

            Ok::<_, OrderError>(created_order)
        })
    }).await;

    match result {
        Ok(order) => /* success response */,
        Err(e) => /* error response with rollback already complete */,
    }
}
```

### Transaction Span Hierarchy

The resulting trace shows clear nesting:

```
orders: POST /orders                              [================] 850ms
  └─ TRANSACTION (checkout)                       [==============] 800ms
       │  otelmart.transaction.outcome: commit
       ├─ SELECT generate_order_number            [=] 5ms
       ├─ INSERT orders                           [=] 35ms
       │    db.response.returned_rows: 1
       ├─ INSERT order_items                      [=] 40ms
       │    otelmart.items.count: 2
       │    db.response.returned_rows: 2
       ├─ INSERT shipping_addresses               [=] 15ms
       ├─ SELECT generate_payment_reference       [=] 5ms
       ├─ INSERT payments                         [=] 20ms
       └─ UPDATE orders                           [=] 15ms
```

If the transaction rolls back, the trace shows:

```
orders: POST /orders                              [========] 450ms  ERROR
  └─ TRANSACTION (checkout)                       [======] 420ms
       │  otelmart.transaction.outcome: rollback
       ├─ SELECT generate_order_number            [=] 5ms
       ├─ INSERT orders                           [=] 35ms
       └─ INSERT order_items                      [=] 30ms  ERROR
```

The `otelmart.transaction.outcome: rollback` attribute immediately signals that this was not a successful commit.

## 5.8 Connection Pool Visibility

A span showing 500ms doesn't tell you whether that time was spent waiting for a connection or executing the query. Under load, pool exhaustion manifests as latency spikes, not errors.

### What SQLx Handles Automatically

For most queries, SQLx acquires and releases connections transparently (you call `pool.fetch_one()` and the pool manages the rest). This means your database spans include both connection wait time *and* query execution time in a single duration. In normal operation, this is fine because connection acquisition takes microseconds when the pool has idle connections.

### When Pool Time Becomes Visible

The problem surfaces under load. SQLx's `PgPool` exposes two methods that reveal pool health:

- `pool.size()`: Total connections currently in the pool
- `pool.num_idle()`: Connections available for immediate use

When `num_idle()` drops to zero, new queries must wait for a connection to be released. This wait time gets folded into your database span duration, making queries *appear* slow when they're actually just queued.

### The Diagnostic Pattern

If you observe a database span taking 500ms, but the same query in PostgreSQL logs shows only 5ms of execution time, the remaining 495ms was spent waiting for a pool connection. This is the signature of pool exhaustion:

1. Span duration >> query execution time
2. Multiple spans from the same service show similar inflation
3. The issue correlates with request volume, not query complexity

At that point, you have two levers: increase `max_connections` in your pool configuration, or reduce connection hold time by optimizing long running queries. We'll build metrics and dashboards to detect this pattern automatically in Chapter 11.

> [!NOTE] Pool sizing, saturation analysis, and targeted pool instrumentation techniques are covered in Chapter 11.

## 5.9 Viewing Database Spans in Jaeger

With instrumentation complete, we can verify the results in Jaeger. Ensure the stack from Chapter 4 is running and Jaeger is accessible at `http://localhost:16686`.

### Triggering Trace Generation

To generate representative traces, perform the following actions in your browser:

1. **Browse Products**: Navigate to [http://localhost:4200/products](http://localhost:4200/products) to trigger read queries.
2. **Complete a Checkout**: Add an item to your cart and complete the checkout flow at [http://localhost:4200/checkout](http://localhost:4200/checkout) to trigger the instrumented transaction.

### Before Database Instrumentation (Chapter 4)

When we first instrumented HTTP boundaries in Chapter 4, our checkout trace looked like this:

```
otelmart: POST /api/orders                        [==================] 920ms
  └─ orders: POST /orders                         [================] 880ms
       ├─ HTTP GET products/{uuid}                [==] 45ms
       └─ HTTP POST inventory/reserve             [==========] 520ms
       
       ⚠️ Missing: 315ms unaccounted for (880ms - 45ms - 520ms = 315ms)
```

The `orders: POST /orders` span shows 880ms total, but only 565ms is visible in child spans. Where did the other 315ms go? Database queries were invisible, they happened inside the handler but left no trace.

### After Database Instrumentation (Chapter 5)

Now, with database spans instrumented, the same trace reveals the full picture:

```
otelmart: POST /api/orders                        [==================] 920ms
  └─ orders: POST /orders                         [================] 880ms
       ├─ HTTP GET products/{uuid}                [==] 45ms
       │    └─ products: GET /products/{uuid}     [=] 40ms
       │         └─ SELECT products               [=] 30ms  ← Now visible!
       ├─ HTTP POST inventory/reserve             [==========] 520ms
       │    └─ inventory: POST /reserve           [=========] 500ms
       │         └─ SELECT reserve_stock          [========] 480ms  ← Now visible!
       └─ TRANSACTION (checkout)                  [======] 280ms  ← Now visible!
            ├─ SELECT generate_order_number       [=] 5ms
            ├─ INSERT orders                      [=] 45ms
            ├─ INSERT order_items                 [=] 35ms
            ├─ INSERT shipping_addresses          [=] 15ms
            ├─ SELECT generate_payment_reference  [=] 5ms
            ├─ INSERT payments                    [=] 20ms
            └─ UPDATE orders                      [=] 15ms
```

The missing 315ms is now accounted for: 30ms (products SELECT) + 5ms (order number) + 45ms (INSERT orders) + 35ms (order items) + 15ms (shipping) + 5ms (payment ref) + 20ms (payment) + 15ms (update) = 170ms of database work, plus ~145ms of application logic between queries.

### Reading the Trace: Healthy Request

A successful product fetch shows:

```
otelmart: GET /api/products/{uuid}          [======] 95ms
  └─ products: GET /products/{uuid}         [====] 75ms
       └─ SELECT products                   [==] 55ms
            db.system.name: postgresql
            db.namespace: products
            db.operation.name: SELECT
            db.collection.name: products
            db.query.text: SELECT * FROM products WHERE uuid = $1
            otelmart.product.uuid: a7f3c5d2-...
```

**What to observe:**

- **Structural Hierarchy**: Clear parent-child nesting showing the flow from the Gateway (HTTP) through the service logic to the final Database query.
- **Aggregatable Span Names**: The low-cardinality `SELECT products` name allows your observability backend to group these spans for performance analysis across all product fetches.
- **Sanitized Transparency**: The `db.query.text` shows the parameterized SQL template, giving you full visibility into the query structure without leaking sensitive literal values.
- **Entity Correlation**: The custom `otelmart.product.uuid` attribute links technical database performance directly to a specific business entity.

### Reading the Trace: Checkout Flow

A successful checkout shows the full hierarchy:

```
otelmart: POST /api/orders                        [==================] 920ms
  └─ orders: POST /orders                         [================] 880ms
       ├─ HTTP GET products/{uuid}                [==] 45ms           (validation)
       │    └─ products: GET /products/...        [=] 40ms
       │         └─ SELECT products               [=] 30ms
       ├─ HTTP POST inventory/reserve             [==========] 520ms  (stock reservation)
       │    └─ inventory: POST /reserve           [=========] 500ms
       └─ TRANSACTION (checkout)                  [======] 280ms
            │  otelmart.transaction.outcome: commit
            ├─ SELECT generate_order_number       [=] 5ms
            ├─ INSERT orders                      [=] 45ms
            ├─ INSERT order_items                 [=] 35ms
            ├─ INSERT shipping_addresses          [=] 15ms
            ├─ SELECT generate_payment_reference  [=] 5ms
            ├─ INSERT payments                    [=] 20ms
            └─ UPDATE orders                      [=] 15ms
```

**What to observe:**

- **Transactional Containment**: The `TRANSACTION (checkout)` span acts as a parent to all database work, providing a unified view of the multi-step atomic operation.
- **Critical Path Identification**: The trace reveals which specific operation (in this case, inventory reservation) dominates the total duration, allowing for targeted optimization.
- **Explicit Outcome Verification**: The `otelmart.transaction.outcome: commit` attribute provides definitive confirmation that the entire unit of work was successfully persisted.
- **Sequence Validation**: The hierarchy preserves the exact order of operations, confirming that order items, shipping details, and payments were processed in the correct logical sequence.

### Reading the Trace: Failed Checkout

A checkout that fails due to insufficient stock:

```
otelmart: POST /api/orders                        [======] 380ms
  └─ orders: POST /orders                         [====] 350ms  ERROR
       ├─ HTTP GET products/{uuid}                [=] 40ms
       └─ HTTP POST inventory/reserve             [==] 280ms  ERROR
            └─ inventory: POST /reserve           [=] 260ms
                 (returns 409 Conflict - insufficient stock)
```

**What to observe:**

- **Logic-Driven Termination**: Observe how the trace ends abruptly at the HTTP layer. The Orders service identified a business conflict (insufficient stock) and returned a 409 status before any database work was even attempted.
- **Fail-Fast Efficiency**: Note the absolute absence of a `TRANSACTION` span or any `INSERT` queries. This confirms your service is "failing fast" saving database resources by validating business state early in the request lifecycle.
- **Error Context**: Although no database error occurred, the HTTP spans are correctly marked as `ERROR`, allowing you to find these failed business attempts in your observability backend.

### Reading the Trace: Leaked Reservation

A distributed consistency mismatch is the most insidious failure: an external side effect (reserved stock) is orphaned when the local order transaction fails to complete.

```
otelmart: POST /api/orders                        [========] 520ms  ERROR
  └─ orders: POST /orders                         [======] 490ms  ERROR
       ├─ HTTP GET products/{uuid}                [=] 35ms
       ├─ HTTP POST inventory/reserve             [===] 220ms  OK
       │    └─ inventory: POST /reserve           [==] 200ms
       │         (stock reserved successfully!)
       └─ TRANSACTION (checkout)                  [===] 200ms
            │  otelmart.transaction.outcome: rollback
            ├─ SELECT generate_order_number       [=] 5ms
            ├─ INSERT orders                      [=] 40ms
            └─ INSERT order_items                 [=] 35ms  ERROR
                 db.response.status_code: 23503
```

**What to observe:**

- **Distributed Side Effects**: Note that the inventory reservation succeeded (HTTP 200) *before* the local transaction started. This highlights the inherent danger in distributed systems: remote state changed while local state failed.
- **The Safety Net in Action**: The `otelmart.transaction.outcome: rollback` attribute is your definitive proof that the database discarded the `INSERT` operations after the failure (PostgreSQL error `23503`), preventing a corrupt "partial order" from being persisted.
- **Visibility into Inconsistency**: Without this trace, your inventory would "leak" (stock reserved but no order created). The telemetry makes this mismatch visible, signaling the need for a compensation call like `release_all_reserved_stock`.

This trace reveals the distributed consistency challenge. The local transaction rolled back, but the remote inventory reservation persists until explicitly released. Without this visibility, such issues are invisible until inventory audits reveal discrepancies.

## Summary

In this chapter, we extended our telemetry skeleton from the HTTP edge down into the data layer. By instrumenting our SQLx repository, we've transformed the database from an opaque "black box" into a transparent component of our distributed traces.

**What we implemented:**

- **Standardized Conventions**: Applied `db.system.name`, `db.namespace`, and `db.operation.name` attributes following OpenTelemetry stable specifications.
- **Aggregatable Naming**: Adopted a low-cardinality span naming strategy (e.g., `SELECT products`) that simplifies performance analysis across thousands of requests.
- **Rich Contextualization**: Injected `otelmart.*` business attributes to link technical query performance directly to entities like specific product UUIDs or order numbers.
- **Execution Visibility**: Leveraged `tracing::field::Empty` to record post-execution data like `db.response.returned_rows`.
- **Atomic Observability**: Implemented a transaction wrapper that tracks `commit` and `rollback` outcomes, revealing the inner workings of complex multi-step operations.

```
HTTP Handler
    ↓
Repository Function (#[instrument] + business context)
    ↓
SQLx Query (parameterized, compile-time checked)
    ↓
Connection Pool
    ↓
PostgreSQL
```

**The observability shift:**

| Before (Chapter 4) | After (Chapter 5) |
|:-------------------|:------------------|
| "Orders service took 880ms" | "INSERT orders: 45ms, INSERT items: 35ms, Reserve: 520ms" |
| Unknown failures within handlers | Specific SQL error codes (e.g., `23503` Foreign Key Violation) |
| Invisible distributed side effects | "Transaction rolled back, but inventory reservation leaked" |

Traces excel at showing you *where* time is spent and *how* operations are related. However, they don't always explain *why* a query is slow or whether your connection pool is actually saturated. To answer those questions, we need to look at our system in aggregate.

In **Chapter 6**, we will add **Metrics** to our observability stack. While traces provide the "what" and "how" of individual requests, metrics will provide the "how many" and "how often," allowing us to build dashboards that detect systemic patterns no single trace can reveal.
