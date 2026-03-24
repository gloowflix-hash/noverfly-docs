# Security

NoverFly takes security seriously. This page outlines how we protect your data, your users, and your business.

---

## Infrastructure Security

### Hosting
- **OVH Cloud VPS** — EU-based hosting (France) with enterprise-grade infrastructure
- **Docker containers** — All services run in isolated containers
- **Private networking** — Internal services communicate on an isolated Docker network
- **Firewall** — Only ports 80 and 443 are exposed to the internet
- **SSH hardening** — Key-based authentication recommended, fail2ban enabled

### CDN & Edge
- **Cloudflare** — All traffic is proxied through Cloudflare's global network
- **DDoS Protection** — Automatic Layer 3/4/7 DDoS mitigation
- **WAF** — Web Application Firewall with managed rulesets
- **Bot Management** — Automated bot detection and blocking

---

## Data Encryption

### In Transit
- **TLS 1.2/1.3** — All connections are encrypted with modern TLS
- **HSTS** — HTTP Strict Transport Security enforced
- **Cloudflare Origin Certificate** — End-to-end encryption from browser to origin server
- **Full (Strict) SSL** — No mixed content or downgrade attacks

### At Rest
- **PostgreSQL** — Database encrypted at disk level via OVH infrastructure
- **Flivex Storage** — Server-side encryption (SSE) for all uploaded files
- **Redis** — In-memory data protected by private network isolation
- **Backups** — Encrypted daily backups retained for 30 days

---

## Authentication Security

- **Password Hashing** — bcrypt with cost factor 12
- **JWT Tokens** — Short-lived access tokens (15 min) with rotating refresh tokens (7 days)
- **Token Rotation** — Refresh tokens are invalidated after single use
- **Rate Limiting** — Login attempts limited to prevent brute-force attacks
- **Session Management** — Server-side session tracking with forced logout capability
- **OAuth 2.0** — Google OAuth with PKCE for enhanced security

---

## API Security

### Rate Limiting
All API endpoints are rate-limited to prevent abuse:

| Endpoint Category | Limit |
|------------------|-------|
| Authentication | 10 requests/minute |
| Read operations | 300 requests/minute (Pro) |
| Write operations | 100 requests/minute (Pro) |
| File uploads | 30 requests/minute |

### Input Validation
- All inputs are validated and sanitized server-side
- SQL injection prevention via Prisma ORM (parameterized queries)
- XSS prevention via output encoding and Content Security Policy
- CSRF protection via SameSite cookies and Origin header validation

### CORS
- Strict CORS policy — only allowed origins can make API requests
- Credentials require explicit origin allowlist
- No wildcard (`*`) origins in production

### Request Security Headers
```http
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Strict-Transport-Security: max-age=31536000; includeSubDomains
Content-Security-Policy: default-src 'self'
Referrer-Policy: strict-origin-when-cross-origin
```

---

## Data Isolation

### Multi-Tenant Architecture
- Every data query is scoped to the authenticated tenant (`tenantId`)
- Database-level row isolation — users can never access another tenant's data
- API middleware enforces tenant boundaries on every request
- Tenant switching requires verified membership

### Role-Based Access Control (RBAC)
| Role | Create | Read | Update | Delete | Admin |
|------|--------|------|--------|--------|-------|
| Owner | ✅ | ✅ | ✅ | ✅ | ✅ |
| Admin | ✅ | ✅ | ✅ | ✅ | ❌ |
| Editor | ✅ | ✅ | ✅ | ❌ | ❌ |
| Viewer | ❌ | ✅ | ❌ | ❌ | ❌ |

---

## Payment Security

- **PCI DSS Compliant** — NoverFly never stores credit card numbers
- **Stripe** — Payment processing delegated to Stripe (PCI Level 1)
- **PayPal** — Payment processing delegated to PayPal
- **No card data on our servers** — Tokenized via Stripe.js / PayPal SDK
- **3D Secure** — Supported for Strong Customer Authentication (SCA)

---

## File Upload Security

- **File type validation** — Only allowed MIME types accepted
- **File size limits** — Per-plan upload limits enforced
- **Virus scanning** — Uploads scanned for malware
- **Isolated storage** — Files stored in Flivex Cloud with per-tenant path isolation
- **Signed URLs** — Private files accessed via time-limited signed URLs
- **No execution** — Uploaded files are never executed server-side

---

## Monitoring & Logging

- **Access logs** — All API requests logged with timestamp, IP, user, and action
- **Audit trail** — Critical operations (login, permission changes, data deletion) fully audited
- **Error tracking** — Real-time error monitoring and alerting
- **Uptime monitoring** — External health checks every 60 seconds
- **Log retention** — 90 days for standard logs, 1 year for audit logs

---

## Incident Response

In case of a security incident:

1. **Detection** — Automated monitoring detects anomalies
2. **Assessment** — Security team evaluates scope and impact
3. **Containment** — Affected systems isolated immediately
4. **Communication** — Affected users notified within 72 hours
5. **Resolution** — Root cause identified and patched
6. **Post-mortem** — Incident report published for transparency

---

## Data Privacy

### GDPR Compliance
- **Data location** — All data stored in the EU (France)
- **Data portability** — Export your data at any time
- **Right to erasure** — Request complete account deletion
- **Data processing** — We only process data necessary for platform operation
- **No data selling** — We never sell user data to third parties

### Data Retention
| Data Type | Retention |
|-----------|-----------|
| User accounts | Until deletion requested |
| Site content | Until deletion requested |
| Access logs | 90 days |
| Audit logs | 1 year |
| Backups | 30 days |
| Deleted accounts | Purged within 30 days |

---

## Responsible Disclosure

If you discover a security vulnerability:

1. **Do not** publicly disclose the vulnerability
2. **Email** security@noverfly.com with details
3. Include steps to reproduce
4. We will acknowledge within 24 hours
5. We will provide a fix timeline within 72 hours

---

## Security Checklist for Users

- ✅ Use a strong, unique password (12+ characters)
- ✅ Enable Google OAuth for stronger authentication
- ✅ Use API keys with minimum required scopes
- ✅ Rotate API keys regularly
- ✅ Review organization members and roles periodically
- ✅ Use Cloudflare for your custom domains
- ✅ Monitor your site analytics for unusual traffic

---

## Next Steps

- [Authentication](authentication.md) — Auth security details
- [API Reference](api.md) — Rate limits and error handling
- [Deployment](deployment.md) — Hosting infrastructure
