# Database & BaaS (Backend as a Service)

NoverFly includes a fully managed **cloud database** powered by PostgreSQL — giving you a complete Backend as a Service (BaaS) similar to Firebase or Supabase, directly integrated into the platform.

---

## Overview

- **Cloud PostgreSQL 16** — Managed database, no server setup required
- **REST API** — CRUD operations on tables via standard HTTP endpoints
- **SQL Queries** — Execute read-only SQL queries via `/database/query`
- **Row-Level Security (RLS)** — Control data access at the row level
- **Real-Time Subscriptions** — Get live updates via WebSocket when data changes
- **Auto-Generated API** — Create a table and get instant REST endpoints
- **Tenant Isolation** — Each organization's data is fully isolated
- **Backups** — Automatic daily backups with 30-day retention

---

## Quick Start

### 1. Create a Table

```bash
curl -X POST https://api.noverfly.com/v1/database/tables \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "X-Tenant-Id: YOUR_TENANT_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "contacts",
    "columns": [
      { "name": "id", "type": "serial" },
      { "name": "name", "type": "text", "nullable": false },
      { "name": "email", "type": "text", "unique": true },
      { "name": "phone", "type": "text" },
      { "name": "company", "type": "text" },
      { "name": "created_at", "type": "timestamp", "default": "now()" }
    ],
    "enableRLS": true
  }'
```

### 2. Insert Rows

```bash
curl -X POST https://api.noverfly.com/v1/database/tables/contacts/rows \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "X-Tenant-Id: YOUR_TENANT_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "rows": [
      { "name": "Alice Martin", "email": "alice@example.com", "company": "Acme Inc" },
      { "name": "Bob Smith", "email": "bob@example.com", "phone": "+33612345678" }
    ]
  }'
```

### 3. Query Rows

```bash
curl "https://api.noverfly.com/v1/database/tables/contacts/rows?select=name,email,company&filter=company=eq.Acme%20Inc&sort=-created_at&limit=10" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "X-Tenant-Id: YOUR_TENANT_ID"
```

**Response:**
```json
{
  "success": true,
  "data": [
    {
      "name": "Alice Martin",
      "email": "alice@example.com",
      "company": "Acme Inc"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 10,
    "total": 1,
    "totalPages": 1
  }
}
```

### 4. Update a Row

```bash
curl -X PATCH https://api.noverfly.com/v1/database/tables/contacts/rows/1 \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "X-Tenant-Id: YOUR_TENANT_ID" \
  -H "Content-Type: application/json" \
  -d '{ "phone": "+33698765432" }'
```

### 5. Delete a Row

```bash
curl -X DELETE https://api.noverfly.com/v1/database/tables/contacts/rows/1 \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "X-Tenant-Id: YOUR_TENANT_ID"
```

---

## Tables

### Column Types

| Type | PostgreSQL | Description | Example |
|------|-----------|-------------|---------|
| `text` | `TEXT` | Variable-length string | Names, emails, URLs |
| `integer` | `INTEGER` | 32-bit integer | Count, quantity, age |
| `float` | `DOUBLE PRECISION` | Floating-point number | Price, coordinates |
| `boolean` | `BOOLEAN` | True/false | Active, verified |
| `date` | `DATE` | Date without time | Birthday, deadline |
| `timestamp` | `TIMESTAMPTZ` | Date with time and timezone | Created at, updated at |
| `json` | `JSONB` | Structured JSON data | Metadata, preferences |
| `uuid` | `UUID` | Universally unique identifier | Primary keys, references |
| `serial` | `SERIAL` | Auto-incrementing integer | Primary keys |

### Column Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `nullable` | boolean | `true` | Allow NULL values |
| `unique` | boolean | `false` | Enforce unique constraint |
| `default` | string | — | Default value expression (e.g., `"now()"`, `"true"`) |
| `primaryKey` | boolean | `false` | Mark as primary key |

### List Tables

```http
GET /v1/database/tables
Authorization: Bearer YOUR_TOKEN
X-Tenant-Id: YOUR_TENANT_ID
```

**Response:**
```json
{
  "success": true,
  "data": [
    {
      "name": "contacts",
      "columns": [
        { "name": "id", "type": "serial", "nullable": false, "primaryKey": true },
        { "name": "name", "type": "text", "nullable": false, "primaryKey": false },
        { "name": "email", "type": "text", "nullable": true, "primaryKey": false }
      ],
      "rowCount": 1542
    },
    {
      "name": "orders",
      "columns": [...],
      "rowCount": 89
    }
  ]
}
```

### Get Table Schema

```http
GET /v1/database/tables/contacts
```

### Drop a Table

```http
DELETE /v1/database/tables/contacts
```

> ⚠️ This permanently deletes the table and all its data. This action cannot be undone.

---

## Querying Data

### Filter Operators

Use the `filter` query parameter with PostgreSQL-style operators:

| Operator | Description | Example |
|----------|-------------|---------|
| `eq` | Equal | `filter=status=eq.active` |
| `neq` | Not equal | `filter=status=neq.deleted` |
| `gt` | Greater than | `filter=price=gt.10` |
| `gte` | Greater than or equal | `filter=age=gte.18` |
| `lt` | Less than | `filter=stock=lt.5` |
| `lte` | Less than or equal | `filter=price=lte.99.99` |
| `like` | Pattern match (case-sensitive) | `filter=name=like.%john%` |
| `ilike` | Pattern match (case-insensitive) | `filter=email=ilike.%@gmail.com` |
| `is` | IS (null/true/false) | `filter=deleted_at=is.null` |
| `in` | In list | `filter=status=in.(active,pending)` |

### Select Specific Columns

```
?select=id,name,email
```

### Sorting

```
?sort=created_at       # Ascending
?sort=-created_at      # Descending
?sort=name,-created_at # Multiple fields
```

### Pagination

```
?page=1&limit=25
```

### Combined Example

```bash
curl "https://api.noverfly.com/v1/database/tables/products/rows?select=id,name,price,stock&filter=price=gte.10&filter=stock=gt.0&sort=-price&page=1&limit=20" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "X-Tenant-Id: YOUR_TENANT_ID"
```

---

## Raw SQL Queries

Execute read-only SQL queries for complex operations:

```bash
curl -X POST https://api.noverfly.com/v1/database/query \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "X-Tenant-Id: YOUR_TENANT_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "SELECT c.name, COUNT(o.id) AS order_count, SUM(o.total) AS total_spent FROM contacts c LEFT JOIN orders o ON o.contact_id = c.id GROUP BY c.name ORDER BY total_spent DESC LIMIT $1",
    "params": [10]
  }'
```

**Response:**
```json
{
  "success": true,
  "data": [
    { "name": "Alice Martin", "order_count": 15, "total_spent": 1250.00 },
    { "name": "Bob Smith", "order_count": 8, "total_spent": 780.50 }
  ],
  "rowCount": 2
}
```

### Security

- Only `SELECT` queries are allowed via `/database/query`
- All queries are parameterized to prevent SQL injection
- Query timeout: 30 seconds max
- Results limited to 1,000 rows per query
- Use the rows endpoints (`POST`, `PATCH`, `DELETE`) for mutations

---

## Row-Level Security (RLS)

Control data access at the row level — useful for multi-user apps where each user should only see their own data.

### Enable RLS on a Table

Set `"enableRLS": true` when creating a table, or:

```http
PATCH /v1/database/tables/contacts
Content-Type: application/json

{
  "enableRLS": true,
  "policies": [
    {
      "name": "users_own_data",
      "operation": "ALL",
      "expression": "user_id = auth.uid()"
    }
  ]
}
```

### Policy Operations

| Operation | Description |
|-----------|-------------|
| `SELECT` | Read access only |
| `INSERT` | Insert access only |
| `UPDATE` | Update access only |
| `DELETE` | Delete access only |
| `ALL` | All operations |

---

## Real-Time Subscriptions

Subscribe to database changes via WebSocket:

```javascript
// JavaScript client example
const ws = new WebSocket('wss://api.noverfly.com/v1/database/realtime');

ws.onopen = () => {
  ws.send(JSON.stringify({
    type: 'subscribe',
    table: 'contacts',
    event: '*', // INSERT, UPDATE, DELETE, or *
    filter: 'company=eq.Acme Inc',
    token: 'YOUR_ACCESS_TOKEN',
    tenantId: 'YOUR_TENANT_ID'
  }));
};

ws.onmessage = (event) => {
  const change = JSON.parse(event.data);
  console.log(change);
  // {
  //   type: "INSERT",
  //   table: "contacts",
  //   old: null,
  //   new: { id: 42, name: "New Contact", email: "new@example.com" },
  //   timestamp: "2025-01-15T10:00:00Z"
  // }
};
```

### Available Events

| Event | Description |
|-------|-------------|
| `INSERT` | New row inserted |
| `UPDATE` | Existing row updated |
| `DELETE` | Row deleted |
| `*` | All changes |

---

## Dashboard UI

You can also manage your database visually from the NoverFly dashboard:

1. Go to your site → **Database**
2. **Table Editor** — Create tables, add columns, edit data in a spreadsheet-like UI
3. **SQL Editor** — Write and run SQL queries with syntax highlighting
4. **Schema Viewer** — Visual diagram of tables and relationships
5. **Backups** — View and restore from automatic daily backups

---

## Quotas

| Plan | Databases | Rows/Database | Storage | Queries/Day |
|------|-----------|--------------|---------|-------------|
| Free | 1 | 1,000 | 50 MB | 500 |
| Pro | 5 | 100,000 | 1 GB | 50,000 |
| Business | Unlimited | 1,000,000 | 10 GB | 500,000 |
| Enterprise | Unlimited | Unlimited | Custom | Unlimited |

---

## Comparison with Other BaaS

| Feature | NoverFly | Firebase | Supabase | PlanetScale |
|---------|----------|----------|----------|-------------|
| Database | PostgreSQL | NoSQL (Firestore) | PostgreSQL | MySQL |
| REST API | ✅ Built-in | ❌ SDK only | ✅ PostgREST | ❌ |
| SQL Queries | ✅ | ❌ | ✅ | ✅ |
| Real-Time | ✅ WebSocket | ✅ | ✅ | ❌ |
| Row-Level Security | ✅ | ✅ (Rules) | ✅ | ❌ |
| Website Builder | ✅ GlowDesign | ❌ | ❌ | ❌ |
| E-Commerce | ✅ Built-in | ❌ | ❌ | ❌ |
| CMS | ✅ Built-in | ❌ | ❌ | ❌ |

**NoverFly is the only platform that combines BaaS + website builder + CMS + e-commerce in one.**

---

## Next Steps

- [CMS & Database](cms.md) — Manage structured content with the CMS
- [API Reference](api.md) — Full endpoint documentation
- [Authentication](authentication.md) — Auth system and API keys
- [Security](security.md) — Row-Level Security and data protection
