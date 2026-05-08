# Chapter 4: Instrumenting the Request Journey

## Introduction

In Chapter 3, we built OpenTel E-Commerce—a system designed to break. We saw how distributed transactions can leak inventory, how network failures create silent inconsistencies, and how `println!` logging leaves us blind when diagnosing failures across service boundaries. We introduced correlation IDs as a first step toward linking requests, but acknowledged their limitation: they tell us what happened, not where the time went. We still had to manually propagate IDs, measure durations with stopwatches, and grep across services to reconstruct timing.

This chapter bridges that gap by automating trace propagation and capturing timing information at every service boundary.

We will integrate OpenTelemetry into our Rust services, transforming scattered log statements into a unified observability pipeline. By the end, every HTTP request flowing through OtelMart, Products, Inventory, and Orders will automatically generate connected traces—complete with timing information, service boundaries, and causal relationships—without requiring manual correlation ID propagation.

We will build what we call the **telemetry skeleton**: the foundational infrastructure that makes every future observability feature possible. Just as a skeleton provides structure for muscles and organs, this telemetry foundation will support the metrics, logs, and alerts we add in later chapters. Without this skeleton, those features would collapse into disconnected data points—metrics that can't explain spikes, logs that can't link to requests.

**In this chapter, we will cover the following key topics:**

* Understanding trace structure, propagation, and span hierarchy
* Adding OpenTelemetry dependencies to Rust services
* Initializing the telemetry pipeline and configuring exporters
* Defining a consistent trace taxonomy for our services
* Instrumenting HTTP boundaries and business logic
* Propagating trace context across service boundaries
* Visualizing end-to-end distributed traces in Jaeger

By the end of this chapter, you'll be able to diagnose distributed failures in minutes instead of hours.

**Before this chapter:**  
"Checkout is slow." → Grep 4 log files → Guess which service → SSH into containers → Calculate time deltas manually.

**After this chapter:**  
"Checkout is slow." → Open Jaeger → Click trace → See that `inventory: db.query` took 550ms (65% of total time).

---

## 4.1 The Transformation: Before and After OpenTelemetry

Before diving into implementation, let's clearly establish where we're starting and where we're going.

### Where We Left Off

Our application consists of four Rust microservices:

| Service | Directory | Port | Role |
|---------|-----------|------|------|
| OtelMart | `otelmart/` | 4200 | API Gateway, proxies to backend services |
| Products | `products/` | 3001 | Product catalog management |
| Inventory | `inventory/` | 3002 | Stock levels and reservations |
| Orders | `orders/` | 3003 | Order processing |

In Chapter 3, our services used `println!` for logging with no structured telemetry. Here's what we're transforming:

**Problems with the Chapter 3 approach:**
- `println!` output disappears into container logs with no structure
- No timing information—we can't see how long operations take
- No trace context—requests across services are disconnected
- Debugging requires grepping multiple log files manually

### The Transformation

Here's how we'll transform the Products service. Notice the inline `← NEW` and `← CHANGED` markers showing what's different:

**products/src/main.rs:**

```rust
use axum::{routing::get, Router};
use tokio::net::TcpListener;  // ← NEW
use tower_http::trace::TraceLayer;  // ← NEW
mod telemetry;  // ← NEW: Telemetry module

#[tokio::main]
async fn main() {
    let tracer_provider = telemetry::init_telemetry("products");  // ← NEW: Initialize first
    
    let config = Config::load().expect("Failed to load config");
    let pool = db::create_pool(&config.database.url).await;
    let state = AppState { pool, config };

    let app = Router::new()
        .route("/products", get(handlers::list_products))
        .route("/products/{id}", get(handlers::get_product_by_id))
        .with_state(state)
        .layer(TraceLayer::new_for_http())  // ← NEW: Automatic HTTP spans
        .layer(CorsLayer::permissive());

    let listener = TcpListener::bind("0.0.0.0:3001").await.unwrap();
    tracing::info!(port = 3001, "Products service listening");  // ← CHANGED: println! → tracing::info!
    axum::serve(listener, app).await.unwrap();
    
    telemetry::shutdown_telemetry(tracer_provider);  // ← NEW: Graceful shutdown
}
```

**products/src/handlers/products.rs:**

```rust
use tracing::instrument;  // ← NEW

#[instrument(  // ← NEW: Automatic span creation
    name = "get_product_by_id",
    skip(state),  // Don't log the entire AppState (too verbose)
    fields(product.uuid = %uuid)  // Attach product UUID as span attribute
)]
pub async fn get_product_by_id(
    State(state): State<AppState>,
    Path(uuid): Path<Uuid>,
) -> Result<Json<ProductDetail>, Response> {
    tracing::info!("Fetching product");  // ← CHANGED: Structured, attached to span
    
    let product = db::products::get_product_by_uuid(&state.pool, uuid).await?;
    Ok(Json(product))
}
```

**New infrastructure added:**

| Component | Purpose |
|-----------|---------|
| `telemetry.rs` | Initializes OpenTelemetry pipeline, exports to Jaeger |
| `TraceLayer` | Automatic spans for all HTTP requests |
| `#[instrument]` | Manual spans for business logic with attributes |
| Jaeger container | Collects and visualizes distributed traces |
| `traceparent` propagation | Links spans across service boundaries |

**After Chapter 4, we can:**
- See all requests in a unified Jaeger UI
- View timing breakdowns across all four services
- Click a trace to see exactly where time was spent
- Correlate logs with their originating spans

**But before we write code, we need to understand the concepts that make this possible.** Section 4.2 explains how traces, spans, and context propagation work under the hood.

---

## 4.2 From Concepts to Implementation

Chapter 1 introduced the three pillars of observability—traces, logs, and metrics—and explained how OpenTelemetry unifies them under a vendor-neutral standard. Chapter 3 showed why we need distributed tracing: correlation IDs help link logs across services, but they don't capture timing or causality. Now we translate those concepts into working Rust code.

This section bridges the gap between conceptual understanding and implementation. We'll examine how spans form hierarchical structures, how their identities propagate across service boundaries, and how Rust's `tracing` ecosystem integrates with OpenTelemetry.

### Traces and Spans: The Hierarchical View

In Chapter 1, we described a trace as a request's journey through your system. Now let's be precise: a **trace** is a tree of spans linked by a shared **TraceID** (128-bit identifier, displayed as 32 hexadecimal characters). Each **span** has its own **SpanID** and a **Parent SpanID** that creates the tree structure. This parent-child linkage is what Jaeger renders as a waterfall.

Consider what happens when a user clicks "Buy" in OpenTel E-Commerce:

```mermaid
flowchart TB
    subgraph trace["TraceID: 4bf92f3577b34da6a3ce929d0e0e4736"]
        A["<b>Span A (Gateway)</b><br/>SpanID: a1b2c3<br/>ParentID: null<br/>Name: POST /api/checkout"]
        B["<b>Span B (Orders)</b><br/>SpanID: d4e5f6<br/>ParentID: a1b2c3<br/>Name: POST /orders"]
        C["<b>Span C (Products)</b><br/>SpanID: 789abc<br/>ParentID: d4e5f6<br/>Name: GET /products"]
        D["<b>Span D (Inventory)</b><br/>SpanID: def012<br/>ParentID: d4e5f6<br/>Name: POST /inventory/reserve"]
        
        A -->|"HTTP call"| B
        B -->|"HTTP call"| C
        B -->|"HTTP call"| D
    end

    style A fill:#4a90d9,stroke:#2c5aa0,color:#fff
    style B fill:#50b86c,stroke:#3a8a50,color:#fff
    style C fill:#f5a623,stroke:#c7851a,color:#fff
    style D fill:#f5a623,stroke:#c7851a,color:#fff
    style trace fill:#f8f9fa,stroke:#333,stroke-width:2px
```

All four spans share the same TraceID (`4bf92f...`). The ParentID linkages create the hierarchy: Gateway spawned Orders, which spawned both Products and Inventory calls. When you view this in Jaeger or another tracing UI, you'll see this tree rendered as a waterfall diagram with timing bars.

This structure answers questions that logs alone cannot:
- **Which service called which?** Follow the ParentID chain.
- **Where did time go?** Compare span durations.
- **What ran in parallel vs. sequentially?** Spans C and D share the same parent and overlapping timestamps—they ran concurrently.

### SpanContext: The Propagation Mechanism

In Chapter 1, we saw that spans share a TraceID and that this ID flows through HTTP headers via the W3C `traceparent` format. But what exactly gets propagated?

**Why not send the entire span?** Spans accumulate data as they execute—events, attributes, timing information. The downstream service doesn't need (and shouldn't see) the upstream service's internal details. What it needs is just enough information to link its own spans to the distributed trace.

**SpanContext** is the minimal data structure that carries trace identity across process boundaries. It contains:

- **TraceID** (128-bit): Identifies the entire distributed trace
- **SpanID** (64-bit): Identifies the current span (becomes the parent for downstream spans)
- **TraceFlags** (8-bit): Contains sampling decisions and other flags
- **TraceState** (optional): Vendor-specific key-value pairs for extended context

When the Orders Service calls the Inventory Service, it doesn't send the entire span—just the SpanContext, serialized into HTTP headers:

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             │  │                                │                │
             │  │                                │                └─ trace flags (sampled)
             │  │                                └─ parent span ID
             │  └─ trace ID
             └─ version
```

The Inventory Service extracts this header, creates a new span with its own SpanID, and sets the extracted SpanID as the **Parent SpanID**. This parent-child linkage is what creates the hierarchical trace visualization you see in Jaeger.

This is precisely what our manual `X-Correlation-ID` header was doing in Chapter 3—but OpenTelemetry standardizes the format, automates the serialization, and handles edge cases like missing headers or malformed values.

### Span Events: Marking Moments in Time

Chapter 1 covered the core span fields: name, timestamps, duration, status, and attributes. OpenTelemetry spans include one additional concept worth understanding before we instrument our services: **events**.

**Events** are timestamped annotations within a span. Unlike attributes (which describe the span as a whole), events mark specific moments during execution:

```rust
span.add_event("cache_miss", vec![KeyValue::new("key", "product:123")]);
// ... fetch from database ...
span.add_event("cache_populated", vec![]);
```

Events are especially useful for recording retries, state transitions, or checkpoints within long-running operations. In our saga pattern from Chapter 3, events could mark each compensation step during rollback, creating a clear audit trail of the failure recovery process:

```rust
// In the Orders service saga compensation
span.add_event("saga.compensation_started", vec![]);
release_inventory_reservation(&state, &reserved_items).await?;
span.add_event("saga.inventory_released", vec![
    KeyValue::new("items_count", reserved_items.len() as i64)
]);
```

When debugging a failed checkout in Jaeger, you'd see these events as markers on the timeline, showing exactly when and why the saga rolled back.

### Seeing Timing and Events in Action

The tree structure shows *relationships*, but traces also capture *timing*. Here's the same checkout flow as a waterfall—the visualization that reveals bottlenecks:

```
Span                                    Timeline (0ms ─────────────────────► 2500ms)
─────────────────────────────────────────────────────────────────────────────────────
Gateway: POST /api/checkout             [██████████████████████████████████████] 2500ms
  └─ Orders: POST /orders                 [████████████████████████████████████] 2350ms
       ├─ Products: GET /products            [██] 200ms
       └─ Inventory: POST /reserve              [██████████████████████████] 2000ms ⚠️
                                                 ▲
                                                 └── BOTTLENECK: 80% of total time
```

The waterfall immediately shows that Inventory consumed 2 seconds—80% of the total request time. Notice that Products (200ms) and Inventory (2000ms) overlap in time—they're concurrent HTTP calls from the Orders service. The Orders span's 2350ms duration includes both calls plus its own processing overhead.

In a real tracing UI like Jaeger, you'd also see events within that span marking when locks were acquired and released, revealing that the span spent almost its entire duration holding a database lock. This is the visibility that Chapter 3's `println!` logging could never provide.

### Understanding Span Lifecycle

Before we start instrumenting code, it's important to understand how spans move through different states. This knowledge is crucial for understanding why certain operations (like `set_parent()`) must happen at specific times.

**The Three Span States:**

1. **Created**: Span exists but is not active
2. **Entered**: Span is active and recording events
3. **Exited**: Span is closed and ready for export

```rust
use tracing::{span, Level};

// State 1: CREATED - span exists but is not active
let span = span!(Level::INFO, "my_operation");

// State 2: ENTERED - span becomes active
let _guard = span.enter();
// Any events or child spans created here will be linked to this span

// State 3: EXITED - when _guard is dropped, span exits
// Span is closed and will be exported
```

**Why This Matters for `set_parent()`:**

Once a span is entered (made active), its parent relationship is **immutable**. You cannot change a span's parent after it has been entered. This is why custom `MakeSpan` implementations are necessary for context extraction—they set the parent during span creation, before `TraceLayer` enters the span.

```rust
// ❌ WRONG: Trying to set parent after entering
let span = span!(Level::INFO, "request");
let _enter = span.enter();  // Span is now active
span.set_parent(parent_context);  // ⚠️ TOO LATE - parent is already set!

// ✅ CORRECT: Set parent before entering
let span = span!(Level::INFO, "request");
span.set_parent(parent_context);  // ✅ Parent set while span is created
let _enter = span.enter();  // Now enter with correct parent
```

This is precisely why we need `ExtractingMakeSpan` (shown in section 4.7)—it creates the span with the correct parent before `TraceLayer` enters it.

**Automatic Lifecycle Management with `#[instrument]`:**

The `#[instrument]` macro handles the entire lifecycle automatically:

```rust
#[instrument]
async fn my_function() {
    // Span is created and entered at function start
    // Span stays active across .await points
    // Span exits when function returns (even on error)
}
```

This is equivalent to the much more verbose manual approach:

```rust
async fn my_function() {
    let span = span!(Level::INFO, "my_function");
    let _enter = span.enter();
    // ... function body ...
    // Span exits when _enter is dropped
}
```

The macro also handles async functions correctly by using task-local storage, which we'll cover in section 4.6.

> [!TIP]
> Use `#[instrument]` for almost all cases. Manual span creation is only needed when you need fine-grained control over span timing or when working with non-async code that has complex control flow.

### Why OpenTelemetry for Rust?

Rust's observability ecosystem centers on the `tracing` crate, which provides structured diagnostics with span support. If you've written `#[tracing::instrument]` or `info!()` in Rust, you've already used `tracing`.

However, `tracing` alone doesn't export data to external systems. It's a *facade*—it defines how to create spans and events, but not where they go. This is where OpenTelemetry enters.

The `tracing-opentelemetry` crate bridges these worlds:

```
┌─────────────────────────────────────────────────────────────┐
│                     Your Application Code                    │
│   #[tracing::instrument], info!(), span.enter()             │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                     tracing-opentelemetry                    │
│   Converts tracing spans → OpenTelemetry spans              │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    OpenTelemetry SDK                         │
│   Batches spans, manages resources, handles sampling        │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      OTLP Exporter                           │
│   Sends spans to Jaeger, Collector, or any OTLP backend     │
└─────────────────────────────────────────────────────────────┘
```

This layered approach means you write idiomatic Rust code using `tracing`'s macros, and OpenTelemetry handles the export. You get the ergonomics of Rust's ecosystem with the interoperability of OpenTelemetry's vendor-neutral protocol.

The alternative—using the `opentelemetry` API directly—works but feels foreign in Rust codebases. The `tracing` integration lets you add observability without changing how you write Rust.

In the next section, we'll add these crates to our services and see this pipeline in action.

---

## 4.3 Adding the `tracing` and OpenTelemetry Crates

The Rust OpenTelemetry ecosystem is modular by design—you compose the exact capabilities you need from specialized crates. This section walks through each dependency, explaining not just *what* it does, but *why* the ecosystem is structured this way and *where* you'll use it in your application code.

### Required Dependencies

Add the following to **each service's** `Cargo.toml` (Products, Inventory, Orders, and OtelMart):

> [!NOTE]
> The versions shown are current as of writing this book. For the latest compatible versions, check the [opentelemetry-rust releases](https://github.com/open-telemetry/opentelemetry-rust/releases).

```toml
[dependencies]
# Core tracing ecosystem
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }

# OpenTelemetry core
opentelemetry = "0.31"
opentelemetry_sdk = { version = "0.31", features = ["rt-tokio"] }

# OTLP exporter (sends traces to collectors/backends)
opentelemetry-otlp = { version = "0.31", features = ["grpc-tonic"] }

# Bridge between tracing and OpenTelemetry
tracing-opentelemetry = "0.32"

# HTTP header propagation
opentelemetry-http = "0.31"
```

Let's understand what each crate provides, organized by **how we'll use it in the application**:

**`tracing`**: The crate for daily use in handlers and business logic. Every `#[instrument]` macro, every `info!()` call, every `span!` uses this crate—it's the only observability API you'll directly interact with in `handlers/`, `db/`, and business logic modules.

```rust
// This is tracing in action - in your handlers
#[instrument(skip(state))]
pub async fn get_product_by_id(...) {
    info!("Fetching product");  // ← tracing macro
}
```

**`tracing-subscriber`**: Lives in `main.rs` or `telemetry.rs` initialization code. Configure it once at startup to control what gets logged (`env-filter` for `RUST_LOG`) and how it's formatted (`json` feature for structured output). After initialization, it runs in the background collecting spans.

```rust
// In telemetry.rs - configure once, forget
let env_filter = EnvFilter::try_from_default_env()
    .unwrap_or_else(|_| EnvFilter::new("info"));
```

#### The Bridge Layer (Invisible Plumbing)

**`tracing-opentelemetry`**: The invisible bridge between `tracing` and OpenTelemetry. Configure it once in `telemetry.rs` as a subscriber layer—it then automatically converts every `#[instrument]` span into an OpenTelemetry span, intercepting `tracing` spans behind the scenes.

```rust
// In telemetry.rs - set up once
let otel_layer = tracing_opentelemetry::layer().with_tracer(tracer);
tracing_subscriber::registry()
    .with(otel_layer)  // ← Intercepts all tracing spans
    .init();
```

**Version note**: This crate is maintained by the Tokio team (not the OpenTelemetry project), so version compatibility matters—always check the [compatibility matrix](https://github.com/tokio-rs/tracing-opentelemetry/blob/main/CHANGELOG.md) when upgrading.

**`opentelemetry-http`**: Appears in two specific places: extracting trace context from *incoming* HTTP requests (in the server's `MakeSpan`), and injecting context into *outgoing* requests (in the HTTP client). Approximately 20 lines total, but critical for distributed tracing—implements the W3C Trace Context specification from section 4.2.

```rust
// Extracting context from incoming requests (server-side)
let parent_context = global::get_text_map_propagator(|propagator| {
    propagator.extract(&HeaderExtractor(request.headers()))
});

// Injecting context into outgoing requests (client-side)
global::get_text_map_propagator(|propagator| {
    propagator.inject_context(&context, &mut HeaderInjector(headers))
});
```

#### Configuration & Export (One-Time Setup)

**`opentelemetry`**: The API traits defining what a tracer, span, and context are. Rarely imported directly—mostly used by `opentelemetry_sdk` and `tracing-opentelemetry`. The main place it appears is in trace context propagation (`global::get_text_map_propagator` shown above).

**`opentelemetry_sdk`**: Where the `TracerProvider` is built in `telemetry.rs`. This is configuration-heavy code written once: setting up the exporter, defining the resource (service name), configuring sampling. After `init_telemetry()` runs, the SDK batches and exports spans in the background. The `rt-tokio` feature integrates with Tokio's async runtime.

```rust
// In telemetry.rs - configuration code
let provider = SdkTracerProvider::builder()
    .with_batch_exporter(exporter)
    .with_resource(Resource::builder_empty()
        .with_service_name("products")
        .build())
    .build();
```

**`opentelemetry-otlp`**: Creates the exporter in `telemetry.rs`—literally 5 lines that configure *where* spans go (Jaeger, Collector) and *how* they're sent (gRPC vs HTTP). OTLP is the standard wire format for OpenTelemetry, supported by virtually all backends:

- **gRPC** (via `grpc-tonic` feature): HTTP/2 with Protocol Buffers—better performance for high-volume telemetry
- **HTTP/Protobuf** (via `http-proto` feature): HTTP/1.1—simpler for environments where gRPC is blocked

```rust
// In telemetry.rs - exporter setup
let exporter = opentelemetry_otlp::SpanExporter::builder()
    .with_tonic()  // ← gRPC transport
    .with_endpoint(&otlp_endpoint)
    .build()?;
```

### Feature Flags Explained

OpenTelemetry crates use feature flags extensively to keep binary sizes small and avoid pulling in unused dependencies. Without features, you'd get every possible transport, serialization format, and integration—most of which you'll never use.

The `grpc-tonic` feature on `opentelemetry-otlp` enables gRPC transport using the Tonic library.

> [!IMPORTANT]
> When using `grpc-tonic`, the OTLP exporter **must** be created from within a Tokio runtime context. Since our Axum services already run on Tokio, this requirement is satisfied. If you're using a different async runtime (e.g., async-std), you'll need the HTTP transport instead.

If your infrastructure doesn't support gRPC, you can use HTTP transport instead:

```toml
# Alternative: HTTP transport instead of gRPC
opentelemetry-otlp = { version = "0.31", features = ["http-proto", "reqwest-blocking-client"] }
```

### Version Compatibility

OpenTelemetry crates follow a coordinated versioning scheme. The versions listed above (0.31 for opentelemetry/sdk/otlp/http, 0.32 for tracing-opentelemetry) are compatible with each other. Mixing mismatched versions can cause subtle runtime errors where spans fail to connect or exports silently fail.

For example, `tracing-opentelemetry` 0.32 is compatible with `opentelemetry` 0.31, but `tracing-opentelemetry` 0.25 expects `opentelemetry` 0.24. Mixing 0.32 with 0.24 will compile but fail at runtime with cryptic errors like "span context not found."

When upgrading, always upgrade the entire OpenTelemetry stack together. Check the [opentelemetry-rust changelog](https://github.com/open-telemetry/opentelemetry-rust/blob/main/CHANGELOG.md) for version compatibility matrices, and the [tracing-opentelemetry changelog](https://github.com/tokio-rs/tracing-opentelemetry/blob/main/CHANGELOG.md) for bridge compatibility.

---

With dependencies in place, we're ready to initialize the OpenTelemetry pipeline. Section 4.4 shows how to configure the `TracerProvider` and connect it to the `tracing` ecosystem.

## 4.4 Initializing the Tracer Provider

With dependencies in place, we need to initialize the OpenTelemetry tracing pipeline. This happens **once** at application startup, before any requests are processed—get this wrong and spans will silently fail to export, or worse, panic at runtime when the first span is created without a configured subscriber.

The initialization involves three components: a **Resource** (metadata about your service), an **Exporter** (where to send traces), and a **TracerProvider** (manages span creation and export). We'll build these step by step, then see the complete module.

### Step 1: Resource - Service Identity

Every span exported from your service carries metadata identifying its origin. This is the **Resource**:

```rust
use opentelemetry_sdk::Resource;

let resource = Resource::builder_empty()
    .with_service_name(service_name.to_string())
    .build();
```

A Resource describes the entity producing telemetry. When viewing traces in Jaeger, spans are grouped by `service.name`—this is how you distinguish Products service spans from Inventory service spans.

The `service.name` is required and should match your service's identity (e.g., "products", "inventory", "orders", "otelmart"). You can add additional attributes following OpenTelemetry's [semantic conventions](https://opentelemetry.io/docs/concepts/semantic-conventions/)—standardized attribute names that backends understand:

```rust
use opentelemetry::KeyValue;

let    .with_resource(
        Resource::builder_empty()
            .with_service_name("products")
            .with_service_version("1.0.0")
            .with_service_namespace("opentel-ecommerce")
            .with_deployment_environment("production")
            .build(),
    )
    .build();
```

**Resource vs. Span Attributes:**

Understanding when to use Resource vs. span attributes is crucial for effective instrumentation:

| Aspect | Resource | Span Attributes |
|--------|----------|-----------------|
| **Scope** | Service-level (all spans) | Span-level (individual operations) |
| **Examples** | `service.name`, `service.version`, `host.name` | `http.method`, `db.statement`, `user.id` |
| **Cardinality** | Low (same for all spans) | High (varies per operation) |
| **When to use** | Metadata that identifies the service | Metadata specific to an operation |
| **Set once** | At initialization | Per span during execution |

**Example:**

```rust
// ✅ Resource: Service-level metadata (set once at startup)
Resource::builder_empty()
    .with_service_name("products")           // Same for all spans
    .with_deployment_environment("prod")     // Same for all spans
    .build()

// ✅ Span Attributes: Operation-level metadata (varies per span)
#[instrument(fields(
    product.id = %product_id,    // Different for each request
    user.id = %user_id,           // Different for each request
    http.method = "GET"           // Specific to this operation
))]
async fn get_product(product_id: Uuid, user_id: Uuid) { ... }
```

Using Resource correctly reduces data volume (no need to repeat `service.name` on every span) and enables better filtering in observability backends.

### Step 2: OTLP Exporter - Where Traces Go

The exporter sends completed spans to an external system. We use OTLP (OpenTelemetry Protocol) over gRPC:

```rust
use opentelemetry_otlp::WithExportConfig;

let otlp_endpoint = std::env::var("OTEL_EXPORTER_OTLP_ENDPOINT")
    .unwrap_or_else(|_| "http://localhost:4317".to_string());

let exporter = opentelemetry_otlp::SpanExporter::builder()
    .with_tonic()
    .with_endpoint(&otlp_endpoint)
    .build()
    .expect("Failed to create OTLP exporter");
```

Port 4317 is the standard OTLP gRPC port. The `OTEL_EXPORTER_OTLP_ENDPOINT` environment variable follows OpenTelemetry's [environment variable specification](https://opentelemetry.io/docs/concepts/sdk-configuration/otlp-exporter-configuration/), allowing operators to configure the export destination without rebuilding the application.

In our Docker Compose setup, this endpoint will point to the OpenTelemetry Collector or directly to Jaeger. For local development without a collector, Jaeger's all-in-one image accepts OTLP on port 4317.

> [!WARNING]
> The code uses `.expect()` for simplicity, which will panic if the exporter fails to initialize (e.g., invalid endpoint URL). In production, consider using `?` and propagating the error to `main()`, or logging a warning and falling back to a no-op tracer.

### Step 3: TracerProvider - The Span Factory

The `SdkTracerProvider` is the central configuration point. It creates `Tracer` instances, which create spans:

```rust
use opentelemetry_sdk::trace::SdkTracerProvider;

let provider = SdkTracerProvider::builder()
    .with_batch_exporter(exporter)
    .with_resource(resource)
    .build();
```

Key configuration options:

**Sampler**: Determines which traces are recorded. By default, the SDK uses `Sampler::ParentBased(Box::new(Sampler::AlwaysOn))`, which records every trace—appropriate for development and low-traffic services. In high-traffic production, configure a ratio-based sampler to reduce storage costs:

```rust
use opentelemetry_sdk::trace::Sampler;

let provider = SdkTracerProvider::builder()
    .with_sampler(Sampler::TraceIdRatioBased(0.1))  // 10% sampling
    .with_batch_exporter(exporter)
    .with_resource(resource)
    .build();
```

**IdGenerator**: Creates TraceIDs and SpanIDs. `RandomIdGenerator` (the default) produces cryptographically random IDs, ensuring uniqueness across distributed services. Alternative generators exist for specific use cases (e.g., AWS X-Ray compatible IDs).

**BatchExporter**: Spans are buffered and sent in batches rather than individually. This significantly reduces network overhead. The SDK's default batch configuration (512 spans per batch, 5-second intervals) works well for most applications.

### Step 4: Connecting to the tracing Ecosystem

This is where `tracing` and OpenTelemetry meet:

```rust
use opentelemetry::trace::TracerProvider as _;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt, EnvFilter};

// Create a tracer from the provider for the OpenTelemetry layer
let tracer = provider.tracer(service_name.to_string());

// Create the OpenTelemetry tracing layer
let otel_layer = tracing_opentelemetry::layer().with_tracer(tracer);

// Create an environment filter for log levels (defaults to "info")
let env_filter = EnvFilter::try_from_default_env()
    .unwrap_or_else(|_| EnvFilter::new("info"));

// Create a formatting layer for console output
let fmt_layer = tracing_subscriber::fmt::layer()
    .with_target(true)
    .with_thread_ids(false)
    .compact();

// Combine all layers into a subscriber and set it as the global default
tracing_subscriber::registry()
    .with(env_filter)
    .with(fmt_layer)
    .with(otel_layer)
    .init();
```

The `tracing_opentelemetry::layer()` creates a subscriber layer that:
1. Intercepts spans created by `tracing` (via `#[instrument]` or manual `span!` macros)
2. Converts them to OpenTelemetry spans
3. Exports them through the configured `TracerProvider`

We stack multiple layers in the subscriber:
- `env_filter`: Controls which spans/events are processed based on `RUST_LOG`
- `fmt_layer`: Prints to stdout for local debugging
- `otel_layer`: Exports to OpenTelemetry (via `tracing-opentelemetry`)

With this setup, a single `#[tracing::instrument]` annotation simultaneously logs to console *and* exports to your observability backend.

### Graceful Shutdown

The shutdown function is critical—it flushes any buffered spans before the application exits:

```rust
pub fn shutdown_telemetry(provider: SdkTracerProvider) {
    if let Err(e) = provider.shutdown() {
        eprintln!("Failed to shutdown tracer provider: {:?}", e);
    }
}
```

Without calling `shutdown_telemetry`, spans created in the last few seconds before shutdown may be lost. The `provider.shutdown()` call blocks until all pending exports complete or a timeout is reached.

### The Complete telemetry.rs Module

{{ ... }}

Here's the complete implementation, combining all the pieces above. Create this file at `src/telemetry.rs` in each service:

> **📁 Repository Reference**: This code is available at `code/chapter_4/products/src/telemetry.rs` (and identically in `orders`, `inventory`, `otelmart`). You can copy it directly or type it out to reinforce understanding.

Here are the key parts of the implementation:

```rust
use opentelemetry::global;
use opentelemetry_sdk::{propagation::TraceContextPropagator, trace::SdkTracerProvider, Resource};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt, EnvFilter};

pub fn init_telemetry(service_name: &str) -> SdkTracerProvider {
    // 1. Set up W3C Trace Context propagator for cross-service correlation
    global::set_text_map_propagator(TraceContextPropagator::new());

    // 2. Create OTLP exporter (sends traces to Jaeger via gRPC)
    let otlp_endpoint = std::env::var("OTEL_EXPORTER_OTLP_ENDPOINT")
        .unwrap_or_else(|_| "http://localhost:4317".to_string());
    
    let exporter = opentelemetry_otlp::SpanExporter::builder()
        .with_tonic()
        .with_endpoint(&otlp_endpoint)
        .build()
        .expect("Failed to create OTLP exporter");

    // 3. Build tracer provider with service resource
    let provider = SdkTracerProvider::builder()
        .with_batch_exporter(exporter)
        .with_resource(
            Resource::builder_empty()
                .with_service_name(service_name.to_string())
                .build(),
        )
        .build();

    // 4. Create layered subscriber (console + OpenTelemetry)
    let tracer = provider.tracer(service_name.to_string());
    let otel_layer = tracing_opentelemetry::layer().with_tracer(tracer);
    let env_filter = EnvFilter::try_from_default_env()
        .unwrap_or_else(|_| EnvFilter::new("info"));
    let fmt_layer = tracing_subscriber::fmt::layer().compact();

    tracing_subscriber::registry()
        .with(env_filter)
        .with(fmt_layer)
        .with(otel_layer)
        .init();

    provider
}

pub fn shutdown_telemetry(provider: SdkTracerProvider) {
    if let Err(e) = provider.shutdown() {
        eprintln!("Failed to shutdown tracer provider: {e}");
    }
}
```

> **💡 Key Concepts:**
> - **Propagator** (line 7): Enables W3C Trace Context format for cross-service traces
> - **Batch Exporter** (line 23): Buffers spans and sends them in batches for efficiency
> - **Resource** (line 25): Service-level metadata attached to all spans
> - **Layered Subscriber** (line 32): Combines console logging with OpenTelemetry export

See `code/chapter_4/products/src/telemetry.rs` for the complete implementation with full documentation.

### Integrating into main.rs

In each service's `main.rs`, add the module declaration and call the initialization:

```rust
mod telemetry;  // ← Declare the module

#[tokio::main]
async fn main() {
    // Initialize telemetry first, before any spans are created
    let tracer_provider = telemetry::init_telemetry("products");
    
    // ... build your Axum app, database connections, etc. ...
    
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3001").await.unwrap();
    tracing::info!(port = 3001, "Products service listening");
    
    axum::serve(listener, app).await.unwrap();
    
    // Gracefully shut down telemetry, flushing any pending spans
    telemetry::shutdown_telemetry(tracer_provider);
}
```

The initialization order matters: call `init_telemetry` before creating any spans (including those from `#[instrument]` macros in your route handlers). The shutdown call ensures buffered spans are exported before the process exits.

With the telemetry pipeline initialized, every `#[instrument]` macro will now create spans that flow through this infrastructure. The next section establishes a **trace taxonomy**—naming conventions and semantic attributes that make your traces searchable and meaningful.

## 4.5 Trace Taxonomy: Naming Conventions & Semantic Attributes

A trace is only as useful as its span names and attributes. Imagine debugging a production outage: you search for "all order creation failures" but find nothing—because each service named its spans differently (`create_order`, `new_order`, `order.create`). Or you try to filter by customer email, but half the services use `customer.email` and half use `user_email`. This is the chaos that trace taxonomy prevents.

This section establishes a **systematic approach** to naming and annotating spans, ensuring your traces are searchable, aggregatable, and consistent across services.

### The Five Fundamental Span Kinds

OpenTelemetry defines five span kinds via the `SpanKind` enum, each serving a distinct role:

| SpanKind | Purpose | Example in OpenTel E-Commerce |
|----------|---------|-------------------------------|
| `SpanKind::Server` | Handles incoming requests | `POST /orders` received by Orders Service |
| `SpanKind::Client` | Represents outbound calls | `HTTP GET products-service/products/123` |
| `SpanKind::Internal` | In-process work with no network I/O | `validate_order`, `calculate_total` |
| `SpanKind::Producer` | Publishing a message to a queue | `kafka.send orders-topic` |
| `SpanKind::Consumer` | Receiving a message from a queue | `kafka.receive orders-topic` |

**Why SpanKind Matters:**

SpanKind isn't just metadata—it has practical implications for trace analysis:

1. **Service Maps**: Observability backends use SpanKind to build service dependency graphs:
   - `CLIENT` → `SERVER` pairs show service-to-service communication
   - `PRODUCER` → `CONSUMER` pairs show async messaging flows

2. **CLIENT/SERVER Pairing**: For synchronous calls, you typically see:
   ```
   orders: HTTP GET products-service (CLIENT)  [====] 50ms
     └─ products: HTTP GET /products/{id} (SERVER) [===] 45ms
   ```
   The CLIENT span in Orders and SERVER span in Products are linked by trace context.

3. **Latency Attribution**: 
   - CLIENT span duration = network + server processing
   - SERVER span duration = just server processing
   - Difference reveals network latency

4. **Boundary Detection**: `INTERNAL` spans never cross process boundaries. If you see an `INTERNAL` span with a parent in a different service, something is wrong with context propagation.

**Example: Choosing the Right SpanKind**

```rust
// ✅ CORRECT: CLIENT span for outgoing HTTP call
#[instrument(
    name = "HTTP GET products-service/products",
    skip(client),
    fields(span.kind = "client")  // Marks this as a client span
)]
async fn fetch_product(client: &reqwest::Client, id: Uuid) -> Result<Product> {
    // ... HTTP call to products service ...
}

// ✅ CORRECT: INTERNAL span for business logic
#[instrument(skip(state))]  // Defaults to INTERNAL
async fn calculate_order_total(state: &AppState, items: &[Item]) -> Decimal {
    // ... local computation, no external calls ...
}
```

When viewing traces in Jaeger, SpanKind determines the span's color and icon, making it easy to distinguish between local processing and external calls at a glance.

**When to use each SpanKind:**
- **Server**: Automatically set by HTTP middleware for incoming requests
- **Client**: Automatically set by HTTP client middleware for outgoing calls
- **Internal**: Explicitly set for business logic, database queries, or any in-process work
- **Producer/Consumer**: For message queue operations (not covered in this chapter)

When using `tracing` with `tracing-opentelemetry`, span kinds are typically inferred:
- **Server spans**: Automatically set by `TraceLayer` for incoming HTTP requests
- **Client spans**: Set by HTTP client middleware (we'll add this in section 4.6)
- **Internal spans**: The default for `#[instrument]` macros

You can explicitly set the span kind using OpenTelemetry attributes:

```rust
use tracing::instrument;

#[instrument(
    skip(state),
    fields(otel.kind = "internal")  // Explicit internal span
)]
async fn validate_order(state: &AppState, order: &Order) -> Result<()> {
    // Business logic
}
```

For OpenTel E-Commerce's synchronous HTTP architecture, we primarily use Server, Client, and Internal spans. Producer and Consumer become relevant when adding message queues.

### Span Hierarchy Rules

Well-formed traces follow these rules:

1. **One root span per trace** — The entry point (typically an HTTP server span) has no parent
2. **Client spans are children of their initiator** — When Orders calls Inventory, the client span is a child of the Orders handler span
3. **Internal spans nest within their service** — `validate_order` is a child of the server span, not a sibling
4. **No orphaned spans** — Every span except the root must have a valid parent

Here's what a well-formed trace looks like:

```
Gateway (Server)                    ← Root span (no parent)
  └─ checkout_orchestration (Internal)
       └─ HTTP POST orders (Client)
            └─ Orders (Server)       ← New service boundary
                 ├─ validate_order (Internal)
                 ├─ HTTP GET products (Client)
                 │    └─ Products (Server)
                 └─ HTTP POST inventory (Client)
                      └─ Inventory (Server)
```

Notice:
- One root (Gateway Server span)
- Client spans are children of their initiator
- Server spans in downstream services are children of Client spans (via propagation)
- Internal spans nest within their service

Violating these rules produces flat, disconnected traces that cannot answer "what called what."

### Span Naming Best Practices

**Cardinality** refers to the number of unique values. A low-cardinality field has few distinct values (e.g., HTTP methods: GET, POST, PUT). A high-cardinality field has many unique values (e.g., UUIDs, timestamps).

Span names should be **low-cardinality** and **operation-centric**, not instance-centric.

**Bad: High-cardinality names**
```rust
// DON'T: Includes variable data in span name
#[tracing::instrument(name = "get_product_a7f3c5")]  // UUID in name!
async fn get_product(id: Uuid) -> Product { ... }
```

This creates a unique span name for every product. You can't search for "all product fetches" because each has a different name.

**Good: Operation-centric names**
```rust
// DO: Static operation name, variables as attributes
#[tracing::instrument(name = "get_product", skip(state), fields(product.id = %id))]
async fn get_product(state: &AppState, id: Uuid) -> Product { ... }
```

Now you can search for all spans named `get_product` and filter by the `product.id` attribute.

### OpenTelemetry Semantic Conventions

OpenTelemetry defines [semantic conventions](https://opentelemetry.io/docs/specs/semconv/)—standardized attribute names for common operations. Using these conventions ensures your traces are compatible with observability tools and comparable across services.

**Why follow semantic conventions?**
- **Tool compatibility**: Observability platforms (Jaeger, Grafana, Datadog) recognize standard attributes and provide built-in dashboards
- **Cross-service consistency**: All services use the same attribute names, enabling unified queries
- **Future-proofing**: New tools automatically understand your traces

**HTTP Conventions** (for server and client spans):

| Attribute | Example | Description |
|-----------|---------|-------------|
| `http.request.method` | `POST` | HTTP method |
| `http.route` | `/orders/{id}` | Route template (not actual path) |
| `http.response.status_code` | `200` | Response status code |
| `url.path` | `/orders/a7f3c5` | Actual request path |
| `server.address` | `orders-service` | Target server hostname |

**Database Conventions**:

| Attribute | Example | Description |
|-----------|---------|-------------|
| `db.system` | `postgresql` | Database type |
| `db.name` | `otelmart` | Database name |
| `db.operation` | `SELECT` | Operation type |
| `db.statement` | `SELECT * FROM...` | Query (sanitized!) |

**Custom Business Attributes**:

For domain-specific data, prefix with your organization or project name:

| Attribute | Example | Description |
|-----------|---------|-------------|
| `otelmart.order.id` | `ord_123` | Order identifier |
| `otelmart.customer.email` | `user@example.com` | Customer email |
| `otelmart.item.count` | `3` | Items in order |

The `otelmart.` prefix prevents collisions with standard OpenTelemetry attributes and clearly identifies domain-specific data. This is especially important when:
- Sending traces to shared observability platforms (multi-tenant SaaS)
- Merging traces from multiple projects
- Avoiding conflicts with future OpenTelemetry conventions

### Designing a Taxonomy for OpenTel E-Commerce

Here's our span naming taxonomy for the four services:

| Service | Span Name | Kind | Key Attributes |
|---------|-----------|------|----------------|
| **Gateway** | `HTTP POST /api/checkout` | Server | `http.route`, `http.method` |
| | `checkout_orchestration` | Internal | `otelmart.order.item_count` |
| | `HTTP POST orders-service` | Client | `server.address` |
| **Products** | `HTTP GET /products/{id}` | Server | `http.route`, `product.id` |
| | `db.query get_product` | Internal | `db.system`, `db.operation` |
| | `cache.lookup product` | Internal | `cache.hit` (if implemented) |
| **Inventory** | `HTTP POST /inventory/reserve` | Server | `http.route` |
| | `db.query reserve_stock` | Internal | `db.statement` |
| | `lock.acquire inventory` | Internal | `lock.key` (if using explicit locks) |
| **Orders** | `HTTP POST /orders` | Server | `http.route` |
| | `validate_order` | Internal | `otelmart.order.id` |
| | `HTTP GET products-service` | Client | `server.address`, `product.id` |
| | `HTTP POST inventory-service/reserve` | Client | `server.address` |
| | `db.query create_order` | Internal | `db.operation` |
| | `saga.compensate` | Internal | `saga.step` (rollback on failure) |

### Applying the Taxonomy in Code

Here's how to apply these conventions in the Orders service's `handlers/orders.rs`:

```rust
use tracing::{instrument, info, Span};

#[instrument(
    name = "create_order",
    skip(state, request),
    fields(
        otelmart.order.item_count = request.items.len(),
        otelmart.customer.email = %request.customer_email,
    )
)]
pub async fn create_order(
    state: &AppState,
    request: CreateOrderRequest,
) -> Result<Order, OrderError> {
    info!("Starting order creation");
    
    // Validate products
    for item in &request.items {
        let product = fetch_product(state, item.product_id).await?;
        
        // Add product info to current span
        Span::current().record("otelmart.product.sku", &product.sku);
    }
    
    // Reserve inventory
    reserve_inventory(state, &request.items).await?;
    
    // Create order in database
    let order = insert_order(state, &request).await?;
    
    info!(order.id = %order.uuid, "Order created successfully");
    Ok(order)
}

#[instrument(
    name = "HTTP GET products-service",
    skip(state),
    fields(
        http.request.method = "GET",
        server.address = "products-service",
        product.id = %product_id,
    )
)]
async fn fetch_product(state: &AppState, product_id: Uuid) -> Result<Product, OrderError> {
    let url = format!("http://products:3001/products/{}", product_id);
    
    let response = state.http_client
        .get(&url)
        .send()
        .await?;
    
    // Record response status
    Span::current().record("http.response.status_code", response.status().as_u16());
    
    // ... parse response
}
```

---

With a consistent taxonomy in place, we're ready to instrument our services. The next section covers **automatic vs. manual instrumentation**—when to use `#[instrument]` macros versus explicit span creation.

## 4.6 Instrumenting HTTP Boundaries

With the telemetry skeleton in place, we're ready to instrument the request journey we mapped in Chapter 3. We'll start at the entry point: HTTP requests arriving at our services.

There are two approaches to adding spans to your code: **automatic instrumentation** (middleware that creates spans for you) and **manual instrumentation** (explicit span creation in your code).

Use only automatic instrumentation and you'll see HTTP requests but not what happens inside them—a 500ms request with no clue where the time went. Use only manual instrumentation and you'll spend hours adding `#[instrument]` to every function, creating noisy traces. The solution: combine both strategically.

### Server-Side: TraceLayer Middleware

For HTTP servers like Axum, automatic instrumentation captures every incoming request without modifying your handlers. We'll use `tower-http`'s tracing middleware:

```toml
# Add to Cargo.toml
[dependencies]
tower-http = { version = "0.6", features = ["trace"] }
```

```rust
use axum::{Router, routing::get, extract::MatchedPath};
use tower_http::trace::TraceLayer;
use tracing::{Level, Span};

pub fn create_router(state: AppState) -> Router {
    Router::new()
        .route("/products/:id", get(get_product))
        .route("/products", get(list_products))
        .with_state(state)
        .layer(
            TraceLayer::new_for_http()
                .make_span_with(|request: &http::Request<_>| {
                    tracing::span!(
                        Level::INFO,
                        "HTTP request",
                        method = %request.method(),
                        uri = %request.uri(),
                        http.route = tracing::field::Empty,  // Filled later
                    )
                })
                .on_request(|request: &http::Request<_>, span: &Span| {
                    // Extract matched route from Axum's extensions
                    if let Some(matched_path) = request.extensions().get::<MatchedPath>() {
                        span.record("http.route", matched_path.as_str());
                    }
                })
                .on_response(|response: &http::Response<_>, latency: std::time::Duration, _span: &Span| {
                    _span.record("http.response.status_code", response.status().as_u16());
                    tracing::info!(
                        latency_ms = latency.as_millis(),
                        "Request completed"
                    );
                })
        )
}
```

This middleware automatically:
- Creates a span for every incoming HTTP request
- Records the HTTP method, URI, route template, and status code
- Measures request duration
- Handles errors and cancellations

**Trace Context Extraction**: By default, `TraceLayer` creates spans for incoming requests but doesn't link them to upstream traces. To extract the `traceparent` header and connect spans across services, we need a custom `MakeSpan` implementation (shown in section 4.7). The W3C Trace Context propagator we configured in `init_telemetry` handles the serialization format, but extraction requires explicit implementation.

### Manual Instrumentation for Business Logic

Automatic instrumentation captures HTTP boundaries, but not what happens *inside* your handlers. For business logic, use `#[tracing::instrument]`.

The `#[instrument]` macro is a procedural macro that wraps your function in a span. It:
- Creates a span with the function name (or custom name)
- Automatically records function arguments as span attributes
- Handles async functions correctly (span stays active across `.await` points)
- Closes the span when the function returns (even on error)

Without `#[instrument]`, you'd need to manually create, enter, and exit spans—error-prone and verbose.

```rust
#[tracing::instrument(
    name = "reserve_stock",
    skip(state),
    fields(
        product.id = %product_id,
        quantity = quantity,
        db.system = "postgresql",
    )
)]
pub async fn reserve_stock(
    state: &AppState,
    product_id: Uuid,
    quantity: i32,
) -> Result<StockReservation, InventoryError> {
    // This entire function is wrapped in a span named "reserve_stock"
    
    let mut tx = state.pool.begin().await?;
    
    // Add event for lock acquisition
    tracing::info!("Acquiring row lock");
    
    let result = sqlx::query_as!(
        StockReservation,
        r#"
        UPDATE inventory 
        SET reserved_quantity = reserved_quantity + $1
        WHERE product_uuid = $2 AND available_quantity >= $1
        RETURNING *
        "#,
        quantity,
        product_id,
    )
    .fetch_optional(&mut *tx)
    .await?;
    
    match result {
        Some(reservation) => {
            tx.commit().await?;
            tracing::info!("Stock reserved successfully");
            Ok(reservation)
        }
        None => {
            tx.rollback().await?;
            tracing::warn!("Insufficient stock");
            
            // Mark span as error for observability platforms
            Span::current().record("error", true);
            Span::current().record("error.type", "InsufficientStock");
            
            Err(InventoryError::InsufficientStock)
        }
    }
}
```

### When to Use Each Approach

| Scenario | Approach | Reason |
|----------|----------|--------|
| HTTP server endpoints | Automatic (middleware) | Consistent, no code changes needed |
| HTTP client calls | Manual | Need to inject trace context into headers |
| Database queries | Manual | Capture query details, timing |
| Business logic | Manual | Domain-specific naming and attributes |
| Third-party libraries | Depends | Check if library has OTel support |
| Simple utility functions | Skip | Adds noise without insight (e.g., `format_email`, `parse_uuid`) |

The goal is **meaningful** instrumentation, not **maximum** instrumentation. A trace with 100 spans for a simple checkout is harder to read than one with 10 well-named spans.

**HTTP Client Instrumentation**: We'll cover this in detail in section 4.7, but the key is injecting the current span's context into outgoing request headers:

```rust
use opentelemetry::global;
use opentelemetry_http::HeaderInjector;
use http::HeaderMap;

let mut headers = HeaderMap::new();
global::get_text_map_propagator(|propagator| {
    propagator.inject_context(
        &opentelemetry::Context::current(),
        &mut HeaderInjector(&mut headers)
    );
});

let response = client.get(url).headers(headers).send().await?;
```

This ensures the downstream service's span becomes a child of the current span, maintaining trace continuity.

### Combining Automatic and Manual Instrumentation

Here's how they work together in a typical request flow:

```
TraceLayer (automatic)           ← HTTP POST /orders
  └─ create_order (manual)       ← #[instrument]
       ├─ validate_order (manual)
       ├─ fetch_product (manual)  ← HTTP client call
       │    └─ TraceLayer (automatic, Products service)
       └─ reserve_stock (manual)
            └─ TraceLayer (automatic, Inventory service)
```

The automatic middleware creates the outer span for each service, while manual instrumentation adds visibility into business logic. Together, they create a complete picture of the request flow.


### Async Functions and Span Context

One of `#[instrument]`'s most powerful features is its seamless handling of async functions. In async Rust, functions can suspend at `.await` points and resume later—potentially on different threads. Without proper instrumentation, span context would be lost across these suspension points.

**The Problem Without `#[instrument]`:**

```rust
// ❌ BROKEN: Span context lost at .await
pub async fn fetch_product(id: Uuid) -> Result<Product> {
    let span = tracing::span!(Level::INFO, "fetch_product");
    let _enter = span.enter();  // Span active here
    
    let product = db::get_product(id).await;  // ⚠️ Span dropped when awaiting!
    // Span is no longer active here - database query won't be linked
    
    Ok(product)
}
```

When the function suspends at `.await`, the `_enter` guard goes out of scope and the span exits. When the function resumes, the span is no longer active, so any child spans (like database queries) won't be linked correctly.

**How `#[instrument]` Solves This:**

```rust
// ✅ CORRECT: Span context preserved across .await
#[instrument(skip(state), fields(product.id = %id))]
pub async fn fetch_product(state: &AppState, id: Uuid) -> Result<Product> {
    tracing::info!("Fetching product");
    
    let product = db::get_product(&state.pool, id).await;  // ✅ Span stays active
    // Span is still active - database query is correctly linked
    
    Ok(product)
}
```

The `#[instrument]` macro uses Tokio's **task-local storage** to preserve span context across `.await` points. The span remains associated with the async task, not the thread, so it survives suspension and resumption.

**Why This Matters for Distributed Tracing:**

In a typical handler, you might have multiple async operations:

```rust
#[instrument(name = "create_order", skip(state))]
pub async fn create_order(state: &AppState, req: OrderRequest) -> Result<Order> {
    // All of these .await points preserve the span context
    let product = fetch_product(state, req.product_id).await?;
    let reservation = reserve_inventory(state, req.product_id, req.quantity).await?;
    let order = insert_order(state, &req).await?;
    
    Ok(order)
}
```

Without `#[instrument]`, each `.await` would break the span hierarchy. With it, you get a clean parent-child relationship:

```
create_order (800ms)
  ├─ fetch_product (50ms)
  │   └─ db.query (30ms)
  ├─ reserve_inventory (600ms)
  │   └─ db.query (550ms)
  └─ insert_order (150ms)
      └─ db.query (120ms)
```

> [!TIP]
> Always use `#[instrument]` on async functions that perform I/O operations. Manual span management in async code is error-prone and easy to get wrong.

### Recording Errors in Spans

When operations fail, recording the error in the span provides crucial debugging context. OpenTelemetry spans have a **status** field that indicates success or failure, and can record exception details.

**Basic Error Recording:**

```rust
use tracing::Span;

#[instrument(skip(state), fields(product.id = %id))]
pub async fn fetch_product(state: &AppState, id: Uuid) -> Result<Product> {
    match db::get_product(&state.pool, id).await {
        Ok(product) => Ok(product),
        Err(e) => {
            // Record the error in the current span
            tracing::error!(error = %e, "Failed to fetch product");
            Span::current().record("error", true);
            Err(e)
        }
    }
}
```

When you view this trace in Jaeger, failed spans will be highlighted in red, and you can see the error message attached to the span.

**Setting Span Status:**

For more explicit error handling, you can set the span's status code:

```rust
use opentelemetry::trace::{Status, StatusCode};
use tracing_opentelemetry::OpenTelemetrySpanExt;

#[instrument(skip(state))]
pub async fn reserve_inventory(state: &AppState, product_id: Uuid, qty: i32) -> Result<()> {
    match try_reserve(state, product_id, qty).await {
        Ok(_) => {
            Span::current().set_status(Status::Ok);
            Ok(())
        }
        Err(e) if e.is_insufficient_stock() => {
            // Business logic error - not a system failure
            Span::current().set_status(Status::error("Insufficient stock"));
            tracing::warn!(product.id = %product_id, "Insufficient stock");
            Err(e)
        }
        Err(e) => {
            // System error - database failure, network issue, etc.
            Span::current().set_status(Status::error(format!("Database error: {}", e)));
            tracing::error!(error = %e, "Failed to reserve inventory");
            Err(e)
        }
    }
}
```

**Error Propagation in Distributed Traces:**

When an error occurs deep in a service call chain, the error status propagates up the span hierarchy:

```
otelmart: checkout (ERROR)
  └─ orders: create_order (ERROR)
       ├─ products: get_product (OK)
       └─ inventory: reserve_stock (ERROR) ← Root cause
```

In Jaeger, you can filter traces by error status to find failures quickly, and the span hierarchy shows exactly where the error originated.

> [!IMPORTANT]
> Distinguish between **business errors** (insufficient stock, invalid input) and **system errors** (database down, network timeout). Use `Status::error()` for system failures that indicate infrastructure problems. Business errors can be logged as warnings without marking the span as failed.

---

With instrumentation in place, we need to ensure traces flow correctly across service boundaries. Server-side instrumentation captures incoming requests, but distributed tracing requires linking those requests across services. Now we'll instrument the outgoing HTTP calls.

## 4.7 Service-to-Service Communication

The magic of distributed tracing is that spans from different services connect into a unified trace. This requires **context propagation**—passing trace identity across process boundaries via HTTP headers.

Here's what propagation looks like:

```
Orders Service                    Inventory Service
┌─────────────────┐              ┌─────────────────┐
│ Span A          │              │                 │
│  ├─ Span B      │              │                 │
│  │   └─ HTTP ───┼──headers────>│ Span C          │
│  │      Client  │  traceparent │  (parent: B)    │
└─────────────────┘              └─────────────────┘
```

Without propagation, Span C would be a new root span, disconnected from the Orders Service trace.

### The Propagation Problem

When the Orders Service calls the Inventory Service, how does the Inventory Service know it should create a child span of the Orders span? The services are separate processes; they don't share memory.

The answer: HTTP headers. The Orders Service **injects** its SpanContext into the outgoing request headers. The Inventory Service **extracts** that context and uses it as the parent for its spans.

**What gets propagated?**
- **TraceID**: The unique identifier for the entire distributed trace
- **SpanID**: The ID of the *current* span (becomes the parent ID in the downstream service)
- **TraceFlags**: Sampling decision (should this trace be recorded?)
- **TraceState**: Optional vendor-specific data

This data is encoded in the `traceparent` header using the W3C Trace Context format we saw in section 4.2.

### Injection: Adding Context to Outgoing Requests

When making HTTP calls to other services, we must inject the current trace context:

```rust
use opentelemetry::global;
use opentelemetry_http::HeaderInjector;
use tracing_opentelemetry::OpenTelemetrySpanExt;
use tracing::Span;

#[tracing::instrument(
    name = "HTTP POST inventory-service/reserve",
    skip(state, items),
    fields(server.address = "inventory-service")
)]
async fn reserve_inventory(
    state: &AppState,
    items: &[OrderItem],
) -> Result<Vec<Reservation>, OrderError> {
    let mut request = state.http_client
        .post("http://inventory:3002/reserve")
        .json(items)
        .build()?;
    
    // Inject trace context into request headers
    let context = tracing::Span::current().context();
    global::get_text_map_propagator(|propagator| {
        propagator.inject_context(&context, &mut HeaderInjector(request.headers_mut()));
    });
    
    let response = state.http_client
        .execute(request)
        .await?;
    
    Span::current().record("http.response.status_code", response.status().as_u16());
    
    // ... handle response
}
```

> **📁 Code Reference**: For a cleaner implementation, see `code/chapter_4/orders/src/traced_client.rs` which provides an `inject_trace_context()` helper function that encapsulates this pattern. The above inline example shows the mechanics for educational purposes.

After injection, the request headers contain:
```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
tracestate: (optional vendor-specific data)
```

> **⚠️ Common Pitfall: Context Timing**
>
> A subtle but critical mistake is injecting context *before* creating the client span. The injected context determines the parent-child relationship in the downstream service. If you extract `Span::current().context()` before your client span exists, you'll propagate the wrong parent:
>
> ```rust
> // ❌ WRONG: Context extracted before client span exists
> let context = tracing::Span::current().context();  // Gets HTTP server span
> let span = info_span!("http_client_call");         // Creates client span
> propagator.inject_context(&context, ...);          // Injects server span, not client!
>
> // ✅ CORRECT: Extract context from the client span
> let span = info_span!("http_client_call");         // Create client span first
> let context = span.context();                      // Get its context
> propagator.inject_context(&context, ...);          // Inject client span as parent
> ```
>
> When using `#[tracing::instrument]`, this is handled correctly because the span is created before your function body executes. The danger arises in manual span creation scenarios like API gateways or proxies.

Visual representation of the wrong vs. right approach:

```
❌ WRONG:
HTTP Server Span (A)
  └─ [extract context from A]  ← Gets wrong parent!
       └─ HTTP Client Span (B)
            └─ Downstream sees A as parent (skips B)

✅ CORRECT:
HTTP Server Span (A)
  └─ HTTP Client Span (B)
       └─ [extract context from B]  ← Gets correct parent!
            └─ Downstream sees B as parent
```

### Extraction: Reading Context from Incoming Requests

On the receiving service, we need to extract the context and use it as the parent span. The key insight is that `set_parent()` must be called **before** the span is entered. This means we can't use a simple middleware that runs after `TraceLayer` creates its span.

**Why before entering?** Once a span is entered (made active), its parent is fixed. OpenTelemetry's span model doesn't allow reparenting active spans because it would break the trace hierarchy. The `MakeSpan` trait is called *before* `TraceLayer` enters the span, giving us the one chance to set the parent.

Instead, we use a custom `MakeSpan` implementation that extracts the parent context and sets it during span creation:

```rust
use axum::http::Request;
use opentelemetry::global;
use opentelemetry_http::HeaderExtractor;
use tower_http::trace::MakeSpan;
use tracing::{Level, Span};
use tracing_opentelemetry::OpenTelemetrySpanExt;

/// Custom MakeSpan that extracts parent trace context from incoming requests.
#[derive(Clone, Debug)]
pub struct ExtractingMakeSpan;

impl<B> MakeSpan<B> for ExtractingMakeSpan {
    fn make_span(&mut self, request: &Request<B>) -> Span {
        // Extract parent context from incoming headers
        let parent_context = global::get_text_map_propagator(|propagator| {
            propagator.extract(&HeaderExtractor(request.headers()))
        });
        
        // If extraction fails (no traceparent header), parent_context will be empty
        // and the span becomes a new root span (which is correct for external requests)

        // Create the span for this request
        let span = tracing::span!(
            Level::INFO,
            "HTTP request",
            method = %request.method(),
            uri = %request.uri(),
            version = ?request.version(),
        );

        // Set the parent context on the span BEFORE it's entered
        let _ = span.set_parent(parent_context);

        span
    }
}
```

Use this custom `MakeSpan` with `TraceLayer`:

```rust
use tower_http::trace::TraceLayer;

pub fn create_router(state: AppState) -> Router {
    Router::new()
        .route("/reserve", post(reserve_stock))
        .with_state(state)
        .layer(TraceLayer::new_for_http().make_span_with(ExtractingMakeSpan))
        .layer(CorsLayer::permissive())
}
```

> [!IMPORTANT]
> **Why not middleware?** A naive approach might use Axum middleware to call `Span::current().set_parent()`. However, due to layer ordering, such middleware runs *after* `TraceLayer` has already created and entered its span. The `set_parent()` call will fail because you cannot change a span's parent after it has been started. The `MakeSpan` approach works because it sets the parent during span creation, before `TraceLayer` enters the span.

### Replacing Manual Correlation IDs

In Chapter 3, we manually passed `X-Correlation-ID` headers:

```rust
// Old approach (Chapter 3)
let correlation_id = Uuid::new_v4();
client.post(&url)
    .header("X-Correlation-ID", correlation_id.to_string())
    .send()
    .await?;
```

You can now remove all manual `X-Correlation-ID` header handling from your services. The `traceparent` header serves the same purpose (and more), and it's automatically injected/extracted by the OpenTelemetry infrastructure.

With OpenTelemetry propagation, this is automatic:

```rust
// New approach (Chapter 4)
let context = tracing::Span::current().context();
global::get_text_map_propagator(|propagator| {
    propagator.inject_context(&context, &mut HeaderInjector(request.headers_mut()));
});
// traceparent header is automatically added
```

The trace context carries more information than a simple correlation ID:
- **TraceID**: Equivalent to our correlation ID
- **SpanID**: Identifies the *specific* operation, not just the request
- **TraceFlags**: Sampling decisions
- **TraceState**: Vendor-specific context

### Verifying Context Propagation

To verify that propagation is working correctly, check the Jaeger UI:

1. **Single TraceID**: All spans across all services should share the same TraceID
2. **Parent-Child Relationships**: The Inventory service's span should be a child of the Orders service's client span
3. **No Orphaned Spans**: Every span except the root should have a parent

If you see multiple traces instead of one unified trace, propagation is broken. Common causes:
- Forgot to inject context in HTTP client calls
- Forgot to extract context in server `MakeSpan`
- Context timing issue (injecting before client span exists)

---

With context propagation in place, we have all the building blocks for distributed tracing. The patterns we've covered—TraceLayer middleware, context extraction, HTTP client instrumentation, and context injection—apply consistently across all services.

> **📁 Repository Reference**: Complete service implementations demonstrating these patterns are available in the code folder:
> - `code/chapter_4/products/` - Basic HTTP server with database queries
> - `code/chapter_4/inventory/` - Database transactions with error recording  
> - `code/chapter_4/orders/` - HTTP client calls with context injection
> - `code/chapter_4/otelmart/` - Request orchestration across services
>
> Each service follows the same pattern: `telemetry::init_telemetry()` → `TraceLayer` with `ExtractingMakeSpan` → instrumented handlers → `inject_trace_context()` for outgoing calls.



## 4.8 Viewing Distributed Traces in Jaeger

### Setting Up the Telemetry Backend

Our services are now instrumented, but we need somewhere to send the traces. We'll add Jaeger to our Docker Compose setup.

**Why Jaeger?**
- **All-in-one**: Single container includes collector, storage, and UI
- **Industry standard**: Widely used CNCF project with strong community
- **Free and self-hosted**: No vendor lock-in or usage costs
- **OTLP support**: Native support for OpenTelemetry Protocol

The `jaegertracing/all-in-one` image is perfect for development but uses in-memory storage (traces disappear on restart). For production, see the [Jaeger documentation](https://www.jaegertracing.io/docs/latest/deployment/) for persistent storage options.

### Adding Jaeger to docker-compose.yml

Here's how to add Jaeger to the `docker-compose.yml`:

```yaml
services:
  # Jaeger for distributed tracing
  jaeger:
    image: jaegertracing/all-in-one:latest
    container_name: jaeger
    environment:
      - COLLECTOR_OTLP_ENABLED=true  # Enable OTLP gRPC/HTTP receivers
    ports:
      - "16686:16686"   # Jaeger UI (web interface)
      - "4317:4317"     # OTLP gRPC receiver (our services use this)
      - "4318:4318"     # OTLP HTTP receiver (alternative to gRPC)
    networks:
      - app-network

  # Services use these OTEL environment variables:
  products:
    build:
      context: .
      dockerfile: products/Dockerfile
    environment:
      - APP_DATABASE_URL=postgres://opentel_user:opentel_pass@postgres:5432/opentel_db
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://jaeger:4317
      - OTEL_SERVICE_NAME=products
      - RUST_LOG=info
    depends_on:
      postgres:
        condition: service_healthy
      jaeger:
        condition: service_started
    # ... ports, networks, etc.

  inventory:
    # Similar configuration with OTEL_EXPORTER_OTLP_ENDPOINT=http://jaeger:4317
    
  orders:
    # Similar configuration with OTEL_EXPORTER_OTLP_ENDPOINT=http://jaeger:4317
    
  otelmart:
    # Similar configuration with OTEL_EXPORTER_OTLP_ENDPOINT=http://jaeger:4317
```

The `depends_on` ensures services wait for Jaeger to start before attempting to export spans. Without this, services might crash on startup if they can't reach the OTLP endpoint.

> **Note**: This configuration matches the `code/chapter_4/docker-compose.yml` file in the code folder.

### Starting the Stack

Build and start all services:

```bash
docker-compose up --build
```

The `--build` flag ensures Docker rebuilds images with the latest code changes.

**Verify Jaeger is running:**
1. Open `http://localhost:16686` in your browser
2. You should see the Jaeger UI with a "Search" page
3. The service dropdown will be empty until you generate your first trace (covered in section 4.11)

**Troubleshooting:**
- If Jaeger UI doesn't load: Check `docker-compose logs jaeger` for errors
- If services fail to start: Ensure `OTEL_EXPORTER_OTLP_ENDPOINT` is set correctly

### Alternative: OpenTelemetry Collector

For production, the **OpenTelemetry Collector** provides advantages over direct-to-Jaeger export:
- **Reliability**: Buffers and retries failed exports
- **Flexibility**: Route traces to multiple backends (Jaeger, Tempo, Honeycomb, etc.)
- **Processing**: Sample, filter, or enrich traces before storage
- **Vendor-neutral**: Swap backends without changing application code

#### Adding the Collector

Add the Collector service to `docker-compose.yml`:

```yaml
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest  # Use specific version in production
    container_name: otel-collector
    command: ["--config=/etc/otel-collector-config.yaml"]
    volumes:
      - ./otel-collector-config.yaml:/etc/otel-collector-config.yaml
    ports:
      - "4317:4317"   # OTLP gRPC (for services)
      - "4318:4318"   # OTLP HTTP
    networks:
      - app-network
```

#### Collector Configuration

Create `otel-collector-config.yaml`:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317  # Accept traces from services
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 5s              # Batch spans for 5s before exporting
    send_batch_size: 512     # Or when 512 spans accumulate

exporters:
  otlp/jaeger:
    endpoint: jaeger:4317    # Forward to Jaeger
    tls:
      insecure: true         # No TLS in dev (use TLS in production!)
  logging:
    loglevel: debug          # Log traces to collector stdout (for debugging)

service:
  pipelines:
    traces:
      receivers: [otlp]                # Receive from services
      processors: [batch]              # Batch before export
      exporters: [otlp/jaeger, logging]  # Send to Jaeger + logs
```

#### Switching Services to Use the Collector

Change service environment variables from:
```yaml
- OTEL_EXPORTER_OTLP_ENDPOINT=http://jaeger:4317
```

to:
```yaml
- OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
```

The Collector will handle forwarding to Jaeger, and you can add more exporters (DataDog, Honeycomb, etc.) without changing service code.

### Triggering a Trace

Start the stack and make a checkout request:

```bash
# Start all services
docker-compose up --build

# In another terminal, get a valid product ID from the database:
docker exec -it opentel-postgres psql -U opentel_user -d opentel_db -c "SELECT uuid, name FROM products LIMIT 1;"

# Then use that UUID in the checkout request (OtelMart gateway on port 4200):
curl -X POST http://localhost:4200/api/checkout \
  -H "Content-Type: application/json" \
  -d '{
    "customer_email": "test@example.com",
    "items": [
      {"product_id": "YOUR_PRODUCT_UUID_HERE", "quantity": 2}
    ]
  }'
```

**Expected response:**
```json
{
  "order_id": "...",
  "status": "created",
  "total": 49.98
}
```

**If the request fails:**
- `Connection refused`: Services aren't running yet (wait ~30s after `docker-compose up`)
- `404 Not Found`: Check the endpoint is `/api/checkout` (not `/checkout`)
- `Product not found`: Use a valid product UUID from the database (see above)

### Viewing in Jaeger

Open `http://localhost:16686` in your browser.

**Navigate to your trace:**
1. **Select Service**: In the left sidebar, find the "Service" dropdown and choose "otelmart"
2. **Set Time Range**: The default "Last Hour" should work, or click "Lookback" and select "Last 15 Minutes"
3. **Find Traces**: Click the blue "Find Traces" button at the bottom of the sidebar
4. **Identify Your Trace**: Look for a trace with:
   - Operation: `checkout_orchestration`
   - Duration: ~800-1000ms (depending on your system)
   - Spans: 10+ spans across 4 services
5. **View Details**: Click on the trace to see the waterfall diagram

You'll see a waterfall diagram showing:

```
otelmart: checkout_orchestration                       [====================] 850ms
  └─ orders: create_order                              [==================] 800ms
       ├─ orders: HTTP GET products-service            [===] 50ms
       │    └─ products: get_product_by_id             [==] 45ms
       │         └─ products: SELECT FROM products     [=] 30ms
       └─ orders: HTTP POST inventory-service/reserve  [============] 600ms
            └─ inventory: reserve_stock                [===========] 590ms
                 └─ inventory: UPDATE inventory        [=========] 550ms ← Slow!
```

### What We Can Now See

Compared to Chapter 3's `println!` logging:

| Before (Chapter 3) | After (Chapter 4) |
|-------------------|-------------------|
| Grep 4 log files | Single trace view |
| Guess timestamps | Precise durations |
| Manual correlation IDs | Automatic propagation |
| "Which service is slow?" | Obvious from waterfall |
| "What called what?" | Parent-child spans |

### Identifying the Bottleneck

In our trace, we can immediately see that `inventory: db.query` took 550ms—65% of the total request time. Without distributed tracing, we would have had to:

1. Check OtelMart logs: "Checkout took 850ms"
2. SSH into Orders, grep for correlation ID
3. SSH into Inventory, grep for same ID
4. Manually calculate time differences
5. Guess where the delay occurred

With tracing, we click once and see the answer.

### Evaluating Trace Quality

Before trusting your traces in production, check for these patterns:

**Signs of healthy traces:**
- Clear parent-child nesting visible in the waterfall
- Consistent span naming across services
- Single TraceID spans all four services in one request
- Client spans from Orders connect to Server spans in Products/Inventory

**Signs of broken traces:**
- Flat timeline with no hierarchy (missing parent links)
- Multiple TraceIDs for one logical request (propagation failure)
- Gaps between services (missing client spans)
- Excessive micro-spans creating visual noise

If your checkout trace shows four disconnected root spans instead of one unified tree, context propagation is broken—revisit section 4.7.

**How to fix broken traces:**

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| Flat timeline (no hierarchy) | Missing `set_parent()` in `MakeSpan` | Review section 4.7 on context extraction |
| Multiple TraceIDs for one request | Context not injected in HTTP client | Review section 4.7 on HTTP client instrumentation |
| Gaps between services | Missing client spans | Add `#[instrument]` to HTTP call functions |
| Excessive micro-spans | Over-instrumentation | Remove `#[instrument]` from utility functions |

---

## Summary

We have built the **telemetry skeleton** for OpenTel E-Commerce:

**What We Implemented:**
- A shared telemetry module with TracerProvider initialization
- Automatic HTTP instrumentation via Tower middleware
- Manual instrumentation for business logic using `#[tracing::instrument]`
- Context propagation across service boundaries
- Jaeger integration for trace visualization
- Graceful shutdown to ensure spans are flushed

**The Trace Taxonomy:**
- Operation-centric span names (low cardinality)
- OpenTelemetry semantic conventions for HTTP and database operations
- Custom `otelmart.*` attributes for business context

**Key Concepts:**
- TraceID links all spans in a request
- SpanID + ParentID create the hierarchy
- SpanContext propagates via W3C `traceparent` headers
- The `tracing` → `tracing-opentelemetry` → OTLP pipeline

The "chaos" we built in Chapter 3 is no longer a black box. We can now see inside. This foundation supports everything we'll build next.
