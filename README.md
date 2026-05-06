# Headless Commerce Engine

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An API-first commerce platform with AI personalisation built into the core, not bolted on as third-party plugins.

The Headless Commerce Engine is an open-source, MACH-aligned commerce backend for engineering teams building custom storefronts, B2B portals, conversational commerce, and multi-channel retail experiences. It targets the gap between expensive enterprise platforms (commercetools, Elastic Path) and the smaller open-source ecosystem (Medusa, Saleor, Vendure) by making AI-native capabilities a first-class part of the API surface.

---

## Why Headless Commerce Engine?

- Enterprise MACH platforms like commercetools and Elastic Path require dedicated engineering teams and contracts that frequently exceed $100k/year, putting composable commerce out of reach for mid-market brands.
- SaaS platforms such as Shopify lock the backend into a single ecosystem with multi-region limitations and constrained tax and pricing logic, even when used in headless mode via the Storefront API.
- Pure-API offerings like Commerce Layer ship without an admin UI or bundled storefront framework, forcing teams to build or buy multiple additional tools before going live.
- Existing open-source options (Medusa, Saleor, Vendure) treat AI as an integration point rather than a built-in capability, so recommendations, search ranking, and personalisation typically require third-party services.
- Composable commerce is forecast to grow from USD 6.44B in 2024 to USD 31.50B by 2034, but no leading open-source project currently combines MACH architecture with native AI features.

---

## Key Features

### Catalog and Inventory APIs

- Product, variant, and SKU management via GraphQL and REST APIs
- Real-time inventory tracking and reservation
- Multi-channel catalog support (web, mobile, marketplace)
- Multi-region and multi-market catalog structures
- Schema introspection for type-safe client code generation

### Cart, Checkout, and Orders

- Shopping cart API with item, quantity, and modifier management
- Checkout flow API with payment gateway integration (Stripe, PayPal)
- Order creation, retrieval, and lifecycle management
- Fulfillment and shipment management
- Webhook events for orders, payments, and inventory changes

### Pricing, Promotions, and B2B

- Multiple price lists per market and customer segment
- Promotion and discount rules engine
- B2B features including pricebooks, quotes, and company accounts
- Multi-vendor and wholesale capabilities
- PCI DSS compliant checkout via tokenisation patterns

### Customer Accounts and Authentication

- Customer account and order history APIs
- OAuth 2.0 and OpenID Connect for API authentication
- Staff portal and admin dashboard for operational tasks
- Plugin and extensibility architecture for custom logic

### AI-Native Capabilities

- Vector search and LLM-ranked product recommendations
- Conversational commerce API for natural-language catalog queries and checkout
- Dynamic bundle and pricing generation based on context, inventory, and demand
- Automated catalog enrichment from supplier data sheets
- ML-based fraud detection and risk-based checkout friction

---

## AI-Native Advantage

Incumbents treat AI features as add-on integrations: search plugins, recommendation services, and external fraud tooling glued together over webhooks. The Headless Commerce Engine embeds vector search, LLM-ranked results, conversational queries, automated attribute extraction, and anomaly detection directly into the commerce APIs. This means a single GraphQL or REST call can return personalised, ranked results or initiate a natural-language checkout, without integrating separate vendors per channel.

---

## Tech Stack & Deployment

The project aligns with the MACH Alliance reference architecture: microservices, API-first, cloud-native, headless. Expected deployment modes include self-hosted (full control over data and infrastructure) and managed cloud, mirroring the patterns used by Medusa, Saleor, and Vendure. APIs are published with OpenAPI 3.0 specifications and a GraphQL schema, secured via OAuth 2.0 / OpenID Connect, and wired between services using CloudEvents (CNCF) for vendor-neutral event integration.

---

## Market Context

The global headless commerce market was valued at approximately USD 1.74 billion in 2025 and is projected to reach USD 7.16 billion by 2032 at a 22.4% CAGR (Swell, 2025). Gartner forecasts that 60% of organisations will rely on composable commerce architectures by 2027. Incumbent pricing ranges from $80/month (Shopify) to $500k+/year for commercetools enterprise contracts, with Commerce Layer and Elastic Path typically starting at $30k–$100k/year. Primary buyers are CTOs, VPs of Engineering, Digital Commerce Directors, and Platform Architects at mid-to-large retailers with multi-region, multi-brand, or B2B requirements.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
