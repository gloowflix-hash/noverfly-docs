---
title: "NoverFly API — Build Websites, Web Apps, E-Commerce & BaaS with One API"
published: true
description: "How to build a SaaS, e-commerce store, or web app using the NoverFly REST API — an all-in-one platform like Webflow + Shopify + Firebase combined."
tags: api, saas, webdev, tutorial
canonical_url: https://github.com/gloowflix-hash/noverfly-docs
cover_image: 
---

# NoverFly API — Build Websites, Web Apps & E-Commerce with One REST API

What if you could replace **Webflow + Shopify + Firebase + Vercel + Stripe + Contentful** with a single API?

That's exactly what **[NoverFly](https://noverfly.com)** does.

## What is NoverFly?

NoverFly is an **all-in-one SaaS platform** that combines:

- 🎨 **Website Builder** — Drag-and-drop visual editor (GlowDesign)
- 🚀 **Application Builder** — Build full-stack web apps (dashboards, SaaS, CRM)
- 🧩 **CMS** — Content management with collections and custom fields
- 🛒 **E-Commerce** — Products, cart, checkout, Stripe & PayPal
- 🗄️ **BaaS Database** — Cloud PostgreSQL with REST API, real-time, SQL queries
- 🤖 **AI Tools** — Content generation, design assistant, image generation
- 📊 **Analytics** — Built-in page views, visitors, conversions

All accessible through **two REST APIs**.

## Two APIs

| API | Base URL | Purpose |
|-----|----------|---------|
| **NoverFly API** | `https://api.noverfly.com/v1` | Auth, sites, apps, CMS, database, e-commerce, analytics |
| **GFK Storage API** | `https://gfk.noverfly.com` | File uploads, images, media storage |

## Quick Start

### 1. Login

```bash
curl -X POST https://api.noverfly.com/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "you@example.com", "password": "your-password"}'
```

### 2. List Your Sites

```bash
curl https://api.noverfly.com/v1/sites \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "X-Tenant-Id: YOUR_TENANT_ID"
```

### 3. Create an Application

```bash
curl -X POST https://api.noverfly.com/v1/apps \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "X-Tenant-Id: YOUR_TENANT_ID" \
  -H "Content-Type: application/json" \
  -d '{"name": "My SaaS Dashboard", "template": "dashboard"}'
```

### 4. Query Your Database

```bash
curl -X POST https://api.noverfly.com/v1/database/query \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "X-Tenant-Id: YOUR_TENANT_ID" \
  -H "Content-Type: application/json" \
  -d '{"query": "SELECT * FROM users WHERE active = true"}'
```

## SDK (Coming Soon)

```javascript
import { NoverFly } from "@noverfly/sdk"

const api = new NoverFly("YOUR_API_KEY")

const sites = await api.sites.list()
const app = await api.apps.create({ name: "My CRM", template: "crm" })
const users = await api.database.query("SELECT * FROM users")
```

## Authentication

3 methods supported:

| Method | Use Case |
|--------|----------|
| **JWT Token** | Web sessions, dashboard |
| **API Key + Secret Key** | Server-to-server, CI/CD |
| **Google OAuth 2.0** | One-click sign-in |

## Use Cases

- **Build a SaaS** — App builder + BaaS + auth + payments
- **Build an E-Commerce Store** — Product catalog + checkout + Stripe/PayPal
- **Build a Marketplace** — Multi-vendor + dashboards + payments
- **Build Internal Tools** — Admin panels + database + role-based access
- **Build a CMS Website** — Collections + custom fields + visual editor

## Pricing

| Plan | Price | Includes |
|------|-------|----------|
| Free | $0/mo | 1 site, 100 pages, 500 MB |
| Pro | $19/mo | 5 sites, e-commerce, AI |
| Business | $49/mo | 20 sites, analytics, teams |
| Enterprise | Custom | Unlimited, SLA |

## Links

- 🌐 **Website:** [noverfly.com](https://noverfly.com)
- 📖 **GitHub Docs:** [github.com/gloowflix-hash/noverfly-docs](https://github.com/gloowflix-hash/noverfly-docs)
- 📮 **Postman:** [NoverFly API on Postman](https://documenter.getpostman.com/view/53467544/2sBXijLYKF)
- 📧 **Support:** support@noverfly.com

---

*NoverFly is built by [Gloowflix Cloud](https://noverfly.com). Try it free at [noverfly.com](https://noverfly.com).*
