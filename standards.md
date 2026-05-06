# Standards & API Reference

> Project: Headless Commerce Engine · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO 4217 — Currency Codes**
- URL: https://www.iso.org/iso-4217-currency-codes.html
- Three-letter alphabetic codes (e.g. USD, EUR, GBP) and three-digit numeric equivalents for all world currencies. Every headless commerce API that exposes price fields, price lists, or multi-currency support must accept and return ISO 4217 currency codes for interoperability with payment gateways, ERP systems, and tax engines.

**ISO 3166 — Country and Subdivision Codes**
- URL: https://www.iso.org/iso-3166-country-codes.html
- Two-letter (alpha-2) and three-letter (alpha-3) codes for countries and their subdivisions. Required for shipping address validation, multi-market configuration, tax jurisdiction mapping, and compliance with regional data privacy regulations. Used by all major commerce platforms (Shopify, commercetools, Commerce Layer) as the canonical country representation in address objects.

**ISO 8601 — Date and Time Representation**
- URL: https://www.iso.org/iso-8601-date-and-time-format.html
- Standard format for representing dates and times (e.g. `2026-05-03T09:00:00Z`). All commerce API endpoints that surface order timestamps, promotion validity windows, inventory reservations, and webhook event times should use ISO 8601 UTC format to avoid timezone ambiguity across multi-region deployments.

### W3C & IETF Standards

**W3C Payment Request API**
- URL: https://www.w3.org/TR/payment-request/
- Browser-native API that standardises the payment form interaction between merchants and users, replacing custom checkout forms with a browser-managed dialog. Headless storefronts that target browser environments should implement the Payment Request API to reduce checkout friction and improve conversion. As of 2024, W3C invited implementations; the spec is supported in Chrome, Safari, Edge, and Opera.

**W3C Web-based Payment Handler API**
- URL: https://www.w3.org/TR/payment-handler/
- Complements the Payment Request API by defining how payment applications (e.g. Apple Pay, Google Pay, browser-based wallets) register themselves as payment handlers. Relevant to any headless commerce engine that integrates wallet-based checkout flows.

**W3C JSON-LD & Schema.org Product Vocabulary**
- URL: https://schema.org/Product
- JSON-LD (JSON Linked Data) is a W3C Recommendation for embedding structured data in web pages. The schema.org `Product` type provides a vendor-neutral vocabulary for describing products, offers, prices, availability, and reviews. Headless storefronts should emit JSON-LD product markup to enable rich search results (Google Shopping, price comparisons) and improve discoverability. Over 45 million web domains use schema.org markup.

**RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The foundational IETF standard for delegated API authorisation. All headless commerce backends use OAuth 2.0 client credentials flow (machine-to-machine) for backend service authentication and authorisation code flow for customer-facing account sessions. commercetools, Saleor, Medusa, and Commerce Layer all mandate OAuth 2.0 token exchange.

**RFC 9449 — OAuth 2.0 DPoP (Demonstrating Proof of Possession)**
- URL: https://datatracker.ietf.org/doc/html/rfc9449
- Binds an OAuth access token to the client's private key, preventing token theft and replay attacks. Considered best practice for mobile and public clients accessing headless commerce APIs in 2025–2026.

**RFC 9110 — HTTP Semantics**
- URL: https://datatracker.ietf.org/doc/html/rfc9110
- The current IETF standard for HTTP semantics, consolidating and superseding earlier RFCs (7230–7235). Defines correct use of HTTP methods, status codes, content negotiation, and caching headers. All REST-based headless commerce APIs should conform to RFC 9110 for correct status code usage (e.g. 201 for order creation, 409 for stock conflicts).

**RFC 8288 — Web Linking**
- URL: https://datatracker.ietf.org/doc/html/rfc8288
- Defines the `Link` HTTP header for expressing relationships between resources (e.g. pagination `rel=next`, `rel=prev`). Used by Commerce Layer's JSON:API implementation and other REST commerce backends to express paginated collection responses.

### Data Model & API Specifications

**OpenAPI Specification 3.1 / 3.2**
- URL: https://spec.openapis.org/oas/v3.1.2.html
- The industry-standard format for describing RESTful APIs in YAML or JSON. OpenAPI 3.1 aligns fully with JSON Schema Draft 2020-12, enabling stricter schema validation. OpenAPI 3.2 adds native streaming media type support (SSE, JSON Lines) and OAuth 2.0 Device Authorization Flow. Commerce Layer, Medusa, and BigCommerce publish OpenAPI schemas that can be used for mock server generation, SDK auto-generation, and contract testing. All new headless commerce REST APIs should publish an OpenAPI 3.1+ document.

**GraphQL Specification (June 2018, draft 2021 onwards)**
- URL: https://spec.graphql.org/
- The specification maintained by the GraphQL Foundation defining the query language, type system, introspection, and execution semantics. A 2024 industry survey found 61% of teams use GraphQL in production, with 10% migrating away from REST entirely. commercetools, Saleor, Shopify Storefront API, Vendure, and BigCommerce all provide GraphQL APIs as their primary or co-primary interface.

**JSON:API Specification v1.0**
- URL: https://jsonapi.org/
- A specification for structuring REST API request and response bodies (compound documents, sparse fieldsets, filtering, sorting, pagination). Commerce Layer is 100% JSON:API-compliant. Provides standardised pagination (`page[size]`, `page[number]`) and relationship patterns useful for multi-resource commerce responses (product + variants + prices).

**AsyncAPI Specification v3.1**
- URL: https://www.asyncapi.com/docs/reference/specification/v3.1.0
- The standard specification for documenting event-driven and message-driven APIs (the async equivalent of OpenAPI). AsyncAPI downloads surpassed 34 million in the past year. Headless commerce platforms that expose webhook events (order.created, inventory.updated, payment.captured) should publish an AsyncAPI document alongside their OpenAPI spec. Supports AMQP, Kafka, WebSockets, and HTTP protocols.

**CloudEvents v1.0 (CNCF)**
- URL: https://cloudevents.io/
- A CNCF-graduated vendor-neutral specification (January 2024) for describing event data in a common format. Provides consistent event envelope metadata (`type`, `source`, `id`, `time`, `datacontenttype`) enabling interoperability across microservice event buses (Kafka, RabbitMQ, Amazon EventBridge). Recommended for headless commerce engines that expose or consume order, inventory, and fulfillment events across composable stacks.

**MACH Alliance Open Data Model**
- URL: https://machalliance.org/
- An open-source framework maintained by the MACH Alliance providing proven integration recipes and data model patterns for composable commerce architectures. The Alliance's MCP Registry (launched 2025) maintains a directory of production-ready Model Context Protocol server implementations enabling AI agents to interact with commerce services. Relevant as the de-facto reference architecture for enterprise headless commerce design.

### Security & Authentication Standards

**PCI DSS v4.0.1 — Payment Card Industry Data Security Standard**
- URL: https://www.pcisecuritystandards.org/
- Mandatory compliance standard for any application that processes, stores, or transmits payment card data. As of 31 March 2025, all v4.0 future-dated requirements are mandatory with no grace period. Requirements 6.4.3 and 11.6.1 are particularly relevant to headless architectures: they require that all scripts on payment pages be authorised, integrity-checked, and monitored for tampering (anti-e-skimming). Headless storefronts that embed payment iframes or load third-party scripts on checkout pages must comply. SAQ A eligibility is now very strict — any JavaScript on the page that can influence the payment form disqualifies SAQ A.

**OpenID Connect Core 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- An identity layer built on OAuth 2.0 that issues ID Tokens (JWTs) containing authenticated user identity claims. Used by headless commerce platforms for customer account authentication, SSO integration, and B2B company account management.

**FAPI 2.0 Security Profile (Financial-grade API)**
- URL: https://openid.net/specs/fapi-security-profile-2_0-final.html
- Became final in February 2025. A high-security profile for OAuth 2.0 mandating DPoP and Pushed Authorization Requests (PAR). Relevant for headless commerce platforms targeting financial services, high-value goods, or markets with strict regulatory requirements.

**OWASP API Security Top 10 (2023)**
- URL: https://owasp.org/API-Security/
- The authoritative reference for API security risks. The most relevant to headless commerce APIs are: Broken Object Level Authorization (BOLA/IDOR) in order and customer APIs; Unrestricted Resource Consumption (rate limiting on cart and checkout endpoints); and Security Misconfiguration in webhook endpoint validation. All headless commerce API implementations should be reviewed against the OWASP API Security Top 10.

**GDPR — General Data Protection Regulation (EU) 2016/679**
- URL: https://gdpr-info.eu/
- EU regulation governing personal data collection, processing, and storage. Headless commerce APIs that collect customer personal data (email, address, order history) must implement: data minimisation, explicit consent for marketing, right-to-erasure (DELETE /customers/{id}), and data portability endpoints. Penalties up to €20M or 4% of annual global turnover.

**CCPA — California Consumer Privacy Act**
- URL: https://oag.ca.gov/privacy/ccpa
- California state law granting consumers rights over their personal data including the right to know, delete, and opt out of data sale. Headless commerce APIs targeting US customers must expose consumer rights endpoints and honour opt-out signals. Less prescriptive about consent than GDPR but requires transparency in data handling disclosures.

### MCP Server Specifications

**Model Context Protocol (MCP)**
- URL: https://modelcontextprotocol.io/
- Anthropic-originated open protocol enabling AI models and agents to interact with external services via a standardised tool/resource interface. The MACH Alliance MCP Registry (https://machalliance.org/mach-alliance-mcp-registry) hosts a vendor-neutral directory of production-ready MCP server implementations for commerce services including catalog management, order processing, payment, search, and inventory. Headless commerce engines that wish to be AI-agent-accessible should implement or publish an MCP server exposing their core commerce capabilities.

---

## Similar Products — Developer Documentation & APIs

### commercetools

- **Description:** Enterprise MACH-native composable commerce platform; offers both REST and GraphQL APIs for all commerce operations.
- **API Documentation:** https://docs.commercetools.com/api
- **GraphQL API:** https://docs.commercetools.com/api/graphql
- **SDKs/Libraries:** Official SDKs in JavaScript/TypeScript, Java, PHP, Python, .NET — https://docs.commercetools.com/sdk
- **Developer Guide:** https://docs.commercetools.com/learning-composable-commerce-developer-essentials
- **API Reference on GitHub:** https://github.com/commercetools/commercetools-api-reference
- **Standards:** REST + GraphQL; OAuth 2.0 client credentials; OpenAPI-compatible; event webhooks
- **Authentication:** OAuth 2.0 client credentials flow; project-scoped access tokens

### Shopify (Storefront API / Hydrogen)

- **Description:** GraphQL Storefront API for building custom headless storefronts backed by Shopify's commerce backend; Hydrogen is the React/Remix-based storefront framework.
- **API Documentation:** https://shopify.dev/docs/api/storefront/latest
- **Hydrogen Framework:** https://shopify.dev/docs/storefronts/headless
- **SDKs/Libraries:** `@shopify/storefront-api-client` (JavaScript); Hydrogen (React); Buy SDK
- **Developer Guide:** https://shopify.dev/docs/storefronts/headless/building-with-the-storefront-api/getting-started
- **Standards:** GraphQL; REST Admin API; OAuth 2.0; versioned API (quarterly releases, e.g. 2025-10)
- **Authentication:** Storefront access tokens (public/private); OAuth 2.0 for Admin API

### Commerce Layer

- **Description:** API-only headless commerce engine optimised for SKU-heavy, multi-market scenarios with granular price list management; 100% JSON:API compliant.
- **API Documentation:** https://docs.commercelayer.io/core
- **API Specification:** https://docs.commercelayer.io/core/api-specification
- **SDKs/Libraries:** `@commercelayer/sdk` (JavaScript/TypeScript) — https://github.com/commercelayer/commercelayer-sdk
- **Developer Guide:** https://docs.commercelayer.io/
- **Standards:** REST/JSON:API v1.0; OpenAPI schema published; OAuth 2.0
- **Authentication:** OAuth 2.0 client credentials and authorization code flows

### Medusa.js

- **Description:** Open-source Node.js/TypeScript headless commerce platform (MIT); self-hosted or Medusa Cloud; exposes REST APIs for store and admin operations.
- **API Documentation:** https://docs.medusajs.com/api/admin (Admin); https://docs.medusajs.com/api/store (Store)
- **SDKs/Libraries:** `@medusajs/js-sdk` — https://docs.medusajs.com/resources/commerce-modules/user/js-sdk
- **Developer Guide:** https://docs.medusajs.com/
- **GitHub:** https://github.com/medusajs/medusa
- **Standards:** REST; OpenAPI schema; plugin architecture
- **Authentication:** JWT-based authentication; API keys for admin

### BigCommerce

- **Description:** SaaS commerce platform with native headless support via GraphQL Storefront API and comprehensive REST APIs; strong B2B features.
- **API Documentation:** https://developer.bigcommerce.com/docs/api
- **GraphQL Storefront API:** https://developer.bigcommerce.com/docs/storefront/graphql
- **SDKs/Libraries:** BigCommerce JS SDK; REST API Postman collections; OpenAPI specs available for import
- **Developer Guide:** https://developer.bigcommerce.com/docs/build
- **Standards:** GraphQL + REST; OpenAPI-compatible; webhook events
- **Authentication:** API keys (REST); customer impersonation tokens (GraphQL Storefront)

### Saleor

- **Description:** Open-source GraphQL-first headless commerce platform (Apache 2.0) built on Django/Python; full GraphQL API with optional REST wrappers.
- **API Documentation:** https://docs.saleor.io/api-reference/
- **GraphQL Overview:** https://docs.saleor.io/api-usage/overview
- **SDKs/Libraries:** Saleor App SDK (TypeScript); official Python client; community JavaScript client
- **Developer Guide:** https://docs.saleor.io/
- **GitHub:** https://github.com/saleor/saleor-docs
- **Standards:** GraphQL (primary); schema introspection; webhook events; OSL 3.0 licence
- **Authentication:** JWT tokens; API keys for apps; OAuth-compatible

### Elastic Path

- **Description:** Enterprise composable commerce with separately deployable microservices for catalog, cart, promotions, and inventory; REST-primary with community GraphQL layer.
- **API Documentation:** https://documentation.elasticpath.com/
- **REST API:** https://developer.elasticpath.com/release-notes/rest-api
- **GraphQL Server:** https://github.com/elasticpath/elasticpath-graphql-server
- **Developer Guide:** https://documentation.elasticpath.com/
- **Standards:** REST; OpenAPI/Swagger specs; Postman collections; Insomnia support
- **Authentication:** OAuth 2.0 implicit and client credentials flows

### Vendure

- **Description:** Open-source TypeScript/Node headless commerce framework (MIT) with GraphQL-first API design and a type-safe plugin architecture.
- **API Documentation:** https://docs.vendure.io/reference/
- **GraphQL Guide:** https://docs.vendure.io/guides/getting-started/graphql-intro/
- **SDKs/Libraries:** Auto-generated TypeScript types from GraphQL schema via code generation
- **Developer Guide:** https://docs.vendure.io/
- **GitHub:** https://github.com/vendurehq/vendure
- **Standards:** GraphQL (primary, Shop API + Admin API); TypeScript type-safety via codegen; plugin architecture
- **Authentication:** Session-based auth; JWT tokens; custom auth strategies via plugin

---

## Notes

- **API versioning patterns:** Shopify uses date-based quarterly versions (e.g. `2025-10`); commercetools GraphQL API is intentionally versionless (additive-only); REST APIs for Medusa, Commerce Layer, and BigCommerce use path versioning (`/v2/`). A new headless engine should adopt a clear versioning strategy from day one.
- **Webhook security:** All major platforms sign webhook payloads with HMAC-SHA256 and include a signature header (e.g. `X-Saleor-Signature`, `X-Shopify-Hmac-Sha256`). Implementing this pattern and documenting it in an AsyncAPI document is recommended.
- **MCP as emerging standard:** The MACH Alliance MCP Registry (2025) signals that MCP server exposure is becoming expected for enterprise commerce platforms that wish to be AI-agent-accessible. This is an emerging, not yet universal, standard.
- **OpenAPI 3.2 streaming support:** The addition of native SSE and JSON Lines support in OpenAPI 3.2 is relevant for AI-native commerce engines that stream product recommendations or real-time inventory updates.
- **PCI DSS SAQ A clarification (2025):** The tightening of SAQ A eligibility in early 2025 means headless storefronts must carefully audit all JavaScript loaded on checkout pages, even from first-party origins, to maintain PCI compliance without full SAQ D assessment obligations.
