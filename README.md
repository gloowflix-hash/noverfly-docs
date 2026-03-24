# NoverFly API Documentation

> **Official developer documentation for the [NoverFly](https://noverfly.com) platform API.**
> Build websites, web applications, sell products, manage content, use a cloud database (BaaS), and integrate AI — through **two REST APIs**.

![API](https://img.shields.io/badge/API-v1-blue)
![License](https://img.shields.io/badge/license-MIT-green)
![Status](https://img.shields.io/badge/status-active-success)
![Node](https://img.shields.io/badge/node-%3E%3D18-brightgreen?logo=node.js)
![PostgreSQL](https://img.shields.io/badge/database-PostgreSQL%2016-336791?logo=postgresql)
![Redis](https://img.shields.io/badge/cache-Redis%207-DC382D?logo=redis)
[![Website](https://img.shields.io/badge/Website-noverfly.com-brightgreen)](https://noverfly.com)
[![API Base URL](https://img.shields.io/badge/API-api.noverfly.com%2Fv1-orange)](https://api.noverfly.com/v1)
[![GFK Storage](https://img.shields.io/badge/Storage-gfk.noverfly.com-blueviolet)](https://gfk.noverfly.com)
[![OpenAPI](https://img.shields.io/badge/OpenAPI-3.0-6BA539?logo=openapi-initiative)](openapi.yaml)
[![Postman](https://img.shields.io/badge/Postman-API%20Docs-FF6C37?logo=postman)](https://documenter.getpostman.com/view/53467544/2sBXijLYKF)
[![Help Center](https://img.shields.io/badge/Help-help.noverfly.com-lightblue)](https://help.noverfly.com)

---

## Why NoverFly?

| Traditional Stack | NoverFly |
|---|---|
| Webflow + Shopify + Firebase + Vercel + Stripe + Contentful | **One platform, one API, one dashboard** |
| 6+ subscriptions, 6+ dashboards | **$0 to start — scale as you grow** |
| Weeks of integration work | **Ship in hours** |
| Limited to US/EU markets | **Built for Africa & global scale** |

- **All-in-one** — Website builder + App builder + CMS + E-commerce + BaaS + AI
- **No-code + API** — Visual drag-and-drop editor for designers, full REST API for developers
- **Built for Africa & global scale** — Lightweight, fast, works on low bandwidth
- **Faster than traditional stack** — Go from idea to production in hours, not weeks
- **Developer-first** — OpenAPI spec, JWT auth, webhooks, SDK (coming soon)

---

## What is NoverFly?

**NoverFly** is an all-in-one **SaaS website & application builder**, **e-commerce platform**, and **Backend as a Service (BaaS)** — like Webflow + Shopify + Firebase combined. NoverFly provides a REST API for developers to programmatically manage every aspect of the platform:

- **Website & App Builder API** — Create and manage sites, web apps, pages, and designs
- **Application API** — Build full-stack web applications with frontend + backend + database
- **CMS API** — Collections, entries, dynamic content
- **Database API (BaaS)** — Cloud PostgreSQL, tables, queries, real-time data, row-level security
- **E-Commerce API** — Products, cart, checkout, orders, payments (Stripe & PayPal)
- **Authentication API** — JWT, OAuth 2.0 (Google), API Key + Secret Key, multi-tenant
- **AI API** — Content generation, design assistance, image generation
- **Analytics API** — Page views, visitors, conversions
- **Asset API** ⚡ — File uploads, media management (**separate GFK Storage API**)
- **Deployment API** — Publish sites, custom domains, SSL

---

## Two APIs

NoverFly exposes **two independent REST APIs**:

| API | Base URL | Purpose |
|-----|----------|ﾭ--------|
| **NoverFly API** | `https://api.noverfly.com/v1` | Main platform — auth, sites, apps, CMS, database, e-commerce, analytics, API keys |
| **GFK Storage API** | `https://gfk.noverfly.com` | Assets & media — file upload, image processing, storage management |

### Why two APIs?

Assets (images, videos, documents) are handled by a **dedicated GFK (Gloowflix) storage service** for performance:
- Optimized for large file uploads and downloads
- Independent scaling from the main API
- CDN-backed delivery for fast global access
- Separate rate limits (not counted against your main API quota)

### Authentication on both APIs

| Method | NoverFly API | GFK Storage API |
|--------|-------------|----------------|
| JWT Bearer Token | ✅ | ✅ |
| API Key + Secret Key | ✅ | ✅ |
| Google OAuth | ✅ | ❌ |

---

## Authentication Methods

NoverFly supports **3 authentication methods**:

### 1. JWT Token (Email + Password)
```bash
curl -X POST https://api.noverfly.com/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "you@example.com", "password": "your-password"}'
```

### 2. API Key + Secret Key
Generate from your **account dashboard** → Settings → API Keys.
```bash
curl https://api.noverfly.com/v1/sites \
  -H "X-API-Key: nf_pk_live_abc123..." \
  -H "X-API-Secret: nf_sk_live_xyz789..." \
  -H "X-Tenant-Id: YOUR_TENANT_ID"
```

### 3. Google OAuth 2.0
```
GET https://api.noverfly.com/v1/auth/google
```

> See [Authentication](docs/authentication.md) for complete details.

NoverFly is built for:

- **Creators & freelancers** — Build client sites and apps fast with GlowDesign visual editor
- **Businesses** — Launch online stores and web applications with built-in e-commerce and payments
- **Developers** — Build full-stack applications via REST API, BaaS, webhooks, and API keys
- **Agencies** — Manage multiple client projects with multi-tenant organizations
- **SaaS builders** — White-label websites, storefronts, and custom web applications
- **Startups** — Ship MVPs fast with built-in auth, database, hosting, and payments

### Keywords

`NoverFly` · `NoverFly API` · `website builder API` · `application builder` · `build web app` · `SaaS platform` · `e-commerce API` · `BaaS` · `Backend as a Service` · `cloud database` · `GlowDesign` · `CMS API` · `Gloowflix Cloud` · `drag-and-drop website builder` · `headless CMS` · `REST API` · `multi-tenant SaaS` · `PostgreSQL` · `database API` · `full-stack platform` · `no-code app builder`

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
| [Build Applications](docs/applications.md) | Full-stack web apps: frontend + backend + database + deploy |
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
| **Sites** | `GET /sites` · `POST /sites` · `POST /sites/:id/publish` | Website and app creation, publishing |
| **Applications** | `GET /apps` · `POST /apps` · `POST /apps/:id/build` · `POST /apps/:id/deploy` | Full-stack web application lifecycle |
| **CMS** | `GET /cms/:collection/entries` · `POST /cms/:collection/entries` | Dynamic content management |
| **Database (BaaS)** | `GET /database/tables` · `POST /database/query` · `POST /database/tables/:table/rows` | Cloud database, queries, real-time |
| **E-Commerce** | `GET /ecommerce/products` · `POST /ecommerce/checkout` · `GET /ecommerce/orders` | Products, checkout, orders |
| **Assets** | `POST /assets/upload` · `GET /assets` | File uploads and media **(GFK Storage API)** |
| **Analytics** | `GET /analytics/overview` · `GET /analytics/pageviews` | Traffic and performance |
| **API Keys** | `POST /api-keys` · `GET /api-keys` | Programmatic access tokens |

> See the complete [API Reference](docs/api.md) for request/response examples, error codes, and rate limits.

---

## Authentication

NoverFly supports **3 authentication methods**:

| Method | How to get it | Use case |
|--------|--------------|----------|
| **JWT Token** | `POST /auth/login` with email + password | Web app sessions, dashboard |
| **API Key + Secret Key** | Account dashboard → Settings → API Keys | Server-to-server, CI/CD, integrations |
| **Google OAuth 2.0** | `GET /auth/google` | One-click sign-in |

**API Key format:**
- **API Key (public):** `nf_pk_live_abc123...` — identifies your account
- **Secret Key (private):** `nf_sk_live_xyz789...` — authenticates requests (never expose in frontend code)

Both keys are generated from your **account dashboard** under **Settings → API Keys**.

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
│  • Help Center│  • Flivex Cloud Storage              │
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

## SDK (Coming Soon)

### JavaScript / TypeScript

```js
import { NoverFly } from "@noverfly/sdk"

const api = new NoverFly("YOUR_API_KEY")

// List all sites
const sites = await api.sites.list()
console.log(sites)

// Create a new application
const app = await api.apps.create({
  name: "My SaaS Dashboard",
  template: "dashboard"
})

// Query database
const users = await api.database.query("SELECT * FROM users WHERE active = true")

// Create a product
const product = await api.ecommerce.products.create({
  name: "T-Shirt",
  price: 2500,
  currency: "XOF"
})
```

### Python

```python
from noverfly import NoverFly

api = NoverFly("YOUR_API_KEY")

# List sites
sites = api.sites.list()

# Create an app
app = api.apps.create(name="My CRM", template="crm")

# Query database
results = api.database.query("SELECT * FROM orders")
```

### Installation

| Language | Package | Status |
|----------|---------|--------|
| JavaScript / TypeScript | `npm install @noverfly/sdk` | 🔜 Coming soon |
| Python | `pip install noverfly` | 🔜 Coming soon |
| PHP | `composer require noverfly/sdk` | 🔜 Coming soon |

---

## Use Cases

| Use Case | How NoverFly Helps |
|----------|--------------------|
| **Build a SaaS** | Visual app builder + BaaS database + auth + payments + deploy |
| **Build an E-Commerce Store** | Product catalog + cart + checkout + Stripe/PayPal + order management |
| **Build a CMS-Powered Website** | Drag-and-drop editor + dynamic collections + custom fields + API |
| **Build a Marketplace** | Multi-vendor products + user auth + payments + dashboards |
| **Build an Internal Tool** | Admin panels + database queries + role-based access + deploy |
| **Build a Client Portal** | Authenticated dashboards + real-time data + custom branding |
| **Build a Landing Page** | GlowDesign editor + responsive design + custom domain + analytics |
| **Build a Blog / Media Site** | CMS collections + markdown + SEO-friendly URLs + AI writer |

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
