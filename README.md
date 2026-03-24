# NoverFly API Documentation

> **Official developer documentation for the [NoverFly](https://noverfly.com) platform API.**
> Build websites, sell products, manage content, use a cloud database (BaaS), and integrate AI — all through one REST API.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Website](https://img.shields.io/badge/Website-noverfly.com-brightgreen)](https://noverfly.com)
[![API Base URL](https://img.shields.io/badge/API-api.noverfly.com%2Fv1-orange)](https://api.noverfly.com/v1)
[![OpenAPI](https://img.shields.io/badge/OpenAPI-3.0-6BA539?logo=openapi-initiative)](openapi.yaml)
[![Help Center](https://img.shields.io/badge/Help-help.noverfly.com-lightblue)](https://help.noverfly.com)

---

## What is NoverFly?

**NoverFly** is an all-in-one **SaaS website builder**, **e-commerce platform**, and **Backend as a Service (BaaS)** — like Webflow + Shopify + Firebase combined. NoverFly provides a REST API for developers to programmatically manage every aspect of the platform:

- **Website Builder API** — Create and manage sites, pages, and designs
- **CMS API** — Collections, entries, dynamic content
- **Database API (BaaS)** — Cloud PostgreSQL, tables, queries, real-time data, row-level security
- **E-Commerce API** — Products, cart, checkout, orders, payments (Stripe & PayPal)
- **Authentication API** — JWT, OAuth 2.0 (Google), API keys, multi-tenant
- **AI API** — Content generation, design assistance, image generation
- **Analytics API** — Page views, visitors, conversions
- **Asset API** — File uploads, media management (S3 storage)
- **Deployment API** — Publish sites, custom domains, SSL

**API Base URL:** `https://api.noverfly.com/v1`

NoverFly is built for:

- **Creators & freelancers** — Build client sites fast with GlowDesign visual editor
- **Businesses** — Launch online stores with built-in e-commerce and payments
- **Developers** — Integrate via REST API, webhooks, and API keys
- **Agencies** — Manage multiple client projects with multi-tenant organizations
- **SaaS builders** — White-label websites and storefronts

### Keywords

`NoverFly` · `NoverFly API` · `website builder API` · `SaaS platform` · `e-commerce API` · `BaaS` · `Backend as a Service` · `cloud database` · `GlowDesign` · `CMS API` · `Gloowflix Cloud` · `drag-and-drop website builder` · `headless CMS` · `REST API` · `multi-tenant SaaS` · `PostgreSQL` · `database API`

---

## Quick Start — API Usage

### 1. Authenticate

```bash
curl -X POST https://api.noverfly.com/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "you@example.com", "password": "your-password"}'
```

### 2. Get Your Profile

```bash
curl https://api.noverfly.com/v1/auth/me \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

### 3. List Your Sites

```bash
curl https://api.noverfly.com/v1/sites \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "X-Tenant-Id: YOUR_TENANT_ID"
```

### 4. Create a CMS Entry

```bash
curl -X POST https://api.noverfly.com/v1/cms/blog-posts/entries \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "X-Tenant-Id: YOUR_TENANT_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "fields": {
      "title": "Hello World",
      "slug": "hello-world",
      "body": "<p>My first post on NoverFly!</p>",
      "published": true
    }
  }'
```

### 5. List Products (E-Commerce)

```bash
curl https://api.noverfly.com/v1/ecommerce/products \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "X-Tenant-Id: YOUR_TENANT_ID"
```

> Full interactive examples in the [API Reference](docs/api.md).

---

## Documentation Index

| Document | Description |
|----------|-------------|
| [Introduction](docs/introduction.md) | Platform overview — what NoverFly is and how it works |
| [Getting Started](docs/getting-started.md) | Step-by-step tutorial: sign up → create site → publish |
| [Authentication](docs/authentication.md) | JWT tokens, Google OAuth, API keys, roles, multi-tenant auth |
| [GlowDesign Editor](docs/glowdesign.md) | Visual drag-and-drop editor, components, responsive design |
| [CMS & Database](docs/cms.md) | Collections, custom fields, entries, CMS API, webhooks |
| [Database & BaaS](docs/database.md) | Cloud PostgreSQL, tables, queries, real-time subscriptions, row-level security |
| [E-Commerce](docs/ecommerce.md) | Products, variants, cart, checkout, orders, Stripe/PayPal |
| [**API Reference**](docs/api.md) | **All REST API endpoints, rate limits, errors, pagination** |
| [Deployment](docs/deployment.md) | Publishing, custom domains, SSL, CI/CD, rollbacks |
| [Security](docs/security.md) | Encryption, GDPR, data isolation, responsible disclosure |
| [OpenAPI Spec](openapi.yaml) | Machine-readable OpenAPI 3.0 specification |

---

## API Endpoints Overview

Base URL: `https://api.noverfly.com/v1`

| Category | Endpoints | Description |
|----------|-----------|-------------|
| **Auth** | `POST /auth/login` · `POST /auth/register` · `GET /auth/me` · `POST /auth/refresh` | Authentication, registration, JWT tokens |
| **Organizations** | `GET /organizations` · `POST /organizations` · `POST /organizations/:id/members` | Multi-tenant team management |
| **Sites** | `GET /sites` · `POST /sites` · `POST /sites/:id/publish` | Website creation and publishing |
| **CMS** | `GET /cms/:collection/entries` · `POST /cms/:collection/entries` | Dynamic content management |
| **Database (BaaS)** | `GET /database/tables` · `POST /database/query` · `POST /database/tables/:table/rows` | Cloud database, queries, real-time |
| **E-Commerce** | `GET /ecommerce/products` · `POST /ecommerce/checkout` · `GET /ecommerce/orders` | Products, checkout, orders |
| **Assets** | `POST /assets/upload` · `GET /assets` | File uploads and media |
| **Analytics** | `GET /analytics/overview` · `GET /analytics/pageviews` | Traffic and performance |
| **API Keys** | `POST /api-keys` · `GET /api-keys` | Programmatic access tokens |

> See the complete [API Reference](docs/api.md) for request/response examples, error codes, and rate limits.

---

## Authentication

NoverFly uses **JWT authentication** and supports:

- Email + password registration
- Google OAuth 2.0
- Session management with refresh tokens
- Multi-tenant organization switching

---

## Pricing & Quotas

| Plan | Price | Quota | Features |
|------|-------|-------|----------|
| **Free** | $0/mo | 1 site, 100 pages, 500 MB storage | GlowDesign Lite, basic CMS, 1 database (1,000 rows) |
| **Pro** | $19/mo | 5 sites, unlimited pages, 10 GB storage | Custom domains, e-commerce, AI tools, 5 databases (100K rows) |
| **Business** | $49/mo | 20 sites, unlimited pages, 50 GB storage | Priority support, analytics, team collab, unlimited databases |
| **Enterprise** | Custom | Unlimited | Dedicated infrastructure, SLA, custom integrations |

> Visit [noverfly.com/pricing](https://noverfly.com/pricing) for detailed plan comparison.

---

## Rate Limits

| Plan | API Requests/min | API Requests/day | Bandwidth/month |
|------|-----------------|-----------------|----------------|
| Free | 60 | 1,000 | 1 GB |
| Pro | 300 | 50,000 | 100 GB |
| Business | 1,000 | 500,000 | 1 TB |
| Enterprise | Custom | Custom | Unlimited |

Rate limit headers are included in every API response:
```
X-RateLimit-Limit: 300
X-RateLimit-Remaining: 295
X-RateLimit-Reset: 1705312800
```

---

## Platform Architecture

```
┌─────────────────────────────────────────────────────┐
│                   Cloudflare CDN                     │
│         (noverfly.com, *.noverfly.com)              │
├───────────────┬─────────────────────────────────────┤
│   Frontend    │           Backend API               │
│  (Next.js)    │      (Node.js / Fastify)            │
│               │                                     │
│  • Marketing  │  • REST API (api.noverfly.com)      │
│  • Dashboard  │  • PostgreSQL 16 (BaaS database)    │
│  • GlowDesign │  • Redis 7 (cache + real-time)      │
│  • Help Center│  • S3 Storage (OVH)                 │
│  • Dev Portal │  • BullMQ Workers + WebSocket        │
└───────────────┴─────────────────────────────────────┘
```

---

## Account System

- **Users** — Individual accounts with email/password or Google OAuth
- **Organizations (Tenants)** — Multi-tenant workspaces for team collaboration
- **Roles** — Owner, Admin, Editor, Viewer
- **API Keys** — Programmatic access for integrations
- **Quotas** — Per-plan resource limits (sites, pages, storage, API calls)

---

## OpenAPI / Swagger

An [OpenAPI 3.0 specification](openapi.yaml) is included for machine-readable API documentation. You can:

- Import into **Postman**, **Insomnia**, or **Swagger UI**
- Generate client SDKs with **openapi-generator**
- Use with any OpenAPI-compatible tool

```bash
# Preview with Swagger UI (Docker)
docker run -p 8080:8080 -e SWAGGER_JSON=/spec/openapi.yaml -v $(pwd):/spec swaggerapi/swagger-ui
```

---

## SDKs (Coming Soon)

| Language | Package | Status |
|----------|---------|--------|
| JavaScript / TypeScript | `npm install @noverfly/sdk` | 🔜 Coming soon |
| Python | `pip install noverfly` | 🔜 Coming soon |
| PHP | `composer require noverfly/sdk` | 🔜 Coming soon |

---

## Links

| Resource | URL |
|----------|-----|
| **Website** | [noverfly.com](https://noverfly.com) |
| **API** | [api.noverfly.com/v1](https://api.noverfly.com/v1) |
| **Help Center** | [help.noverfly.com](https://help.noverfly.com) |
| **Documentation** | [docs.noverfly.com](https://docs.noverfly.com) |
| **Status** | [status.noverfly.com](https://status.noverfly.com) |
| **Email Support** | support@noverfly.com |
| **Bug Reports** | [GitHub Issues](https://github.com/gloowflix-hash/noverfly-docs/issues) |

---

## License

This documentation is licensed under the [MIT License](LICENSE).

> **Note:** This repository contains documentation only. The NoverFly platform source code is proprietary and owned by **Gloowflix Cloud**.

---

<p align="center">
  <strong>NoverFly</strong> — Build, sell, and manage your digital business.<br>
  Made with ❤️ by <a href="https://noverfly.com">Gloowflix Cloud</a>
</p>
