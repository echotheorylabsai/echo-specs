# SaaS Product Specification (v2)

## Cross-Platform Application (Web + Desktop + Mobile)

> **Principle:** Data structures first. Ship the smallest thing that works. Earn complexity, don't assume it.

---

## 1. Core Data Model

Everything starts here. Get this right and the architecture follows. Get this wrong and no pattern saves you.

### 1.1 Entity Relationships

```
Tenant (org/workspace)
  ├── Users (belong to exactly one tenant, have roles)
  │     ├── Sessions (device-bound, tracks auth state)
  │     └── Preferences (notification, UI, timezone)
  ├── Resources (your core domain objects — define these per product)
  │     ├── owned by User, scoped to Tenant
  │     └── access controlled via Permissions
  ├── Subscription (one active per tenant)
  │     ├── Plan (tier + entitlements)
  │     └── Payment history
  └── Audit log (append-only, immutable)
```

### 1.2 Key Constraints

- **Tenant isolation is row-level** — single database, `tenant_id` on every table, enforced at query layer. Graduate to schema-per-tenant or DB-per-tenant only when you have data residency requirements.
- **Soft deletes everywhere** — `deleted_at` timestamp, never hard delete until retention window expires.
- **All timestamps UTC** — convert at display layer only.
- **UUIDs for external IDs, bigint for internal FKs** — UUIDs prevent enumeration, bigints keep joins fast.

### 1.3 MVP Data Model

Define only the tables you need to ship v1. Every table must answer: "What user action creates/reads/updates this?"

If you can't answer that, the table doesn't exist yet.

---

## 2. What Ships First (MVP) vs What Waits

| Ships in MVP | Earns its way in later |
| --- | --- |
| Email/password auth + social login (Google) | SSO/SAML, passkeys |
| Single API server (no BFF, no gateway) | BFF pattern when platform-specific auth diverges |
| Web app only | Mobile (RN), desktop (Tauri) |
| Stripe Checkout for billing (hosted) | Custom billing UI, usage-based pricing |
| Console logging + Sentry | OpenTelemetry, distributed tracing |
| Single region | Multi-region, data residency |
| Basic RBAC (owner, admin, member) | Custom roles, fine-grained permissions |
| Email notifications only | Push, in-app, webhooks |
| PostgreSQL full-text search | Dedicated search engine |
| Manual deploys to single environment | Canary deployments, feature flags |

**Rule:** Nothing from the "later" column enters the codebase until the MVP is live and a real user is asking for it.

---

## 3. Auth — Simple, Then Evolve

### MVP Auth (What Ships)

```
Client → API Server → Managed IdP (Auth0/Clerk)
```

- Authorization Code + PKCE for web
- Tokens stored in httpOnly secure cookies, set by the API server directly
- Access token: 15 min, JWT, validated by API
- Refresh token: 7 days, opaque, rotated on use with reuse detection
- No BFF. The API server handles session management. One server, one responsibility.

### When to Add Complexity

| Trigger | Response |
| --- | --- |
| Ship mobile app | Add mobile-specific token handling, evaluate BFF split |
| Enterprise customer needs SSO | Add SAML/OIDC federation via IdP config |
| Security audit flags token handling | Add API gateway for centralized validation |
| Passkey adoption hits critical mass | Add WebAuthn at credential layer, session layer unchanged |

Don't build the airport before you know where people want to fly.

---

## 4. Architecture — Start Simple

### MVP Architecture

```
React (Next.js)  →  Single API Server (Node/Go/Rust)  →  PostgreSQL
                                                        →  Redis (sessions + cache)
                                                        →  S3 (file storage)
```

That's it. One deployable. One database. One cache.

### Code Organization

```
/src
  /modules
    /auth        ← signup, login, session, password reset
    /tenants     ← workspace CRUD, member management
    /billing     ← Stripe integration, plan enforcement
    /[domain]    ← your core product modules
  /shared
    /db          ← query helpers, migrations
    /middleware   ← auth, tenant scoping, rate limiting, error handling
    /types       ← shared TypeScript types
```

**Colocation:** each module owns its routes, handlers, queries, and tests. No `/controllers`, `/services`, `/repositories` separation. A module is a vertical slice.

### What's NOT Here (By Design)

- No repository pattern — direct SQL queries (parameterized) in handlers. Abstract only when you have two storage backends, which you don't.
- No event bus — call the audit logger and notification sender directly. Extract to events when you have 5+ consumers of the same action.
- No CQRS — same model reads and writes. Split when a dashboard query is too slow against the write-optimized schema.
- No microservices — extract a service only when a module has a genuinely different scaling/deployment need.

---

## 5. Non-Functional Requirements

### What's Measurable (MVP Targets)

| Metric | Target |
| --- | --- |
| API P95 latency | < 300ms |
| Web TTI | < 3s on 4G |
| Uptime | 99.5% (43h downtime/year — honest for a small team) |
| Deploy frequency | Multiple times per day |
| Recovery time | < 1 hour from backup |

### Security Baseline (MVP)

- HTTPS everywhere, HSTS headers
- Parameterized queries (no ORMs hiding SQL injection)
- Rate limiting on auth endpoints (10 attempts/min)
- CSP headers on web
- Dependency audit in CI (`npm audit`, `cargo audit`)
- Secrets in environment variables, never in code

### What Scales With the Product

- Observability graduates from logs → traces → dashboards as team grows
- Uptime target increases with paying customer count
- Security posture escalates toward SOC2 when enterprise pipeline demands it

---

## 6. Cross-Platform Strategy

### Phase 1: Web Only

React + Next.js. Ship, get users, validate the product.

### Phase 2: Mobile

React Native. Share types and validation logic from the monorepo. UI is platform-native, not forced parity.

### Phase 3: Desktop

Tauri wrapping the web app with native OS integrations (notifications, file system, system tray). Only if desktop-specific workflows justify it.

**Shared code lives in `/packages/core`** — validation, types, business logic. UI code is never shared across platforms. Forced UI parity across platforms produces mediocre experiences everywhere.

---

## 7. Things Most People Forget

### Billing Entitlements

Every feature check should go through:

```
canAccess(tenant, feature) → boolean
```

Not scattered `if (plan === 'pro')` checks. One function, one place to update.

### Tenant Lifecycle

Define these states and transitions upfront:

```
trial → active → past_due → suspended → cancelled → deleted
                                          ↓
                                     reactivated → active
```

### Data Export

Every table the user touches must be exportable as JSON/CSV. Design for this from the start — it's a constraint on your data model, not a feature bolted on later.

### Audit Log

Append-only table:

```sql
CREATE TABLE audit_log (
    id          bigserial PRIMARY KEY,
    tenant_id   uuid NOT NULL,
    user_id     uuid NOT NULL,
    action      text NOT NULL,       -- 'invoice.created', 'member.removed'
    resource    text NOT NULL,       -- 'invoice:uuid', 'member:uuid'
    metadata    jsonb,               -- action-specific context
    created_at  timestamptz NOT NULL DEFAULT now()
);
```

Log at the handler level, not through middleware magic. Explicit is better than implicit.

---

## Appendix: Decision Log

Track every architectural decision with:

| Decision | Chosen | Rejected | Why |
| --- | --- | --- | --- |
| Database | PostgreSQL (single instance) | Multi-DB, DynamoDB | Simplicity, SQL power, scale later |
| Auth | Managed IdP | Self-hosted Keycloak | Operational burden not justified at MVP |
| Architecture | Modular monolith | Microservices | No scaling need justifies the complexity tax |
| Mobile | React Native | Flutter, native | Team already knows React, type sharing |
| API style | REST | GraphQL | Simpler tooling, add GraphQL if frontend needs demand it |

> **This spec is intentionally short.** If the spec is longer than the first version of the codebase, you're designing instead of building. Ship, learn, adapt.