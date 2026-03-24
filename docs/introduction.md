# Introduction to NoverFly

NoverFly is a **cloud-native SaaS platform** that brings together website building, application development, content management, e-commerce, a cloud database (BaaS), and AI-powered tools into one seamless experience — exposed through **two REST APIs**.

| API | Base URL | Purpose |
|-----|----------|--------|
| **NoverFly API** | `https://api.noverfly.com/v1` | Main platform — auth, sites, apps, CMS, database, e-commerce, analytics |
| **GFK Storage API** | `https://gfk.noverfly.com` | Assets & media — file upload, image processing, storage |

---

## Why NoverFly?

Building an online presence typically requires juggling multiple tools:
- A website builder (Webflow, Wix)
- An application builder (Bubble, Retool)
- An e-commerce platform (Shopify, WooCommerce)
- A CMS (Contentful, Strapi)
- A backend database / BaaS (Firebase, Supabase)
- A hosting provider (Vercel, Netlify)
- Analytics tools (Google Analytics, Mixpanel)

**NoverFly replaces all of them** with a single platform.

---

## Core Features

### 🎨 GlowDesign Lite — Visual Editor
A powerful drag-and-drop editor that lets you design pixel-perfect websites and web applications without writing code. Supports responsive design, component systems, custom CSS, and real-time preview.

### 🚀 Build Applications
Build complete **full-stack web applications** — dashboards, admin panels, SaaS products, CRMs, client portals — with a visual frontend, cloud backend (BaaS), database, built-in authentication, and one-click deployment. No server management needed.

### 🧩 CMS & Dynamic Content
Structure your content with collections, custom fields, and relationships. Manage blog posts, products, team members, FAQs — anything you need.

### �️ Database & BaaS (Backend as a Service)
A fully managed **cloud PostgreSQL database** with REST API, SQL queries, Row-Level Security, and real-time WebSocket subscriptions. Create tables, insert data, query with filters — no backend code required. Like Firebase or Supabase, built into NoverFly.

### �🛒 E-Commerce
Full-featured online store with product catalog, inventory management, cart, checkout, Stripe & PayPal integration, order management, and customer accounts.

### 🤖 AI-Powered Tools
- **AI Writer** — Generate marketing copy, product descriptions, blog posts
- **AI Design Assistant** — Suggest layouts, color palettes, typography
- **AI Image Generation** — Create visuals with DALL-E and Stable Diffusion
- **AI Code Assistant** — Generate custom components and logic

### 🌐 Instant Deployment
Publish your site with one click. Get a free `*.noverfly.com` subdomain or connect your own custom domain with automatic SSL.

### 📊 Built-In Analytics
Track page views, unique visitors, bounce rates, conversion funnels, and more — directly in your dashboard.

### 👥 Team Collaboration
Invite team members, assign roles (Owner, Admin, Editor, Viewer), and work together in real-time across multiple projects.

---

## Platform Architecture

NoverFly is built on modern, battle-tested technologies:

| Layer | Technology |
|-------|-----------|
| Frontend | Next.js 14 (React) |
| Backend | Node.js / Fastify |
| Database | PostgreSQL 16 |
| Cache | Redis 7 |
| Storage | Flivex Cloud Storage |
| Workers | BullMQ (background jobs) |
| CDN | Cloudflare |
| Reverse Proxy | Caddy 2 |
| AI | OpenAI, DeepSeek, Google Gemini |
| Payments | Stripe, PayPal |

---

## Multi-Tenant Architecture

NoverFly uses a **shared-database, tenant-isolated** architecture:

- Each user can belong to multiple **organizations (tenants)**
- Data is isolated per tenant using a `tenantId` foreign key
- API requests require an `X-Tenant-Id` header for tenant-scoped operations
- Users can switch between tenants seamlessly from the dashboard

---

## What's Next?

- [Getting Started](getting-started.md) — Create your first project
- [Build Applications](applications.md) — Create full-stack web apps
- [Authentication](authentication.md) — Understand the auth system
- [Database & BaaS](database.md) — Cloud database and REST API
- [API Reference](api.md) — Explore the REST API
