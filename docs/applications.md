# Build Applications

NoverFly is not just a website builder — it's a **full-stack application platform**. Build complete web applications with a visual frontend, cloud backend, database, authentication, and one-click deployment.

---

## Overview

With NoverFly, you can build:

- **Web Apps** — Dashboards, admin panels, internal tools, client portals
- **SaaS Products** — Multi-tenant applications with user auth and billing
- **Marketplaces** — Two-sided platforms with products, sellers, and buyers
- **Mobile-Friendly Apps** — Responsive PWA applications
- **Customer Portals** — Branded portals for your clients with custom domains
- **Data-Driven Apps** — Apps backed by PostgreSQL with real-time updates

---

## Full-Stack Architecture

Every NoverFly application is a complete stack:

```
┌──────────────────────────────────────────────────────┐
│                    Your Application                   │
├──────────────┬──────────────┬────────────────────────┤
│   Frontend   │   Backend    │      Database          │
│              │              │                        │
│  GlowDesign  │  REST API    │  Cloud PostgreSQL      │
│  (visual     │  (auto-      │  (tables, queries,     │
│   editor)    │   generated) │   real-time, RLS)      │
├──────────────┼──────────────┼────────────────────────┤
│     Auth     │   Storage    │     Deployment         │
│              │              │                        │
│  JWT + OAuth │  Flivex Files │  One-click publish     │
│  API Keys    │  CDN Delivery│  Custom domains        │
│  Roles/RBAC  │  Uploads     │  SSL + Cloudflare      │
└──────────────┴──────────────┴────────────────────────┘
```

---

## Getting Started — Build Your First App

### Step 1: Create an Application

```bash
curl -X POST https://api.noverfly.com/v1/apps \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "X-Tenant-Id: YOUR_TENANT_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "My Task Manager",
    "type": "application",
    "template": "blank"
  }'
```

Or from the dashboard:
1. Click **+ New Project**
2. Select **Application** (not Website)
3. Choose a template or start blank
4. Your app is created with built-in auth, database, and hosting

### Step 2: Design the Frontend

Open **GlowDesign** to build your app's UI:

- Drag-and-drop **forms**, **tables**, **charts**, **lists**
- Bind components to **database tables** or **API endpoints**
- Add **user authentication** pages (login, register, profile)
- Create **role-based views** (admin vs user)

### Step 3: Set Up the Database

Create your data model:

```bash
# Create a "tasks" table
curl -X POST https://api.noverfly.com/v1/database/tables \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "X-Tenant-Id: YOUR_TENANT_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "tasks",
    "columns": [
      { "name": "id", "type": "serial" },
      { "name": "title", "type": "text", "nullable": false },
      { "name": "description", "type": "text" },
      { "name": "status", "type": "text", "default": "todo" },
      { "name": "priority", "type": "integer", "default": "0" },
      { "name": "assigned_to", "type": "uuid" },
      { "name": "due_date", "type": "date" },
      { "name": "created_at", "type": "timestamp", "default": "now()" }
    ],
    "enableRLS": true
  }'
```

### Step 4: Connect Frontend to Data

In GlowDesign, bind your UI components to database tables:

1. Select a **Table** or **List** component
2. Click **Data Source** → select your `tasks` table
3. Map columns to display fields
4. Add **Create**, **Update**, **Delete** buttons with form bindings
5. Data flows automatically between UI and database

### Step 5: Add Authentication

Every application includes built-in auth:

- **Login / Register pages** — Pre-built or custom
- **Protected routes** — Only authenticated users can access
- **Role-based access** — Admin, Editor, Viewer
- **User data isolation** — Row-Level Security ensures users see only their data

### Step 6: Build & Deploy

```bash
# Build the application
curl -X POST https://api.noverfly.com/v1/apps/YOUR_APP_ID/build \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "X-Tenant-Id: YOUR_TENANT_ID"

# Deploy to production
curl -X POST https://api.noverfly.com/v1/apps/YOUR_APP_ID/deploy \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "X-Tenant-Id: YOUR_TENANT_ID"
```

Your app is live at `https://your-app.noverfly.com` or your custom domain.

---

## Application API

### Manage Applications

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/apps` | List all applications |
| `POST` | `/apps` | Create a new application |
| `GET` | `/apps/:id` | Get application details |
| `PATCH` | `/apps/:id` | Update application settings |
| `DELETE` | `/apps/:id` | Delete an application |
| `POST` | `/apps/:id/build` | Build the application |
| `POST` | `/apps/:id/deploy` | Deploy to production |
| `GET` | `/apps/:id/builds` | List build history |
| `POST` | `/apps/:id/rollback/:buildId` | Rollback to a previous build |

### Application Object

```json
{
  "id": "app_abc123",
  "name": "My Task Manager",
  "type": "application",
  "url": "https://my-task-manager.noverfly.com",
  "customDomain": "app.mycompany.com",
  "status": "live",
  "stack": {
    "frontend": "glowdesign",
    "backend": "auto-api",
    "database": "postgresql",
    "auth": "jwt+oauth"
  },
  "tables": ["tasks", "users", "comments"],
  "pages": 8,
  "lastBuild": "2025-01-15T10:00:00Z",
  "lastDeploy": "2025-01-15T10:00:03Z",
  "createdAt": "2025-01-10T08:00:00Z"
}
```

---

## App Templates

Start faster with pre-built templates:

| Template | Description | Includes |
|----------|-------------|----------|
| **Blank** | Empty application canvas | Auth, database |
| **Task Manager** | Kanban board + task tracking | Tasks table, status views |
| **CRM** | Contact and deal management | Contacts, deals, pipeline |
| **Admin Dashboard** | Data management panel | Charts, tables, CRUD |
| **E-Commerce Store** | Full online shop | Products, cart, checkout |
| **Blog/CMS** | Content publishing platform | Posts, categories, editor |
| **Portfolio** | Personal/agency showcase | Projects, gallery, contact |
| **SaaS Starter** | Multi-tenant SaaS boilerplate | Auth, billing, teams, API |
| **Booking System** | Appointment scheduling | Calendar, bookings, notifications |
| **Inventory Manager** | Stock tracking system | Products, stock, orders |

---

## Sites vs Applications

| Feature | Sites (Websites) | Applications (Web Apps) |
|---------|-----------------|------------------------|
| **Purpose** | Marketing, landing pages, blogs | Dashboards, tools, SaaS products |
| **Frontend** | GlowDesign (static + CMS) | GlowDesign (dynamic + data-bound) |
| **Backend** | CMS API only | Full REST API + Database |
| **Database** | CMS collections | Cloud PostgreSQL (full BaaS) |
| **Auth** | Optional | Built-in (JWT + OAuth + RLS) |
| **Real-Time** | ❌ | ✅ WebSocket subscriptions |
| **User Roles** | ❌ | ✅ RBAC (Admin, Editor, Viewer) |
| **API Auto-Gen** | ❌ | ✅ CRUD endpoints per table |
| **Custom Domain** | ✅ | ✅ |
| **Deployment** | One-click publish | Build + Deploy pipeline |

---

## Use Cases

### Internal Tools
Build custom admin panels, reporting dashboards, and back-office tools for your team. Connect to your database, add charts and tables, protect with authentication.

### Client Portals
Create branded portals where your clients can log in, view their data, upload files, and communicate with your team. Each client sees only their own data via Row-Level Security.

### SaaS Products
Build multi-tenant SaaS applications with:
- User registration and authentication
- Team/organization management
- Subscription billing (Stripe)
- API key generation
- Usage quota enforcement
- Custom domains per tenant

### Marketplaces
Two-sided platforms with:
- Seller and buyer accounts
- Product listings
- Shopping cart and checkout
- Order management
- Seller dashboards
- Commission tracking

### Data Applications
Build apps that collect, process, and display data:
- Survey/form builders
- Analytics dashboards
- Inventory management
- Project management
- CRM systems

---

## Quotas

| Plan | Applications | Tables/App | Rows/Table | Builds/Month |
|------|-------------|-----------|-----------|-------------|
| Free | 1 | 5 | 1,000 | 10 |
| Pro | 10 | 25 | 100,000 | 100 |
| Business | 50 | Unlimited | 1,000,000 | 1,000 |
| Enterprise | Unlimited | Unlimited | Unlimited | Unlimited |

---

## Next Steps

- [Database & BaaS](database.md) — Cloud PostgreSQL and data management
- [Authentication](authentication.md) — Auth system for your applications
- [GlowDesign](glowdesign.md) — Design your app's frontend
- [Deployment](deployment.md) — Build, deploy, and manage releases
- [API Reference](api.md) — Full endpoint documentation
