# Data Model Suggestion 4: Microservice-Decomposed (MACH-Aligned)

> Project: Headless Commerce Engine · Created: 2026-05-19

## Philosophy

This model decomposes the commerce domain into independent bounded contexts, each owning its own database schema. Instead of a single monolithic database, each microservice (Catalog, Pricing, Inventory, Cart, Order, Customer, Payment, Fulfillment) has its own schema with no cross-schema foreign keys. Services communicate through asynchronous events (CloudEvents over a message bus) and synchronous API calls where necessary.

This approach directly implements the MACH Alliance architecture (Microservices, API-first, Cloud-native, Headless) and mirrors how enterprise platforms like Elastic Path and commercetools structure their backends. Each service can be developed, deployed, scaled, and versioned independently. The Catalog service can use a different storage strategy (JSONB-heavy) than the Order service (fully normalised) because each schema is optimised for its specific workload.

The trade-off is operational complexity: cross-service queries require API composition rather than JOINs, data consistency is eventual rather than immediate, and the infrastructure footprint (message bus, service discovery, per-service databases or schemas) is significantly larger. However, for a headless commerce engine that aspires to enterprise scale and composability — where customers may want to use only the Catalog and Pricing services while bringing their own Order management — this decomposition is the natural architecture.

**Best for:** Teams building for enterprise scale, composable commerce, and independent service deployment where customers may adopt individual services a la carte.

**Trade-offs:**
- Pro: Each service scales independently (inventory spikes don't affect catalog reads)
- Pro: Services can be replaced individually (swap in a different payment provider)
- Pro: Teams can work on services in parallel without merge conflicts
- Pro: Failure isolation — one service going down doesn't take down the entire platform
- Pro: Natural alignment with MACH Alliance architecture
- Con: Cross-service queries require API composition (no database JOINs)
- Con: Eventual consistency across services requires saga patterns for multi-step operations
- Con: Significantly more infrastructure (message bus, per-service schemas, service mesh)
- Con: Data duplication across services (product name in catalog AND in order snapshots)
- Con: Higher operational complexity (monitoring, debugging, distributed tracing)
- Con: Overkill for small teams or early-stage products

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| MACH Alliance | Architecture directly implements MACH principles; each service is independently deployable |
| CloudEvents v1.0 | All inter-service events use CloudEvents envelope format |
| AsyncAPI v3.1 | Each service publishes an AsyncAPI document for its event channels |
| OpenAPI 3.1 | Each service publishes its own OpenAPI spec for synchronous API |
| ISO 4217 | Currency codes in Pricing and Order service schemas |
| ISO 3166-1/2 | Country/subdivision codes in Customer, Fulfillment, and Tax services |
| ISO 8601 | All timestamps across all services use ISO 8601 UTC |
| PCI DSS v4.0.1 | Payment service is the only service that handles payment tokens; isolated for PCI scope reduction |
| OAuth 2.0 / OIDC | API Gateway handles auth; service-to-service uses mTLS or signed JWTs |
| GDPR | Customer service owns all PII; data erasure is a single-service operation with event broadcast |

---

## Service Decomposition

```
┌─────────────────────────────────────────────────────────────┐
│                        API Gateway                          │
│              (OAuth 2.0, Rate Limiting, Routing)            │
└──────┬────────┬────────┬────────┬────────┬────────┬────────┘
       │        │        │        │        │        │
  ┌────▼──┐ ┌──▼───┐ ┌──▼───┐ ┌──▼──┐ ┌──▼───┐ ┌──▼────┐
  │Catalog│ │Price │ │Inven │ │Cart │ │Order │ │Custom │
  │Service│ │Serv. │ │tory  │ │Serv.│ │Serv. │ │er Srv │
  └───┬───┘ └──┬───┘ └──┬───┘ └──┬──┘ └──┬───┘ └──┬────┘
      │        │        │        │        │        │
  ┌───▼───┐ ┌──▼───┐ ┌──▼───┐ ┌──▼──┐ ┌──▼───┐ ┌──▼────┐
  │catalog│ │price │ │inven │ │cart  │ │order │ │custom │
  │_db    │ │_db   │ │tory  │ │_db   │ │_db   │ │er_db  │
  └───────┘ └──────┘ │_db   │ └─────┘ └──────┘ └───────┘
                     └──────┘
  ┌────────┐ ┌──────────┐ ┌─────────┐ ┌──────────┐
  │Payment │ │Fulfillmnt│ │Search & │ │Analytics │
  │Service │ │Service   │ │AI Serv. │ │Service   │
  └───┬────┘ └────┬─────┘ └────┬────┘ └────┬─────┘
  ┌───▼────┐ ┌────▼─────┐ ┌────▼────┐ ┌────▼─────┐
  │payment │ │fulfill   │ │search   │ │analytics │
  │_db     │ │_db       │ │_db      │ │_db       │
  └────────┘ └──────────┘ └─────────┘ └──────────┘

         ╔══════════════════════════════════╗
         ║     Event Bus (Kafka/NATS)       ║
         ║   CloudEvents v1.0 envelope      ║
         ╚══════════════════════════════════╝
```

---

## Catalog Service Schema (`catalog_db`)

```sql
-- Owns: products, variants, categories, product types, media
-- Publishes: product.created, product.updated, product.archived,
--            product.variant_added, product.variant_updated

CREATE SCHEMA catalog;

CREATE TABLE catalog.product_types (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) NOT NULL UNIQUE,
    description TEXT,
    is_digital BOOLEAN NOT NULL DEFAULT FALSE,
    is_shipping_required BOOLEAN NOT NULL DEFAULT TRUE,
    attribute_schema JSONB NOT NULL DEFAULT '{}',
    variant_attributes TEXT[] NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE catalog.categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parent_id UUID REFERENCES catalog.categories(id) ON DELETE SET NULL,
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) NOT NULL UNIQUE,
    path TEXT NOT NULL DEFAULT '',
    level INTEGER NOT NULL DEFAULT 0,
    sort_order INTEGER NOT NULL DEFAULT 0,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    metadata JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE catalog.products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_type_id UUID NOT NULL REFERENCES catalog.product_types(id),
    name VARCHAR(500) NOT NULL,
    slug VARCHAR(500) NOT NULL UNIQUE,
    description TEXT,
    status VARCHAR(20) NOT NULL DEFAULT 'draft' CHECK (status IN ('draft', 'active', 'archived')),
    attributes JSONB NOT NULL DEFAULT '{}',
    metadata JSONB NOT NULL DEFAULT '{}',
    category_ids UUID[] NOT NULL DEFAULT '{}',
    media JSONB NOT NULL DEFAULT '[]',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cat_products_status ON catalog.products(status);
CREATE INDEX idx_cat_products_name ON catalog.products USING gin(name gin_trgm_ops);
CREATE INDEX idx_cat_products_attrs ON catalog.products USING gin(attributes);

CREATE TABLE catalog.product_variants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id UUID NOT NULL REFERENCES catalog.products(id) ON DELETE CASCADE,
    sku VARCHAR(255) NOT NULL UNIQUE,
    name VARCHAR(500),
    barcode VARCHAR(255),
    weight_grams INTEGER,
    is_master BOOLEAN NOT NULL DEFAULT FALSE,
    attributes JSONB NOT NULL DEFAULT '{}',
    sort_order INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cat_variants_product ON catalog.product_variants(product_id);
CREATE INDEX idx_cat_variants_sku ON catalog.product_variants(sku);

-- Outbox table for reliable event publishing (transactional outbox pattern)
CREATE TABLE catalog.outbox (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type VARCHAR(100) NOT NULL,
    aggregate_id UUID NOT NULL,
    event_type VARCHAR(255) NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    published_at TIMESTAMPTZ  -- NULL until consumed by event relay
);

CREATE INDEX idx_cat_outbox_unpublished ON catalog.outbox(created_at) WHERE published_at IS NULL;
```

## Pricing Service Schema (`price_db`)

```sql
-- Owns: price lists, prices, promotions
-- Consumes: product.created, product.variant_added (to validate variant IDs)
-- Publishes: price.set, price.changed, promotion.created, promotion.activated

CREATE SCHEMA pricing;

-- Local reference of known variant IDs (populated from catalog events)
CREATE TABLE pricing.known_variants (
    variant_id UUID PRIMARY KEY,
    sku VARCHAR(255) NOT NULL,
    last_seen_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE pricing.channels (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) NOT NULL UNIQUE,
    currency_code CHAR(3) NOT NULL,
    country_code CHAR(2),
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    config JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE pricing.price_lists (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    channel_id UUID NOT NULL REFERENCES pricing.channels(id),
    customer_group_id UUID,  -- no FK — customer group is owned by Customer Service
    currency_code CHAR(3) NOT NULL,
    priority INTEGER NOT NULL DEFAULT 0,
    valid_from TIMESTAMPTZ,
    valid_until TIMESTAMPTZ,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE pricing.prices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    variant_id UUID NOT NULL,  -- no FK — variant is owned by Catalog Service
    price_list_id UUID NOT NULL REFERENCES pricing.price_lists(id) ON DELETE CASCADE,
    amount_cents BIGINT NOT NULL,
    compare_at_cents BIGINT,
    min_quantity INTEGER NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (variant_id, price_list_id, min_quantity)
);

CREATE INDEX idx_pr_prices_variant ON pricing.prices(variant_id);

CREATE TABLE pricing.promotions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    code VARCHAR(100) UNIQUE,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    valid_from TIMESTAMPTZ NOT NULL,
    valid_until TIMESTAMPTZ,
    max_uses INTEGER,
    current_uses INTEGER NOT NULL DEFAULT 0,
    channel_ids UUID[] NOT NULL DEFAULT '{}',
    rules JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE pricing.outbox (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type VARCHAR(100) NOT NULL,
    aggregate_id UUID NOT NULL,
    event_type VARCHAR(255) NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    published_at TIMESTAMPTZ
);

CREATE INDEX idx_pr_outbox_unpublished ON pricing.outbox(created_at) WHERE published_at IS NULL;
```

## Inventory Service Schema (`inventory_db`)

```sql
-- Owns: stock locations, inventory levels, reservations
-- Consumes: order.placed (decrement stock), cart.converted (confirm reservation)
-- Publishes: inventory.reserved, inventory.sold, inventory.low_stock

CREATE SCHEMA inventory;

CREATE TABLE inventory.stock_locations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    country_code CHAR(2) NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    details JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE inventory.levels (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    variant_id UUID NOT NULL,  -- no FK — owned by Catalog Service
    stock_location_id UUID NOT NULL REFERENCES inventory.stock_locations(id),
    quantity_on_hand INTEGER NOT NULL DEFAULT 0,
    quantity_reserved INTEGER NOT NULL DEFAULT 0,
    quantity_available INTEGER GENERATED ALWAYS AS (quantity_on_hand - quantity_reserved) STORED,
    low_stock_threshold INTEGER,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (variant_id, stock_location_id)
);

CREATE INDEX idx_inv_variant ON inventory.levels(variant_id);
CREATE INDEX idx_inv_low_stock ON inventory.levels(quantity_available) WHERE quantity_available <= 0;

-- Reservations (temporary holds for carts, released on expiry or conversion)
CREATE TABLE inventory.reservations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    variant_id UUID NOT NULL,
    stock_location_id UUID NOT NULL REFERENCES inventory.stock_locations(id),
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    reference_type VARCHAR(50) NOT NULL,  -- 'cart', 'order'
    reference_id UUID NOT NULL,           -- cart_id or order_id
    expires_at TIMESTAMPTZ NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'confirmed', 'released', 'expired')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inv_res_variant ON inventory.reservations(variant_id);
CREATE INDEX idx_inv_res_reference ON inventory.reservations(reference_type, reference_id);
CREATE INDEX idx_inv_res_expires ON inventory.reservations(expires_at) WHERE status = 'active';

CREATE TABLE inventory.movements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    level_id UUID NOT NULL REFERENCES inventory.levels(id),
    quantity_change INTEGER NOT NULL,
    reason VARCHAR(50) NOT NULL,
    reference_type VARCHAR(50),
    reference_id UUID,
    context JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by UUID
);

CREATE INDEX idx_inv_mov_level ON inventory.movements(level_id);
CREATE INDEX idx_inv_mov_created ON inventory.movements(created_at DESC);

CREATE TABLE inventory.outbox (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type VARCHAR(100) NOT NULL,
    aggregate_id UUID NOT NULL,
    event_type VARCHAR(255) NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    published_at TIMESTAMPTZ
);

CREATE INDEX idx_inv_outbox_unpublished ON inventory.outbox(created_at) WHERE published_at IS NULL;
```

## Cart Service Schema (`cart_db`)

```sql
-- Owns: shopping carts
-- Consumes: inventory.reserved (confirmation), pricing events
-- Publishes: cart.created, cart.item_added, cart.converted, cart.abandoned

CREATE SCHEMA cart;

CREATE TABLE cart.carts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id UUID,  -- no FK — owned by Customer Service
    channel_id UUID NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'merged', 'converted', 'abandoned')),
    currency_code CHAR(3) NOT NULL,
    email VARCHAR(320),
    items JSONB NOT NULL DEFAULT '[]',
    shipping_address JSONB,
    billing_address JSONB,
    subtotal_cents BIGINT NOT NULL DEFAULT 0,
    discount_cents BIGINT NOT NULL DEFAULT 0,
    shipping_cents BIGINT NOT NULL DEFAULT 0,
    tax_cents BIGINT NOT NULL DEFAULT 0,
    total_cents BIGINT NOT NULL DEFAULT 0,
    applied_promotions JSONB NOT NULL DEFAULT '[]',
    notes TEXT,
    expires_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cart_customer ON cart.carts(customer_id);
CREATE INDEX idx_cart_status ON cart.carts(status);
CREATE INDEX idx_cart_expires ON cart.carts(expires_at) WHERE status = 'active';

CREATE TABLE cart.outbox (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type VARCHAR(100) NOT NULL,
    aggregate_id UUID NOT NULL,
    event_type VARCHAR(255) NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    published_at TIMESTAMPTZ
);

CREATE INDEX idx_cart_outbox_unpublished ON cart.outbox(created_at) WHERE published_at IS NULL;
```

## Order Service Schema (`order_db`)

```sql
-- Owns: orders, order lifecycle
-- Consumes: cart.converted, payment.captured, fulfillment.shipped
-- Publishes: order.placed, order.confirmed, order.cancelled, order.fulfilled

CREATE SCHEMA orders;

CREATE TABLE orders.orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_number VARCHAR(50) NOT NULL UNIQUE,
    customer_id UUID,
    channel_id UUID NOT NULL,
    company_id UUID,
    status VARCHAR(30) NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending', 'confirmed', 'processing', 'partially_fulfilled',
        'fulfilled', 'cancelled', 'refunded'
    )),
    payment_status VARCHAR(30) NOT NULL DEFAULT 'pending',
    fulfillment_status VARCHAR(30) NOT NULL DEFAULT 'unfulfilled',
    currency_code CHAR(3) NOT NULL,
    subtotal_cents BIGINT NOT NULL,
    discount_cents BIGINT NOT NULL DEFAULT 0,
    shipping_cents BIGINT NOT NULL DEFAULT 0,
    tax_cents BIGINT NOT NULL DEFAULT 0,
    total_cents BIGINT NOT NULL,
    -- Snapshot data (denormalized from other services at order placement time)
    items JSONB NOT NULL,
    shipping_address JSONB NOT NULL,
    billing_address JSONB NOT NULL,
    applied_promotions JSONB NOT NULL DEFAULT '[]',
    tax_breakdown JSONB NOT NULL DEFAULT '[]',
    notes TEXT,
    placed_at TIMESTAMPTZ,
    cancelled_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ord_customer ON orders.orders(customer_id);
CREATE INDEX idx_ord_status ON orders.orders(status);
CREATE INDEX idx_ord_placed ON orders.orders(placed_at DESC);
CREATE INDEX idx_ord_number ON orders.orders(order_number);

-- Order status history (audit trail within order service)
CREATE TABLE orders.status_history (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES orders.orders(id) ON DELETE CASCADE,
    from_status VARCHAR(30),
    to_status VARCHAR(30) NOT NULL,
    reason TEXT,
    actor_id UUID,
    actor_type VARCHAR(50),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ord_hist_order ON orders.status_history(order_id);

-- Saga state for multi-service order orchestration
CREATE TABLE orders.sagas (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES orders.orders(id),
    saga_type VARCHAR(100) NOT NULL,  -- 'place_order', 'cancel_order', 'refund_order'
    status VARCHAR(30) NOT NULL DEFAULT 'started' CHECK (status IN (
        'started', 'payment_pending', 'inventory_pending',
        'completed', 'compensating', 'failed'
    )),
    state JSONB NOT NULL DEFAULT '{}',
    -- Example state:
    -- {
    --   "steps_completed": ["inventory_reserved", "payment_authorized"],
    --   "steps_pending": ["payment_capture"],
    --   "compensation_needed": [],
    --   "error": null
    -- }
    started_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at TIMESTAMPTZ,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ord_sagas_order ON orders.sagas(order_id);
CREATE INDEX idx_ord_sagas_status ON orders.sagas(status) WHERE status NOT IN ('completed', 'failed');

CREATE TABLE orders.outbox (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type VARCHAR(100) NOT NULL,
    aggregate_id UUID NOT NULL,
    event_type VARCHAR(255) NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    published_at TIMESTAMPTZ
);

CREATE INDEX idx_ord_outbox_unpublished ON orders.outbox(created_at) WHERE published_at IS NULL;
```

## Customer Service Schema (`customer_db`)

```sql
-- Owns: customers, addresses, customer groups, companies, authentication
-- Publishes: customer.registered, customer.updated, customer.data_erased

CREATE SCHEMA customer;

CREATE TABLE customer.customers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(320) NOT NULL UNIQUE,
    password_hash VARCHAR(500),
    first_name VARCHAR(255),
    last_name VARCHAR(255),
    phone VARCHAR(50),
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    accepts_marketing BOOLEAN NOT NULL DEFAULT FALSE,
    marketing_consent_at TIMESTAMPTZ,
    customer_group_ids UUID[] NOT NULL DEFAULT '{}',
    addresses JSONB NOT NULL DEFAULT '[]',
    custom_fields JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cust_email ON customer.customers(email);
CREATE INDEX idx_cust_groups ON customer.customers USING gin(customer_group_ids);

CREATE TABLE customer.customer_groups (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) NOT NULL UNIQUE,
    description TEXT,
    config JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE customer.companies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(500) NOT NULL,
    tax_id VARCHAR(100),
    registration_number VARCHAR(100),
    customer_group_id UUID REFERENCES customer.customer_groups(id),
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    settings JSONB NOT NULL DEFAULT '{}',
    members JSONB NOT NULL DEFAULT '[]',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- API clients and OAuth (could also be a separate Auth service)
CREATE TABLE customer.api_clients (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id VARCHAR(255) NOT NULL UNIQUE,
    client_secret_hash VARCHAR(500) NOT NULL,
    name VARCHAR(255) NOT NULL,
    grant_types TEXT[] NOT NULL DEFAULT '{client_credentials}',
    scopes TEXT[] NOT NULL DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    config JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE customer.outbox (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type VARCHAR(100) NOT NULL,
    aggregate_id UUID NOT NULL,
    event_type VARCHAR(255) NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    published_at TIMESTAMPTZ
);

CREATE INDEX idx_cust_outbox_unpublished ON customer.outbox(created_at) WHERE published_at IS NULL;
```

## Payment Service Schema (`payment_db`)

```sql
-- Owns: payment transactions, refunds
-- Isolated for PCI DSS scope reduction
-- Consumes: order.placed (initiate payment)
-- Publishes: payment.authorized, payment.captured, payment.failed, payment.refunded

CREATE SCHEMA payment;

CREATE TABLE payment.transactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL,  -- no FK — owned by Order Service
    gateway VARCHAR(50) NOT NULL,
    status VARCHAR(30) NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending', 'authorized', 'captured', 'partially_refunded',
        'refunded', 'voided', 'failed'
    )),
    method VARCHAR(50) NOT NULL,
    amount_cents BIGINT NOT NULL,
    currency_code CHAR(3) NOT NULL,
    gateway_data JSONB NOT NULL DEFAULT '{}',
    idempotency_key VARCHAR(255) UNIQUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pay_order ON payment.transactions(order_id);
CREATE INDEX idx_pay_status ON payment.transactions(status);

CREATE TABLE payment.refunds (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    transaction_id UUID NOT NULL REFERENCES payment.transactions(id),
    amount_cents BIGINT NOT NULL,
    reason TEXT,
    gateway_refund_id VARCHAR(255),
    status VARCHAR(20) NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'processed', 'failed')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pay_refund_txn ON payment.refunds(transaction_id);

CREATE TABLE payment.outbox (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type VARCHAR(100) NOT NULL,
    aggregate_id UUID NOT NULL,
    event_type VARCHAR(255) NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    published_at TIMESTAMPTZ
);

CREATE INDEX idx_pay_outbox_unpublished ON payment.outbox(created_at) WHERE published_at IS NULL;
```

## Fulfillment Service Schema (`fulfill_db`)

```sql
-- Owns: fulfillments, shipments
-- Consumes: order.confirmed (create fulfillment tasks)
-- Publishes: fulfillment.created, fulfillment.shipped, fulfillment.delivered

CREATE SCHEMA fulfillment;

CREATE TABLE fulfillment.fulfillments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL,
    stock_location_id UUID,
    status VARCHAR(30) NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending', 'processing', 'shipped', 'delivered', 'cancelled'
    )),
    tracking_number VARCHAR(255),
    tracking_url TEXT,
    carrier VARCHAR(100),
    items JSONB NOT NULL,
    shipped_at TIMESTAMPTZ,
    delivered_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ful_order ON fulfillment.fulfillments(order_id);
CREATE INDEX idx_ful_status ON fulfillment.fulfillments(status);

CREATE TABLE fulfillment.outbox (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type VARCHAR(100) NOT NULL,
    aggregate_id UUID NOT NULL,
    event_type VARCHAR(255) NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    published_at TIMESTAMPTZ
);

CREATE INDEX idx_ful_outbox_unpublished ON fulfillment.outbox(created_at) WHERE published_at IS NULL;
```

## Search and AI Service Schema (`search_db`)

```sql
-- Owns: search indexes, embeddings, recommendation models
-- Consumes: product.*, pricing.*, inventory.* events to build search index
-- Publishes: search.query_logged (for analytics)

CREATE SCHEMA search;

-- Denormalized product search index (rebuilt from catalog + pricing + inventory events)
CREATE TABLE search.product_index (
    product_id UUID PRIMARY KEY,
    variant_ids UUID[] NOT NULL DEFAULT '{}',
    name VARCHAR(500) NOT NULL,
    slug VARCHAR(500) NOT NULL,
    description TEXT,
    status VARCHAR(20) NOT NULL,
    product_type_slug VARCHAR(255),
    category_ids UUID[] NOT NULL DEFAULT '{}',
    category_paths TEXT[] NOT NULL DEFAULT '{}',
    attributes JSONB NOT NULL DEFAULT '{}',
    channel_ids UUID[] NOT NULL DEFAULT '{}',
    -- Denormalized pricing (min/max across all price lists)
    min_price_cents BIGINT,
    max_price_cents BIGINT,
    prices_by_channel JSONB NOT NULL DEFAULT '{}',
    -- Denormalized inventory
    total_available INTEGER NOT NULL DEFAULT 0,
    in_stock BOOLEAN NOT NULL DEFAULT TRUE,
    -- Search-specific fields
    search_text TSVECTOR,
    boost_score NUMERIC NOT NULL DEFAULT 1.0,
    last_synced_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_search_status ON search.product_index(status);
CREATE INDEX idx_search_categories ON search.product_index USING gin(category_ids);
CREATE INDEX idx_search_attrs ON search.product_index USING gin(attributes);
CREATE INDEX idx_search_channels ON search.product_index USING gin(channel_ids);
CREATE INDEX idx_search_text ON search.product_index USING gin(search_text);
CREATE INDEX idx_search_price ON search.product_index(min_price_cents);
CREATE INDEX idx_search_stock ON search.product_index(in_stock);

-- Vector embeddings for AI-powered search
CREATE TABLE search.embeddings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id UUID NOT NULL,
    variant_id UUID,
    model_version VARCHAR(100) NOT NULL,
    embedding vector(1536) NOT NULL,
    ai_metadata JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (product_id, variant_id, model_version)
);

CREATE INDEX idx_search_emb_product ON search.embeddings(product_id);
CREATE INDEX idx_search_emb_vector ON search.embeddings USING ivfflat (embedding vector_cosine_ops);

-- Search query log (for analytics and AI training)
CREATE TABLE search.query_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    query_text TEXT NOT NULL,
    filters JSONB NOT NULL DEFAULT '{}',
    result_count INTEGER NOT NULL,
    result_product_ids UUID[] NOT NULL DEFAULT '{}',
    clicked_product_id UUID,
    customer_id UUID,
    session_id VARCHAR(255),
    channel_id UUID,
    response_time_ms INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_search_qlog_created ON search.query_log(created_at DESC);
```

---

## Inter-Service Event Flow (Order Placement Saga)

```
Customer places order via API Gateway
    │
    ▼
Cart Service ──[cart.converted]──► Order Service creates order (status: pending)
    │
    ▼
Order Service starts PlaceOrder saga
    │
    ├──[order.placement_requested]──► Inventory Service reserves stock
    │                                    │
    │                                    ▼
    │                              [inventory.reserved] or [inventory.insufficient]
    │
    ├──[order.payment_requested]──► Payment Service authorises payment
    │                                    │
    │                                    ▼
    │                              [payment.authorized] or [payment.failed]
    │
    ▼
All steps succeed → Order Service confirms order
    │
    ├──[order.confirmed]──► Fulfillment Service creates fulfillment tasks
    ├──[order.confirmed]──► Inventory Service confirms reservation
    ├──[order.confirmed]──► Analytics Service updates metrics
    │
    ▼
If payment fails → Order Service starts compensation
    │
    ├──[order.cancelled]──► Inventory Service releases reservation
    └──[order.cancelled]──► Customer Service sends notification
```

---

## Table Count Summary

| Service | Tables | Notes |
|---------|--------|-------|
| Catalog Service | 5 | Products, variants, product types, categories, outbox |
| Pricing Service | 6 | Channels, price lists, prices, promotions, known_variants, outbox |
| Inventory Service | 5 | Stock locations, levels, reservations, movements, outbox |
| Cart Service | 2 | Carts, outbox |
| Order Service | 4 | Orders, status history, sagas, outbox |
| Customer Service | 5 | Customers, groups, companies, API clients, outbox |
| Payment Service | 3 | Transactions, refunds, outbox |
| Fulfillment Service | 2 | Fulfillments, outbox |
| Search & AI Service | 3 | Product index, embeddings, query log |
| **Total** | **35** | Across 9 independent schemas/databases |

---

## Key Design Decisions

1. **No cross-schema foreign keys** — each service owns its data completely. The Pricing service stores `variant_id` as a plain UUID, not as a foreign key to the Catalog service. Data consistency is enforced through events and application logic, not database constraints.

2. **Transactional outbox pattern for reliable events** — every service has an `outbox` table. When a service changes state, it writes the entity update AND the outbox event in a single database transaction. A separate event relay process reads unpublished outbox rows and publishes them to the event bus. This guarantees exactly-once event publishing without two-phase commits.

3. **Saga pattern for multi-service operations** — the Order service maintains a `sagas` table that tracks the state of multi-step operations (place order, cancel order, refund). If any step fails, the saga orchestrator triggers compensating actions (release inventory, void payment).

4. **Payment service isolated for PCI scope** — by keeping all payment-related data in a separate `payment_db` schema/database, the PCI DSS audit scope is limited to the Payment service infrastructure. Other services never see or store payment tokens.

5. **Search service as a CQRS read model** — the Search service maintains a denormalized `product_index` that combines catalog, pricing, and inventory data from events. This read-optimised index serves all product search and browse queries without hitting the Catalog, Pricing, or Inventory services.

6. **Each service can use different storage** — while all schemas here use PostgreSQL for consistency, the architecture allows each service to choose its optimal storage. The Search service could use Elasticsearch, the Cart service could use Redis, and the Analytics service could use ClickHouse.

7. **Inventory reservations as first-class entities** — the Inventory service has a dedicated `reservations` table with expiry tracking, separating temporary cart holds from permanent order decrements. This prevents phantom stock issues where abandoned carts permanently reduce available inventory.

8. **Event-driven data synchronisation** — the Pricing service maintains a `known_variants` table populated from Catalog service events. This allows the Pricing service to validate that a variant exists before setting a price, without making synchronous API calls to the Catalog service.

9. **Order data is fully snapshotted** — when an order is placed, all relevant data (items, addresses, prices, promotions, tax breakdown) is snapshotted into the order record. The order is self-contained and does not depend on data from any other service.

10. **9 independent deployment units** — each service can be developed by a different team, deployed on a different schedule, and scaled based on its specific load pattern. The Inventory service might need 10x the instances during flash sales, while the Catalog service remains stable.
