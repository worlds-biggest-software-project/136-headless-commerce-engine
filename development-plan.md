# Headless Commerce Engine — Phased Development Plan

> Project: 136-headless-commerce-engine · Created: 2026-05-25
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | TypeScript (Node.js 22 LTS) | Commerce APIs demand type safety across API boundaries; TypeScript enables shared types between API, admin, and client SDKs. Aligns with Medusa.js and Vendure precedent. Node.js 22 LTS provides native ESM, built-in test runner, and stable fetch API. |
| API framework | Fastify 5 | High-throughput HTTP framework with built-in schema validation (via JSON Schema/Ajv), OpenAPI 3.1 auto-generation via `@fastify/swagger`, lifecycle hooks for auth middleware, and first-class TypeScript support. Preferred over Express for performance and over NestJS for lower abstraction overhead. |
| GraphQL | Mercurius (on Fastify) | Fastify-native GraphQL adapter with JIT compilation, subscription support (WebSocket), and federation-ready. Co-exists with REST on the same Fastify instance. |
| Database | PostgreSQL 16 + pgvector | Relational integrity for commerce core (orders, inventory, pricing); JSONB columns for extensible attributes; pgvector for AI embedding search; generated columns for computed inventory. Matches Data Model Suggestion 3 (Hybrid Relational + JSONB). |
| Database ORM / query builder | Drizzle ORM | Type-safe SQL query builder with zero runtime overhead, native PostgreSQL JSONB operators, migration generation, and introspection. Lighter than Prisma, more type-safe than Knex. |
| Migrations | Drizzle Kit | Generates SQL migration files from the Drizzle schema, supports diffing against a live database, and produces plain `.sql` files for review. |
| Task queue | BullMQ (Redis-backed) | Handles async workloads: webhook delivery, inventory reservation expiry, AI embedding generation, email notifications. Redis also serves as the caching layer. |
| Cache | Redis 7 | Session cache, cart TTL, rate limiting, and BullMQ backing store. Single dependency for multiple concerns. |
| Search | PostgreSQL full-text + pgvector | MVP uses PostgreSQL tsvector/GIN indexes for text search and pgvector for semantic search. Avoids Elasticsearch dependency until scale demands it. |
| Authentication | Custom OAuth 2.0 (RFC 6749) + OIDC | API client credentials flow for machine-to-machine; authorization code flow for customer sessions. JWT access tokens, refresh tokens in Redis. No third-party auth service dependency. |
| Payment integration | Stripe SDK + PayPal REST API | Stripe as primary gateway (widest coverage); PayPal as secondary. Payment provider abstraction layer allows adding others. Never store raw card data (PCI DSS v4.0.1). |
| Event system | CloudEvents v1.0 + internal event bus | Domain events follow CloudEvents envelope format. Internal pub/sub via Node.js EventEmitter for in-process; BullMQ for async cross-module delivery; webhook delivery for external consumers. |
| Admin dashboard | React 19 + Vite + TanStack Router | Lightweight SPA for operational tasks (orders, inventory, customers). TanStack Query for data fetching. Separate from the API — communicates via the same GraphQL/REST APIs. |
| Testing | Vitest + Supertest + Testcontainers | Vitest for unit/integration tests (fast, ESM-native, compatible with Fastify). Supertest for HTTP-level integration tests. Testcontainers for PostgreSQL and Redis in CI. |
| Code quality | Biome (lint + format) + tsc --noEmit | Biome replaces ESLint + Prettier with a single, fast tool. TypeScript strict mode for type checking. |
| Containerisation | Docker + docker-compose | Multi-service local development (API, PostgreSQL, Redis). Production Dockerfile with multi-stage build. |
| Package manager | pnpm 9 | Workspace support for monorepo (API, admin, shared types, SDK). Faster and more disk-efficient than npm. |
| Monorepo structure | pnpm workspaces | Packages: `packages/api`, `packages/admin`, `packages/sdk`, `packages/shared`. Shared types and utilities across packages. |
| CI | GitHub Actions | Lint, type-check, test (with Testcontainers), build Docker image, publish SDK to npm. |

---

### Project Structure

```
headless-commerce-engine/
├── pnpm-workspace.yaml
├── package.json                         # Root workspace config
├── tsconfig.base.json                   # Shared TypeScript config
├── biome.json                           # Linting and formatting
├── docker-compose.yml                   # Local dev: API + PostgreSQL + Redis
├── Dockerfile                           # Multi-stage production build
├── .github/
│   └── workflows/
│       ├── ci.yml                       # Lint, type-check, test, build
│       └── release.yml                  # SDK publish, Docker image push
├── packages/
│   ├── shared/                          # Shared types, constants, utilities
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── src/
│   │       ├── types/
│   │       │   ├── catalog.ts           # Product, Variant, Category types
│   │       │   ├── pricing.ts           # PriceList, Price, Promotion types
│   │       │   ├── inventory.ts         # StockLocation, InventoryLevel types
│   │       │   ├── customer.ts          # Customer, Company, Address types
│   │       │   ├── cart.ts              # Cart, CartItem types
│   │       │   ├── order.ts             # Order, OrderItem, Payment types
│   │       │   ├── events.ts            # CloudEvents envelope types
│   │       │   └── common.ts            # Money, ISO codes, pagination
│   │       ├── constants/
│   │       │   ├── currencies.ts        # ISO 4217 currency codes
│   │       │   └── countries.ts         # ISO 3166-1 country codes
│   │       └── utils/
│   │           ├── money.ts             # Monetary arithmetic (cents-based)
│   │           └── slug.ts              # URL-safe slug generation
│   ├── api/                             # Fastify API server
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   ├── drizzle.config.ts            # Drizzle Kit migration config
│   │   └── src/
│   │       ├── index.ts                 # Server entry point
│   │       ├── config.ts                # Environment config with defaults
│   │       ├── db/
│   │       │   ├── connection.ts        # PostgreSQL connection pool
│   │       │   ├── schema/              # Drizzle schema definitions
│   │       │   │   ├── catalog.ts
│   │       │   │   ├── pricing.ts
│   │       │   │   ├── inventory.ts
│   │       │   │   ├── customers.ts
│   │       │   │   ├── carts.ts
│   │       │   │   ├── orders.ts
│   │       │   │   ├── tax.ts
│   │       │   │   ├── shipping.ts
│   │       │   │   ├── events.ts
│   │       │   │   └── ai.ts
│   │       │   └── migrations/          # Generated SQL migrations
│   │       ├── modules/                 # Business logic by domain
│   │       │   ├── catalog/
│   │       │   │   ├── catalog.service.ts
│   │       │   │   ├── catalog.routes.ts     # REST routes
│   │       │   │   ├── catalog.resolvers.ts  # GraphQL resolvers
│   │       │   │   └── catalog.schema.ts     # JSON Schema for validation
│   │       │   ├── pricing/
│   │       │   ├── inventory/
│   │       │   ├── customers/
│   │       │   ├── cart/
│   │       │   ├── orders/
│   │       │   ├── payments/
│   │       │   ├── fulfillment/
│   │       │   ├── tax/
│   │       │   ├── shipping/
│   │       │   ├── webhooks/
│   │       │   ├── search/
│   │       │   └── ai/
│   │       ├── graphql/
│   │       │   ├── schema.ts            # Merged GraphQL schema
│   │       │   └── scalars.ts           # Custom scalars (DateTime, Money, JSON)
│   │       ├── plugins/                 # Fastify plugins
│   │       │   ├── auth.ts              # OAuth 2.0 / JWT validation
│   │       │   ├── rate-limit.ts        # Redis-backed rate limiting
│   │       │   ├── event-bus.ts         # CloudEvents pub/sub
│   │       │   └── error-handler.ts     # Standardised error responses
│   │       └── workers/                 # BullMQ workers
│   │           ├── webhook-delivery.ts
│   │           ├── inventory-expiry.ts
│   │           ├── embedding-generator.ts
│   │           └── order-notifications.ts
│   ├── admin/                           # React admin dashboard
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   ├── vite.config.ts
│   │   └── src/
│   │       ├── main.tsx
│   │       ├── routes/
│   │       ├── components/
│   │       └── api/                     # API client (generated from OpenAPI)
│   └── sdk/                             # TypeScript client SDK
│       ├── package.json
│       ├── tsconfig.json
│       └── src/
│           ├── client.ts                # HTTP client wrapper
│           ├── resources/               # Resource-specific methods
│           └── types.ts                 # Re-exports from shared
└── tests/
    ├── fixtures/                        # Shared test data
    │   ├── products.json
    │   ├── orders.json
    │   └── customers.json
    └── e2e/                             # End-to-end test suites
        ├── catalog.e2e.test.ts
        ├── checkout.e2e.test.ts
        └── order-lifecycle.e2e.test.ts
```

---

## Phase 1: Project Foundation and Core Infrastructure

### Purpose

Establish the monorepo structure, database connection, configuration management, and Fastify server skeleton. After this phase, the API server starts, connects to PostgreSQL and Redis, serves a health check endpoint, and has the complete database schema migrated. Every subsequent phase builds on this foundation without restructuring.

### Tasks

#### 1.1 — Monorepo Setup and Tooling Configuration

**What**: Initialise the pnpm workspace with all packages, configure TypeScript, Biome, and Docker.

**Design**:

Root `pnpm-workspace.yaml`:
```yaml
packages:
  - "packages/*"
```

Root `tsconfig.base.json`:
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "outDir": "dist",
    "rootDir": "src"
  }
}
```

`biome.json`:
```json
{
  "$schema": "https://biomejs.dev/schemas/2.0.0/schema.json",
  "organizeImports": { "enabled": true },
  "linter": {
    "enabled": true,
    "rules": { "recommended": true }
  },
  "formatter": {
    "enabled": true,
    "indentStyle": "space",
    "indentWidth": 2,
    "lineWidth": 100
  }
}
```

`docker-compose.yml`:
```yaml
services:
  postgres:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_DB: commerce
      POSTGRES_USER: commerce
      POSTGRES_PASSWORD: commerce_dev
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
    volumes: ["redisdata:/data"]

  api:
    build: .
    depends_on: [postgres, redis]
    ports: ["3000:3000"]
    environment:
      DATABASE_URL: postgresql://commerce:commerce_dev@postgres:5432/commerce
      REDIS_URL: redis://redis:6379
    volumes:
      - ./packages:/app/packages

volumes:
  pgdata:
  redisdata:
```

Multi-stage `Dockerfile`:
```dockerfile
FROM node:22-alpine AS base
RUN corepack enable && corepack prepare pnpm@9 --activate
WORKDIR /app

FROM base AS deps
COPY pnpm-workspace.yaml package.json pnpm-lock.yaml ./
COPY packages/shared/package.json packages/shared/
COPY packages/api/package.json packages/api/
RUN pnpm install --frozen-lockfile

FROM base AS build
COPY --from=deps /app/node_modules ./node_modules
COPY --from=deps /app/packages/shared/node_modules ./packages/shared/node_modules
COPY --from=deps /app/packages/api/node_modules ./packages/api/node_modules
COPY . .
RUN pnpm --filter @commerce/shared build && pnpm --filter @commerce/api build

FROM base AS production
COPY --from=build /app/packages/api/dist ./dist
COPY --from=build /app/node_modules ./node_modules
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

**Testing**:
- `Unit: pnpm install completes without errors`
- `Unit: pnpm --filter @commerce/shared build produces dist/ with .js and .d.ts files`
- `Unit: biome check --apply passes with no errors on all packages`
- `Unit: tsc --noEmit passes across all packages`
- `Integration: docker-compose up starts PostgreSQL, Redis, and API containers`
- `Integration: docker-compose health checks pass for all services`

#### 1.2 — Configuration Management

**What**: Implement environment-based configuration with validation and sensible defaults.

**Design**:

```typescript
// packages/api/src/config.ts
import { z } from "zod";

const configSchema = z.object({
  // Server
  PORT: z.coerce.number().default(3000),
  HOST: z.string().default("0.0.0.0"),
  NODE_ENV: z.enum(["development", "production", "test"]).default("development"),
  LOG_LEVEL: z.enum(["fatal", "error", "warn", "info", "debug", "trace"]).default("info"),

  // Database
  DATABASE_URL: z.string().url(),
  DB_POOL_MIN: z.coerce.number().default(2),
  DB_POOL_MAX: z.coerce.number().default(10),

  // Redis
  REDIS_URL: z.string().url(),

  // Auth
  JWT_SECRET: z.string().min(32),
  JWT_ACCESS_TOKEN_TTL: z.string().default("15m"),
  JWT_REFRESH_TOKEN_TTL: z.string().default("7d"),

  // Payments
  STRIPE_SECRET_KEY: z.string().optional(),
  STRIPE_WEBHOOK_SECRET: z.string().optional(),
  PAYPAL_CLIENT_ID: z.string().optional(),
  PAYPAL_CLIENT_SECRET: z.string().optional(),

  // AI
  OPENAI_API_KEY: z.string().optional(),
  EMBEDDING_MODEL: z.string().default("text-embedding-3-small"),
  EMBEDDING_DIMENSIONS: z.coerce.number().default(1536),

  // Cart
  CART_TTL_HOURS: z.coerce.number().default(72),
  INVENTORY_RESERVATION_TTL_MINUTES: z.coerce.number().default(15),
});

export type Config = z.infer<typeof configSchema>;

export function loadConfig(): Config {
  const result = configSchema.safeParse(process.env);
  if (!result.success) {
    const formatted = result.error.issues
      .map((i) => `  ${i.path.join(".")}: ${i.message}`)
      .join("\n");
    throw new Error(`Configuration validation failed:\n${formatted}`);
  }
  return result.data;
}
```

**Testing**:
- `Unit: valid env vars -> Config object with correct types and defaults`
- `Unit: missing DATABASE_URL -> throws with "DATABASE_URL" in error message`
- `Unit: invalid NODE_ENV value -> throws with valid options listed`
- `Unit: JWT_SECRET shorter than 32 chars -> throws validation error`
- `Unit: all optional fields omitted -> Config has undefined for optionals, defaults for others`
- `Unit: numeric coercion from string env vars -> correct number values`

#### 1.3 — Database Schema and Migrations

**What**: Implement the complete Hybrid Relational + JSONB database schema (Data Model Suggestion 3) using Drizzle ORM and generate initial migration.

**Design**:

The schema follows Data Model Suggestion 3 with 22 core tables. Each domain area is a separate Drizzle schema file.

```typescript
// packages/api/src/db/schema/catalog.ts
import { pgTable, uuid, varchar, text, boolean, integer, jsonb, timestamp, index } from "drizzle-orm/pg-core";

export const productTypes = pgTable("product_types", {
  id: uuid("id").primaryKey().defaultRandom(),
  name: varchar("name", { length: 255 }).notNull(),
  slug: varchar("slug", { length: 255 }).notNull().unique(),
  description: text("description"),
  isDigital: boolean("is_digital").notNull().default(false),
  isShippingRequired: boolean("is_shipping_required").notNull().default(true),
  attributeSchema: jsonb("attribute_schema").notNull().default({}),
  variantAttributes: text("variant_attributes").array().notNull().default([]),
  filterableAttributes: text("filterable_attributes").array().notNull().default([]),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});

export const categories = pgTable("categories", {
  id: uuid("id").primaryKey().defaultRandom(),
  parentId: uuid("parent_id").references(() => categories.id, { onDelete: "set null" }),
  name: varchar("name", { length: 255 }).notNull(),
  slug: varchar("slug", { length: 255 }).notNull().unique(),
  description: text("description"),
  path: text("path").notNull().default(""),
  level: integer("level").notNull().default(0),
  sortOrder: integer("sort_order").notNull().default(0),
  isActive: boolean("is_active").notNull().default(true),
  metadata: jsonb("metadata").notNull().default({}),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index("idx_categories_parent").on(table.parentId),
]);

export const products = pgTable("products", {
  id: uuid("id").primaryKey().defaultRandom(),
  productTypeId: uuid("product_type_id").notNull().references(() => productTypes.id),
  name: varchar("name", { length: 500 }).notNull(),
  slug: varchar("slug", { length: 500 }).notNull().unique(),
  description: text("description"),
  status: varchar("status", { length: 20 }).notNull().default("draft"),
  attributes: jsonb("attributes").notNull().default({}),
  metadata: jsonb("metadata").notNull().default({}),
  channelConfig: jsonb("channel_config").notNull().default({}),
  media: jsonb("media").notNull().default([]),
  categoryIds: uuid("category_ids").array().notNull().default([]),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index("idx_products_status").on(table.status),
  index("idx_products_type").on(table.productTypeId),
]);

export const productVariants = pgTable("product_variants", {
  id: uuid("id").primaryKey().defaultRandom(),
  productId: uuid("product_id").notNull().references(() => products.id, { onDelete: "cascade" }),
  sku: varchar("sku", { length: 255 }).notNull().unique(),
  name: varchar("name", { length: 500 }),
  barcode: varchar("barcode", { length: 255 }),
  weightGrams: integer("weight_grams"),
  isMaster: boolean("is_master").notNull().default(false),
  sortOrder: integer("sort_order").notNull().default(0),
  attributes: jsonb("attributes").notNull().default({}),
  prices: jsonb("prices").notNull().default([]),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index("idx_variants_product").on(table.productId),
]);
```

Additional schema files follow the same pattern for all 22 tables from Data Model Suggestion 3: `pricing.ts` (channels, promotions), `inventory.ts` (stock_locations, inventory_levels, inventory_movements), `customers.ts` (customers, customer_groups, companies, api_clients), `carts.ts` (carts), `orders.ts` (orders, payments, fulfillments), `tax.ts` (tax_configurations), `shipping.ts` (shipping_methods), `events.ts` (domain_events, webhook_subscriptions), `ai.ts` (product_embeddings).

Database connection pool:
```typescript
// packages/api/src/db/connection.ts
import { drizzle } from "drizzle-orm/node-postgres";
import pg from "pg";
import type { Config } from "../config.js";

export function createDb(config: Config) {
  const pool = new pg.Pool({
    connectionString: config.DATABASE_URL,
    min: config.DB_POOL_MIN,
    max: config.DB_POOL_MAX,
  });
  return drizzle(pool);
}

export type Db = ReturnType<typeof createDb>;
```

**Testing**:
- `Unit: Drizzle schema compiles without type errors`
- `Integration (Testcontainers): drizzle-kit push applies schema to empty PostgreSQL`
- `Integration (Testcontainers): drizzle-kit generate produces migration SQL`
- `Integration (Testcontainers): migration runs idempotently (apply twice without error)`
- `Integration (Testcontainers): all 22 tables created with correct columns and indexes`
- `Integration (Testcontainers): JSONB GIN indexes exist on products.attributes, product_variants.prices`
- `Integration (Testcontainers): pgvector extension enabled, product_embeddings.embedding column has vector type`
- `Integration (Testcontainers): foreign key constraints enforce referential integrity (insert product with invalid product_type_id fails)`

#### 1.4 — Fastify Server with Health Check and OpenAPI

**What**: Create the Fastify server with request logging, error handling, CORS, OpenAPI spec generation, and health check endpoints.

**Design**:

```typescript
// packages/api/src/index.ts
import Fastify from "fastify";
import cors from "@fastify/cors";
import swagger from "@fastify/swagger";
import swaggerUi from "@fastify/swagger-ui";
import { loadConfig } from "./config.js";
import { createDb } from "./db/connection.js";
import { createRedisClient } from "./plugins/redis.js";

export async function buildServer() {
  const config = loadConfig();
  const server = Fastify({
    logger: {
      level: config.LOG_LEVEL,
      transport: config.NODE_ENV === "development"
        ? { target: "pino-pretty" }
        : undefined,
    },
    genReqId: () => crypto.randomUUID(),
  });

  // CORS
  await server.register(cors, {
    origin: config.NODE_ENV === "development" ? true : [],
    credentials: true,
  });

  // OpenAPI 3.1
  await server.register(swagger, {
    openapi: {
      openapi: "3.1.0",
      info: {
        title: "Headless Commerce Engine API",
        version: "0.1.0",
        description: "API-first commerce platform with AI personalisation",
      },
      servers: [{ url: `http://${config.HOST}:${config.PORT}` }],
      components: {
        securitySchemes: {
          bearerAuth: {
            type: "http",
            scheme: "bearer",
            bearerFormat: "JWT",
          },
          oauth2: {
            type: "oauth2",
            flows: {
              clientCredentials: {
                tokenUrl: "/oauth/token",
                scopes: {
                  "catalog:read": "Read catalog data",
                  "catalog:write": "Manage catalog data",
                  "orders:read": "Read orders",
                  "orders:write": "Manage orders",
                  "customers:read": "Read customers",
                  "customers:write": "Manage customers",
                  "inventory:read": "Read inventory",
                  "inventory:write": "Manage inventory",
                },
              },
            },
          },
        },
      },
    },
  });

  await server.register(swaggerUi, { routePrefix: "/docs" });

  // Database and Redis
  const db = createDb(config);
  const redis = createRedisClient(config);
  server.decorate("db", db);
  server.decorate("redis", redis);
  server.decorate("config", config);

  // Health check
  server.get("/health", {
    schema: {
      response: {
        200: {
          type: "object",
          properties: {
            status: { type: "string", enum: ["ok", "degraded"] },
            version: { type: "string" },
            timestamp: { type: "string", format: "date-time" },
            services: {
              type: "object",
              properties: {
                database: { type: "string", enum: ["up", "down"] },
                redis: { type: "string", enum: ["up", "down"] },
              },
            },
          },
        },
      },
    },
  }, async () => {
    const dbStatus = await db.execute("SELECT 1").then(() => "up" as const).catch(() => "down" as const);
    const redisStatus = await redis.ping().then(() => "up" as const).catch(() => "down" as const);
    return {
      status: dbStatus === "up" && redisStatus === "up" ? "ok" : "degraded",
      version: "0.1.0",
      timestamp: new Date().toISOString(),
      services: { database: dbStatus, redis: redisStatus },
    };
  });

  // Error handler
  server.setErrorHandler((error, request, reply) => {
    const statusCode = error.statusCode ?? 500;
    request.log.error({ err: error, req: request }, "Request error");
    reply.status(statusCode).send({
      error: {
        code: error.code ?? "INTERNAL_ERROR",
        message: statusCode >= 500 ? "Internal server error" : error.message,
        requestId: request.id,
      },
    });
  });

  return server;
}

const server = await buildServer();
await server.listen({ port: server.config.PORT, host: server.config.HOST });
```

Standardised error response type:
```typescript
// packages/shared/src/types/common.ts
export interface ApiError {
  error: {
    code: string;
    message: string;
    requestId: string;
    details?: Record<string, unknown>;
  };
}

export interface PaginatedResponse<T> {
  data: T[];
  pagination: {
    total: number;
    limit: number;
    offset: number;
    hasMore: boolean;
  };
}

export interface Money {
  amountCents: number;
  currencyCode: string; // ISO 4217
}
```

**Testing**:
- `Unit: buildServer() creates Fastify instance without throwing`
- `Integration (Testcontainers): GET /health returns 200 with status "ok" when DB and Redis are up`
- `Integration (mocked DB): GET /health returns 200 with status "degraded" when DB connection fails`
- `Integration: GET /docs returns Swagger UI HTML`
- `Integration: GET /docs/json returns valid OpenAPI 3.1 document`
- `Unit: error handler returns 500 with "Internal server error" for unhandled exceptions (no stack leak)`
- `Unit: error handler returns 400 with original message for validation errors`
- `Unit: every response includes requestId (UUID format)`

#### 1.5 — Shared Types and Utility Functions

**What**: Implement shared TypeScript types for all domain entities, monetary arithmetic utilities, and ISO code constants.

**Design**:

```typescript
// packages/shared/src/types/catalog.ts
export interface ProductType {
  id: string;
  name: string;
  slug: string;
  description?: string;
  isDigital: boolean;
  isShippingRequired: boolean;
  attributeSchema: Record<string, unknown>; // JSON Schema
  variantAttributes: string[];
  filterableAttributes: string[];
  createdAt: string; // ISO 8601
  updatedAt: string;
}

export interface Product {
  id: string;
  productTypeId: string;
  name: string;
  slug: string;
  description?: string;
  status: "draft" | "active" | "archived";
  attributes: Record<string, unknown>;
  metadata: Record<string, unknown>;
  channelConfig: Record<string, { visible: boolean; featured?: boolean }>;
  media: ProductMedia[];
  categoryIds: string[];
  createdAt: string;
  updatedAt: string;
}

export interface ProductMedia {
  type: "image" | "video" | "file";
  url: string;
  alt?: string;
  sort: number;
}

export interface ProductVariant {
  id: string;
  productId: string;
  sku: string;
  name?: string;
  barcode?: string;
  weightGrams?: number;
  isMaster: boolean;
  sortOrder: number;
  attributes: Record<string, unknown>;
  prices: VariantPrice[];
  createdAt: string;
  updatedAt: string;
}

export interface VariantPrice {
  priceList: string;
  currency: string; // ISO 4217
  amountCents: number;
  compareAtCents?: number;
  minQty: number;
}

export interface Category {
  id: string;
  parentId?: string;
  name: string;
  slug: string;
  description?: string;
  path: string;
  level: number;
  sortOrder: number;
  isActive: boolean;
  metadata: Record<string, unknown>;
  createdAt: string;
  updatedAt: string;
}
```

```typescript
// packages/shared/src/utils/money.ts
export function addCents(a: number, b: number): number {
  return a + b;
}

export function subtractCents(a: number, b: number): number {
  return a - b;
}

export function multiplyCents(amount: number, quantity: number): number {
  return Math.round(amount * quantity);
}

export function applyPercentageDiscount(amountCents: number, percentOff: number): number {
  return Math.round(amountCents * (1 - percentOff / 100));
}

export function formatMoney(amountCents: number, currencyCode: string, locale = "en-US"): string {
  return new Intl.NumberFormat(locale, {
    style: "currency",
    currency: currencyCode,
  }).format(amountCents / 100);
}
```

**Testing**:
- `Unit: multiplyCents(2999, 3) -> 8997 (no floating point error)`
- `Unit: applyPercentageDiscount(10000, 20) -> 8000`
- `Unit: applyPercentageDiscount(9999, 33) -> 6699 (rounds correctly)`
- `Unit: formatMoney(2999, "USD") -> "$29.99"`
- `Unit: formatMoney(100, "JPY", "ja-JP") -> handles zero-decimal currency`
- `Unit: all shared types compile without errors`
- `Unit: slug generator produces URL-safe strings from Unicode input`

---

## Phase 2: Catalog Management API

### Purpose

Implement the product catalog CRUD operations — the most fundamental commerce domain. After this phase, API consumers can create product types, define attribute schemas, manage categories, create products with variants, and query the catalog via both REST and GraphQL endpoints with pagination, filtering, and text search.

### Tasks

#### 2.1 — Product Types CRUD

**What**: REST and GraphQL endpoints for creating, reading, updating, and deleting product types with JSON Schema attribute definitions.

**Design**:

REST endpoints:
```
POST   /api/v1/product-types          # Create product type
GET    /api/v1/product-types           # List product types (paginated)
GET    /api/v1/product-types/:id       # Get product type by ID
PATCH  /api/v1/product-types/:id       # Update product type
DELETE /api/v1/product-types/:id       # Delete product type (fails if products exist)
```

Service layer:
```typescript
// packages/api/src/modules/catalog/catalog.service.ts
import type { Db } from "../../db/connection.js";
import { productTypes, products } from "../../db/schema/catalog.js";
import { eq, count } from "drizzle-orm";

export interface CreateProductTypeInput {
  name: string;
  slug: string;
  description?: string;
  isDigital?: boolean;
  isShippingRequired?: boolean;
  attributeSchema?: Record<string, unknown>;
  variantAttributes?: string[];
  filterableAttributes?: string[];
}

export class CatalogService {
  constructor(private db: Db) {}

  async createProductType(input: CreateProductTypeInput) {
    const [result] = await this.db.insert(productTypes).values({
      name: input.name,
      slug: input.slug,
      description: input.description,
      isDigital: input.isDigital ?? false,
      isShippingRequired: input.isShippingRequired ?? true,
      attributeSchema: input.attributeSchema ?? {},
      variantAttributes: input.variantAttributes ?? [],
      filterableAttributes: input.filterableAttributes ?? [],
    }).returning();
    return result;
  }

  async getProductType(id: string) {
    const [result] = await this.db.select().from(productTypes).where(eq(productTypes.id, id));
    return result ?? null;
  }

  async listProductTypes(limit: number, offset: number) {
    const [items, [{ total }]] = await Promise.all([
      this.db.select().from(productTypes).limit(limit).offset(offset).orderBy(productTypes.name),
      this.db.select({ total: count() }).from(productTypes),
    ]);
    return { items, total };
  }

  async deleteProductType(id: string) {
    const [productCount] = await this.db
      .select({ count: count() })
      .from(products)
      .where(eq(products.productTypeId, id));
    if (productCount.count > 0) {
      throw new ConflictError("Cannot delete product type with existing products");
    }
    await this.db.delete(productTypes).where(eq(productTypes.id, id));
  }
}
```

**Testing**:
- `Unit: createProductType with valid input -> returns product type with generated UUID`
- `Unit: createProductType with duplicate slug -> throws unique constraint error`
- `Integration: POST /api/v1/product-types -> 201 with product type body`
- `Integration: GET /api/v1/product-types -> 200 with paginated list`
- `Integration: GET /api/v1/product-types/:id -> 200 with single product type`
- `Integration: GET /api/v1/product-types/:invalid-uuid -> 404`
- `Integration: PATCH /api/v1/product-types/:id with new attributeSchema -> 200 with updated schema`
- `Integration: DELETE /api/v1/product-types/:id with no products -> 204`
- `Integration: DELETE /api/v1/product-types/:id with existing products -> 409 Conflict`
- `E2E: create product type via GraphQL mutation -> query returns it`

#### 2.2 — Category Management with Hierarchy

**What**: CRUD for categories with parent-child relationships, materialised paths, and tree retrieval.

**Design**:

REST endpoints:
```
POST   /api/v1/categories              # Create category
GET    /api/v1/categories              # List root categories (flat or tree)
GET    /api/v1/categories/:id          # Get category with children
GET    /api/v1/categories/:id/tree     # Get full subtree
PATCH  /api/v1/categories/:id          # Update category
DELETE /api/v1/categories/:id          # Delete (reassigns children to parent)
```

Path computation on insert/update:
```typescript
async createCategory(input: CreateCategoryInput) {
  let path = "/";
  let level = 0;
  if (input.parentId) {
    const parent = await this.getCategory(input.parentId);
    if (!parent) throw new NotFoundError("Parent category not found");
    path = `${parent.path}${parent.slug}/`;
    level = parent.level + 1;
  }
  const [result] = await this.db.insert(categories).values({
    ...input,
    path,
    level,
  }).returning();
  return result;
}
```

**Testing**:
- `Unit: root category path is "/"`
- `Unit: child of "/electronics/" with slug "phones" gets path "/electronics/phones/"`
- `Integration: create parent -> create child -> child.level is 1, path includes parent slug`
- `Integration: GET /api/v1/categories?tree=true -> returns nested tree structure`
- `Integration: delete parent category -> children reparented to grandparent`
- `Integration: move category (change parentId) -> path and level updated for entire subtree`

#### 2.3 — Products and Variants CRUD

**What**: Full product lifecycle management with variant creation, attribute validation against product type schema, and multi-format media attachments.

**Design**:

REST endpoints:
```
POST   /api/v1/products                   # Create product
GET    /api/v1/products                   # List products (paginated, filtered)
GET    /api/v1/products/:id               # Get product with variants
PATCH  /api/v1/products/:id               # Update product
DELETE /api/v1/products/:id               # Archive product (soft delete via status)
POST   /api/v1/products/:id/variants      # Add variant
PATCH  /api/v1/products/:id/variants/:vid # Update variant
DELETE /api/v1/products/:id/variants/:vid # Remove variant
POST   /api/v1/products/:id/media         # Add media item
DELETE /api/v1/products/:id/media/:index  # Remove media item
```

Attribute validation:
```typescript
import Ajv from "ajv";

const ajv = new Ajv();

async createProduct(input: CreateProductInput) {
  const productType = await this.getProductType(input.productTypeId);
  if (!productType) throw new NotFoundError("Product type not found");

  // Validate attributes against product type's JSON Schema
  if (Object.keys(productType.attributeSchema).length > 0) {
    const validate = ajv.compile(productType.attributeSchema);
    if (!validate(input.attributes)) {
      throw new ValidationError("Invalid product attributes", validate.errors);
    }
  }

  const [product] = await this.db.insert(products).values({
    productTypeId: input.productTypeId,
    name: input.name,
    slug: input.slug,
    description: input.description,
    attributes: input.attributes ?? {},
    metadata: input.metadata ?? {},
    channelConfig: input.channelConfig ?? {},
    media: input.media ?? [],
    categoryIds: input.categoryIds ?? [],
  }).returning();

  return product;
}
```

Product list filtering:
```typescript
async listProducts(filters: ProductFilters) {
  let query = this.db.select().from(products);

  if (filters.status) query = query.where(eq(products.status, filters.status));
  if (filters.productTypeId) query = query.where(eq(products.productTypeId, filters.productTypeId));
  if (filters.categoryId) query = query.where(sql`${products.categoryIds} @> ARRAY[${filters.categoryId}]::uuid[]`);
  if (filters.attributes) query = query.where(sql`${products.attributes} @> ${JSON.stringify(filters.attributes)}::jsonb`);
  if (filters.search) query = query.where(sql`${products.name} ILIKE ${"%" + filters.search + "%"}`);

  return query.limit(filters.limit).offset(filters.offset);
}
```

**Testing**:
- `Unit: create product with valid attributes matching product type schema -> succeeds`
- `Unit: create product with attributes violating schema (missing required field) -> ValidationError`
- `Unit: create product with invalid productTypeId -> NotFoundError`
- `Integration: POST product -> GET product returns it with all fields`
- `Integration: POST variant with sku -> variant.sku is unique`
- `Integration: PATCH product status "draft" -> "active" -> status updated`
- `Integration: DELETE product -> status set to "archived" (not hard deleted)`
- `Integration: filter products by attributes @> '{"color":"black"}' -> returns matching products`
- `Integration: filter products by categoryId -> returns products in that category`
- `Integration: pagination with limit=10, offset=20 -> correct subset returned`
- `E2E: create product type -> create product -> add 3 variants -> GET product returns product with 3 variants`

#### 2.4 — GraphQL Schema for Catalog

**What**: GraphQL schema and resolvers for querying the entire catalog, supporting nested queries (product -> variants -> prices, category -> children).

**Design**:

```graphql
type Query {
  product(id: ID!): Product
  products(
    filter: ProductFilter
    sort: ProductSort
    pagination: PaginationInput
  ): ProductConnection!

  category(id: ID!): Category
  categories(parentId: ID): [Category!]!

  productType(id: ID!): ProductType
  productTypes: [ProductType!]!
}

type Mutation {
  createProduct(input: CreateProductInput!): Product!
  updateProduct(id: ID!, input: UpdateProductInput!): Product!
  archiveProduct(id: ID!): Product!

  createVariant(productId: ID!, input: CreateVariantInput!): ProductVariant!
  updateVariant(id: ID!, input: UpdateVariantInput!): ProductVariant!
  deleteVariant(id: ID!): Boolean!
}

type Product {
  id: ID!
  name: String!
  slug: String!
  description: String
  status: ProductStatus!
  attributes: JSON!
  media: [ProductMedia!]!
  variants: [ProductVariant!]!
  categories: [Category!]!
  productType: ProductType!
  createdAt: DateTime!
  updatedAt: DateTime!
}

type ProductVariant {
  id: ID!
  sku: String!
  name: String
  attributes: JSON!
  prices: [VariantPrice!]!
  inventoryAvailable: Int
}

type ProductConnection {
  edges: [ProductEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}
```

**Testing**:
- `Integration: GraphQL query product(id) -> returns product with nested variants`
- `Integration: GraphQL query products with filter -> returns filtered results`
- `Integration: GraphQL mutation createProduct -> returns created product`
- `Integration: GraphQL query categories -> returns tree structure`
- `Unit: GraphQL schema introspection query succeeds`
- `Unit: invalid GraphQL query -> returns error with field path`

---

## Phase 3: Pricing, Inventory, and Channels

### Purpose

Add the commercial core: multi-channel support, price list management, inventory tracking with stock locations, and promotion rules. After this phase, the platform can represent real commerce scenarios — products with different prices per market, real-time stock levels, and promotional discounts.

### Tasks

#### 3.1 — Channel Management

**What**: CRUD for sales channels with currency, country, and configuration management.

**Design**:

```typescript
// packages/shared/src/types/pricing.ts
export interface Channel {
  id: string;
  name: string;
  slug: string;
  currencyCode: string; // ISO 4217
  countryCode?: string; // ISO 3166-1 alpha-2
  isActive: boolean;
  config: ChannelConfig;
  createdAt: string;
  updatedAt: string;
}

export interface ChannelConfig {
  supportedCurrencies?: string[];
  supportedCountries?: string[];
  taxInclusivePricing?: boolean;
  defaultLocale?: string;
  checkoutConfig?: {
    requirePhone?: boolean;
    allowGuestCheckout?: boolean;
    minOrderCents?: number;
  };
}
```

REST endpoints:
```
POST   /api/v1/channels
GET    /api/v1/channels
GET    /api/v1/channels/:id
PATCH  /api/v1/channels/:id
DELETE /api/v1/channels/:id
```

**Testing**:
- `Unit: create channel with valid ISO 4217 currency -> succeeds`
- `Unit: create channel with invalid currency code "XXX" -> validation error`
- `Integration: POST channel -> GET returns channel with config`
- `Integration: PATCH channel config -> nested JSONB updated correctly`

#### 3.2 — Price List and Pricing Management

**What**: Price lists per channel and customer group, with variant-level pricing including quantity breaks and compare-at prices. Prices stored as JSONB array on variants.

**Design**:

Service operations:
```typescript
export interface SetPriceInput {
  variantId: string;
  priceList: string;
  currency: string;
  amountCents: number;
  compareAtCents?: number;
  minQty?: number;
}

async setPrice(input: SetPriceInput) {
  // Read current prices JSONB array from variant
  const variant = await this.db.select().from(productVariants).where(eq(productVariants.id, input.variantId));
  if (!variant[0]) throw new NotFoundError("Variant not found");

  const prices: VariantPrice[] = variant[0].prices as VariantPrice[];

  // Upsert: find existing entry for this priceList + currency + minQty
  const existingIdx = prices.findIndex(
    (p) => p.priceList === input.priceList && p.currency === input.currency && p.minQty === (input.minQty ?? 1)
  );

  const newEntry: VariantPrice = {
    priceList: input.priceList,
    currency: input.currency,
    amountCents: input.amountCents,
    compareAtCents: input.compareAtCents,
    minQty: input.minQty ?? 1,
  };

  if (existingIdx >= 0) {
    prices[existingIdx] = newEntry;
  } else {
    prices.push(newEntry);
  }

  await this.db.update(productVariants)
    .set({ prices, updatedAt: new Date() })
    .where(eq(productVariants.id, input.variantId));
}

async resolvePrice(variantId: string, channel: string, currency: string, quantity: number, customerGroupId?: string): Promise<VariantPrice | null> {
  // Returns the best matching price: channel-specific > customer-group > default, highest minQty <= quantity
}
```

REST endpoints:
```
PUT    /api/v1/variants/:id/prices     # Set/update a price entry
GET    /api/v1/variants/:id/prices     # Get all prices for a variant
DELETE /api/v1/variants/:id/prices     # Remove a specific price entry
POST   /api/v1/prices/resolve          # Resolve best price for context
```

**Testing**:
- `Unit: setPrice creates new entry in JSONB prices array`
- `Unit: setPrice with existing priceList+currency+minQty -> updates existing entry`
- `Unit: resolvePrice picks channel-specific price over default`
- `Unit: resolvePrice picks highest minQty <= requested quantity`
- `Integration: set 3 prices for variant -> GET returns all 3`
- `Integration: resolve price for quantity=10 with minQty thresholds at 1, 5, 25 -> picks minQty=5`

#### 3.3 — Inventory Management

**What**: Stock locations, inventory levels with on-hand/reserved/available tracking, and an append-only movement audit trail.

**Design**:

```typescript
// packages/shared/src/types/inventory.ts
export interface InventoryLevel {
  id: string;
  variantId: string;
  stockLocationId: string;
  quantityOnHand: number;
  quantityReserved: number;
  quantityAvailable: number; // computed: onHand - reserved
  lowStockThreshold?: number;
  updatedAt: string;
}

export interface InventoryMovement {
  id: string;
  inventoryLevelId: string;
  quantityChange: number;
  reason: "received" | "sold" | "returned" | "adjusted" | "reserved" | "unreserved" | "damaged" | "transferred";
  referenceType?: string;
  referenceId?: string;
  context: Record<string, unknown>;
  createdAt: string;
  createdBy?: string;
}
```

Service with transactional stock updates:
```typescript
async adjustStock(input: AdjustStockInput): Promise<InventoryLevel> {
  return this.db.transaction(async (tx) => {
    // Lock the inventory row for update
    const [level] = await tx
      .select()
      .from(inventoryLevels)
      .where(and(
        eq(inventoryLevels.variantId, input.variantId),
        eq(inventoryLevels.stockLocationId, input.stockLocationId)
      ))
      .for("update");

    if (!level) throw new NotFoundError("Inventory level not found");

    const newOnHand = level.quantityOnHand + input.quantityChange;
    if (newOnHand < 0) throw new ConflictError("Insufficient stock");

    // Update level
    const [updated] = await tx.update(inventoryLevels)
      .set({ quantityOnHand: newOnHand, updatedAt: new Date() })
      .where(eq(inventoryLevels.id, level.id))
      .returning();

    // Record movement
    await tx.insert(inventoryMovements).values({
      inventoryLevelId: level.id,
      quantityChange: input.quantityChange,
      reason: input.reason,
      referenceType: input.referenceType,
      referenceId: input.referenceId,
      context: input.context ?? {},
    });

    return updated;
  });
}
```

REST endpoints:
```
POST   /api/v1/stock-locations                              # Create stock location
GET    /api/v1/stock-locations                              # List stock locations
GET    /api/v1/inventory/:variantId                         # Get all levels for variant
POST   /api/v1/inventory/:variantId/locations/:locationId   # Set initial inventory
PATCH  /api/v1/inventory/:variantId/locations/:locationId   # Adjust stock
GET    /api/v1/inventory/:variantId/movements               # Get movement history
POST   /api/v1/inventory/reserve                            # Reserve stock for cart
POST   /api/v1/inventory/release                            # Release reservation
```

**Testing**:
- `Unit: adjustStock +10 -> quantityOnHand increases by 10, movement recorded`
- `Unit: adjustStock -5 when onHand=3 -> ConflictError "Insufficient stock"`
- `Unit: reserve 2 units -> quantityReserved increases by 2, quantityAvailable decreases`
- `Integration: concurrent stock adjustments with FOR UPDATE locking -> no race condition`
- `Integration: GET movements -> returns chronological audit trail with reasons`
- `Integration: reserve + release -> net quantityReserved returns to original`
- `Fixture: create location, set stock, adjust, verify movement trail matches`

#### 3.4 — Promotions Engine

**What**: JSONB-based promotion rules with support for percentage off, fixed amount off, buy-X-get-Y, free shipping, and bundle pricing, scoped to channels and customer groups.

**Design**:

```typescript
export interface PromotionRules {
  type: "percentage_off" | "fixed_amount_off" | "buy_x_get_y" | "free_shipping" | "bundle_price";
  value: number;
  conditions?: {
    minOrderCents?: number;
    productTypeSlugs?: string[];
    categoryIds?: string[];
    customerGroups?: string[];
    variantIds?: string[];
  };
  limits?: {
    maxDiscountCents?: number;
    onePerCustomer?: boolean;
  };
}

async applyPromotion(cartId: string, code: string): Promise<DiscountResult> {
  const promotion = await this.findActivePromotion(code);
  if (!promotion) throw new NotFoundError("Invalid promotion code");
  if (!this.isValidForChannel(promotion, cart.channelId)) throw new ValidationError("Promotion not valid for this channel");
  if (promotion.maxUses && promotion.currentUses >= promotion.maxUses) throw new ConflictError("Promotion usage limit reached");

  const discount = this.calculateDiscount(promotion.rules, cart);
  return discount;
}
```

**Testing**:
- `Unit: percentage_off 20% on $100 cart -> $20 discount`
- `Unit: percentage_off 20% with maxDiscountCents 1500 -> capped at $15`
- `Unit: fixed_amount_off $10 on $8 cart -> discount capped at cart total`
- `Unit: promotion with expired valid_until -> throws "Promotion expired"`
- `Unit: promotion with maxUses=100 and currentUses=100 -> throws usage limit error`
- `Unit: promotion scoped to channel "web" applied on channel "mobile" -> throws channel error`
- `Integration: apply code -> cart discount updated, promotion.currentUses incremented`

---

## Phase 4: Customer Accounts and Authentication

### Purpose

Implement customer registration, login, OAuth 2.0 API authentication, B2B company accounts, and GDPR compliance features. After this phase, the API is secured with proper authentication and authorization, customers can manage their accounts, and B2B buyers can operate within company spending limits.

### Tasks

#### 4.1 — OAuth 2.0 Token Endpoint and JWT Middleware

**What**: Token issuance endpoint supporting client credentials (API clients) and password grant (customer login), JWT generation, and Fastify auth plugin.

**Design**:

```typescript
// packages/api/src/plugins/auth.ts
import fp from "fastify-plugin";
import jwt from "@fastify/jwt";

export default fp(async (fastify) => {
  await fastify.register(jwt, {
    secret: fastify.config.JWT_SECRET,
    sign: { expiresIn: fastify.config.JWT_ACCESS_TOKEN_TTL },
  });

  fastify.decorate("authenticate", async (request, reply) => {
    try {
      await request.jwtVerify();
    } catch (err) {
      reply.status(401).send({ error: { code: "UNAUTHORIZED", message: "Invalid or expired token" } });
    }
  });

  fastify.decorate("authorize", (...scopes: string[]) => {
    return async (request, reply) => {
      await request.jwtVerify();
      const tokenScopes: string[] = request.user.scopes ?? [];
      const hasScope = scopes.some((s) => tokenScopes.includes(s));
      if (!hasScope) {
        reply.status(403).send({ error: { code: "FORBIDDEN", message: `Required scope: ${scopes.join(" or ")}` } });
      }
    };
  });
});

// Token endpoint
// POST /oauth/token
interface TokenRequest {
  grant_type: "client_credentials" | "password";
  client_id?: string;
  client_secret?: string;
  email?: string;
  password?: string;
  scope?: string;
}

interface TokenResponse {
  access_token: string;
  token_type: "Bearer";
  expires_in: number;
  refresh_token?: string;
  scope: string;
}
```

**Testing**:
- `Unit: client_credentials grant with valid client_id/secret -> returns access_token`
- `Unit: client_credentials with invalid secret -> 401`
- `Unit: password grant with valid email/password -> returns access_token + refresh_token`
- `Unit: password grant with wrong password -> 401`
- `Unit: JWT token with expired TTL -> authenticate middleware returns 401`
- `Unit: token with scope "catalog:read" accessing "orders:write" endpoint -> 403`
- `Integration: full flow: create API client -> request token -> use token on protected endpoint -> 200`
- `Integration: rate limiting on /oauth/token -> 429 after threshold`

#### 4.2 — Customer Registration and Profile Management

**What**: Customer CRUD with password hashing, address management (JSONB), GDPR consent tracking, and customer group membership.

**Design**:

REST endpoints:
```
POST   /api/v1/customers                   # Register customer
GET    /api/v1/customers                   # List customers (admin)
GET    /api/v1/customers/:id               # Get customer (admin or self)
PATCH  /api/v1/customers/:id               # Update profile
DELETE /api/v1/customers/:id               # GDPR erasure
POST   /api/v1/customers/:id/addresses     # Add address
PATCH  /api/v1/customers/:id/addresses/:aid # Update address
DELETE /api/v1/customers/:id/addresses/:aid # Remove address
GET    /api/v1/customers/:id/orders        # Customer order history
POST   /api/v1/customers/:id/data-export   # GDPR data export
```

```typescript
export interface CreateCustomerInput {
  email: string;
  password: string;
  firstName?: string;
  lastName?: string;
  phone?: string;
  acceptsMarketing?: boolean;
}

async createCustomer(input: CreateCustomerInput): Promise<Customer> {
  const existingEmail = await this.db.select().from(customers).where(eq(customers.email, input.email.toLowerCase()));
  if (existingEmail.length > 0) throw new ConflictError("Email already registered");

  const passwordHash = await argon2.hash(input.password);

  const [customer] = await this.db.insert(customers).values({
    email: input.email.toLowerCase(),
    passwordHash,
    firstName: input.firstName,
    lastName: input.lastName,
    phone: input.phone,
    acceptsMarketing: input.acceptsMarketing ?? false,
    marketingConsentAt: input.acceptsMarketing ? new Date() : null,
  }).returning();

  // Emit customer.registered event
  await this.eventBus.emit({
    type: "customer.registered",
    source: "/customers",
    subject: customer.id,
    data: { customerId: customer.id, email: customer.email },
  });

  return this.sanitize(customer); // strip passwordHash
}
```

GDPR erasure (right-to-erasure):
```typescript
async eraseCustomer(id: string): Promise<void> {
  await this.db.update(customers).set({
    email: `erased-${id}@deleted.local`,
    firstName: null,
    lastName: null,
    phone: null,
    addresses: [],
    customFields: {},
    isActive: false,
    updatedAt: new Date(),
  }).where(eq(customers.id, id));

  await this.eventBus.emit({
    type: "customer.data_erased",
    source: "/customers",
    subject: id,
    data: { customerId: id, erasedAt: new Date().toISOString() },
  });
}
```

**Testing**:
- `Unit: createCustomer hashes password with argon2 (hash != plaintext)`
- `Unit: createCustomer with duplicate email -> ConflictError`
- `Unit: customer response never includes passwordHash field`
- `Unit: eraseCustomer replaces PII with anonymised data`
- `Integration: register -> login -> get profile -> profile matches input`
- `Integration: add 2 addresses -> JSONB array has 2 entries`
- `Integration: GDPR export -> returns JSON with all customer data`
- `Integration: GDPR erasure -> customer email replaced, addresses cleared`

#### 4.3 — B2B Company Accounts

**What**: Company management with member roles, spending limits, and purchase order requirements.

**Design**:

```typescript
export interface Company {
  id: string;
  name: string;
  taxId?: string;
  registrationNumber?: string;
  customerGroupId?: string;
  isActive: boolean;
  settings: CompanySettings;
  members: CompanyMember[];
  createdAt: string;
  updatedAt: string;
}

export interface CompanySettings {
  paymentTerms?: "prepaid" | "net_15" | "net_30" | "net_60";
  creditLimitCents?: number;
  autoApproveOrdersUnderCents?: number;
  requiredPoNumber?: boolean;
  approvedShippingMethods?: string[];
}

export interface CompanyMember {
  customerId: string;
  role: "admin" | "buyer" | "approver" | "viewer";
  spendingLimitCents?: number;
}
```

REST endpoints:
```
POST   /api/v1/companies                      # Create company
GET    /api/v1/companies                      # List companies
GET    /api/v1/companies/:id                  # Get company
PATCH  /api/v1/companies/:id                  # Update company
POST   /api/v1/companies/:id/members          # Add member
PATCH  /api/v1/companies/:id/members/:cid     # Update member role/limits
DELETE /api/v1/companies/:id/members/:cid     # Remove member
```

**Testing**:
- `Unit: buyer with spendingLimit=50000 placing 60000 order -> rejected`
- `Unit: approver can approve orders over autoApproveOrdersUnderCents threshold`
- `Unit: requiredPoNumber=true and no PO on order -> validation error`
- `Integration: create company -> add 2 members -> GET returns company with both members`
- `Integration: member with role "viewer" cannot create orders for company`

---

## Phase 5: Cart, Checkout, and Order Management

### Purpose

Implement the transactional heart of the commerce engine: shopping carts with inventory reservation, checkout with address/shipping/tax computation, payment gateway integration, and order lifecycle management. After this phase, the platform supports end-to-end purchase flows.

### Tasks

#### 5.1 — Shopping Cart API

**What**: Cart creation, item management, promotion application, and automatic expiry with inventory reservation.

**Design**:

```typescript
// packages/shared/src/types/cart.ts
export interface Cart {
  id: string;
  customerId?: string;
  channelId: string;
  status: "active" | "merged" | "converted" | "abandoned";
  currencyCode: string;
  email?: string;
  items: CartItem[];
  shippingAddress?: Address;
  billingAddress?: Address;
  subtotalCents: number;
  discountCents: number;
  shippingCents: number;
  taxCents: number;
  totalCents: number;
  appliedPromotions: AppliedPromotion[];
  notes?: string;
  expiresAt?: string;
  createdAt: string;
  updatedAt: string;
}

export interface CartItem {
  id: string;
  variantId: string;
  productId: string;
  sku: string;
  productName: string;
  variantName?: string;
  quantity: number;
  unitPriceCents: number;
  totalCents: number;
  attributes: Record<string, unknown>;
}
```

Cart recalculation on every mutation:
```typescript
async addItem(cartId: string, input: AddItemInput): Promise<Cart> {
  return this.db.transaction(async (tx) => {
    const cart = await this.getCartForUpdate(tx, cartId);

    // Resolve price for this variant in cart's channel/currency
    const price = await this.pricingService.resolvePrice(
      input.variantId, cart.channelId, cart.currencyCode, input.quantity
    );
    if (!price) throw new ValidationError("No price available for this variant in this channel");

    // Check inventory availability
    const available = await this.inventoryService.getAvailableQuantity(input.variantId);
    if (available < input.quantity) throw new ConflictError("Insufficient stock");

    // Add or merge item
    const items = [...cart.items];
    const existingIdx = items.findIndex((i) => i.variantId === input.variantId);
    if (existingIdx >= 0) {
      items[existingIdx].quantity += input.quantity;
      items[existingIdx].totalCents = items[existingIdx].quantity * items[existingIdx].unitPriceCents;
    } else {
      const variant = await this.catalogService.getVariant(input.variantId);
      const product = await this.catalogService.getProduct(variant.productId);
      items.push({
        id: crypto.randomUUID(),
        variantId: input.variantId,
        productId: product.id,
        sku: variant.sku,
        productName: product.name,
        variantName: variant.name,
        quantity: input.quantity,
        unitPriceCents: price.amountCents,
        totalCents: price.amountCents * input.quantity,
        attributes: variant.attributes,
      });
    }

    // Reserve inventory
    await this.inventoryService.reserve({
      variantId: input.variantId,
      quantity: input.quantity,
      referenceType: "cart",
      referenceId: cartId,
      ttlMinutes: this.config.INVENTORY_RESERVATION_TTL_MINUTES,
    });

    // Recalculate totals
    const totals = this.calculateTotals(items, cart.appliedPromotions);

    await tx.update(carts).set({
      items,
      ...totals,
      updatedAt: new Date(),
    }).where(eq(carts.id, cartId));

    return this.getCart(cartId);
  });
}
```

REST endpoints:
```
POST   /api/v1/carts                         # Create cart
GET    /api/v1/carts/:id                     # Get cart
POST   /api/v1/carts/:id/items               # Add item
PATCH  /api/v1/carts/:id/items/:itemId       # Update item quantity
DELETE /api/v1/carts/:id/items/:itemId       # Remove item
POST   /api/v1/carts/:id/promotions          # Apply promotion code
DELETE /api/v1/carts/:id/promotions/:code    # Remove promotion
PATCH  /api/v1/carts/:id/shipping-address    # Set shipping address
PATCH  /api/v1/carts/:id/billing-address     # Set billing address
```

**Testing**:
- `Unit: addItem calculates totalCents = unitPriceCents * quantity`
- `Unit: addItem same variant twice -> merges quantities instead of duplicate`
- `Unit: addItem with insufficient stock -> ConflictError`
- `Unit: removeItem releases inventory reservation`
- `Unit: recalculate totals after discount -> subtotal - discount + shipping + tax = total`
- `Integration: create cart -> add 3 items -> GET returns cart with correct totals`
- `Integration: cart expiry after CART_TTL_HOURS -> status set to "abandoned", reservations released`
- `Integration: apply promotion code -> discountCents updated, totalCents recalculated`
- `E2E: full cart lifecycle: create -> add items -> set addresses -> apply promo -> verify totals`

#### 5.2 — Tax Calculation

**What**: Address-based tax computation using JSONB tax configuration rules, supporting VAT (tax-inclusive) and sales tax (tax-exclusive) models.

**Design**:

```typescript
export interface TaxResult {
  totalTaxCents: number;
  breakdown: TaxBreakdownEntry[];
  isIncludedInPrice: boolean;
}

export interface TaxBreakdownEntry {
  name: string;
  ratePercent: number;
  amountCents: number;
}

async calculateTax(
  items: CartItem[],
  shippingAddress: Address,
  channelConfig: ChannelConfig
): Promise<TaxResult> {
  const taxConfig = await this.getActiveTaxConfig();
  const matchingZone = this.findMatchingZone(taxConfig.rules.zones, shippingAddress);

  if (!matchingZone) return { totalTaxCents: 0, breakdown: [], isIncludedInPrice: false };

  const breakdown: TaxBreakdownEntry[] = [];
  const itemsTotal = items.reduce((sum, i) => sum + i.totalCents, 0);

  for (const rate of matchingZone.rates) {
    // Check if rate has more specific postal code match
    if (rate.match?.postalCodePrefix && !shippingAddress.postalCode?.startsWith(rate.match.postalCodePrefix)) {
      continue;
    }
    const taxAmount = Math.round(itemsTotal * (rate.ratePercent / 100));
    breakdown.push({ name: rate.name, ratePercent: rate.ratePercent, amountCents: taxAmount });
  }

  return {
    totalTaxCents: breakdown.reduce((sum, b) => sum + b.amountCents, 0),
    breakdown,
    isIncludedInPrice: matchingZone.isIncludedInPrice ?? false,
  };
}
```

**Testing**:
- `Unit: US-CA address with 7.25% state + 1.25% county -> two breakdown entries`
- `Unit: DE address with 19% VAT inclusive -> tax included in existing price`
- `Unit: address with no matching tax zone -> zero tax`
- `Unit: compound tax (tax on tax) calculated correctly when isCompound=true`
- `Integration: set tax config JSONB -> calculate tax for matching address -> correct amounts`

#### 5.3 — Checkout and Payment Integration

**What**: Checkout flow that validates the cart, initiates payment via Stripe or PayPal, and converts the cart to an order upon successful payment.

**Design**:

Payment provider abstraction:
```typescript
export interface PaymentProvider {
  name: string;
  createPaymentIntent(input: CreatePaymentInput): Promise<PaymentIntent>;
  capturePayment(transactionId: string): Promise<PaymentResult>;
  refundPayment(transactionId: string, amountCents: number): Promise<RefundResult>;
  handleWebhook(payload: Buffer, signature: string): Promise<WebhookEvent>;
}

export interface CreatePaymentInput {
  amountCents: number;
  currencyCode: string;
  orderId: string;
  customerEmail: string;
  metadata?: Record<string, string>;
}

export interface PaymentIntent {
  gatewayTransactionId: string;
  clientSecret: string; // for frontend confirmation
  status: "requires_payment_method" | "requires_confirmation" | "requires_action" | "succeeded";
}

// Stripe implementation
export class StripeProvider implements PaymentProvider {
  name = "stripe";
  constructor(private stripe: Stripe) {}

  async createPaymentIntent(input: CreatePaymentInput): Promise<PaymentIntent> {
    const intent = await this.stripe.paymentIntents.create({
      amount: input.amountCents,
      currency: input.currencyCode.toLowerCase(),
      metadata: { order_id: input.orderId, ...input.metadata },
    });
    return {
      gatewayTransactionId: intent.id,
      clientSecret: intent.client_secret!,
      status: intent.status as PaymentIntent["status"],
    };
  }
}
```

Checkout endpoint:
```
POST /api/v1/checkout
```

```typescript
async checkout(cartId: string, paymentMethod: string): Promise<{ order: Order; paymentIntent: PaymentIntent }> {
  return this.db.transaction(async (tx) => {
    const cart = await this.cartService.getCartForUpdate(tx, cartId);

    // Validate cart is ready for checkout
    if (cart.items.length === 0) throw new ValidationError("Cart is empty");
    if (!cart.shippingAddress) throw new ValidationError("Shipping address required");
    if (!cart.billingAddress) throw new ValidationError("Billing address required");

    // Final tax calculation
    const tax = await this.taxService.calculateTax(cart.items, cart.shippingAddress, channelConfig);

    // Create order from cart (snapshot all data)
    const order = await this.orderService.createFromCart(tx, cart, tax);

    // Create payment intent
    const paymentIntent = await this.paymentProvider.createPaymentIntent({
      amountCents: order.totalCents,
      currencyCode: order.currencyCode,
      orderId: order.id,
      customerEmail: cart.email ?? "",
    });

    // Record payment
    await tx.insert(payments).values({
      orderId: order.id,
      gateway: this.paymentProvider.name,
      status: "pending",
      method: paymentMethod,
      amountCents: order.totalCents,
      currencyCode: order.currencyCode,
      gatewayData: { transactionId: paymentIntent.gatewayTransactionId },
    });

    // Mark cart as converted
    await tx.update(carts).set({ status: "converted", updatedAt: new Date() }).where(eq(carts.id, cartId));

    return { order, paymentIntent };
  });
}
```

**Testing**:
- `Unit: checkout with empty cart -> ValidationError`
- `Unit: checkout without shipping address -> ValidationError`
- `Unit (mocked Stripe): checkout creates Stripe PaymentIntent with correct amount`
- `Unit: cart status changes to "converted" after checkout`
- `Unit: order contains snapshot of cart items, addresses, promotions`
- `Integration (mocked Stripe): full checkout flow -> order created, payment recorded`
- `Integration: payment webhook (payment_intent.succeeded) -> order status updated to "confirmed"`
- `Integration: payment webhook (payment_intent.payment_failed) -> order status "cancelled", inventory released`
- `Unit: payment data contains only gateway token, no raw card numbers (PCI DSS compliance)`

#### 5.4 — Order Lifecycle Management

**What**: Order CRUD, status transitions (pending -> confirmed -> processing -> fulfilled -> cancelled/refunded), fulfillment creation, and order event emission.

**Design**:

Order state machine:
```
pending ──► confirmed ──► processing ──► partially_fulfilled ──► fulfilled
   │            │              │                                      │
   └─► cancelled └─► cancelled └─► cancelled                   refunded
```

```typescript
// Valid transitions
const ORDER_TRANSITIONS: Record<string, string[]> = {
  pending: ["confirmed", "cancelled"],
  confirmed: ["processing", "cancelled"],
  processing: ["partially_fulfilled", "fulfilled", "cancelled"],
  partially_fulfilled: ["fulfilled", "cancelled"],
  fulfilled: ["refunded"],
};

async transitionOrder(orderId: string, newStatus: string, reason?: string) {
  const order = await this.getOrder(orderId);
  const allowed = ORDER_TRANSITIONS[order.status];
  if (!allowed?.includes(newStatus)) {
    throw new ConflictError(`Cannot transition from "${order.status}" to "${newStatus}"`);
  }

  await this.db.update(orders).set({
    status: newStatus,
    ...(newStatus === "cancelled" ? { cancelledAt: new Date() } : {}),
    updatedAt: new Date(),
  }).where(eq(orders.id, orderId));

  await this.eventBus.emit({
    type: `order.${newStatus}`,
    source: "/orders",
    subject: orderId,
    data: { orderId, fromStatus: order.status, toStatus: newStatus, reason },
  });
}
```

REST endpoints:
```
GET    /api/v1/orders                        # List orders (admin, paginated)
GET    /api/v1/orders/:id                    # Get order detail
PATCH  /api/v1/orders/:id/status             # Transition order status
POST   /api/v1/orders/:id/fulfillments       # Create fulfillment/shipment
PATCH  /api/v1/orders/:id/fulfillments/:fid  # Update fulfillment (tracking, status)
POST   /api/v1/orders/:id/refund             # Initiate refund
GET    /api/v1/orders/:id/events             # Get order event history
```

**Testing**:
- `Unit: transition pending -> confirmed -> succeeds`
- `Unit: transition pending -> fulfilled -> ConflictError (invalid transition)`
- `Unit: cancel order -> inventory reservations released`
- `Unit: create fulfillment with items -> fulfillmentStatus updated`
- `Integration: full order lifecycle: checkout -> confirm -> create fulfillment -> mark shipped -> deliver -> complete`
- `Integration: order.confirmed event emitted with correct CloudEvents envelope`
- `Integration: refund -> payment provider refund called, order status "refunded"`
- `E2E: create cart -> add items -> checkout -> confirm payment -> create fulfillment -> order fulfilled`

---

## Phase 6: Event System and Webhooks

### Purpose

Implement the CloudEvents-based domain event system, webhook subscription management, and reliable webhook delivery with retry logic. After this phase, external systems can subscribe to commerce events (order.placed, inventory.low_stock, payment.captured) and receive HMAC-signed webhook notifications.

### Tasks

#### 6.1 — Domain Event Bus

**What**: Internal event emitter that records events to the domain_events table in CloudEvents v1.0 format and dispatches to registered handlers.

**Design**:

```typescript
// packages/api/src/plugins/event-bus.ts
import { CloudEvent } from "cloudevents";

export interface DomainEvent {
  type: string;          // e.g. "order.placed"
  source: string;        // e.g. "/orders"
  subject: string;       // e.g. order ID
  data: Record<string, unknown>;
}

export class EventBus {
  private handlers = new Map<string, Array<(event: CloudEvent) => Promise<void>>>();

  constructor(private db: Db, private webhookDeliveryQueue: Queue) {}

  async emit(event: DomainEvent): Promise<void> {
    const cloudEvent = new CloudEvent({
      specversion: "1.0",
      type: event.type,
      source: event.source,
      subject: event.subject,
      time: new Date().toISOString(),
      datacontenttype: "application/json",
      data: event.data,
    });

    // Persist event
    await this.db.insert(domainEvents).values({
      type: cloudEvent.type,
      source: cloudEvent.source,
      subject: cloudEvent.subject ?? "",
      time: new Date(cloudEvent.time!),
      data: cloudEvent.data,
    });

    // Dispatch to in-process handlers
    const handlers = this.handlers.get(event.type) ?? [];
    await Promise.allSettled(handlers.map((h) => h(cloudEvent)));

    // Enqueue webhook deliveries
    await this.webhookDeliveryQueue.add("deliver", {
      eventType: event.type,
      payload: cloudEvent,
    });
  }

  on(eventType: string, handler: (event: CloudEvent) => Promise<void>): void {
    const existing = this.handlers.get(eventType) ?? [];
    existing.push(handler);
    this.handlers.set(eventType, existing);
  }
}
```

**Testing**:
- `Unit: emit persists event to domain_events table with CloudEvents fields`
- `Unit: emit calls all registered handlers for the event type`
- `Unit: handler failure does not prevent other handlers from executing`
- `Unit: emit enqueues webhook delivery job`
- `Integration: emit order.placed -> event queryable in domain_events table`

#### 6.2 — Webhook Subscription Management

**What**: CRUD for webhook subscriptions with HMAC-SHA256 secret generation and event type filtering.

**Design**:

REST endpoints:
```
POST   /api/v1/webhooks                      # Create subscription
GET    /api/v1/webhooks                      # List subscriptions
GET    /api/v1/webhooks/:id                  # Get subscription
PATCH  /api/v1/webhooks/:id                  # Update (URL, event types)
DELETE /api/v1/webhooks/:id                  # Remove subscription
POST   /api/v1/webhooks/:id/test             # Send test event
```

```typescript
export interface CreateWebhookInput {
  targetUrl: string;
  eventTypes: string[];  // ["order.placed", "order.cancelled", "inventory.low_stock"]
}

async createWebhook(apiClientId: string, input: CreateWebhookInput) {
  const secret = crypto.randomBytes(32).toString("hex");
  const secretHash = await argon2.hash(secret);

  const [webhook] = await this.db.insert(webhookSubscriptions).values({
    apiClientId,
    targetUrl: input.targetUrl,
    eventTypes: input.eventTypes,
    secretHash,
  }).returning();

  // Return secret only once at creation time
  return { ...webhook, secret };
}
```

**Testing**:
- `Unit: createWebhook generates 64-char hex secret`
- `Unit: secret returned only at creation, not on GET`
- `Integration: create webhook for "order.placed" -> webhook stored with event type filter`
- `Integration: list webhooks for API client -> returns only that client's webhooks`

#### 6.3 — Webhook Delivery Worker with Retries

**What**: BullMQ worker that delivers webhook payloads to subscriber URLs with HMAC-SHA256 signatures, retry with exponential backoff, and delivery logging.

**Design**:

```typescript
// packages/api/src/workers/webhook-delivery.ts
import { Worker } from "bullmq";
import crypto from "node:crypto";

// HMAC signature header: X-Commerce-Signature-256
function signPayload(payload: string, secret: string): string {
  return crypto.createHmac("sha256", secret).update(payload).digest("hex");
}

const worker = new Worker("webhook-delivery", async (job) => {
  const { eventType, payload } = job.data;

  // Find all active subscriptions matching this event type
  const subscriptions = await db.select().from(webhookSubscriptions)
    .where(and(
      eq(webhookSubscriptions.isActive, true),
      sql`${webhookSubscriptions.eventTypes} @> ARRAY[${eventType}]::text[]`
    ));

  for (const sub of subscriptions) {
    const body = JSON.stringify(payload);
    const signature = signPayload(body, sub.secretHash);

    const response = await fetch(sub.targetUrl, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "X-Commerce-Signature-256": `sha256=${signature}`,
        "X-Commerce-Event": eventType,
        "X-Commerce-Delivery-Id": job.id,
      },
      body,
      signal: AbortSignal.timeout(5000),
    });

    if (!response.ok) throw new Error(`Webhook delivery failed: ${response.status}`);
  }
}, {
  connection: redis,
  attempts: 5,
  backoff: { type: "exponential", delay: 1000 }, // 1s, 2s, 4s, 8s, 16s
});
```

**Testing**:
- `Unit: signPayload produces consistent HMAC-SHA256 hex string`
- `Unit: webhook delivery includes X-Commerce-Signature-256 header`
- `Integration (mocked HTTP): successful delivery -> job completed`
- `Integration (mocked HTTP): 500 response -> job retried with backoff`
- `Integration (mocked HTTP): 5 consecutive failures -> job moved to failed state`
- `Integration: timeout after 5000ms -> treated as failure, retried`

---

## Phase 7: Search and Product Discovery

### Purpose

Build full-text search and faceted filtering for the product catalog using PostgreSQL's native tsvector/GIN capabilities. After this phase, storefronts can search products by text, filter by attributes, sort by price or relevance, and present faceted navigation.

### Tasks

#### 7.1 — Full-Text Search Index

**What**: PostgreSQL tsvector-based full-text search with trigram matching for typo tolerance, combined with JSONB attribute filtering.

**Design**:

Add tsvector column and trigger:
```sql
ALTER TABLE products ADD COLUMN search_vector tsvector
  GENERATED ALWAYS AS (
    setweight(to_tsvector('english', coalesce(name, '')), 'A') ||
    setweight(to_tsvector('english', coalesce(description, '')), 'B')
  ) STORED;

CREATE INDEX idx_products_search ON products USING gin(search_vector);
```

Search service:
```typescript
async searchProducts(query: SearchQuery): Promise<SearchResult> {
  const conditions: SQL[] = [eq(products.status, "active")];

  if (query.text) {
    conditions.push(sql`${products.searchVector} @@ websearch_to_tsquery('english', ${query.text})`);
  }
  if (query.categoryId) {
    conditions.push(sql`${products.categoryIds} @> ARRAY[${query.categoryId}]::uuid[]`);
  }
  if (query.attributes) {
    conditions.push(sql`${products.attributes} @> ${JSON.stringify(query.attributes)}::jsonb`);
  }
  if (query.minPriceCents !== undefined || query.maxPriceCents !== undefined) {
    // Filter via variant prices JSONB
  }
  if (query.channelId) {
    conditions.push(sql`${products.channelConfig} ? ${query.channelId}`);
  }

  const orderBy = query.text
    ? sql`ts_rank(${products.searchVector}, websearch_to_tsquery('english', ${query.text})) DESC`
    : sql`${products.createdAt} DESC`;

  const items = await this.db.select().from(products)
    .where(and(...conditions))
    .orderBy(orderBy)
    .limit(query.limit ?? 20)
    .offset(query.offset ?? 0);

  // Compute facets
  const facets = await this.computeFacets(conditions, query);

  return { items, facets, total: items.length };
}
```

REST and GraphQL:
```
GET /api/v1/search/products?q=headphones&category=electronics&color=black&minPrice=2000&maxPrice=10000&sort=price_asc&limit=20&offset=0
```

**Testing**:
- `Unit: search "wireless headphone" matches "Wireless Headphones Pro"`
- `Unit: search with attribute filter {"color":"black"} -> only black products`
- `Unit: search with categoryId filter -> products in that category only`
- `Unit: empty search text -> returns all active products`
- `Integration: create 50 products -> search with pagination -> correct pages`
- `Integration: search ranking -> product with query in name ranks higher than description-only match`
- `Integration: facet computation -> returns available attribute values with counts`

#### 7.2 — Faceted Navigation

**What**: Compute available filter facets (attribute values, price ranges, categories) dynamically from the current search result set.

**Design**:

```typescript
export interface Facet {
  field: string;
  values: FacetValue[];
}

export interface FacetValue {
  value: string;
  count: number;
}

async computeFacets(baseConditions: SQL[], query: SearchQuery): Promise<Facet[]> {
  // For each filterable attribute defined in product types,
  // count distinct values across matching products
  const facets: Facet[] = [];

  // Category facet
  const categoryFacet = await this.db.execute(sql`
    SELECT unnest(category_ids) AS category_id, COUNT(*) AS count
    FROM products
    WHERE ${and(...baseConditions)}
    GROUP BY category_id
    ORDER BY count DESC
  `);

  // Attribute facets (from JSONB)
  for (const attr of filterableAttributes) {
    const attrFacet = await this.db.execute(sql`
      SELECT attributes->>${attr} AS value, COUNT(*) AS count
      FROM products
      WHERE ${and(...baseConditions)} AND attributes ? ${attr}
      GROUP BY value
      ORDER BY count DESC
      LIMIT 50
    `);
    facets.push({ field: attr, values: attrFacet.rows });
  }

  return facets;
}
```

**Testing**:
- `Unit: facets include correct counts per attribute value`
- `Unit: facets exclude values with zero matching products`
- `Integration: 10 red, 5 blue products -> color facet shows red:10, blue:5`
- `Integration: applying a filter narrows subsequent facet counts`

---

## Phase 8: AI-Native Features

### Purpose

Implement the AI differentiators that set this engine apart from existing solutions: vector-based semantic search, LLM-ranked product recommendations, automated catalog enrichment from unstructured data, and conversational commerce queries. After this phase, the platform delivers personalised, AI-driven shopping experiences through the same API surface.

### Tasks

#### 8.1 — Product Embedding Generation

**What**: Generate vector embeddings for products using OpenAI's text-embedding-3-small model (1536 dimensions), stored in pgvector for similarity search.

**Design**:

```typescript
// packages/api/src/modules/ai/embedding.service.ts
import OpenAI from "openai";

export class EmbeddingService {
  private openai: OpenAI;

  constructor(config: Config) {
    this.openai = new OpenAI({ apiKey: config.OPENAI_API_KEY });
  }

  async generateProductEmbedding(product: Product, variant?: ProductVariant): Promise<number[]> {
    const text = this.buildEmbeddingText(product, variant);
    const response = await this.openai.embeddings.create({
      model: "text-embedding-3-small",
      input: text,
      dimensions: 1536,
    });
    return response.data[0].embedding;
  }

  private buildEmbeddingText(product: Product, variant?: ProductVariant): string {
    const parts = [
      product.name,
      product.description ?? "",
      ...Object.entries(product.attributes).map(([k, v]) => `${k}: ${v}`),
    ];
    if (variant) {
      parts.push(`SKU: ${variant.sku}`);
      parts.push(...Object.entries(variant.attributes).map(([k, v]) => `${k}: ${v}`));
    }
    return parts.join(" | ");
  }

  async storeEmbedding(productId: string, variantId: string | null, embedding: number[]): Promise<void> {
    await this.db.insert(productEmbeddings).values({
      productId,
      variantId,
      modelVersion: "text-embedding-3-small",
      embedding: sql`${JSON.stringify(embedding)}::vector`,
      aiMetadata: {},
    }).onConflictDoUpdate({
      target: [productEmbeddings.productId, productEmbeddings.variantId, productEmbeddings.modelVersion],
      set: { embedding: sql`${JSON.stringify(embedding)}::vector`, createdAt: new Date() },
    });
  }
}
```

BullMQ worker for batch embedding generation:
```typescript
// packages/api/src/workers/embedding-generator.ts
// Triggered by product.created and product.updated events
// Processes in batches of 100 to optimise API calls
```

**Testing**:
- `Unit: buildEmbeddingText includes product name, description, and attributes`
- `Unit (mocked OpenAI): generateProductEmbedding returns 1536-dimension array`
- `Integration (mocked OpenAI): store embedding -> queryable via pgvector cosine similarity`
- `Integration: product.created event -> embedding generation job enqueued`
- `Unit: batch processing of 100 products -> correct number of API calls`

#### 8.2 — Semantic Search API

**What**: Vector similarity search endpoint that accepts natural language queries, generates a query embedding, and returns semantically similar products ranked by cosine similarity.

**Design**:

```typescript
async semanticSearch(query: string, limit = 20): Promise<SemanticSearchResult[]> {
  const queryEmbedding = await this.embeddingService.generateQueryEmbedding(query);

  const results = await this.db.execute(sql`
    SELECT
      pe.product_id,
      pe.variant_id,
      1 - (pe.embedding <=> ${JSON.stringify(queryEmbedding)}::vector) AS similarity,
      p.name,
      p.slug,
      p.description,
      p.attributes,
      p.media
    FROM product_embeddings pe
    JOIN products p ON p.id = pe.product_id
    WHERE p.status = 'active'
    ORDER BY pe.embedding <=> ${JSON.stringify(queryEmbedding)}::vector
    LIMIT ${limit}
  `);

  return results.rows.map((r) => ({
    product: { id: r.product_id, name: r.name, slug: r.slug },
    similarity: r.similarity,
  }));
}
```

REST endpoint:
```
GET /api/v1/search/semantic?q=comfortable+over-ear+headphones+for+working+from+home&limit=10
```

GraphQL:
```graphql
type Query {
  semanticSearch(query: String!, limit: Int): [SemanticSearchResult!]!
}

type SemanticSearchResult {
  product: Product!
  similarity: Float!
}
```

**Testing**:
- `Unit (mocked OpenAI): semantic search for "noise cancelling" returns headphone products`
- `Unit: results sorted by similarity descending`
- `Unit: only active products returned`
- `Integration (mocked OpenAI): insert 3 product embeddings -> query returns them ranked by similarity`
- `Integration: semantic search respects limit parameter`

#### 8.3 — Automated Catalog Enrichment

**What**: LLM-powered attribute extraction and description generation from raw supplier data (unstructured text, CSV data sheets).

**Design**:

```typescript
export interface EnrichmentInput {
  productId: string;
  rawText: string;  // supplier data sheet text
}

export interface EnrichmentResult {
  generatedDescription: string;
  extractedAttributes: Record<string, unknown>;
  suggestedCategories: string[];
  confidenceScore: number;
}

async enrichProduct(input: EnrichmentInput): Promise<EnrichmentResult> {
  const product = await this.catalogService.getProduct(input.productId);
  const productType = await this.catalogService.getProductType(product.productTypeId);

  const prompt = `You are a product catalog enrichment assistant. Given the following raw supplier data,
extract structured product attributes according to the schema below, generate a compelling product description,
and suggest relevant categories.

Product Type: ${productType.name}
Attribute Schema: ${JSON.stringify(productType.attributeSchema)}

Raw Supplier Data:
${input.rawText}

Respond in JSON format:
{
  "description": "...",
  "attributes": { ... },
  "suggestedCategories": ["...", "..."],
  "confidence": 0.0-1.0
}`;

  const response = await this.openai.chat.completions.create({
    model: "gpt-4o-mini",
    messages: [{ role: "user", content: prompt }],
    response_format: { type: "json_object" },
    temperature: 0.2,
  });

  const result = JSON.parse(response.choices[0].message.content!);
  return result;
}
```

REST endpoint:
```
POST /api/v1/ai/enrich
```

**Testing**:
- `Unit (mocked LLM): enrichment with clothing data sheet -> extracts color, size, material`
- `Unit (mocked LLM): generated description is non-empty and under 1000 chars`
- `Unit: extracted attributes validate against product type schema`
- `Unit: confidence score between 0 and 1`
- `Integration (mocked LLM): enrich product -> attributes saved to product record`

#### 8.4 — Conversational Commerce API

**What**: Natural-language query endpoint that lets chatbots and voice interfaces search products, check inventory, and initiate checkout through conversation.

**Design**:

```typescript
export interface ConversationMessage {
  role: "user" | "assistant" | "system";
  content: string;
}

export interface ConversationResponse {
  message: string;
  actions: ConversationAction[];
  products?: Product[];
}

export interface ConversationAction {
  type: "search" | "add_to_cart" | "check_inventory" | "view_product";
  data: Record<string, unknown>;
}

async processConversation(
  messages: ConversationMessage[],
  sessionContext: { cartId?: string; channelId: string; customerId?: string }
): Promise<ConversationResponse> {
  const systemPrompt = `You are a commerce assistant for an online store.
You can help customers search for products, check availability, and add items to their cart.
When the user asks about products, search the catalog.
When they want to buy something, add it to their cart.
Respond with structured actions when appropriate.

Available tools:
- search_products(query: string, filters: object)
- check_inventory(variant_id: string)
- add_to_cart(cart_id: string, variant_id: string, quantity: number)
- get_product_details(product_id: string)`;

  // Use function calling to map conversation to commerce actions
  const response = await this.openai.chat.completions.create({
    model: "gpt-4o-mini",
    messages: [{ role: "system", content: systemPrompt }, ...messages],
    tools: this.getToolDefinitions(),
    temperature: 0.3,
  });

  // Execute tool calls against commerce services
  const actions = await this.executeToolCalls(response, sessionContext);

  return {
    message: response.choices[0].message.content ?? "",
    actions,
    products: actions.filter((a) => a.type === "search").flatMap((a) => a.data.results as Product[]),
  };
}
```

REST endpoint:
```
POST /api/v1/ai/conversation
```

**Testing**:
- `Unit (mocked LLM): "Show me black headphones" -> search action with color filter`
- `Unit (mocked LLM): "Add the first one to my cart" -> add_to_cart action`
- `Unit (mocked LLM): "Is the XL in stock?" -> check_inventory action`
- `Integration (mocked LLM): multi-turn conversation maintains context`
- `Unit: conversation without sessionContext.cartId -> creates new cart`

---

## Phase 9: Shipping and Tax Configuration

### Purpose

Complete the commerce operations layer with configurable shipping methods, rate calculation, and robust tax configuration management. After this phase, the platform handles multi-zone shipping with weight-based and order-total-based rates, and supports both VAT-inclusive and sales-tax-exclusive pricing models.

### Tasks

#### 9.1 — Shipping Method and Rate Management

**What**: CRUD for shipping methods with JSONB rate rules supporting zone-based, weight-based, and free-shipping-threshold rates.

**Design**:

```typescript
export interface ShippingMethod {
  id: string;
  name: string;
  carrier?: string;
  description?: string;
  isActive: boolean;
  rateRules: ShippingRateRules;
  channelIds: string[];
  createdAt: string;
  updatedAt: string;
}

export interface ShippingRateRules {
  rates: ShippingRate[];
}

export interface ShippingRate {
  zone: string;           // country code or region name
  minOrderCents: number;
  maxOrderCents?: number;
  rateCents: number;
  currency: string;
  maxWeightGrams?: number;
}

async calculateShipping(
  items: CartItem[],
  shippingAddress: Address,
  channelId: string
): Promise<ShippingOption[]> {
  const methods = await this.db.select().from(shippingMethods)
    .where(and(
      eq(shippingMethods.isActive, true),
      sql`${shippingMethods.channelIds} @> ARRAY[${channelId}]::uuid[]`
    ));

  const orderTotal = items.reduce((sum, i) => sum + i.totalCents, 0);
  const totalWeight = items.reduce((sum, i) => sum + (i.weightGrams ?? 0) * i.quantity, 0);

  return methods
    .map((method) => {
      const matchingRate = this.findMatchingRate(method.rateRules, shippingAddress, orderTotal, totalWeight);
      if (!matchingRate) return null;
      return { methodId: method.id, name: method.name, carrier: method.carrier, rateCents: matchingRate.rateCents, currency: matchingRate.currency };
    })
    .filter(Boolean) as ShippingOption[];
}
```

REST endpoints:
```
POST   /api/v1/shipping-methods
GET    /api/v1/shipping-methods
PATCH  /api/v1/shipping-methods/:id
DELETE /api/v1/shipping-methods/:id
POST   /api/v1/shipping/calculate         # Calculate available options for an address + items
```

**Testing**:
- `Unit: US order $50 with free shipping threshold $75 -> standard rate applied`
- `Unit: US order $80 with free shipping threshold $75 -> $0 shipping`
- `Unit: EU address with no EU rates -> method not returned`
- `Unit: weight exceeding maxWeightGrams -> method excluded`
- `Integration: create 3 shipping methods -> calculate for address -> correct subset returned`

#### 9.2 — Tax Configuration Management

**What**: CRUD for tax configurations with JSONB rules supporting multi-zone, multi-rate, compound, and tax-inclusive pricing scenarios.

**Design**:

REST endpoints:
```
POST   /api/v1/tax-configurations
GET    /api/v1/tax-configurations
GET    /api/v1/tax-configurations/:id
PATCH  /api/v1/tax-configurations/:id
DELETE /api/v1/tax-configurations/:id
POST   /api/v1/tax/calculate              # Calculate tax for address + items
```

**Testing**:
- `Unit: EU VAT inclusive pricing -> tax extracted from existing price, not added`
- `Unit: US sales tax -> tax added on top of item prices`
- `Unit: compound tax (provincial + federal) calculated correctly`
- `Integration: create tax config -> calculate for matching address -> correct breakdown`
- `Unit: no matching zone -> zero tax returned`

---

## Phase 10: Admin Dashboard

### Purpose

Build the React-based admin dashboard for operational tasks: managing products, viewing orders, monitoring inventory, and handling customer service. After this phase, non-technical staff can operate the commerce platform through a web UI.

### Tasks

#### 10.1 — Dashboard Shell and Authentication

**What**: React app with authentication flow, navigation, and layout components.

**Design**:

```typescript
// packages/admin/src/main.tsx
import { createRouter, RouterProvider } from "@tanstack/react-router";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { AuthProvider } from "./auth/AuthProvider";

const queryClient = new QueryClient();

const router = createRouter({
  routeTree: rootRoute,
  context: { queryClient },
});

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <AuthProvider>
        <RouterProvider router={router} />
      </AuthProvider>
    </QueryClientProvider>
  );
}
```

Navigation structure:
```
/dashboard              # Overview: recent orders, inventory alerts, revenue
/products               # Product list with search and filters
/products/:id           # Product detail/edit
/products/new           # Create product
/orders                 # Order list with status filters
/orders/:id             # Order detail with fulfillment actions
/customers              # Customer list
/customers/:id          # Customer detail with order history
/inventory              # Inventory levels across locations
/promotions             # Promotion management
/settings               # Channels, tax, shipping, webhooks, API clients
```

**Testing**:
- `Unit: unauthenticated user redirected to login`
- `Unit: navigation renders correct routes`
- `Integration: login -> dashboard renders with data from API`
- `E2E: login -> navigate to products -> see product list`

#### 10.2 — Product Management UI

**What**: Product list with search, product detail form with variant management, media upload, and attribute editing.

**Design**:

Product list page features:
- Search bar with real-time filtering
- Status filter (draft/active/archived)
- Category filter (tree selector)
- Bulk actions (archive, change status)
- Pagination

Product edit form:
- Dynamic attribute fields generated from product type's JSON Schema
- Inline variant management (add/edit/remove)
- Media upload with drag-and-drop reordering
- Price management per variant
- Channel visibility toggles

**Testing**:
- `E2E: create product type -> create product with 3 variants -> all displayed in UI`
- `E2E: edit product name -> save -> list shows updated name`
- `E2E: search "headphones" -> filtered results shown`
- `Unit: dynamic form fields match product type attribute schema`

#### 10.3 — Order Management UI

**What**: Order list with status filtering, order detail with timeline, fulfillment creation, and refund initiation.

**Design**:

Order detail page:
- Order status badge with state machine transitions
- Item list with product thumbnails
- Shipping and billing address display
- Payment status and gateway details
- Fulfillment creation form (select items, enter tracking)
- Order event timeline (from domain_events)
- Refund button with amount input

**Testing**:
- `E2E: view order -> create fulfillment with tracking number -> order status updates`
- `E2E: filter orders by status "pending" -> only pending orders shown`
- `E2E: order timeline shows events in chronological order`

---

## Phase 11: Client SDK and Developer Experience

### Purpose

Build the TypeScript client SDK and comprehensive developer documentation. After this phase, frontend developers can integrate with the commerce engine using a type-safe SDK, and the API is fully documented with OpenAPI and AsyncAPI specifications.

### Tasks

#### 11.1 — TypeScript Client SDK

**What**: Auto-generated TypeScript SDK from the OpenAPI spec, with resource-specific convenience methods.

**Design**:

```typescript
// packages/sdk/src/client.ts
export class CommerceClient {
  private baseUrl: string;
  private accessToken?: string;

  constructor(options: { baseUrl: string; clientId: string; clientSecret: string }) {
    this.baseUrl = options.baseUrl;
  }

  async authenticate(): Promise<void> {
    const response = await this.request("POST", "/oauth/token", {
      grant_type: "client_credentials",
      client_id: this.options.clientId,
      client_secret: this.options.clientSecret,
    });
    this.accessToken = response.access_token;
  }

  products = {
    list: (params?: ListParams) => this.request<PaginatedResponse<Product>>("GET", "/api/v1/products", params),
    get: (id: string) => this.request<Product>("GET", `/api/v1/products/${id}`),
    create: (input: CreateProductInput) => this.request<Product>("POST", "/api/v1/products", input),
    update: (id: string, input: UpdateProductInput) => this.request<Product>("PATCH", `/api/v1/products/${id}`, input),
    search: (query: SearchParams) => this.request<SearchResult>("GET", "/api/v1/search/products", query),
    semanticSearch: (query: string) => this.request<SemanticSearchResult[]>("GET", "/api/v1/search/semantic", { q: query }),
  };

  cart = {
    create: (channelId: string, currency: string) => this.request<Cart>("POST", "/api/v1/carts", { channelId, currencyCode: currency }),
    get: (id: string) => this.request<Cart>("GET", `/api/v1/carts/${id}`),
    addItem: (cartId: string, variantId: string, quantity: number) => this.request<Cart>("POST", `/api/v1/carts/${cartId}/items`, { variantId, quantity }),
    checkout: (cartId: string, paymentMethod: string) => this.request("POST", "/api/v1/checkout", { cartId, paymentMethod }),
  };

  orders = {
    list: (params?: ListParams) => this.request<PaginatedResponse<Order>>("GET", "/api/v1/orders", params),
    get: (id: string) => this.request<Order>("GET", `/api/v1/orders/${id}`),
  };
}
```

**Testing**:
- `Unit: SDK authenticate() stores access token`
- `Unit: SDK includes Authorization header on subsequent requests`
- `Unit: SDK types match API response shapes`
- `Integration: SDK products.list() -> returns paginated product list from API`
- `Integration: SDK cart.create() -> cart.addItem() -> cart.checkout() flow`

#### 11.2 — AsyncAPI Specification for Events

**What**: AsyncAPI 3.1 document describing all webhook event channels, payload schemas, and security (HMAC signatures).

**Design**:

```yaml
# asyncapi.yaml
asyncapi: 3.1.0
info:
  title: Headless Commerce Engine Events
  version: 0.1.0
channels:
  orderEvents:
    address: /webhooks
    messages:
      orderPlaced:
        payload:
          $ref: "#/components/schemas/OrderPlacedEvent"
      orderConfirmed:
        payload:
          $ref: "#/components/schemas/OrderConfirmedEvent"
  inventoryEvents:
    address: /webhooks
    messages:
      inventoryLowStock:
        payload:
          $ref: "#/components/schemas/InventoryLowStockEvent"
components:
  schemas:
    OrderPlacedEvent:
      type: object
      properties:
        specversion: { type: string, const: "1.0" }
        type: { type: string, const: "order.placed" }
        source: { type: string }
        subject: { type: string }
        time: { type: string, format: date-time }
        data:
          $ref: "#/components/schemas/OrderPlacedData"
```

**Testing**:
- `Unit: AsyncAPI document validates against AsyncAPI 3.1 specification`
- `Unit: all event types documented in AsyncAPI match actual emitted events`

---

## Phase 12: Production Readiness and Performance

### Purpose

Harden the platform for production deployment: rate limiting, security headers, database connection pooling, monitoring, and load testing. After this phase, the engine is deployable to production environments with confidence.

### Tasks

#### 12.1 — Rate Limiting and Security Headers

**What**: Redis-backed rate limiting per API client, OWASP-recommended security headers, and request validation hardening.

**Design**:

```typescript
// Rate limiting configuration
const RATE_LIMITS = {
  "storefront": { windowMs: 60_000, max: 100 },   // 100 req/min for storefront clients
  "admin": { windowMs: 60_000, max: 300 },         // 300 req/min for admin clients
  "oauth": { windowMs: 60_000, max: 20 },          // 20 req/min for token endpoint
};

// Security headers (Helmet-compatible)
const SECURITY_HEADERS = {
  "X-Content-Type-Options": "nosniff",
  "X-Frame-Options": "DENY",
  "X-XSS-Protection": "0",
  "Strict-Transport-Security": "max-age=31536000; includeSubDomains",
  "Content-Security-Policy": "default-src 'none'; frame-ancestors 'none'",
  "Referrer-Policy": "strict-origin-when-cross-origin",
};
```

**Testing**:
- `Integration: exceed rate limit -> 429 with Retry-After header`
- `Integration: all responses include security headers`
- `Unit: rate limit resets after window expires`
- `Unit: different rate limits for different client types`

#### 12.2 — Health Monitoring and Metrics

**What**: Prometheus-compatible metrics endpoint, structured logging, and readiness/liveness probes for Kubernetes deployment.

**Design**:

Metrics:
```
commerce_http_requests_total{method, path, status}
commerce_http_request_duration_seconds{method, path}
commerce_orders_total{channel, status}
commerce_inventory_low_stock_total{location}
commerce_payment_transactions_total{gateway, status}
commerce_webhook_deliveries_total{event_type, status}
```

Endpoints:
```
GET /health          # Liveness probe (already exists)
GET /ready           # Readiness probe (DB + Redis + all workers healthy)
GET /metrics         # Prometheus metrics
```

**Testing**:
- `Integration: GET /metrics returns Prometheus text format`
- `Integration: GET /ready returns 503 when DB is down`
- `Integration: HTTP request increments request counter metric`
- `Unit: order placement increments commerce_orders_total`

#### 12.3 — Database Optimisation and Connection Pooling

**What**: Query performance tuning, index verification, connection pool configuration, and slow query logging.

**Design**:

Connection pool settings:
```typescript
const pool = new pg.Pool({
  connectionString: config.DATABASE_URL,
  min: config.DB_POOL_MIN,        // 2 in dev, 5 in prod
  max: config.DB_POOL_MAX,        // 10 in dev, 25 in prod
  idleTimeoutMillis: 30_000,
  connectionTimeoutMillis: 5_000,
  statement_timeout: 10_000,       // 10s max query time
});
```

Key query optimisations:
- Products search uses composite GIN index on (status, attributes)
- Order listing uses partial index on (status) WHERE status NOT IN ('fulfilled', 'cancelled')
- Inventory queries use FOR UPDATE SKIP LOCKED for concurrent access
- Cart expiry uses partial index on (expires_at) WHERE status = 'active'

**Testing**:
- `Integration: product search with 10,000 products completes under 100ms`
- `Integration: concurrent cart operations with connection pool -> no pool exhaustion`
- `Integration: slow query (>10s) is killed and returns timeout error`
- `Unit: EXPLAIN ANALYZE for critical queries shows index scans, not sequential scans`

#### 12.4 — CI/CD Pipeline

**What**: GitHub Actions workflow for continuous integration (lint, type-check, test, build) and deployment (Docker image push, SDK publish).

**Design**:

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  lint-and-typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: "22" }
      - run: pnpm install --frozen-lockfile
      - run: pnpm biome check .
      - run: pnpm tsc --noEmit

  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: pgvector/pgvector:pg16
        env:
          POSTGRES_DB: commerce_test
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports: ["5432:5432"]
      redis:
        image: redis:7-alpine
        ports: ["6379:6379"]
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: "22" }
      - run: pnpm install --frozen-lockfile
      - run: pnpm --filter @commerce/api test
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/commerce_test
          REDIS_URL: redis://localhost:6379
          JWT_SECRET: test-secret-minimum-32-characters-long

  build:
    runs-on: ubuntu-latest
    needs: [lint-and-typecheck, test]
    steps:
      - uses: actions/checkout@v4
      - uses: docker/build-push-action@v6
        with:
          push: false
          tags: commerce-engine:latest
```

**Testing**:
- `Unit: CI pipeline passes on clean repository`
- `Unit: CI fails if Biome reports errors`
- `Unit: CI fails if any test fails`
- `Integration: Docker build produces runnable image`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation                    ─── required by everything
    │
    ├── Phase 2: Catalog API           ─── requires Phase 1
    │       │
    │       ├── Phase 3: Pricing/Inventory/Channels ─── requires Phase 2
    │       │       │
    │       │       ├── Phase 4: Customers & Auth    ─── requires Phase 1, can parallel with Phase 3
    │       │       │       │
    │       │       │       └── Phase 5: Cart/Checkout/Orders ─── requires Phases 3 + 4
    │       │       │               │
    │       │       │               ├── Phase 6: Events & Webhooks ─── requires Phase 5
    │       │       │               │
    │       │       │               └── Phase 9: Shipping & Tax   ─── can parallel with Phase 6
    │       │       │
    │       │       └── Phase 7: Search & Discovery  ─── requires Phase 2, can parallel with Phases 3-5
    │       │               │
    │       │               └── Phase 8: AI Features ─── requires Phase 7
    │       │
    │       └── Phase 10: Admin Dashboard ─── requires Phases 5 + 6 (needs data to display)
    │
    └── Phase 11: SDK & Docs           ─── requires Phases 5 + 6 (needs stable API surface)
         │
         └── Phase 12: Production Readiness ─── requires all prior phases

Parallelism opportunities:
  - Phases 3 and 4 can be developed concurrently after Phase 2
  - Phase 7 can be developed concurrently with Phases 3-5 (only needs catalog)
  - Phases 6 and 9 can be developed concurrently after Phase 5
  - Phases 10 and 11 can be developed concurrently once the API is stable
```

---

## Definition of Done (per phase)

1. All tasks in the phase are implemented with complete functionality.
2. All unit tests pass (`pnpm --filter @commerce/api test`).
3. All integration tests pass (with Testcontainers or docker-compose services).
4. Biome linting and formatting pass (`pnpm biome check .`).
5. TypeScript strict-mode type checking passes (`pnpm tsc --noEmit`).
6. Docker build succeeds (`docker build .`).
7. All new REST endpoints appear in the auto-generated OpenAPI 3.1 specification.
8. All new GraphQL types and resolvers are introspectable.
9. Database migrations generated and tested (apply and rollback).
10. All new environment variables documented in `config.ts` with defaults.
11. CloudEvents emitted for all state-changing operations (create, update, delete, transition).
12. No raw card data or secrets stored in the database (PCI DSS v4.0.1 compliance).
13. ISO standards applied: ISO 4217 for currencies, ISO 3166-1/2 for countries, ISO 8601 for timestamps.
14. Error responses follow the standardised `ApiError` format with request IDs.
