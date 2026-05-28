# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Headless Commerce Engine · Created: 2026-05-19

## Philosophy

This model treats every state change as an immutable event appended to an event store. The current state of any entity (product, order, cart, inventory) is derived by replaying its event stream. Separate materialised read models (projections) are maintained for fast query access, following the Command Query Responsibility Segregation (CQRS) pattern. The event store is the single source of truth; read models are disposable and can be rebuilt from the event log at any time.

This approach is used by financial systems, audit-heavy platforms, and enterprises that need to answer temporal questions like "what was this product's price on March 15?" or "show me the complete history of this order." Event sourcing naturally produces a complete audit trail — a critical requirement for PCI DSS compliance, GDPR data processing records, and B2B commerce where approval chains must be traceable. The CloudEvents specification provides the event envelope format, ensuring interoperability with external event buses (Kafka, RabbitMQ, Amazon EventBridge).

The trade-off is increased complexity: developers must think in terms of events and projections rather than simple CRUD, and eventual consistency between the write and read sides requires careful UX design. However, for a headless commerce engine that needs to support AI analytics on change patterns, fraud detection on order event sequences, and full regulatory compliance, this approach provides the strongest foundation.

**Best for:** Commerce platforms requiring complete audit trails, temporal queries, regulatory compliance, and AI-powered analytics on event sequences.

**Trade-offs:**
- Pro: Complete, immutable audit trail of every change — nothing is ever lost
- Pro: Temporal queries are trivial ("what was the state at time T?")
- Pro: Event replay enables AI/ML training on real event sequences
- Pro: Read models can be optimised independently for different query patterns
- Pro: Natural fit for CloudEvents and async microservice communication
- Con: Higher complexity; developers must understand event sourcing patterns
- Con: Eventual consistency between write and read models
- Con: Event schema evolution requires careful versioning (upcasters)
- Con: Read model rebuild can be slow for large event stores
- Con: Debugging requires event replay tooling rather than simple SQL queries

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CloudEvents v1.0 | Event store envelope follows CloudEvents spec exactly (type, source, id, time, data) |
| ISO 4217 | Currency codes embedded in price and order event payloads |
| ISO 3166-1/2 | Country and subdivision codes in address-related events |
| ISO 8601 | All event timestamps use ISO 8601 UTC format |
| AsyncAPI v3.1 | Event streams documented as AsyncAPI channels |
| PCI DSS v4.0.1 | Payment events never contain raw card data; only tokenised references |
| GDPR | Event store supports right-to-erasure via crypto-shredding (encrypted customer data, key deletion) |
| OAuth 2.0 / OIDC | Command authentication uses OAuth 2.0; actor identity recorded in every event |

---

## Event Store (Write Side)

```sql
-- The core event store: single append-only table for all domain events
CREATE TABLE event_store (
    event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    -- CloudEvents envelope fields
    spec_version VARCHAR(10) NOT NULL DEFAULT '1.0',
    type VARCHAR(255) NOT NULL,           -- e.g. 'product.created', 'order.item_added'
    source VARCHAR(500) NOT NULL,         -- e.g. '/products', '/orders'
    subject VARCHAR(500) NOT NULL,        -- aggregate ID (product ID, order ID, etc.)
    time TIMESTAMPTZ NOT NULL DEFAULT now(),
    data_content_type VARCHAR(100) NOT NULL DEFAULT 'application/json',

    -- Event sourcing metadata
    aggregate_type VARCHAR(100) NOT NULL,  -- 'product', 'order', 'cart', 'customer', 'inventory'
    aggregate_id UUID NOT NULL,            -- the entity this event belongs to
    sequence_number BIGINT NOT NULL,       -- monotonically increasing per aggregate
    correlation_id UUID,                   -- links related commands/events across aggregates
    causation_id UUID,                     -- the event that caused this event
    actor_id UUID,                         -- who/what triggered this event
    actor_type VARCHAR(50),               -- 'customer', 'staff', 'system', 'api_client'

    -- Event payload
    data JSONB NOT NULL,                   -- the event-specific payload
    metadata JSONB NOT NULL DEFAULT '{}',  -- additional context (IP, user agent, channel)

    -- Optimistic concurrency: unique per aggregate ensures no conflicting writes
    UNIQUE (aggregate_type, aggregate_id, sequence_number)
);

-- Primary query pattern: load all events for an aggregate in order
CREATE INDEX idx_es_aggregate ON event_store(aggregate_type, aggregate_id, sequence_number);

-- Query by event type (for projections that consume specific event types)
CREATE INDEX idx_es_type ON event_store(type, time);

-- Query by time range (for replay, analytics, debugging)
CREATE INDEX idx_es_time ON event_store(time DESC);

-- Query by correlation (trace a command through the system)
CREATE INDEX idx_es_correlation ON event_store(correlation_id) WHERE correlation_id IS NOT NULL;

-- Partition by month for performance at scale
-- In production, use declarative partitioning:
-- CREATE TABLE event_store (...) PARTITION BY RANGE (time);
-- CREATE TABLE event_store_2026_05 PARTITION OF event_store
--     FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
```

### Event Type Taxonomy

```
-- Product events
product.created           -- initial product creation with type, name, slug
product.updated           -- attribute changes (name, description, status)
product.archived          -- soft delete
product.variant_added     -- new variant with SKU, price, attributes
product.variant_updated   -- variant attribute/price change
product.variant_removed   -- variant discontinued
product.media_added       -- image/video attached
product.media_removed     -- image/video detached
product.category_assigned -- product placed in category
product.category_removed  -- product removed from category

-- Pricing events
price.set                 -- price established for variant + price_list
price.changed             -- price modified (old and new values in payload)
price.removed             -- price removed from a price list
promotion.created         -- new promotion with rules
promotion.activated       -- promotion goes live
promotion.deactivated     -- promotion paused
promotion.expired         -- promotion end date reached

-- Inventory events
inventory.received        -- stock received at location
inventory.reserved        -- stock reserved for a cart/order
inventory.unreserved      -- reservation released (cart abandoned/expired)
inventory.sold            -- stock decremented for fulfilled order
inventory.returned        -- stock returned and restocked
inventory.adjusted        -- manual adjustment (damage, count correction)
inventory.transferred     -- stock moved between locations

-- Cart events
cart.created              -- new cart initialised
cart.item_added           -- item added with quantity and price snapshot
cart.item_updated         -- quantity changed
cart.item_removed         -- item removed from cart
cart.promotion_applied    -- discount code applied
cart.promotion_removed    -- discount code removed
cart.address_set          -- shipping/billing address attached
cart.abandoned            -- cart expired without conversion
cart.converted            -- cart converted to order

-- Order events
order.placed              -- order submitted (contains full order snapshot)
order.confirmed           -- order confirmed by system/staff
order.payment_authorized  -- payment gateway authorised
order.payment_captured    -- payment captured
order.payment_failed      -- payment failed
order.item_fulfilled      -- items shipped
order.fully_fulfilled     -- all items shipped
order.cancelled           -- order cancelled (with reason)
order.refund_requested    -- refund initiated
order.refund_processed    -- refund completed
order.note_added          -- internal note added

-- Customer events
customer.registered       -- new customer account
customer.updated          -- profile changes
customer.address_added    -- new address
customer.address_updated  -- address modified
customer.address_removed  -- address deleted
customer.consent_given    -- GDPR marketing consent
customer.consent_revoked  -- GDPR consent withdrawn
customer.deactivated      -- account deactivated
customer.data_exported    -- GDPR data export requested
customer.data_erased      -- GDPR erasure executed (crypto-shredding)
```

### Example Event Payloads

```json
-- product.created event
{
    "product_id": "550e8400-e29b-41d4-a716-446655440001",
    "product_type_id": "550e8400-e29b-41d4-a716-446655440010",
    "name": "Wireless Headphones Pro",
    "slug": "wireless-headphones-pro",
    "description": "Premium wireless headphones with ANC",
    "status": "draft"
}

-- order.placed event (contains full snapshot)
{
    "order_id": "550e8400-e29b-41d4-a716-446655440100",
    "order_number": "ORD-2026-001234",
    "customer_id": "550e8400-e29b-41d4-a716-446655440050",
    "channel": "web",
    "currency_code": "USD",
    "items": [
        {
            "variant_id": "...",
            "sku": "WHP-BLK-001",
            "product_name": "Wireless Headphones Pro",
            "variant_name": "Black",
            "quantity": 1,
            "unit_price_cents": 29999,
            "total_cents": 29999
        }
    ],
    "subtotal_cents": 29999,
    "discount_cents": 0,
    "shipping_cents": 999,
    "tax_cents": 2480,
    "total_cents": 33478,
    "shipping_address": {
        "first_name": "Jane",
        "last_name": "Smith",
        "address_line1": "123 Main St",
        "city": "San Francisco",
        "subdivision_code": "US-CA",
        "country_code": "US",
        "postal_code": "94102"
    }
}

-- inventory.reserved event
{
    "variant_id": "...",
    "stock_location_id": "...",
    "quantity": 1,
    "reason": "cart_reservation",
    "reference_type": "cart",
    "reference_id": "550e8400-e29b-41d4-a716-446655440200",
    "previous_available": 42,
    "new_available": 41
}
```

---

## Read Models (Projections)

Read models are materialised views rebuilt from events. They are optimised for specific query patterns.

### Product Catalog Projection

```sql
-- Materialised product read model (rebuilt from product.* events)
CREATE TABLE rm_products (
    product_id UUID PRIMARY KEY,
    product_type_id UUID NOT NULL,
    name VARCHAR(500) NOT NULL,
    slug VARCHAR(500) NOT NULL UNIQUE,
    description TEXT,
    status VARCHAR(20) NOT NULL,
    category_ids UUID[] NOT NULL DEFAULT '{}',
    variant_count INTEGER NOT NULL DEFAULT 0,
    min_price_cents BIGINT,
    max_price_cents BIGINT,
    default_currency CHAR(3),
    primary_image_url TEXT,
    attributes JSONB NOT NULL DEFAULT '{}',
    last_event_sequence BIGINT NOT NULL,
    last_event_time TIMESTAMPTZ NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_products_status ON rm_products(status);
CREATE INDEX idx_rm_products_slug ON rm_products(slug);
CREATE INDEX idx_rm_products_categories ON rm_products USING gin(category_ids);
CREATE INDEX idx_rm_products_name ON rm_products USING gin(name gin_trgm_ops);

-- Product variants read model
CREATE TABLE rm_product_variants (
    variant_id UUID PRIMARY KEY,
    product_id UUID NOT NULL REFERENCES rm_products(product_id),
    sku VARCHAR(255) NOT NULL UNIQUE,
    name VARCHAR(500),
    barcode VARCHAR(255),
    weight_grams INTEGER,
    is_master BOOLEAN NOT NULL DEFAULT FALSE,
    attributes JSONB NOT NULL DEFAULT '{}',
    prices JSONB NOT NULL DEFAULT '[]',
    -- Denormalized price example in prices:
    -- [{"price_list_id": "...", "amount_cents": 2999, "currency": "USD", "min_qty": 1}]
    inventory_available INTEGER NOT NULL DEFAULT 0,
    sort_order INTEGER NOT NULL DEFAULT 0,
    last_event_sequence BIGINT NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_variants_product ON rm_product_variants(product_id);
CREATE INDEX idx_rm_variants_sku ON rm_product_variants(sku);
```

### Order Projection

```sql
-- Materialised order read model (rebuilt from order.* events)
CREATE TABLE rm_orders (
    order_id UUID PRIMARY KEY,
    order_number VARCHAR(50) NOT NULL UNIQUE,
    customer_id UUID,
    customer_email VARCHAR(320),
    channel_id UUID NOT NULL,
    status VARCHAR(30) NOT NULL,
    payment_status VARCHAR(30) NOT NULL DEFAULT 'pending',
    fulfillment_status VARCHAR(30) NOT NULL DEFAULT 'unfulfilled',
    currency_code CHAR(3) NOT NULL,
    subtotal_cents BIGINT NOT NULL,
    discount_cents BIGINT NOT NULL DEFAULT 0,
    shipping_cents BIGINT NOT NULL DEFAULT 0,
    tax_cents BIGINT NOT NULL DEFAULT 0,
    total_cents BIGINT NOT NULL,
    item_count INTEGER NOT NULL,
    shipping_address JSONB,
    billing_address JSONB,
    notes TEXT,
    placed_at TIMESTAMPTZ,
    cancelled_at TIMESTAMPTZ,
    last_event_sequence BIGINT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_orders_customer ON rm_orders(customer_id);
CREATE INDEX idx_rm_orders_status ON rm_orders(status);
CREATE INDEX idx_rm_orders_placed ON rm_orders(placed_at DESC);
CREATE INDEX idx_rm_orders_number ON rm_orders(order_number);

-- Order items read model
CREATE TABLE rm_order_items (
    id UUID PRIMARY KEY,
    order_id UUID NOT NULL REFERENCES rm_orders(order_id),
    variant_id UUID,
    sku VARCHAR(255) NOT NULL,
    product_name VARCHAR(500) NOT NULL,
    variant_name VARCHAR(500),
    quantity INTEGER NOT NULL,
    unit_price_cents BIGINT NOT NULL,
    discount_cents BIGINT NOT NULL DEFAULT 0,
    tax_cents BIGINT NOT NULL DEFAULT 0,
    total_cents BIGINT NOT NULL,
    fulfillment_status VARCHAR(30) NOT NULL DEFAULT 'unfulfilled'
);

CREATE INDEX idx_rm_oi_order ON rm_order_items(order_id);
```

### Inventory Projection

```sql
-- Real-time inventory read model (rebuilt from inventory.* events)
CREATE TABLE rm_inventory (
    variant_id UUID NOT NULL,
    stock_location_id UUID NOT NULL,
    quantity_on_hand INTEGER NOT NULL DEFAULT 0,
    quantity_reserved INTEGER NOT NULL DEFAULT 0,
    quantity_available INTEGER NOT NULL DEFAULT 0,
    last_event_sequence BIGINT NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (variant_id, stock_location_id)
);

CREATE INDEX idx_rm_inv_variant ON rm_inventory(variant_id);
CREATE INDEX idx_rm_inv_low_stock ON rm_inventory(quantity_available) WHERE quantity_available <= 0;
```

### Customer Projection

```sql
-- Customer read model (rebuilt from customer.* events)
CREATE TABLE rm_customers (
    customer_id UUID PRIMARY KEY,
    email VARCHAR(320) NOT NULL UNIQUE,
    first_name VARCHAR(255),
    last_name VARCHAR(255),
    phone VARCHAR(50),
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    accepts_marketing BOOLEAN NOT NULL DEFAULT FALSE,
    order_count INTEGER NOT NULL DEFAULT 0,
    total_spent_cents BIGINT NOT NULL DEFAULT 0,
    last_order_at TIMESTAMPTZ,
    addresses JSONB NOT NULL DEFAULT '[]',
    customer_group_ids UUID[] NOT NULL DEFAULT '{}',
    last_event_sequence BIGINT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_customers_email ON rm_customers(email);
```

### Analytics Projection

```sql
-- Daily sales aggregation (rebuilt from order.placed events)
CREATE TABLE rm_daily_sales (
    date DATE NOT NULL,
    channel_id UUID NOT NULL,
    currency_code CHAR(3) NOT NULL,
    order_count INTEGER NOT NULL DEFAULT 0,
    item_count INTEGER NOT NULL DEFAULT 0,
    gross_revenue_cents BIGINT NOT NULL DEFAULT 0,
    discount_total_cents BIGINT NOT NULL DEFAULT 0,
    net_revenue_cents BIGINT NOT NULL DEFAULT 0,
    avg_order_value_cents BIGINT NOT NULL DEFAULT 0,
    new_customer_count INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY (date, channel_id, currency_code)
);

-- Product performance (rebuilt from order.placed + product events)
CREATE TABLE rm_product_performance (
    product_id UUID NOT NULL,
    period_start DATE NOT NULL,
    period_type VARCHAR(10) NOT NULL CHECK (period_type IN ('daily', 'weekly', 'monthly')),
    units_sold INTEGER NOT NULL DEFAULT 0,
    revenue_cents BIGINT NOT NULL DEFAULT 0,
    view_count INTEGER NOT NULL DEFAULT 0,
    cart_add_count INTEGER NOT NULL DEFAULT 0,
    conversion_rate NUMERIC(5,4) NOT NULL DEFAULT 0,
    PRIMARY KEY (product_id, period_start, period_type)
);
```

---

## Projection Infrastructure

```sql
-- Tracks the last processed event for each projection (checkpointing)
CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(255) PRIMARY KEY,
    last_event_id UUID NOT NULL,
    last_event_time TIMESTAMPTZ NOT NULL,
    last_processed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    status VARCHAR(20) NOT NULL DEFAULT 'running' CHECK (status IN ('running', 'paused', 'rebuilding', 'error')),
    error_message TEXT
);

-- Snapshot store for aggregates with long event histories
-- Instead of replaying 10,000 events, load the latest snapshot + events since
CREATE TABLE aggregate_snapshots (
    aggregate_type VARCHAR(100) NOT NULL,
    aggregate_id UUID NOT NULL,
    sequence_number BIGINT NOT NULL,
    state JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (aggregate_type, aggregate_id, sequence_number)
);

-- Dead letter queue for events that failed projection processing
CREATE TABLE projection_dead_letters (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    projection_name VARCHAR(255) NOT NULL,
    event_id UUID NOT NULL REFERENCES event_store(event_id),
    error_message TEXT NOT NULL,
    retry_count INTEGER NOT NULL DEFAULT 0,
    last_retry_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pdl_projection ON projection_dead_letters(projection_name);
```

## GDPR Crypto-Shredding Support

```sql
-- Customer encryption keys (for crypto-shredding on erasure)
-- Customer PII in events is encrypted with per-customer keys
-- Deleting the key makes all PII in the event store unrecoverable
CREATE TABLE customer_encryption_keys (
    customer_id UUID PRIMARY KEY,
    encrypted_key BYTEA NOT NULL,  -- AES key encrypted with master key
    key_version INTEGER NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    rotated_at TIMESTAMPTZ
);
```

---

## Example: Temporal Query (Replaying State at a Point in Time)

```sql
-- "What was the inventory for variant X at location Y on March 15, 2026?"
-- Replay all inventory events for this aggregate up to the target time

SELECT
    (e.data->>'variant_id')::UUID AS variant_id,
    (e.data->>'stock_location_id')::UUID AS stock_location_id,
    SUM(
        CASE
            WHEN e.type IN ('inventory.received', 'inventory.returned', 'inventory.unreserved')
                THEN (e.data->>'quantity')::INTEGER
            WHEN e.type IN ('inventory.sold', 'inventory.reserved', 'inventory.damaged')
                THEN -(e.data->>'quantity')::INTEGER
            WHEN e.type = 'inventory.adjusted'
                THEN (e.data->>'quantity_change')::INTEGER
            ELSE 0
        END
    ) AS computed_quantity_on_hand
FROM event_store e
WHERE e.aggregate_type = 'inventory'
  AND e.aggregate_id = '...'  -- aggregate ID for this variant+location
  AND e.time <= '2026-03-15T23:59:59Z'
ORDER BY e.sequence_number;

-- "Show me the complete history of order ORD-2026-001234"
SELECT
    e.type,
    e.time,
    e.actor_type,
    e.actor_id,
    e.data
FROM event_store e
WHERE e.aggregate_type = 'order'
  AND e.subject = 'ORD-2026-001234'
ORDER BY e.sequence_number;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store (Write) | 1 | Single append-only event table (partitioned by month) |
| Product Read Models | 2 | rm_products, rm_product_variants |
| Order Read Models | 2 | rm_orders, rm_order_items |
| Inventory Read Model | 1 | rm_inventory |
| Customer Read Model | 1 | rm_customers |
| Analytics Read Models | 2 | rm_daily_sales, rm_product_performance |
| Infrastructure | 4 | projection_checkpoints, aggregate_snapshots, dead_letters, encryption_keys |
| **Total** | **13** | Plus reference/config tables for channels, tax, shipping (~8 more) |

---

## Key Design Decisions

1. **Single event store table** — all domain events share one table, distinguished by `aggregate_type` and `type`. This simplifies infrastructure (one append path, one replication stream) and enables cross-aggregate queries. Monthly partitioning prevents the table from becoming unwieldy.

2. **CloudEvents envelope as the event schema** — every event follows the CloudEvents v1.0 specification for `type`, `source`, `subject`, `time`, and `data`. This means events can be published directly to external event buses (Kafka, EventBridge) without transformation.

3. **Optimistic concurrency via sequence numbers** — the `UNIQUE (aggregate_type, aggregate_id, sequence_number)` constraint prevents conflicting concurrent writes to the same aggregate. If two commands try to append event #5 for the same order, one fails and must retry.

4. **Projections are disposable** — all `rm_*` read model tables can be dropped and rebuilt from the event store. The `projection_checkpoints` table tracks how far each projection has consumed the event stream, enabling incremental catch-up after downtime.

5. **Aggregate snapshots for performance** — for aggregates with long event histories (e.g., a product with 5,000 price changes), snapshots store a pre-computed state at a given sequence number. Replay only needs to process events after the snapshot.

6. **Crypto-shredding for GDPR** — customer PII in event payloads is encrypted with per-customer keys. To honour a right-to-erasure request, the customer's key is deleted, making their PII in all historical events permanently unrecoverable without modifying the immutable event store.

7. **Dead letter queue for projection failures** — if a projection handler fails to process an event (schema mismatch, bug), the event goes to a dead letter table for manual investigation rather than blocking the entire projection pipeline.

8. **Analytics projections built from events** — sales aggregations and product performance metrics are computed by projections consuming order and interaction events, not by querying operational tables. This decouples analytics from transactional workloads.

9. **Correlation and causation IDs for tracing** — every event carries optional `correlation_id` (links all events from one user action) and `causation_id` (the specific event that caused this one), enabling full distributed tracing across aggregates.

10. **Read models store denormalized data** — `rm_product_variants.prices` is a JSONB array of all prices, and `rm_customers.addresses` is a JSONB array of all addresses. This avoids JOINs on the read path, optimising for the API response pattern where a single GraphQL query fetches a product with all its variants and prices.
