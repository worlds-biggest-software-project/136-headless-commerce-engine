# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Headless Commerce Engine · Created: 2026-05-19

## Philosophy

This model follows classical relational database design principles: every concept gets its own table, relationships are expressed through foreign keys and junction tables, and data integrity is enforced at the schema level through constraints, unique indexes, and check constraints. The schema is fully normalized to Third Normal Form (3NF), eliminating data duplication and ensuring that every fact is stored exactly once.

This approach mirrors how mature enterprise commerce platforms like commercetools and BigCommerce structure their internal data, even when they expose it through GraphQL or REST APIs. It prioritises query flexibility and referential integrity over write throughput, making it well-suited for commerce workloads where reads vastly outnumber writes and data consistency is critical (inventory counts, pricing, order state).

The trade-off is a higher table count and more complex JOIN queries, but PostgreSQL handles this efficiently with proper indexing. This model is the most straightforward for teams with strong SQL experience and provides the clearest migration path to other relational databases.

**Best for:** Teams that want a conventional, well-understood relational schema with strong data integrity guarantees and maximum query flexibility.

**Trade-offs:**
- Pro: Strong referential integrity; no orphaned records possible
- Pro: Standard SQL tooling, ORMs, and reporting tools work out of the box
- Pro: Clear schema evolution path via standard migrations
- Pro: Easy to reason about for new developers
- Con: Higher table count (~45-55 tables) increases JOIN complexity
- Con: Schema changes require migrations; adding new product attributes means ALTER TABLE
- Con: Multi-tenant isolation requires careful RLS policy design rather than schema separation
- Con: Extensible/custom fields require EAV pattern or additional tables

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 4217 | `currency_code CHAR(3)` on all monetary fields references ISO 4217 |
| ISO 3166-1 | `country_code CHAR(2)` on addresses and tax zones uses ISO 3166-1 alpha-2 |
| ISO 3166-2 | `subdivision_code VARCHAR(6)` for state/province in tax jurisdiction mapping |
| ISO 8601 | All `TIMESTAMPTZ` columns store UTC timestamps in ISO 8601 format |
| OAuth 2.0 / OIDC | `api_clients` and `customer_sessions` tables model OAuth token grants |
| PCI DSS v4.0.1 | No raw card data stored; `payment_methods` stores only gateway tokens |
| CloudEvents v1.0 | `domain_events` table envelope matches CloudEvents metadata fields |
| Schema.org Product | Product attributes align with schema.org Product vocabulary for JSON-LD export |
| JSON:API / OpenAPI 3.1 | Table structure maps cleanly to JSON:API resource objects |

---

## Catalog Management

```sql
-- Product Types define the attribute schema for a group of products
CREATE TABLE product_types (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) NOT NULL UNIQUE,
    description TEXT,
    is_digital BOOLEAN NOT NULL DEFAULT FALSE,
    is_shipping_required BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Attribute definitions (reusable across product types)
CREATE TABLE attribute_definitions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) NOT NULL UNIQUE,
    input_type VARCHAR(50) NOT NULL CHECK (input_type IN (
        'text', 'numeric', 'boolean', 'date', 'datetime',
        'rich_text', 'dropdown', 'multiselect', 'swatch', 'file'
    )),
    is_required BOOLEAN NOT NULL DEFAULT FALSE,
    is_filterable BOOLEAN NOT NULL DEFAULT FALSE,
    is_variant_attribute BOOLEAN NOT NULL DEFAULT FALSE,
    sort_order INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Junction: which attributes belong to which product types
CREATE TABLE product_type_attributes (
    product_type_id UUID NOT NULL REFERENCES product_types(id) ON DELETE CASCADE,
    attribute_definition_id UUID NOT NULL REFERENCES attribute_definitions(id) ON DELETE CASCADE,
    sort_order INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY (product_type_id, attribute_definition_id)
);

-- Categories (hierarchical via parent_id adjacency list)
CREATE TABLE categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parent_id UUID REFERENCES categories(id) ON DELETE SET NULL,
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) NOT NULL UNIQUE,
    description TEXT,
    level INTEGER NOT NULL DEFAULT 0,
    path TEXT NOT NULL DEFAULT '',  -- materialized path e.g. '/electronics/phones/'
    sort_order INTEGER NOT NULL DEFAULT 0,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_categories_parent ON categories(parent_id);
CREATE INDEX idx_categories_path ON categories USING gist (path gist_trgm_ops);

-- Products
CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_type_id UUID NOT NULL REFERENCES product_types(id),
    name VARCHAR(500) NOT NULL,
    slug VARCHAR(500) NOT NULL UNIQUE,
    description TEXT,
    status VARCHAR(20) NOT NULL DEFAULT 'draft' CHECK (status IN ('draft', 'active', 'archived')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_products_status ON products(status);
CREATE INDEX idx_products_product_type ON products(product_type_id);
CREATE INDEX idx_products_name_trgm ON products USING gin (name gin_trgm_ops);

-- Product-Category junction (many-to-many)
CREATE TABLE product_categories (
    product_id UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    category_id UUID NOT NULL REFERENCES categories(id) ON DELETE CASCADE,
    sort_order INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY (product_id, category_id)
);

-- Product Variants (the sellable unit)
CREATE TABLE product_variants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    sku VARCHAR(255) NOT NULL UNIQUE,
    name VARCHAR(500),
    barcode VARCHAR(255),
    weight_grams INTEGER,
    is_master BOOLEAN NOT NULL DEFAULT FALSE,
    sort_order INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_variants_product ON product_variants(product_id);
CREATE INDEX idx_variants_sku ON product_variants(sku);

-- Product Attribute Values (stores actual values for each product)
CREATE TABLE product_attribute_values (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    attribute_definition_id UUID NOT NULL REFERENCES attribute_definitions(id),
    variant_id UUID REFERENCES product_variants(id) ON DELETE CASCADE,  -- NULL = product-level
    value_text TEXT,
    value_numeric NUMERIC,
    value_boolean BOOLEAN,
    value_date DATE,
    value_datetime TIMESTAMPTZ,
    UNIQUE (product_id, attribute_definition_id, variant_id)
);

CREATE INDEX idx_pav_product ON product_attribute_values(product_id);
CREATE INDEX idx_pav_attribute ON product_attribute_values(attribute_definition_id);

-- Product Media (images, videos, files)
CREATE TABLE product_media (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    variant_id UUID REFERENCES product_variants(id) ON DELETE CASCADE,
    media_type VARCHAR(20) NOT NULL CHECK (media_type IN ('image', 'video', 'file')),
    url TEXT NOT NULL,
    alt_text VARCHAR(500),
    sort_order INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_media_product ON product_media(product_id);
```

## Pricing and Promotions

```sql
-- Channels represent distinct sales channels (web, mobile, marketplace, B2B portal)
CREATE TABLE channels (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) NOT NULL UNIQUE,
    currency_code CHAR(3) NOT NULL,  -- ISO 4217
    country_code CHAR(2),            -- ISO 3166-1 alpha-2
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Price Lists (per channel, per customer group)
CREATE TABLE price_lists (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    channel_id UUID NOT NULL REFERENCES channels(id),
    customer_group_id UUID REFERENCES customer_groups(id),
    currency_code CHAR(3) NOT NULL,  -- ISO 4217
    priority INTEGER NOT NULL DEFAULT 0,
    valid_from TIMESTAMPTZ,
    valid_until TIMESTAMPTZ,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_price_lists_channel ON price_lists(channel_id);

-- Prices (variant + price list = price)
CREATE TABLE prices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    variant_id UUID NOT NULL REFERENCES product_variants(id) ON DELETE CASCADE,
    price_list_id UUID NOT NULL REFERENCES price_lists(id) ON DELETE CASCADE,
    amount_cents BIGINT NOT NULL,
    compare_at_cents BIGINT,  -- original/compare-at price for display
    min_quantity INTEGER NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (variant_id, price_list_id, min_quantity)
);

CREATE INDEX idx_prices_variant ON prices(variant_id);
CREATE INDEX idx_prices_list ON prices(price_list_id);

-- Promotions
CREATE TABLE promotions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    code VARCHAR(100) UNIQUE,
    type VARCHAR(50) NOT NULL CHECK (type IN (
        'percentage_off', 'fixed_amount_off', 'buy_x_get_y',
        'free_shipping', 'bundle_price'
    )),
    value NUMERIC NOT NULL,
    currency_code CHAR(3),  -- for fixed_amount_off
    min_order_cents BIGINT,
    max_uses INTEGER,
    current_uses INTEGER NOT NULL DEFAULT 0,
    valid_from TIMESTAMPTZ NOT NULL,
    valid_until TIMESTAMPTZ,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_promotions_code ON promotions(code);
CREATE INDEX idx_promotions_active ON promotions(is_active, valid_from, valid_until);

-- Promotion-Channel junction
CREATE TABLE promotion_channels (
    promotion_id UUID NOT NULL REFERENCES promotions(id) ON DELETE CASCADE,
    channel_id UUID NOT NULL REFERENCES channels(id) ON DELETE CASCADE,
    PRIMARY KEY (promotion_id, channel_id)
);
```

## Inventory

```sql
-- Warehouses / stock locations
CREATE TABLE stock_locations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    address_line1 VARCHAR(500),
    address_line2 VARCHAR(500),
    city VARCHAR(255),
    subdivision_code VARCHAR(6),  -- ISO 3166-2
    country_code CHAR(2) NOT NULL,  -- ISO 3166-1
    postal_code VARCHAR(20),
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Inventory levels per variant per location
CREATE TABLE inventory_levels (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    variant_id UUID NOT NULL REFERENCES product_variants(id) ON DELETE CASCADE,
    stock_location_id UUID NOT NULL REFERENCES stock_locations(id) ON DELETE CASCADE,
    quantity_on_hand INTEGER NOT NULL DEFAULT 0,
    quantity_reserved INTEGER NOT NULL DEFAULT 0,
    quantity_available INTEGER GENERATED ALWAYS AS (quantity_on_hand - quantity_reserved) STORED,
    low_stock_threshold INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (variant_id, stock_location_id)
);

CREATE INDEX idx_inventory_variant ON inventory_levels(variant_id);
CREATE INDEX idx_inventory_location ON inventory_levels(stock_location_id);
CREATE INDEX idx_inventory_low_stock ON inventory_levels(quantity_available) WHERE quantity_available <= 0;

-- Inventory movements (audit trail for stock changes)
CREATE TABLE inventory_movements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inventory_level_id UUID NOT NULL REFERENCES inventory_levels(id),
    quantity_change INTEGER NOT NULL,
    reason VARCHAR(50) NOT NULL CHECK (reason IN (
        'received', 'sold', 'returned', 'adjusted',
        'reserved', 'unreserved', 'damaged', 'transferred'
    )),
    reference_type VARCHAR(50),  -- 'order', 'transfer', 'adjustment'
    reference_id UUID,
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by UUID
);

CREATE INDEX idx_inv_movements_level ON inventory_movements(inventory_level_id);
CREATE INDEX idx_inv_movements_created ON inventory_movements(created_at);
```

## Customers and Authentication

```sql
-- Customer groups (wholesale, VIP, etc.)
CREATE TABLE customer_groups (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) NOT NULL UNIQUE,
    description TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Customers
CREATE TABLE customers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(320) NOT NULL UNIQUE,
    first_name VARCHAR(255),
    last_name VARCHAR(255),
    phone VARCHAR(50),
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    accepts_marketing BOOLEAN NOT NULL DEFAULT FALSE,
    marketing_consent_at TIMESTAMPTZ,  -- GDPR consent timestamp
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_customers_email ON customers(email);

-- Customer-Group junction
CREATE TABLE customer_group_memberships (
    customer_id UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    customer_group_id UUID NOT NULL REFERENCES customer_groups(id) ON DELETE CASCADE,
    PRIMARY KEY (customer_id, customer_group_id)
);

-- Customer addresses
CREATE TABLE customer_addresses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    label VARCHAR(100),  -- 'home', 'work', 'billing'
    first_name VARCHAR(255) NOT NULL,
    last_name VARCHAR(255) NOT NULL,
    company VARCHAR(255),
    address_line1 VARCHAR(500) NOT NULL,
    address_line2 VARCHAR(500),
    city VARCHAR(255) NOT NULL,
    subdivision_code VARCHAR(6),  -- ISO 3166-2
    country_code CHAR(2) NOT NULL,  -- ISO 3166-1
    postal_code VARCHAR(20),
    phone VARCHAR(50),
    is_default_shipping BOOLEAN NOT NULL DEFAULT FALSE,
    is_default_billing BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_addresses_customer ON customer_addresses(customer_id);

-- B2B: Company accounts
CREATE TABLE companies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(500) NOT NULL,
    tax_id VARCHAR(100),
    registration_number VARCHAR(100),
    customer_group_id UUID REFERENCES customer_groups(id),
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Company-Customer junction (employees of a company)
CREATE TABLE company_members (
    company_id UUID NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    customer_id UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    role VARCHAR(50) NOT NULL DEFAULT 'buyer' CHECK (role IN ('admin', 'buyer', 'approver', 'viewer')),
    spending_limit_cents BIGINT,
    PRIMARY KEY (company_id, customer_id)
);

-- API clients (OAuth 2.0)
CREATE TABLE api_clients (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id VARCHAR(255) NOT NULL UNIQUE,
    client_secret_hash VARCHAR(500) NOT NULL,
    name VARCHAR(255) NOT NULL,
    grant_types VARCHAR(50)[] NOT NULL DEFAULT '{client_credentials}',
    scopes VARCHAR(100)[] NOT NULL DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Cart and Checkout

```sql
-- Shopping carts
CREATE TABLE carts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id UUID REFERENCES customers(id) ON DELETE SET NULL,
    channel_id UUID NOT NULL REFERENCES channels(id),
    status VARCHAR(20) NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'merged', 'converted', 'abandoned')),
    currency_code CHAR(3) NOT NULL,  -- ISO 4217
    email VARCHAR(320),
    shipping_address_id UUID REFERENCES customer_addresses(id),
    billing_address_id UUID REFERENCES customer_addresses(id),
    promotion_id UUID REFERENCES promotions(id),
    subtotal_cents BIGINT NOT NULL DEFAULT 0,
    discount_cents BIGINT NOT NULL DEFAULT 0,
    shipping_cents BIGINT NOT NULL DEFAULT 0,
    tax_cents BIGINT NOT NULL DEFAULT 0,
    total_cents BIGINT NOT NULL DEFAULT 0,
    notes TEXT,
    expires_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_carts_customer ON carts(customer_id);
CREATE INDEX idx_carts_status ON carts(status);
CREATE INDEX idx_carts_expires ON carts(expires_at) WHERE status = 'active';

-- Cart line items
CREATE TABLE cart_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cart_id UUID NOT NULL REFERENCES carts(id) ON DELETE CASCADE,
    variant_id UUID NOT NULL REFERENCES product_variants(id),
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    unit_price_cents BIGINT NOT NULL,
    total_cents BIGINT NOT NULL,
    -- Snapshot of product data at time of adding (denormalized for display)
    product_name VARCHAR(500) NOT NULL,
    variant_name VARCHAR(500),
    sku VARCHAR(255) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cart_items_cart ON cart_items(cart_id);
```

## Orders and Fulfillment

```sql
-- Orders
CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_number VARCHAR(50) NOT NULL UNIQUE,
    customer_id UUID REFERENCES customers(id) ON DELETE SET NULL,
    channel_id UUID NOT NULL REFERENCES channels(id),
    company_id UUID REFERENCES companies(id),
    status VARCHAR(30) NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending', 'confirmed', 'processing', 'partially_fulfilled',
        'fulfilled', 'cancelled', 'refunded'
    )),
    currency_code CHAR(3) NOT NULL,  -- ISO 4217
    subtotal_cents BIGINT NOT NULL,
    discount_cents BIGINT NOT NULL DEFAULT 0,
    shipping_cents BIGINT NOT NULL DEFAULT 0,
    tax_cents BIGINT NOT NULL DEFAULT 0,
    total_cents BIGINT NOT NULL,
    -- Snapshot addresses (independent of customer address changes)
    shipping_first_name VARCHAR(255),
    shipping_last_name VARCHAR(255),
    shipping_address_line1 VARCHAR(500),
    shipping_address_line2 VARCHAR(500),
    shipping_city VARCHAR(255),
    shipping_subdivision_code VARCHAR(6),
    shipping_country_code CHAR(2),
    shipping_postal_code VARCHAR(20),
    billing_first_name VARCHAR(255),
    billing_last_name VARCHAR(255),
    billing_address_line1 VARCHAR(500),
    billing_address_line2 VARCHAR(500),
    billing_city VARCHAR(255),
    billing_subdivision_code VARCHAR(6),
    billing_country_code CHAR(2),
    billing_postal_code VARCHAR(20),
    notes TEXT,
    placed_at TIMESTAMPTZ,
    cancelled_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_orders_customer ON orders(customer_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_placed ON orders(placed_at DESC);
CREATE INDEX idx_orders_number ON orders(order_number);

-- Order line items (snapshot of product at time of purchase)
CREATE TABLE order_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    variant_id UUID REFERENCES product_variants(id) ON DELETE SET NULL,
    product_name VARCHAR(500) NOT NULL,
    variant_name VARCHAR(500),
    sku VARCHAR(255) NOT NULL,
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    unit_price_cents BIGINT NOT NULL,
    discount_cents BIGINT NOT NULL DEFAULT 0,
    tax_cents BIGINT NOT NULL DEFAULT 0,
    total_cents BIGINT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_order_items_order ON order_items(order_id);

-- Payments
CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    gateway VARCHAR(50) NOT NULL,  -- 'stripe', 'paypal', 'braintree'
    gateway_transaction_id VARCHAR(255),
    gateway_token VARCHAR(500),  -- tokenized reference, never raw card data (PCI DSS)
    method VARCHAR(50) NOT NULL CHECK (method IN (
        'credit_card', 'debit_card', 'paypal', 'apple_pay',
        'google_pay', 'bank_transfer', 'gift_card'
    )),
    status VARCHAR(30) NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending', 'authorized', 'captured', 'partially_refunded',
        'refunded', 'voided', 'failed'
    )),
    amount_cents BIGINT NOT NULL,
    currency_code CHAR(3) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payments_order ON payments(order_id);
CREATE INDEX idx_payments_gateway_txn ON payments(gateway, gateway_transaction_id);

-- Fulfillments (shipments)
CREATE TABLE fulfillments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    stock_location_id UUID REFERENCES stock_locations(id),
    status VARCHAR(30) NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending', 'processing', 'shipped', 'delivered', 'cancelled'
    )),
    tracking_number VARCHAR(255),
    tracking_url TEXT,
    carrier VARCHAR(100),
    shipped_at TIMESTAMPTZ,
    delivered_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_fulfillments_order ON fulfillments(order_id);

-- Fulfillment items (which items are in which shipment)
CREATE TABLE fulfillment_items (
    fulfillment_id UUID NOT NULL REFERENCES fulfillments(id) ON DELETE CASCADE,
    order_item_id UUID NOT NULL REFERENCES order_items(id) ON DELETE CASCADE,
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    PRIMARY KEY (fulfillment_id, order_item_id)
);
```

## Tax Configuration

```sql
-- Tax zones (groups of jurisdictions)
CREATE TABLE tax_zones (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Tax zone members (which countries/subdivisions are in which zone)
CREATE TABLE tax_zone_members (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tax_zone_id UUID NOT NULL REFERENCES tax_zones(id) ON DELETE CASCADE,
    country_code CHAR(2) NOT NULL,  -- ISO 3166-1
    subdivision_code VARCHAR(6),     -- ISO 3166-2 (NULL = entire country)
    postal_code_pattern VARCHAR(50)  -- optional postal code range
);

CREATE INDEX idx_tzm_zone ON tax_zone_members(tax_zone_id);
CREATE INDEX idx_tzm_country ON tax_zone_members(country_code, subdivision_code);

-- Tax rates
CREATE TABLE tax_rates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tax_zone_id UUID NOT NULL REFERENCES tax_zones(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,  -- 'GST', 'VAT', 'Sales Tax'
    rate_percent NUMERIC(6,4) NOT NULL,
    is_compound BOOLEAN NOT NULL DEFAULT FALSE,
    is_included_in_price BOOLEAN NOT NULL DEFAULT FALSE,  -- tax-inclusive pricing
    product_type_id UUID REFERENCES product_types(id),  -- NULL = applies to all
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tax_rates_zone ON tax_rates(tax_zone_id);
```

## Shipping

```sql
-- Shipping methods
CREATE TABLE shipping_methods (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    carrier VARCHAR(100),
    description TEXT,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Shipping rates (method + zone = rate)
CREATE TABLE shipping_rates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shipping_method_id UUID NOT NULL REFERENCES shipping_methods(id) ON DELETE CASCADE,
    tax_zone_id UUID NOT NULL REFERENCES tax_zones(id),
    min_order_cents BIGINT NOT NULL DEFAULT 0,
    max_order_cents BIGINT,
    rate_cents BIGINT NOT NULL,
    currency_code CHAR(3) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_shipping_rates_method ON shipping_rates(shipping_method_id);
```

## Domain Events and Webhooks

```sql
-- Domain events log (CloudEvents-aligned envelope)
CREATE TABLE domain_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    spec_version VARCHAR(10) NOT NULL DEFAULT '1.0',  -- CloudEvents specversion
    type VARCHAR(255) NOT NULL,        -- e.g. 'com.commerce.order.created'
    source VARCHAR(500) NOT NULL,      -- e.g. '/orders'
    subject VARCHAR(500),              -- e.g. order ID
    time TIMESTAMPTZ NOT NULL DEFAULT now(),
    data_content_type VARCHAR(100) NOT NULL DEFAULT 'application/json',
    data JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_events_type ON domain_events(type);
CREATE INDEX idx_events_time ON domain_events(time DESC);
CREATE INDEX idx_events_subject ON domain_events(subject);

-- Webhook subscriptions
CREATE TABLE webhook_subscriptions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    api_client_id UUID NOT NULL REFERENCES api_clients(id) ON DELETE CASCADE,
    target_url TEXT NOT NULL,
    event_types VARCHAR(255)[] NOT NULL,  -- array of event types to subscribe to
    secret_hash VARCHAR(500) NOT NULL,     -- HMAC-SHA256 signing secret
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_webhooks_client ON webhook_subscriptions(api_client_id);
```

## AI Features

```sql
-- Product embeddings for vector search
CREATE TABLE product_embeddings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    variant_id UUID REFERENCES product_variants(id) ON DELETE CASCADE,
    model_version VARCHAR(100) NOT NULL,
    embedding vector(1536) NOT NULL,  -- pgvector extension
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (product_id, variant_id, model_version)
);

CREATE INDEX idx_embeddings_product ON product_embeddings(product_id);
CREATE INDEX idx_embeddings_vector ON product_embeddings USING ivfflat (embedding vector_cosine_ops);

-- AI recommendation logs
CREATE TABLE recommendation_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id UUID REFERENCES customers(id),
    session_id VARCHAR(255),
    algorithm VARCHAR(100) NOT NULL,
    input_context JSONB,
    recommended_product_ids UUID[] NOT NULL,
    clicked_product_id UUID REFERENCES products(id),
    converted BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_reclogs_customer ON recommendation_logs(customer_id);
CREATE INDEX idx_reclogs_created ON recommendation_logs(created_at DESC);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Catalog Management | 8 | Products, variants, types, attributes, categories, media |
| Pricing & Promotions | 5 | Channels, price lists, prices, promotions, promotion-channels |
| Inventory | 3 | Stock locations, inventory levels, inventory movements |
| Customers & Auth | 6 | Customers, groups, addresses, companies, members, API clients |
| Cart & Checkout | 2 | Carts, cart items |
| Orders & Fulfillment | 5 | Orders, order items, payments, fulfillments, fulfillment items |
| Tax & Shipping | 5 | Tax zones, zone members, tax rates, shipping methods, rates |
| Events & Webhooks | 2 | Domain events, webhook subscriptions |
| AI Features | 2 | Product embeddings, recommendation logs |
| **Total** | **38** | |

---

## Key Design Decisions

1. **UUID primary keys everywhere** — enables distributed ID generation, safe for multi-region deployments, and avoids sequential ID enumeration (OWASP BOLA mitigation).

2. **Monetary values stored as BIGINT cents** — avoids floating-point rounding errors. All amounts are in the smallest currency unit (cents for USD/EUR, yen for JPY). The currency code is always stored alongside the amount.

3. **Order items snapshot product data** — `product_name`, `sku`, and `unit_price_cents` are denormalized onto order items because products change after orders are placed. The `variant_id` FK is SET NULL on delete so historical orders survive product deletion.

4. **Adjacency list + materialized path for categories** — `parent_id` provides simple parent-child queries while `path` enables efficient subtree queries without recursive CTEs for common read operations.

5. **Separate price lists rather than price columns on variants** — supports multi-market, multi-currency, and customer-group-specific pricing without schema changes. Adding a new market means inserting a new price list row, not altering the variants table.

6. **Inventory movements as append-only audit trail** — every stock change creates an `inventory_movements` record, providing a complete history of why inventory changed. The `inventory_levels` table maintains the current computed state.

7. **CloudEvents-aligned domain events table** — the `domain_events` table follows the CloudEvents v1.0 envelope structure, making it straightforward to publish events to external systems (Kafka, webhooks) in a standards-compliant format.

8. **No raw payment card data** — only gateway tokens and transaction IDs are stored, maintaining PCI DSS compliance. The payment gateway handles all card data.

9. **pgvector for AI search** — product embeddings use the pgvector extension with IVFFlat indexing, keeping vector search in the same database rather than requiring a separate vector store for MVP.

10. **ISO standards for all geography and currency fields** — country codes (ISO 3166-1 alpha-2), subdivision codes (ISO 3166-2), currency codes (ISO 4217), and timestamps (ISO 8601 via TIMESTAMPTZ) are used consistently across all tables.
