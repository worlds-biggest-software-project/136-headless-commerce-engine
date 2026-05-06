# Headless Commerce Engine — Feature & Functionality Survey

> Candidate #136 · Researched: 2026-05-03

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| commercetools | Commercial SaaS | Proprietary | https://commercetools.com |
| Shopify (Hydrogen) | Commercial SaaS | Proprietary | https://shopify.com |
| Commerce Layer | Commercial SaaS | Proprietary | https://commercelayer.io |
| Medusa.js | Open-source / Commercial Cloud | MIT | https://medusajs.com |
| BigCommerce | Commercial SaaS | Proprietary | https://bigcommerce.com |
| Saleor | Open-source / Commercial Cloud | Apache 2.0 | https://saleor.io |
| Elastic Path | Commercial SaaS | Proprietary | https://elasticpath.com |
| Vendure | Open-source | MIT | https://vendure.io |

## Feature Analysis by Solution

### commercetools

**Core features**
- Fully API-first commerce platform (GraphQL and REST APIs)
- Multi-region catalog and inventory management
- Price list management with customer-specific pricing
- Cart and checkout API with custom logic support
- Order and fulfillment management
- Real-time inventory sync and reservations
- Advanced search with custom faceting
- Tax calculation API
- Payment integration with PCI DSS compliance

**Differentiating features**
- Unlimited API-driven customisation; no UI constraints
- Global scale with CDN-backed API endpoints
- Fine-grained permission model for multi-vendor scenarios

**UX patterns**
- Pure API model; no admin UI included (requires separate build or integration)
- Event-driven architecture with webhook support for order and inventory events
- GraphQL-first design allowing clients to request exactly what data they need

**Integration points**
- REST and GraphQL APIs
- Webhook system for event-driven integrations
- OAuth 2.0 for client authentication
- Connectors to payment gateways, shipping providers, ERPs
- Custom payment and tax provider plugins

**Known gaps**
- High implementation cost requiring dedicated engineering team
- No bundled admin UI or operational dashboards
- Requires external CMS, PIM, or admin tooling
- Learning curve steep for non-API-first teams

**Licence / IP notes**
- Proprietary SaaS; no open-source components

### Shopify (Hydrogen / Storefront API)

**Core features**
- Shopify as headless backend via Storefront API (GraphQL)
- Hydrogen React framework (Remix-based) for storefront building
- Oxygen hosting platform bundled with Hydrogen
- Product catalog, variant, and inventory management
- Shopping cart and checkout APIs
- Order and fulfillment management
- Payment processing (Shopify Payments or third-party gateways)
- Customer account and order history APIs
- Collections, discounts, and automation rules

**Differentiating features**
- Huge ecosystem of Shopify apps and integrations
- Hydrogen simplifies React storefront development with Remix conventions
- Oxygen provides edge-rendered hosting reducing latency
- Rapid time-to-market for standard retail scenarios

**UX patterns**
- GraphQL Storefront API for client access
- Hydrogen framework templates for rapid frontend scaffolding
- Admin UI for operational tasks (orders, fulfillment, customer service)

**Integration points**
- GraphQL Storefront API for custom frontends
- REST Admin API for integrations
- App marketplace for third-party extensions
- Stripe, PayPal, and alternative payment integrations
- Shipping provider APIs (Shopify rates)

**Known gaps**
- Backend locked into Shopify ecosystem
- Multi-region limitations compared to truly global solutions
- Limited customisation of tax and pricing logic
- Hydrogen framework ties frontend to React/Remix

**Licence / IP notes**
- Proprietary SaaS; Hydrogen framework is open-source (MIT-like license pending verification)

### Commerce Layer

**Core features**
- API-only commerce engine (no admin UI)
- Fine-grained price list management (per market, customer segment, promotion)
- Multi-market inventory management
- Shopping carts and checkout APIs
- Order management
- SKU and variant catalogs
- Promotions and discount rules
- Payment and shipping integrations
- Webhook event system

**Differentiating features**
- Designed specifically for SKU-heavy, multi-market scenarios
- Price list granularity enables per-market, per-customer pricing
- Pure API model forces clear separation of concerns (no UI coupling)

**UX patterns**
- Pure REST API with predictable naming conventions
- Event webhooks for async order processing
- Token-based authentication (OAuth 2.0)

**Integration points**
- REST API (OpenAPI documentation)
- Webhook events (orders, payments, promotions)
- Payment gateway connectors
- Shipping and fulfillment integrations
- Custom PIM and CMS via API integration

**Known gaps**
- No admin UI; requires separate solution for operational dashboards
- No bundled storefront framework; full custom build required
- Usage-based pricing can become expensive at scale
- Smaller ecosystem compared to Shopify or commercetools

**Licence / IP notes**
- Proprietary SaaS

### Medusa.js

**Core features**
- Open-source Node.js headless commerce platform
- REST API for products, variants, carts, orders, and customers
- Admin dashboard (open-source) for order and inventory management
- Multi-channel support (web, mobile, social)
- Plugin architecture for extensibility
- Fulfillment and shipment management
- Payment gateway integrations (Stripe, PayPal, etc.)
- Search and faceting with Meilisearch/Elasticsearch
- Customer accounts and authentication
- Analytics hooks for event tracking

**Differentiating features**
- Developer-friendly TypeScript codebase
- Modular plugin system allowing selective feature adoption
- Active open-source community with regular updates
- Self-hosted option eliminates vendor lock-in

**UX patterns**
- REST API-first design with comprehensive documentation
- Modular backend allowing teams to use only needed services
- Admin dashboard for operational tasks
- Event emitter system for extending core functionality

**Integration points**
- REST API with OpenAPI documentation
- Plugin marketplace for integrations
- Payment processor integrations (Stripe, Square, PayPal)
- Shipping providers (EasyPost, ShipStation)
- Search integrations (Meilisearch, Algolia)
- Custom modules via plugin system

**Known gaps**
- Smaller enterprise support ecosystem compared to commercial solutions
- Self-hosting requires operational overhead (database, infrastructure)
- Limited native multi-region support
- Medusa Cloud offering still maturing

**Licence / IP notes**
- MIT License for open-source components; Medusa Cloud uses proprietary backend

### BigCommerce

**Core features**
- Native headless support via GraphQL and REST APIs
- Stencil theme framework for custom storefronts (CLI-based)
- Product catalog, variants, and inventory management
- Advanced search with faceting
- Shopping cart and checkout APIs
- Order and fulfillment management
- No transaction fees (unlike Shopify)
- B2B-specific features (pricebooks, quotes, company accounts)
- Multi-vendor and wholesale capabilities
- Built-in SEO, analytics, and reporting

**Differentiating features**
- No transaction fees unlike Shopify
- Strong B2B and wholesale features out-of-box
- Stencil theme framework bridges custom and platform-provided functionality

**UX patterns**
- GraphQL and REST API with admin control panel
- Stencil CLI for theme/storefront development
- Clear separation between theme code and API
- Admin UI for order management and marketing operations

**Integration points**
- GraphQL and REST APIs
- Stencil theme framework
- Payment gateway integrations (Stripe, PayPal, Authorize.net, etc.)
- Shipping provider integrations
- Apps and extensions from BigCommerce marketplace
- Webhook events for custom integrations

**Known gaps**
- Less composable than pure-MACH solutions (some features bundled)
- Stencil learning curve steeper than React/Hydrogen
- Multi-region support limited compared to global platforms

**Licence / IP notes**
- Proprietary SaaS

### Saleor

**Core features**
- Open-source GraphQL-first headless commerce platform
- Django/Python backend with plugin architecture
- GraphQL API (primary interface) and REST APIs
- Product catalog, variants, and attributes
- Multi-channel support (web, mobile, marketplace)
- Shopping cart and checkout workflows
- Order and fulfillment management
- Customer accounts and authentication
- Staff portal for admin operations
- Webhook events and event webhooks

**Differentiating features**
- Full GraphQL API designed from ground-up (not REST retrofitted)
- Python/Django backend appeals to teams with Python expertise
- Open-source with active community contribution
- Self-hosted with no vendor lock-in

**UX patterns**
- GraphQL-first design; REST wrappers available via layer
- Dashboard (Saleor Dashboard) for operational tasks
- Plugin system for extending core functionality
- Event-driven architecture with webhooks

**Integration points**
- GraphQL API with introspection and schema documentation
- REST API wrappers (via Saleor Cloud)
- Webhook events for integrations
- Payment integrations (Stripe, Braintree, PayPal)
- Shipping provider integrations
- Search integrations (Elasticsearch, Meilisearch)
- Custom plugins via Python/Django

**Known gaps**
- Python stack limits some JavaScript/Node integrations
- Smaller ecosystem compared to Medusa or commercial solutions
- Self-hosting requires Python expertise
- Saleor Cloud offering newer and less mature than alternatives

**Licence / IP notes**
- OSL 3.0 (Open Software License); strong copyleft model

### Elastic Path

**Core features**
- Enterprise composable commerce with microservices architecture
- Separate services for catalog, cart, checkout, promotions, inventory
- Flexible pricing and promotions engine
- Multi-tenant and multi-brand support
- API-first design with REST and GraphQL options
- Order and fulfillment management
- Advanced search and discovery
- Webhook events for async processing

**Differentiating features**
- Highly decoupled service architecture (buy only what you need)
- Strong in complex multi-vendor B2B scenarios
- Fine-grained pricing and promotion engine

**UX patterns**
- Pure API model with microservice separation
- Event-driven architecture for scale
- Admin tools available but primarily API-centric

**Integration points**
- REST and GraphQL APIs
- Webhook events
- Custom microservice integration points
- Payment gateway connectors
- Shipping and inventory systems

**Known gaps**
- Complex multi-vendor integration overhead
- Higher implementation costs
- Smaller ecosystem than Shopify or commercetools
- Learning curve steep for organisations new to microservices

**Licence / IP notes**
- Proprietary SaaS

### Vendure

**Core features**
- Open-source TypeScript/Node headless commerce framework
- GraphQL API (primary interface)
- Admin UI built on Angular
- Product catalog and variant management
- Multi-channel support
- Shopping cart and checkout
- Order and fulfillment management
- Plugin architecture for customisation
- Customer accounts and authentication
- Payment and shipping provider integrations

**Differentiating features**
- Type-safe TypeScript codebase appeals to modern development teams
- Modular plugin system with type safety
- Active maintenance and community support
- Self-hosted with vendor lock-in avoidance

**UX patterns**
- GraphQL API with type-safe client code generation
- Plugin-based extensibility
- Admin dashboard for operational tasks

**Integration points**
- GraphQL API with comprehensive schema
- Plugin system for extending functionality
- Payment integrations (Stripe, Braintree)
- Shipping provider integrations
- Custom plugins via TypeScript

**Known gaps**
- Smaller community than Medusa
- Fewer third-party integrations available
- Less mature SaaS offering compared to alternatives
- Smaller ecosystem

**Licence / IP notes**
- MIT License for open-source core

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Headless API (REST and/or GraphQL) for product catalog, cart, and checkout
- Multi-channel support (web, mobile, marketplace)
- Product variants and SKU management
- Inventory tracking and real-time sync
- Shopping cart and checkout workflows
- Order and fulfillment management
- Customer account and authentication
- Payment gateway integrations
- Shipping provider integrations
- Admin operational dashboards (order, customer, inventory management)

### Differentiating Features
- GraphQL API with efficient client data fetching
- Multi-region and global scale support
- Advanced pricing and promotion engine
- B2B and wholesale capabilities
- PCI DSS compliant checkout
- Multi-vendor and marketplace support
- Search and discovery with advanced faceting
- Webhook event system for async integrations
- Event-driven architecture supporting high-volume transactions

### Underserved Areas / Opportunities
- AI-driven product recommendations and search ranking
- Conversational commerce API (natural-language queries)
- Dynamic bundle and pricing generation in real-time
- Automated catalog enrichment (AI extraction from supplier data)
- Fraud and anomaly detection in order processing
- Predictive inventory and demand forecasting
- Automated product attribute mapping across channels
- Multi-currency and tax calculation at edge
- Real-time personalisation API layer

### AI-Augmentation Candidates
- Vector search and LLM-ranked product recommendations
- AI bundle generation and dynamic pricing
- Conversational checkout and natural-language queries
- Automated catalog enrichment from unstructured data
- ML-based fraud detection and dynamic risk-based checkout friction
- Demand forecasting and inventory optimisation
- Automated channel-specific product attribute generation

## Legal & IP Summary

Open-source options (Medusa, Saleor, Vendure) use standard permissive (MIT) or copyleft (OSL 3.0 for Saleor) licenses without IP encumbrances. Proprietary SaaS platforms (commercetools, Shopify, BigCommerce, Commerce Layer, Elastic Path) are closed-source. No known software patents directly encumber headless commerce architecture; MACH architecture is an open industry reference model promoted by a vendor-neutral alliance. All major solutions support PCI DSS compliance through standard tokenisation patterns.

## Recommended Feature Scope

**Must-have (MVP)**
- GraphQL or REST API for products, variants, and catalog
- Shopping cart API with basic item management
- Checkout flow API (payment processing via gateway integration)
- Customer account and authentication API
- Order creation and retrieval APIs
- Basic inventory tracking and real-time sync
- Payment gateway integrations (Stripe, PayPal minimum)
- Webhook events for order and fulfillment notifications
- Admin dashboard for order and inventory management

**Should-have (v1.1)**
- Advanced search with basic faceting
- Multi-channel support (web, mobile at minimum)
- Multiple price lists per market or customer segment
- Promotions and discount rules engine
- Shipping provider integrations (EasyPost, ShipStation)
- B2B features (pricebooks, company accounts)
- Analytics and reporting dashboards
- Plugin or extensibility architecture
- GraphQL-first API design with schema introspection

**Nice-to-have (backlog)**
- AI-driven product recommendations
- Conversational commerce API (natural-language queries)
- Dynamic pricing and bundle generation
- Automated catalog enrichment
- Multi-region and global scale support
- Fraud detection and risk-based checkout
- Predictive inventory and demand forecasting
- Multi-vendor and marketplace features
- Edge-based rendering for global storefront latency
