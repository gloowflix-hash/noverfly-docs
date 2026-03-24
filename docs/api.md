# API Reference

NoverFly provides a **RESTful API** for programmatic access to all platform features.

---

## Base URL

```
https://api.noverfly.com/v1
```

All API endpoints are versioned under `/v1`.

---

## Authentication

Every API request requires authentication via Bearer token:

```http
Authorization: Bearer YOUR_ACCESS_TOKEN
```

For multi-tenant operations, include the tenant header:

```http
X-Tenant-Id: YOUR_TENANT_ID
```

See [Authentication](authentication.md) for details on obtaining tokens.

---

## Request Format

- **Content-Type:** `application/json`
- **Character Encoding:** UTF-8
- **Date Format:** ISO 8601 (`2025-01-15T10:00:00Z`)

---

## Response Format

All responses follow a consistent structure:

**Success:**
```json
{
  "success": true,
  "data": { ... }
}
```

**Success (paginated):**
```json
{
  "success": true,
  "data": [ ... ],
  "pagination": {
    "page": 1,
    "limit": 25,
    "total": 142,
    "totalPages": 6
  }
}
```

**Error:**
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Email is required",
    "details": [
      { "field": "email", "message": "This field is required" }
    ]
  }
}
```

---

## Endpoints

### Auth

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/auth/register` | Create new account |
| `POST` | `/auth/login` | Login with email/password |
| `POST` | `/auth/logout` | Logout and invalidate session |
| `POST` | `/auth/refresh` | Refresh access token |
| `GET` | `/auth/me` | Get current user |
| `PATCH` | `/auth/me` | Update profile |
| `POST` | `/auth/forgot-password` | Request password reset |
| `POST` | `/auth/reset-password` | Reset password with token |
| `GET` | `/auth/google` | Initiate Google OAuth |
| `GET` | `/auth/google/callback` | Google OAuth callback |

### Organizations

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/organizations` | List user's organizations |
| `POST` | `/organizations` | Create organization |
| `GET` | `/organizations/:id` | Get organization details |
| `PATCH` | `/organizations/:id` | Update organization |
| `DELETE` | `/organizations/:id` | Delete organization |
| `GET` | `/organizations/:id/members` | List members |
| `POST` | `/organizations/:id/members` | Invite member |
| `PATCH` | `/organizations/:id/members/:userId` | Update member role |
| `DELETE` | `/organizations/:id/members/:userId` | Remove member |

### Sites

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/sites` | List sites |
| `POST` | `/sites` | Create site |
| `GET` | `/sites/:id` | Get site details |
| `PATCH` | `/sites/:id` | Update site |
| `DELETE` | `/sites/:id` | Delete site |
| `POST` | `/sites/:id/publish` | Publish site |
| `POST` | `/sites/:id/unpublish` | Unpublish site |
| `GET` | `/sites/:id/pages` | List pages |
| `POST` | `/sites/:id/pages` | Create page |
| `PATCH` | `/sites/:id/pages/:pageId` | Update page |
| `DELETE` | `/sites/:id/pages/:pageId` | Delete page |

### CMS

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/cms/collections` | List collections |
| `POST` | `/cms/collections` | Create collection |
| `GET` | `/cms/collections/:slug` | Get collection schema |
| `PATCH` | `/cms/collections/:slug` | Update collection |
| `DELETE` | `/cms/collections/:slug` | Delete collection |
| `GET` | `/cms/:collectionSlug/entries` | List entries |
| `POST` | `/cms/:collectionSlug/entries` | Create entry |
| `GET` | `/cms/:collectionSlug/entries/:id` | Get entry |
| `PATCH` | `/cms/:collectionSlug/entries/:id` | Update entry |
| `DELETE` | `/cms/:collectionSlug/entries/:id` | Delete entry |

### Database (BaaS)

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/database/tables` | List all tables |
| `POST` | `/database/tables` | Create a table |
| `GET` | `/database/tables/:name` | Get table schema |
| `DELETE` | `/database/tables/:name` | Drop a table |
| `GET` | `/database/tables/:name/rows` | Query rows (with filters, sort, pagination) |
| `POST` | `/database/tables/:name/rows` | Insert rows |
| `PATCH` | `/database/tables/:name/rows/:id` | Update a row |
| `DELETE` | `/database/tables/:name/rows/:id` | Delete a row |
| `POST` | `/database/query` | Execute a read-only SQL query |

### E-Commerce

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/ecommerce/products` | List products |
| `POST` | `/ecommerce/products` | Create product |
| `GET` | `/ecommerce/products/:id` | Get product |
| `PATCH` | `/ecommerce/products/:id` | Update product |
| `DELETE` | `/ecommerce/products/:id` | Delete product |
| `POST` | `/ecommerce/cart` | Create/update cart |
| `POST` | `/ecommerce/cart/discount` | Apply discount code |
| `POST` | `/ecommerce/checkout` | Initiate checkout |
| `GET` | `/ecommerce/orders` | List orders |
| `GET` | `/ecommerce/orders/:id` | Get order details |
| `PATCH` | `/ecommerce/orders/:id` | Update order status |

### Assets

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/assets` | List uploaded assets |
| `POST` | `/assets/upload` | Upload file |
| `GET` | `/assets/:id` | Get asset details |
| `DELETE` | `/assets/:id` | Delete asset |

### API Keys

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api-keys` | List API keys |
| `POST` | `/api-keys` | Create API key |
| `DELETE` | `/api-keys/:id` | Revoke API key |

### Analytics

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/analytics/overview` | Dashboard overview |
| `GET` | `/analytics/pageviews` | Page view statistics |
| `GET` | `/analytics/visitors` | Visitor statistics |
| `GET` | `/analytics/events` | Custom event tracking |

---

## Rate Limits

API requests are rate-limited per plan:

| Plan | Requests/Minute | Requests/Day |
|------|----------------|-------------|
| Free | 60 | 1,000 |
| Pro | 300 | 50,000 |
| Business | 1,000 | 500,000 |
| Enterprise | Custom | Custom |

### Rate Limit Headers

Every API response includes rate limit information:

```http
X-RateLimit-Limit: 300
X-RateLimit-Remaining: 295
X-RateLimit-Reset: 1705312800
```

When rate limited, you'll receive:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30

{
  "success": false,
  "error": {
    "code": "RATE_LIMITED",
    "message": "Rate limit exceeded. Try again in 30 seconds."
  }
}
```

---

## Pagination

List endpoints support pagination:

| Parameter | Default | Max | Description |
|-----------|---------|-----|-------------|
| `page` | 1 | — | Page number |
| `limit` | 25 | 100 | Items per page |

```http
GET /v1/sites?page=2&limit=10
```

---

## Filtering & Sorting

### Filtering

Use query parameters to filter results:

```http
GET /v1/cms/blog-posts/entries?filter[published]=true&filter[tags]=tutorial
```

### Sorting

Use the `sort` parameter. Prefix with `-` for descending:

```http
GET /v1/cms/blog-posts/entries?sort=-publishDate
GET /v1/ecommerce/products?sort=price
```

---

## Error Codes

| HTTP | Code | Description |
|------|------|-------------|
| 400 | `BAD_REQUEST` | Invalid request body or parameters |
| 401 | `UNAUTHORIZED` | Missing or invalid authentication |
| 403 | `FORBIDDEN` | Insufficient permissions |
| 404 | `NOT_FOUND` | Resource not found |
| 409 | `CONFLICT` | Resource already exists |
| 422 | `VALIDATION_ERROR` | Input validation failed |
| 429 | `RATE_LIMITED` | Too many requests |
| 500 | `INTERNAL_ERROR` | Server error |

---

## Webhooks

Configure webhooks in **Settings > Webhooks** to receive real-time notifications.

### Events

| Event | Trigger |
|-------|---------|
| `site.published` | Site published to production |
| `site.unpublished` | Site taken offline |
| `entry.created` | CMS entry created |
| `entry.updated` | CMS entry updated |
| `entry.deleted` | CMS entry deleted |
| `order.created` | New order placed |
| `order.paid` | Payment confirmed |
| `order.shipped` | Order marked as shipped |
| `order.refunded` | Order refunded |
| `member.invited` | Team member invited |
| `member.joined` | Member accepted invite |

### Webhook Security

Every webhook includes a signature header for verification:

```http
X-NoverFly-Signature: sha256=abc123...
```

Verify the signature using your webhook secret:
```javascript
const crypto = require('crypto');
const signature = crypto
  .createHmac('sha256', WEBHOOK_SECRET)
  .update(requestBody)
  .digest('hex');
```

---

## SDKs & Libraries

Official SDKs (coming soon):
- **JavaScript/TypeScript** — `npm install @noverfly/sdk`
- **Python** — `pip install noverfly`
- **PHP** — `composer require noverfly/sdk`

---

## Next Steps

- [Authentication](authentication.md) — Auth endpoints in detail
- [CMS](cms.md) — CMS API usage
- [E-Commerce](ecommerce.md) — E-commerce API usage
- [Security](security.md) — API security practices
