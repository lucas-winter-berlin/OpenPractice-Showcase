# Architecture: Cloudflare Edge & D1

Technical view of Open Practice as an **SSR worker on the Cloudflare edge**. No production IDs, no seeds, no handler implementations.

## 1. Edge infrastructure

Two workers share the same logical database and fill different roles.

```mermaid
flowchart LR
  subgraph prod [Production_Worker]
    SSR[Astro_SSR]
    API["/api/*"]
    HTML[HTML_Pages]
  end

  subgraph cron [Scheduled_Worker]
    Daily[Daily_03UTC]
    Hourly[Hourly]
  end

  subgraph bindings [Shared_and_App_Bindings]
    D1[(D1)]
    KV[(KV_rate_limits)]
    Img[Images]
    AI[Workers_AI]
    AE[Analytics_Engine]
  end

  SSR --> HTML
  SSR --> API
  API --> D1
  API --> KV
  API --> AI
  API --> AE
  HTML --> D1
  HTML --> Img
  Daily --> D1
  Hourly --> D1
```

| Component | Role |
| :--- | :--- |
| **App worker** | Astro `output: 'server'` + Cloudflare adapter. Serves HTML (SSR injection of public catalogs) and JSON/ICS under `/api/*`. Custom domains only - no public `*.workers.dev`. |
| **Scheduled worker** | Cron: daily at 03:00 UTC, set events older than five days to `draft` (rows are kept) and delete expired sessions and magic-link tokens. Hourly, publish due blog posts and methods from `scheduled` to `published`. Not deployed with the app worker. |
| **D1 `op_db`** | Users, sessions, magic-link tokens, events, blog, methods, categories. |
| **KV `SESSION`** | Rate limits only (magic-link, contact, studio translate). **Not** used for session storage. |
| **Workers AI** | Studio translate. Does not write D1. |
| **Analytics Engine** | Same-origin telemetry. Studio analytics reads it back. |
| **Images** | Media binding for the app. |
| **ASSETS** | Client bundle from the Astro build. |

### Environments

| Environment | Pattern |
| :--- | :--- |
| Production | Custom domain, production D1, mail delivery required for magic link and contact |
| Preview / staging | Separate worker name, **separate** D1, same `op_db` binding. Without mail config: safe no-op (no send, API still returns `ok`) |
| Local | Wrangler-local D1 for dev and E2E - isolated from remote data |

Outbound (capabilities only, no URLs): transactional email (primary) and an optional notify webhook `{ kind, payload }`.

## 2. Request flows

### Public page

```mermaid
sequenceDiagram
  participant Browser
  participant Worker as AstroSSR
  participant D1 as D1
  Browser->>Worker: GET listing_or_detail
  Worker->>D1: published content by locale
  D1-->>Worker: rows
  Worker-->>Browser: HTML plus SSR payload
```

### Magic link (auth)

```mermaid
sequenceDiagram
  participant Browser
  participant API as AppWorker
  participant KV as KV
  participant D1 as D1
  participant Mail as EmailProvider
  Browser->>API: POST /api/auth/magic/request
  API->>API: CSRF same-origin
  API->>KV: email rate limit
  API->>D1: lookup user by email
  Note over API,D1: Unknown email still returns ok
  API->>D1: store hashed token TTL 15min
  API->>Mail: magic_login_link
  Browser->>API: GET /api/auth/magic/verify?token
  API->>D1: consume token create session
  API-->>Browser: Set-Cookie redirect studio
```

Password login (`POST /api/auth/login`) is permanently `410 Gone`.

### Mutating studio APIs

1. Session cookie (HttpOnly, `SameSite=Lax`, `Secure`, 7-day TTL) checked against `user_sessions` ⋈ `users`
2. CSRF: reject `Sec-Fetch-Site: cross-site`; if `Origin` is set, it must match the request origin
3. Authorization: `is_admin` bypasses grants and owner checks. Otherwise show-tab plus the matching create / edit / publish flag, and owner (`user_id`) for updates and deletes
4. Events use the same grant model as posts and methods. There is no "any signed-in user may create events" exception

### Content-read API

Bearer token, GET only, six routes under `/api/content/{posts,methods,events}`. Published rows only. No session, no CSRF, no CORS. Empty or missing token secret → `503`; wrong Bearer → `401`. Query: `lang` (`de` default), list `q` and `limit` (1–100), event list also `dateFrom` / `dateTo`. Responses omit `userId`, status, rejection reasons, and email.

### Telemetry

`POST /api/telemetry` is same-origin. CSRF applies. Always `204` for a trusted request, including when the body is discarded. No visitor id. Verified bots are not written.

## 3. i18n and public surfaces

| Mode | Examples | Language switch |
| :--- | :--- | :--- |
| Toggle | `/`, `/blog`, `/events` | Same URL, UI locale |
| Navigate | `/favoriten` ↔ `/favorites`, site pages | Partner URL |
| Detail | `/methoden/{slug}`, `/methods/{slug}`, `/blog/{slug}`, `/events/{slug}` | Resource partner URL |

Static legal/content pages (about, privacy, imprint, contact) live as content, **not** in D1. The sitemap (`/sitemap.xml`) mixes registry URLs with published D1 entries at runtime.

Favorites stay client-side. Legacy paths `/praxis` and `/practice` 301-redirect to the saved-items pages.

## 4. D1 data model

Abstract relational model. No seed rows, no dumps.

```mermaid
erDiagram
  users ||--o{ user_sessions : has
  users ||--o{ magic_login_tokens : has
  users ||--o{ events : authors
  users ||--o{ blog_posts : authors
  users ||--o{ methods : authors
  blog_categories ||--o{ blog_posts : classifies
  blog_categories ||--o{ methods : classifies
  blog_posts }o--o{ methods : related_ids

  users {
    text id PK
    text email UK
    int is_admin
    text trust_level
    int can_events
    int can_events_create
    int can_events_edit
    int can_events_publish
    int can_posts
    int can_posts_create
    int can_posts_edit
    int can_posts_publish
    int can_methods
    int can_methods_create
    int can_methods_edit
    int can_methods_publish
    int can_categories
    int can_categories_create
    int can_categories_edit
    int can_analytics
    datetime created_at
  }

  user_sessions {
    text token PK
    text user_id FK
    datetime expires_at
    datetime created_at
  }

  magic_login_tokens {
    text id PK
    text user_id FK
    text token_hash UK
    datetime expires_at
    datetime used_at
    text requested_ip
    datetime created_at
  }

  events {
    text id PK
    text user_id FK
    text status
    text rejection_reason
    date event_date
    time start_time
    time end_time
    text location
    text city
    text price
    text spoken_language
    text title_de
    text title_en
    text description_short_de
    text description_short_en
    text description_long_de
    text description_long_en
    text external_link
    datetime created_at
    datetime updated_at
  }

  blog_categories {
    text id PK
    text name_de
    text name_en
    text slug UK
    datetime created_at
  }

  blog_posts {
    text id PK
    text user_id FK
    text category_id FK
    text status
    text rejection_reason
    date post_date
    datetime publish_at
    text title_de
    text title_en
    text description_short_de
    text description_short_en
    text content_de
    text content_en
    text slug
    text related_method_ids
    datetime created_at
    datetime updated_at
  }

  methods {
    int id PK
    text user_id FK
    text category_id FK
    text status
    text rejection_reason
    date post_date
    datetime publish_at
    text slug_de
    text slug_en
    text title_de
    text title_en
    text preview_de
    text preview_en
    text time_de
    text time_en
    text description_de
    text description_en
    text steps_de
    text steps_en
    text note_de
    text note_en
    text source_de
    text source_en
    text quick_help_tags
    datetime created_at
    datetime updated_at
  }
```

### Semantics (no DDL script)

| Topic | Rule |
| :--- | :--- |
| IDs | Users, events, posts, categories: text/UUID. Methods: stable **integer** IDs (favorites and related methods). |
| Bilingual | `_de` / `_en` columns, no locale tables. |
| Status | Content: `draft` \| `scheduled` \| `published` \| `rejected` (events have no `scheduled`). |
| Trust | `draft` (review), `approved`, `verified`. Independent of `can_*`. Publish is the matching `can_*_publish` flag, not trust. |
| Related methods | `blog_posts.related_method_ids` is a serialized ID list, not a join table. |
| Categories | Delete is blocked while posts or methods still reference the category. |
| Auth tokens | Magic link stores a **hash**, not plaintext; one-time use via `used_at`. |

Useful index families: `(status, post_date)` and `user_id` / `category_id` on posts and methods; slugs and `(status, publish_at)` on methods.

## 5. Security architecture (as built)

```mermaid
flowchart TB
  Req[Incoming_Request] --> CSRF[SameOrigin_CSRF]
  CSRF --> Session[D1_Session_Cookie]
  Session --> RBAC[Admin_Owner_canFlags]
  RBAC --> Handler[Route_Contract]
  Handler --> Sanitize[RichText_and_Field_Limits]
  Handler --> RL[KV_RateLimit]
```

| Layer | Contract |
| :--- | :--- |
| CSRF | Mutating routes except documented public exceptions |
| Session | Cookie `op_session`, validated against D1, not KV |
| XSS | Server-side rich-text allowlist; tight field length limits |
| HTTP | CSP, `X-Frame-Options: DENY`, `nosniff`, JSON `Cache-Control: no-store` |
| Rate limit | Magic link (email, fail-open without KV); contact (IP, fail-closed in production without KV); studio translate (per user, fail-closed without KV) |
| Enumeration | Unknown magic-link email → same `ok` as a known address |
| Contact | No D1 insert; honeypot field still answered with `ok` |

Left open by design: no global API throttling; CSP is maintained by hand.

## 6. Quality gates (no pipeline code)

Typical chain before a production deploy: dependency audit (high) → unit and security coverage → worker build → bundle/cron smoke → E2E. Remote D1 writes are operationally gated (confirmation), separate from this showcase.

## 7. API reference

Full HTTP contracts: [../openapi.yaml](../openapi.yaml). Product map: [features.md](features.md). Pages and workers: [structure.md](structure.md). Grants and publish: [studio.md](studio.md).
