# Headless Commerce Engine

> Candidate #136 · Researched: 2026-05-02

## Existing Products and Software Packages

| Tool | Description | Type | Pricing | Strengths / Weaknesses |
|------|-------------|------|---------|------------------------|
| commercetools | Pioneered MACH architecture; fully API-first commerce platform targeting enterprise brands | SaaS | Custom enterprise ($$$) | Strength: unlimited customisation, global scale; Weakness: high implementation cost, requires dedicated engineering team |
| Shopify (+ Hydrogen) | Full commerce platform with headless option via Storefront API and Hydrogen React framework (Remix-based) | SaaS | $79–$399/month + custom | Strength: huge ecosystem, Oxygen hosting bundled; Weakness: locked into Shopify backend, multi-region limits |
| Commerce Layer | API-only commerce engine for SKU-heavy, multi-market scenarios; no admin UI included | SaaS | Usage-based ($$$) | Strength: fine-grained price lists and inventory per market; Weakness: requires separate CMS/admin tooling |
| Medusa.js | Open-source Node.js headless commerce platform, self-hosted or cloud | Open-source / Cloud | Free (self-host); Medusa Cloud custom | Strength: developer-friendly, composable modules; Weakness: smaller enterprise support ecosystem |
| BigCommerce | API-driven SaaS platform with native headless support via Stencil and REST/GraphQL APIs | SaaS | $39–$400/month + enterprise | Strength: no transaction fees, B2B features; Weakness: less composable than pure-MACH solutions |
| Saleor | Open-source, GraphQL-first headless commerce built for Python/Django environments | Open-source / Cloud | Free (self-host); Cloud custom | Strength: full GraphQL API, active community; Weakness: Python stack limits some integrations |
| Elastic Path | Enterprise composable commerce with separate microservices for catalog, cart, checkout, and promotions | SaaS | Custom | Strength: highly decoupled services; Weakness: complex multi-vendor integration overhead |
| Vendure | Open-source TypeScript/Node headless commerce framework with plugin architecture | Open-source | Free (self-host) | Strength: type-safe API, extensible; Weakness: smaller community than Medusa |

## Relevant Industry Standards or Protocols

- **MACH Architecture** — Microservices, API-first, Cloud-native, Headless; defined and promoted by the MACH Alliance (111+ members); the dominant architectural reference for composable commerce stacks
- **GraphQL** — query language used by Saleor, Shopify Storefront API, and others to allow clients to request precisely the data they need, reducing over-fetching
- **OpenAPI 3.0** — REST API specification standard; recommended for all headless commerce backends to publish alongside changelogs for interoperability
- **OAuth 2.0 / OpenID Connect** — standard authentication and authorisation protocols for securing API access between headless frontend and backend services
- **PCI DSS** — Payment Card Industry Data Security Standard; headless architectures must ensure tokenisation and compliant checkout flows even when frontend is fully decoupled
- **CloudEvents (CNCF)** — vendor-neutral event format used to wire microservices (cart events, order events) in composable stacks

## Available Research Materials

1. MACH Alliance (2025). *State of Composable Commerce Survey*. MACH Alliance. https://www.swell.is/content/mach-architecture-trends — industry survey; 87% of companies report implementing MACH technologies; not peer-reviewed
2. Gartner (2026). *Predicts: 60% of Organisations Will Rely on Composable Commerce Architectures by 2027*. Referenced in: https://www.bigcommerce.com/articles/headless-commerce/ — analyst forecast, not peer-reviewed
3. BigCommerce (2026). *Headless Commerce in 2026: Everything You Need to Know*. https://www.bigcommerce.com/articles/headless-commerce/ — vendor whitepaper
4. Digital Applied (2026). *Headless Commerce 2026: API-First eCommerce Guide*. https://www.digitalapplied.com/blog/headless-commerce-2026-api-first-ecommerce-guide — industry guide, not peer-reviewed
5. Strapi (2024). *A Developer's Guide to Headless Commerce*. https://strapi.io/blog/headless-commerce-guide — technical practitioner guide
6. Branch8 (2026). *Headless vs Composable Commerce 2026: Cost and Readiness Guide*. https://branch8.com/posts/headless-commerce-vs-composable-commerce-explained-2026 — comparative industry analysis
7. Swell (2025). *28 Headless Commerce Trends and Statistics*. https://www.swell.is/content/headless-commerce-trends-statistics — industry data aggregation

## Market Research

**Market Size:** The global headless commerce market was valued at approximately USD 1.74 billion in 2025 and is projected to reach USD 7.16 billion by 2032 at a 22.4% CAGR. The broader composable commerce market is forecast to grow from USD 6.44 billion in 2024 to USD 31.50 billion by 2034 at 17.2% CAGR.

**Funding:** commercetools raised $140M Series C (2021) at $1.9B valuation; Medusa raised $9.5M Seed (2023); Elastic Path raised $60M in growth funding. The category attracts both venture and private equity given enterprise deal sizes.

**Pricing Landscape:** Open-source options (Medusa, Saleor, Vendure) are free to self-host but incur engineering costs. SaaS platforms range from $80/month (Shopify) to $500k+/year for commercetools enterprise contracts. Commerce Layer and Elastic Path are custom-quoted at typical mid-market starting points of $30k–$100k/year.

**Key Buyer Personas:** CTOs and VP Engineering at mid-to-large retailers; Digital Commerce Directors; Platform Architects at brands with complex multi-region, multi-brand, or B2B needs.

**Notable Trends:** Composable commerce has become the default for enterprise new builds; AI personalisation layers (product recommendations, search ranking) are increasingly expected as first-class API services; edge rendering (Cloudflare Workers, Vercel Edge) is reducing storefront latency; TikTok Shop and social commerce channels require API-first backends to integrate quickly.

## AI-Native Opportunity

- **AI-driven product recommendations and search** — embedding vector search and LLM-ranked results natively in the commerce API layer, replacing third-party search plugins
- **Dynamic bundle and pricing generation** — AI that assembles product bundles and generates promotional pricing rules in real time based on customer context, inventory levels, and demand signals
- **Conversational commerce API** — a natural-language query layer over the commerce graph, enabling chatbots and voice interfaces to query catalog, check inventory, and initiate checkout without custom integration per channel
- **Automated catalog enrichment** — LLM-powered attribute extraction and description generation from supplier data sheets, dramatically reducing time-to-publish for new SKUs
- **AI-native fraud and anomaly detection** — ML models embedded in the order and checkout APIs to flag suspicious sessions, apply friction dynamically, and reduce chargebacks without third-party add-ons
