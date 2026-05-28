# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Headless Commerce Engine · Created: 2026-05-19

## Philosophy

This model combines the strengths of normalised relational tables for core commerce entities with PostgreSQL JSONB columns for extensible, variable, and domain-specific fields. The core schema (products, orders, customers, inventory) uses relational columns with proper types, constraints, and foreign keys. But wherever the data varies by product type, channel, market, or tenant — attributes, metadata, custom fields, configuration — a JSONB column provides schema-on-write flexibility without ALTER TABLE migrations.

This is the approach used internally by Medusa.js (which allows custom data via its module system) and is conceptually similar to how commercetools extends its core resources with Custom Types and Custom Fields. PostgreSQL's JSONB is not a NoSQL escape hatch; it supports GIN indexing, containment queries (`@>`), path queries (`->>`, `#>>`), and partial updates (`jsonb_set`), making it a first-class query target alongside relational columns.

The key insight is that a headless commerce engine must support wildly different product attribute schemas (a t-shirt has size and colour; a laptop has RAM, CPU, and screen size; a wine has vintage, region, and grape variety) without requiring schema migrations for each new product type. JSONB columns absorb this variability while the relational skeleton ensures referential integrity, efficient JOINs, and standard SQL reporting.

**Best for:** Teams that want rapid iteration, multi-market flexibility, and the ability to add custom fields without migrations — while keeping strong relational integrity for core commerce operations.

**Trade-offs:**
- Pro: No schema migrations needed for new product attributes, custom fields, or tenant-specific data
- Pro: Fewer tables than a fully normalised model (~25-30 vs. ~38+)
- Pro: JSONB columns with GIN indexes support efficient containment and path queries
- Pro: Natural fit for GraphQL APIs that return nested, variable-shape objects
- Pro: Easy to add per-market or per-channel configuration without new tables
- Con: JSONB data is not constrained by the database; validation must happen at the application layer
- Con: No foreign key constraints inside JSONB — referential integrity is application-enforced
- Con: Complex JSONB queries can be harder to optimise than simple relational JOINs
- Con: Schema documentation for JSONB fields requires discipline (comments, JSON Schema)
- Con: Reporting/BI tools may struggle with nested JSONB structures

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 4217 | `currency_code CHAR(3)` on relational columns; currency in JSONB pricing arrays |
| ISO 3166-1/2 | Country/subdivision codes in relational address columns and JSONB address objects |
| ISO 8601 | All TIMESTAMPTZ columns; date strings in JSONB use ISO 8601 format |
| JSON Schema Draft 2020-12 | JSONB column structure documented via JSON Schema (application-level validation) |
| Schema.org Product | Product `attributes` JSONB aligns with schema.org property names for easy JSON-LD export |
| OpenAPI 3.1 | JSONB structures documented in OpenAPI schemas using JSON Schema integration |
| CloudEvents v1.0 | Event payloads use JSONB with CloudEvents envelope |
| PCI DSS v4.0.1 | Payment data limited to gateway tokens; no card data in JSONB |
| OAuth 2.0 | Scopes and grants stored as JSONB arrays on API client records |

---

## Catalog Management

```sql
-- Product types define validation rules for the attributes JSONB column
CREATE TABLE product_types (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) NOT NULL UNIQUE,
    description TEXT,
    is_digital BOOLEAN NOT NULL DEFAULT FALSE,
    is_shipping_required BOOLEAN NOT NULL DEFAULT TRUE,
    -- JSON Schema that defines and validates the attributes JSONB on products of this type
    -- Example:
    -- {
    --   "type": "object",
    --   "properties": {
    --     "color": {"type": "string", "enum": ["red", "blue", "green", "black", "white"]},
    --     "size": {"type": "string", "enum": ["XS", "S", "M", "L", "XL", "XXL"]},
    --     "material": {"type": "string"},
    --     "weight_kg": {"type": "number", "minimum": 0}
    --   },
    --   "required": ["color", "size"]
    -- }
    attribute_schema JSONB NOT NULL DEFAULT '{}',
    -- Which attributes from the schema are used to distinguish variants
    variant_attributes TEXT[] NOT NULL DEFAULT '{}',  -- e.g. ['color', 'size']
    -- Which attributes are filterable in search/browse
    filterable_attributes TEXT[] NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Categories (adjacency list with materialised path)
CREATE TABLE categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parent_id UUID REFERENCES categories(id) ON DELETE SET NULL,
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) NOT NULL UNIQUE,
    description TEXT,
    path TEXT NOT NULL DEFAULT '',
    level INTEGER NOT NULL DEFAULT 0,
    sort_order INTEGER NOT NULL DEFAULT 0,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    -- Category-specific metadata (display settings, SEO, etc.)
    metadata JSONB NOT NULL DEFAULT '{}',
    -- Example metadata:
    -- {
    --   "seo_title": "Wireless Headphones | Best Deals",
    --   "seo_description": "Shop premium wireless headphones...",
    --   "banner_image_url": "https://cdn.example.com/banners/headphones.jpg",
    --   "display_mode": "grid"
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_categories_parent ON categories(parent_id);
CREATE INDEX idx_categories_path ON categories(path);

-- Products (core relational + JSONB attributes)
CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_type_id UUID NOT NULL REFERENCES product_types(id),
    name VARCHAR(500) NOT NULL,
    slug VARCHAR(500) NOT NULL UNIQUE,
    description TEXT,
    status VARCHAR(20) NOT NULL DEFAULT 'draft' CHECK (status IN ('draft', 'active', 'archived')),
    -- Variable product-level attributes (validated against product_type.attribute_schema)
    attributes JSONB NOT NULL DEFAULT '{}',
    -- Example for a clothing product:
    -- {
    --   "brand": "Acme Apparel",
    --   "material": "100% organic cotton",
    --   "care_instructions": "Machine wash cold",
    --   "country_of_origin": "PT"
    -- }

    -- SEO and display metadata
    metadata JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "seo_title": "Organic Cotton T-Shirt - Acme Apparel",
    --   "seo_description": "...",
    --   "tags": ["sustainable", "organic", "basics"],
    --   "schema_org": { "@type": "Product", "brand": {"@type": "Brand", "name": "Acme"} }
    -- }

    -- Channel availability and visibility
    channel_config JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "web": {"visible": true, "featured": false},
    --   "mobile": {"visible": true, "featured": true},
    --   "b2b_portal": {"visible": false}
    -- }

    -- Media as JSONB array (avoids separate media table)
    media JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {"type": "image", "url": "https://cdn.example.com/img1.jpg", "alt": "Front view", "sort": 0},
    --   {"type": "image", "url": "https://cdn.example.com/img2.jpg", "alt": "Side view", "sort": 1},
    --   {"type": "video", "url": "https://cdn.example.com/demo.mp4", "alt": "Product demo", "sort": 2}
    -- ]

    category_ids UUID[] NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_products_status ON products(status);
CREATE INDEX idx_products_type ON products(product_type_id);
CREATE INDEX idx_products_name ON products USING gin(name gin_trgm_ops);
CREATE INDEX idx_products_attributes ON products USING gin(attributes);
CREATE INDEX idx_products_categories ON products USING gin(category_ids);
CREATE INDEX idx_products_channel ON products USING gin(channel_config);

-- Product variants
CREATE TABLE product_variants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    sku VARCHAR(255) NOT NULL UNIQUE,
    name VARCHAR(500),
    barcode VARCHAR(255),
    weight_grams INTEGER,
    is_master BOOLEAN NOT NULL DEFAULT FALSE,
    sort_order INTEGER NOT NULL DEFAULT 0,

    -- Variant-specific attributes (subset defined by product_type.variant_attributes)
    attributes JSONB NOT NULL DEFAULT '{}',
    -- Example: {"color": "black", "size": "M"}

    -- All prices for this variant across price lists, markets, and currencies
    prices JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {"price_list": "default", "currency": "USD", "amount_cents": 2999, "compare_at_cents": 3999, "min_qty": 1},
    --   {"price_list": "wholesale", "currency": "USD", "amount_cents": 1999, "min_qty": 10},
    --   {"price_list": "eu_market", "currency": "EUR", "amount_cents": 2799, "min_qty": 1}
    -- ]

    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_variants_product ON product_variants(product_id);
CREATE INDEX idx_variants_sku ON product_variants(sku);
CREATE INDEX idx_variants_attributes ON product_variants USING gin(attributes);
CREATE INDEX idx_variants_prices ON product_variants USING gin(prices);
```

### Example: Querying Products by JSONB Attributes

```sql
-- Find all black, size M products of type 'clothing'
SELECT p.id, p.name, v.sku, v.attributes, v.prices
FROM products p
JOIN product_variants v ON v.product_id = p.id
WHERE p.status = 'active'
  AND v.attributes @> '{"color": "black", "size": "M"}';

-- Find all products with a specific brand
SELECT id, name, attributes->>'brand' AS brand
FROM products
WHERE attributes @> '{"brand": "Acme Apparel"}';

-- Find all products visible on the B2B portal
SELECT id, name
FROM products
WHERE channel_config @> '{"b2b_portal": {"visible": true}}';

-- Find the USD price for a variant from the default price list
SELECT
    v.sku,
    price_entry->>'amount_cents' AS price_cents,
    price_entry->>'currency' AS currency
FROM product_variants v,
     jsonb_array_elements(v.prices) AS price_entry
WHERE v.id = '...'
  AND price_entry->>'price_list' = 'default'
  AND price_entry->>'currency' = 'USD'
  AND (price_entry->>'min_qty')::int <= 1;
```

## Channels and Pricing Configuration

```sql
-- Channels (relational for FK integrity, JSONB for channel-specific config)
CREATE TABLE channels (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) NOT NULL UNIQUE,
    currency_code CHAR(3) NOT NULL,  -- ISO 4217, default currency
    country_code CHAR(2),            -- ISO 3166-1, default country
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    -- Channel-specific configuration
    config JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "supported_currencies": ["USD", "CAD"],
    --   "supported_countries": ["US", "CA"],
    --   "tax_inclusive_pricing": false,
    --   "default_locale": "en-US",
    --   "checkout_config": {
    --     "require_phone": true,
    --     "allow_guest_checkout": true,
    --     "min_order_cents": 1000
    --   }
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Promotions (rules and conditions as JSONB)
CREATE TABLE promotions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    code VARCHAR(100) UNIQUE,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    valid_from TIMESTAMPTZ NOT NULL,
    valid_until TIMESTAMPTZ,
    max_uses INTEGER,
    current_uses INTEGER NOT NULL DEFAULT 0,
    channel_ids UUID[] NOT NULL DEFAULT '{}',

    -- Flexible promotion rules as JSONB
    rules JSONB NOT NULL,
    -- Example for "20% off orders over $50 on clothing":
    -- {
    --   "type": "percentage_off",
    --   "value": 20,
    --   "conditions": {
    --     "min_order_cents": 5000,
    --     "product_type_slugs": ["clothing"],
    --     "customer_groups": ["vip", "wholesale"]
    --   },
    --   "limits": {
    --     "max_discount_cents": 10000,
    --     "one_per_customer": true
    --   }
    -- }

    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_promotions_code ON promotions(code);
CREATE INDEX idx_promotions_active ON promotions(is_active, valid_from, valid_until);
CREATE INDEX idx_promotions_channels ON promotions USING gin(channel_ids);
```

## Inventory

```sql
-- Stock locations
CREATE TABLE stock_locations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    country_code CHAR(2) NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    -- Full address and config as JSONB (varies by location type)
    details JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "address": {"line1": "123 Warehouse Rd", "city": "Chicago", "state": "IL", "postal_code": "60601"},
    --   "type": "warehouse",
    --   "fulfillment_priority": 1,
    --   "supports_pickup": false
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Inventory levels (relational for performance-critical stock queries)
CREATE TABLE inventory_levels (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    variant_id UUID NOT NULL REFERENCES product_variants(id) ON DELETE CASCADE,
    stock_location_id UUID NOT NULL REFERENCES stock_locations(id) ON DELETE CASCADE,
    quantity_on_hand INTEGER NOT NULL DEFAULT 0,
    quantity_reserved INTEGER NOT NULL DEFAULT 0,
    quantity_available INTEGER GENERATED ALWAYS AS (quantity_on_hand - quantity_reserved) STORED,
    low_stock_threshold INTEGER,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (variant_id, stock_location_id)
);

CREATE INDEX idx_inv_variant ON inventory_levels(variant_id);
CREATE INDEX idx_inv_low_stock ON inventory_levels(quantity_available) WHERE quantity_available <= 0;

-- Inventory movements (append-only audit log)
CREATE TABLE inventory_movements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inventory_level_id UUID NOT NULL REFERENCES inventory_levels(id),
    quantity_change INTEGER NOT NULL,
    reason VARCHAR(50) NOT NULL CHECK (reason IN (
        'received', 'sold', 'returned', 'adjusted',
        'reserved', 'unreserved', 'damaged', 'transferred'
    )),
    reference_type VARCHAR(50),
    reference_id UUID,
    -- Additional context as JSONB
    context JSONB NOT NULL DEFAULT '{}',
    -- Example: {"po_number": "PO-2026-001", "supplier": "Acme Supplies", "notes": "Damaged in transit"}
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by UUID
);

CREATE INDEX idx_inv_mov_level ON inventory_movements(inventory_level_id);
CREATE INDEX idx_inv_mov_created ON inventory_movements(created_at DESC);
```

## Customers and Authentication

```sql
-- Customers (relational core + JSONB for extensible profile data)
CREATE TABLE customers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(320) NOT NULL UNIQUE,
    first_name VARCHAR(255),
    last_name VARCHAR(255),
    phone VARCHAR(50),
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    accepts_marketing BOOLEAN NOT NULL DEFAULT FALSE,
    marketing_consent_at TIMESTAMPTZ,

    -- Customer group memberships as array (avoids junction table)
    customer_group_ids UUID[] NOT NULL DEFAULT '{}',

    -- Addresses as JSONB array (avoids separate addresses table)
    addresses JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {
    --     "id": "addr_01", "label": "home", "is_default_shipping": true, "is_default_billing": true,
    --     "first_name": "Jane", "last_name": "Smith", "company": null,
    --     "line1": "123 Main St", "line2": "Apt 4", "city": "San Francisco",
    --     "subdivision_code": "US-CA", "country_code": "US", "postal_code": "94102",
    --     "phone": "+14155551234"
    --   },
    --   {
    --     "id": "addr_02", "label": "work", "is_default_shipping": false, "is_default_billing": false,
    --     "first_name": "Jane", "last_name": "Smith", "company": "Acme Corp",
    --     "line1": "456 Market St", "city": "San Francisco",
    --     "subdivision_code": "US-CA", "country_code": "US", "postal_code": "94105"
    --   }
    -- ]

    -- Extensible custom profile data
    custom_fields JSONB NOT NULL DEFAULT '{}',
    -- Example: {"loyalty_tier": "gold", "preferred_language": "en", "birthday": "1990-05-15"}

    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_customers_email ON customers(email);
CREATE INDEX idx_customers_groups ON customers USING gin(customer_group_ids);

-- Customer groups
CREATE TABLE customer_groups (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) NOT NULL UNIQUE,
    description TEXT,
    -- Group-specific config (discount rules, feature flags, etc.)
    config JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- B2B companies
CREATE TABLE companies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(500) NOT NULL,
    tax_id VARCHAR(100),
    registration_number VARCHAR(100),
    customer_group_id UUID REFERENCES customer_groups(id),
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    -- Company-specific settings
    settings JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "payment_terms": "net_30",
    --   "credit_limit_cents": 5000000,
    --   "auto_approve_orders_under_cents": 100000,
    --   "required_po_number": true,
    --   "approved_shipping_methods": ["standard", "express"]
    -- }

    -- Members as JSONB array (avoids junction table for small member lists)
    members JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {"customer_id": "...", "role": "admin", "spending_limit_cents": null},
    --   {"customer_id": "...", "role": "buyer", "spending_limit_cents": 50000}
    -- ]

    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- API clients (OAuth 2.0)
CREATE TABLE api_clients (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id VARCHAR(255) NOT NULL UNIQUE,
    client_secret_hash VARCHAR(500) NOT NULL,
    name VARCHAR(255) NOT NULL,
    grant_types TEXT[] NOT NULL DEFAULT '{client_credentials}',
    scopes TEXT[] NOT NULL DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    -- Client-specific config
    config JSONB NOT NULL DEFAULT '{}',
    -- Example: {"rate_limit_per_minute": 100, "allowed_channels": ["web"], "webhook_url": "..."}
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Cart and Checkout

```sql
-- Shopping carts (JSONB for line items — keeps the cart as a single document)
CREATE TABLE carts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id UUID REFERENCES customers(id) ON DELETE SET NULL,
    channel_id UUID NOT NULL REFERENCES channels(id),
    status VARCHAR(20) NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'merged', 'converted', 'abandoned')),
    currency_code CHAR(3) NOT NULL,
    email VARCHAR(320),

    -- Line items as JSONB array (cart is a short-lived document)
    items JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {
    --     "id": "item_01", "variant_id": "...", "product_id": "...",
    --     "sku": "WHP-BLK-001", "product_name": "Wireless Headphones Pro",
    --     "variant_name": "Black", "quantity": 1,
    --     "unit_price_cents": 29999, "total_cents": 29999,
    --     "attributes": {"color": "black"}
    --   }
    -- ]

    -- Addresses as JSONB (snapshot, may differ from customer saved addresses)
    shipping_address JSONB,
    billing_address JSONB,

    -- Pricing summary
    subtotal_cents BIGINT NOT NULL DEFAULT 0,
    discount_cents BIGINT NOT NULL DEFAULT 0,
    shipping_cents BIGINT NOT NULL DEFAULT 0,
    tax_cents BIGINT NOT NULL DEFAULT 0,
    total_cents BIGINT NOT NULL DEFAULT 0,

    -- Applied promotions
    applied_promotions JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"promotion_id": "...", "code": "SAVE20", "discount_cents": 5000}]

    notes TEXT,
    expires_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_carts_customer ON carts(customer_id);
CREATE INDEX idx_carts_status ON carts(status);
CREATE INDEX idx_carts_expires ON carts(expires_at) WHERE status = 'active';
```

## Orders and Fulfillment

```sql
-- Orders (relational core for queryable fields, JSONB for snapshots and details)
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
    payment_status VARCHAR(30) NOT NULL DEFAULT 'pending',
    fulfillment_status VARCHAR(30) NOT NULL DEFAULT 'unfulfilled',
    currency_code CHAR(3) NOT NULL,

    -- Pricing (relational for reporting/aggregation)
    subtotal_cents BIGINT NOT NULL,
    discount_cents BIGINT NOT NULL DEFAULT 0,
    shipping_cents BIGINT NOT NULL DEFAULT 0,
    tax_cents BIGINT NOT NULL DEFAULT 0,
    total_cents BIGINT NOT NULL,

    -- Line items as JSONB (snapshot at time of order)
    items JSONB NOT NULL,
    -- Example:
    -- [
    --   {
    --     "id": "oi_01", "variant_id": "...", "sku": "WHP-BLK-001",
    --     "product_name": "Wireless Headphones Pro", "variant_name": "Black",
    --     "quantity": 1, "unit_price_cents": 29999,
    --     "discount_cents": 0, "tax_cents": 2400, "total_cents": 32399,
    --     "fulfillment_status": "unfulfilled",
    --     "attributes": {"color": "black"}
    --   }
    -- ]

    -- Address snapshots (JSONB — independent of customer profile changes)
    shipping_address JSONB NOT NULL,
    billing_address JSONB NOT NULL,

    -- Applied promotions snapshot
    applied_promotions JSONB NOT NULL DEFAULT '[]',

    -- Tax breakdown
    tax_breakdown JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {"name": "CA State Tax", "rate_percent": 7.25, "amount_cents": 2174},
    --   {"name": "SF County Tax", "rate_percent": 1.25, "amount_cents": 375}
    -- ]

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
CREATE INDEX idx_orders_payment ON orders(payment_status);

-- Payments
CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    gateway VARCHAR(50) NOT NULL,
    status VARCHAR(30) NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending', 'authorized', 'captured', 'partially_refunded',
        'refunded', 'voided', 'failed'
    )),
    method VARCHAR(50) NOT NULL,
    amount_cents BIGINT NOT NULL,
    currency_code CHAR(3) NOT NULL,
    -- Gateway-specific data (transaction IDs, tokens, metadata)
    gateway_data JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "transaction_id": "pi_3abc123",
    --   "payment_method_id": "pm_xyz789",
    --   "card_last4": "4242",
    --   "card_brand": "visa",
    --   "receipt_url": "https://pay.stripe.com/receipts/..."
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payments_order ON payments(order_id);

-- Fulfillments
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
    -- Which items are in this fulfillment (references item IDs from order.items JSONB)
    items JSONB NOT NULL,
    -- Example: [{"order_item_id": "oi_01", "quantity": 1}]
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_fulfillments_order ON fulfillments(order_id);
```

## Tax and Shipping Configuration

```sql
-- Tax configuration (JSONB rules for maximum flexibility)
CREATE TABLE tax_configurations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    -- Tax rules as JSONB (jurisdiction matching + rates)
    rules JSONB NOT NULL,
    -- Example:
    -- {
    --   "zones": [
    --     {
    --       "name": "California",
    --       "match": {"country_code": "US", "subdivision_code": "US-CA"},
    --       "rates": [
    --         {"name": "State Tax", "rate_percent": 7.25, "is_compound": false},
    --         {"name": "County Tax (SF)", "rate_percent": 1.25, "match": {"postal_code_prefix": "941"}}
    --       ]
    --     },
    --     {
    --       "name": "EU VAT",
    --       "match": {"country_codes": ["DE", "FR", "NL", "IT"]},
    --       "rates": [
    --         {"name": "VAT Standard", "rate_percent": 19, "country_code": "DE"},
    --         {"name": "VAT Standard", "rate_percent": 20, "country_code": "FR"}
    --       ],
    --       "is_included_in_price": true
    --     }
    --   ]
    -- }
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Shipping methods with JSONB rate tables
CREATE TABLE shipping_methods (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    carrier VARCHAR(100),
    description TEXT,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    -- Rate rules as JSONB (by zone, weight, order total)
    rate_rules JSONB NOT NULL,
    -- Example:
    -- {
    --   "rates": [
    --     {"zone": "US", "min_order_cents": 0, "rate_cents": 999, "currency": "USD"},
    --     {"zone": "US", "min_order_cents": 7500, "rate_cents": 0, "currency": "USD"},
    --     {"zone": "EU", "min_order_cents": 0, "rate_cents": 1499, "currency": "EUR"}
    --   ]
    -- }
    channel_ids UUID[] NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Events, Webhooks, and AI

```sql
-- Domain events (CloudEvents envelope with JSONB payload)
CREATE TABLE domain_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    type VARCHAR(255) NOT NULL,
    source VARCHAR(500) NOT NULL,
    subject VARCHAR(500),
    time TIMESTAMPTZ NOT NULL DEFAULT now(),
    data JSONB NOT NULL
);

CREATE INDEX idx_events_type ON domain_events(type);
CREATE INDEX idx_events_time ON domain_events(time DESC);
CREATE INDEX idx_events_subject ON domain_events(subject);

-- Webhook subscriptions
CREATE TABLE webhook_subscriptions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    api_client_id UUID NOT NULL REFERENCES api_clients(id) ON DELETE CASCADE,
    target_url TEXT NOT NULL,
    event_types TEXT[] NOT NULL,
    secret_hash VARCHAR(500) NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    -- Delivery config as JSONB
    config JSONB NOT NULL DEFAULT '{}',
    -- Example: {"retry_count": 3, "timeout_ms": 5000, "headers": {"X-Custom": "value"}}
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Product embeddings for AI-native search
CREATE TABLE product_embeddings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    variant_id UUID REFERENCES product_variants(id) ON DELETE CASCADE,
    model_version VARCHAR(100) NOT NULL,
    embedding vector(1536) NOT NULL,
    -- Additional AI metadata
    ai_metadata JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "generated_tags": ["wireless", "noise-cancelling", "premium"],
    --   "enriched_description": "...",
    --   "category_confidence": 0.95,
    --   "similar_product_ids": ["...", "..."]
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (product_id, variant_id, model_version)
);

CREATE INDEX idx_embeddings_product ON product_embeddings(product_id);
CREATE INDEX idx_embeddings_vector ON product_embeddings USING ivfflat (embedding vector_cosine_ops);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Catalog Management | 4 | Products, variants, product types, categories |
| Channels & Promotions | 2 | Channels, promotions |
| Inventory | 3 | Stock locations, inventory levels, inventory movements |
| Customers & Auth | 4 | Customers, customer groups, companies, API clients |
| Cart | 1 | Carts (items as JSONB array) |
| Orders & Fulfillment | 3 | Orders (items as JSONB), payments, fulfillments |
| Tax & Shipping | 2 | Tax configurations, shipping methods |
| Events & AI | 3 | Domain events, webhooks, product embeddings |
| **Total** | **22** | Significantly fewer tables than normalised (~38) |

---

## Key Design Decisions

1. **JSONB for product attributes instead of EAV tables** — product attributes (`{"color": "black", "size": "M"}`) are stored as JSONB on the product/variant, validated at the application layer against the `product_types.attribute_schema` (JSON Schema). This eliminates the need for `attribute_definitions`, `attribute_values`, and `product_attribute_values` junction tables (3 tables saved), while supporting GIN-indexed containment queries.

2. **Prices as JSONB array on variants** — all prices for a variant (across markets, currencies, quantity breaks) are stored in a single JSONB array on `product_variants.prices`. This trades normalised price list tables for a simpler model where loading a variant returns all its prices in one row without JOINs. Price list management happens at the application layer.

3. **Cart items and order items as JSONB arrays** — carts and orders store their line items as JSONB arrays rather than in separate `cart_items`/`order_items` tables. This matches the API response shape (a cart is always returned as a complete document) and reduces JOIN count. Cart items are ephemeral; order items are immutable snapshots.

4. **Addresses as JSONB on customers** — customer addresses are stored as a JSONB array on the customer record rather than in a separate `customer_addresses` table. Most customers have 1-3 addresses, making this efficient. Order addresses are snapshots (JSONB) independent of the customer profile.

5. **Tax rules and shipping rates as JSONB configuration** — instead of normalised `tax_zones`, `tax_zone_members`, `tax_rates`, `shipping_rates` tables (4+ tables), tax and shipping rules are stored as structured JSONB documents. This is simpler to manage for multi-region deployments where rules vary significantly by jurisdiction.

6. **Promotion rules as JSONB** — promotion conditions, limits, and targeting rules are stored as JSONB rather than in normalised condition tables. This allows arbitrarily complex promotion rules without schema changes.

7. **`attribute_schema` as JSON Schema on product types** — the product type stores a JSON Schema document that defines what attributes are valid for products of that type. Application-layer validation enforces the schema before writes. This is the same pattern commercetools uses with Custom Types.

8. **Channel configuration as JSONB** — per-channel settings (supported currencies, checkout behaviour, locale) are stored as JSONB on the channel record rather than in separate configuration tables.

9. **GIN indexes on all queryable JSONB columns** — every JSONB column that participates in filtering or search has a GIN index, enabling efficient containment (`@>`) queries.

10. **22 tables vs. 38+ in the normalised model** — the JSONB hybrid approach reduces table count by ~42% while preserving relational integrity for core entities (products, orders, customers, inventory levels) where referential integrity and aggregation queries matter most.
