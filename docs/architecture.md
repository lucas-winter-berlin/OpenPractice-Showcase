# Architektur: Cloudflare Edge & D1

Technische Sicht auf Open Practice als **SSR-Worker am Cloudflare-Edge**. Keine produktiven IDs, keine Seeds, keine Handler-Implementierung.

## 1. Edge-Infrastruktur

Zwei Worker teilen sich dieselbe logische Datenbank, erfüllen aber unterschiedliche Rollen.

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
  end

  SSR --> HTML
  SSR --> API
  API --> D1
  API --> KV
  HTML --> D1
  HTML --> Img
  Daily --> D1
  Hourly --> D1
```

| Komponente | Rolle |
| :--- | :--- |
| **App-Worker** | Astro `output: 'server'` + Cloudflare-Adapter. Liefert HTML (SSR-Injection öffentlicher Kataloge) und JSON/ICS unter `/api/*`. Custom Domains, kein öffentliches `*.workers.dev`. |
| **Scheduled Worker** | Cron: täglich 03:00 UTC Events mit Datum älter als fünf Tage auf `draft` setzen (Datensatz bleibt). Stündlich fällige Blog-Posts und Methoden von `scheduled` nach `published`. |
| **D1 `op_db`** | Nutzer, Sessions, Magic-Link-Tokens, Events, Blog, Methoden, Kategorien. |
| **KV `SESSION`** | Nur Rate-Limits. **Keine** Session-Speicherung. |
| **Images** | Medien-Binding der App. |
| **ASSETS** | Client-Bundle des Astro-Builds. |

### Umgebungen

| Umgebung | Muster |
| :--- | :--- |
| Production | Custom Domain, D1-Produktion, Mail-Versand Pflicht für Magic Link und Kontakt |
| Preview / Staging | Eigener Worker-Name, **eigene** D1, gleiches Binding `op_db`. Ohne Mail-Konfiguration: sicheres No-Op (kein Versand, API bleibt `ok`) |
| Lokal | Wrangler-lokale D1 für Dev und E2E — getrennt von Remote-Daten |

Outbound (nur als Capabilities, ohne URLs): transaktionale E-Mail (primär) und optionaler Notify-Webhook `{ kind, payload }`.

## 2. Request-Flüsse

### Öffentliche Seite

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

### Magic Link (Auth)

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
  API-->>Browser: Set-Cookie redirect dashboard
```

Passwort-Login (`POST /api/auth/login`) ist dauerhaft `410 Gone`.

### Mutierende Dashboard-APIs

1. Session-Cookie (HttpOnly, `SameSite=Lax`, `Secure`, TTL 7 Tage) gegen `user_sessions` ⋈ `users`
2. CSRF: `Sec-Fetch-Site: cross-site` ablehnen; gesetztes `Origin` muss Request-Origin entsprechen
3. Autorisierung: `is_admin`, Owner (`user_id`) oder `can_posts` / `can_methods` / `can_categories`
4. Events anlegen: jeder authentifizierte Account (kein `can_events`-Gate auf API-Ebene)

## 3. i18n und öffentliche Oberflächen

| Modus | Beispiele | Sprachwechsel |
| :--- | :--- | :--- |
| Toggle | `/`, `/blog`, `/events` | Gleiche URL, UI-Locale |
| Navigate | `/favoriten` ↔ `/favorites`, Site Pages | Partner-URL |
| Detail | `/methoden/{slug}`, `/methods/{slug}`, `/blog/{slug}`, `/events/{slug}` | Partner-URL der Ressource |

Statische Rechtstexte (About, Datenschutz, Impressum, Kontakt) liegen als Content, **nicht** in D1. Die Sitemap (`/sitemap.xml`) mischt Registry-URLs mit zur Laufzeit veröffentlichten D1-Einträgen.

Favoriten bleiben clientseitig. Legacy-Pfade `/praxis` und `/practice` leiten per 301 auf die Gespeichert-Seiten.

## 4. D1-Datenmodell

Abstraktes Relationenmodell. Keine Seed-Zeilen, keine Dumps.

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
    int can_posts
    int can_methods
    int can_categories
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

### Semantik (ohne DDL-Skript)

| Thema | Regel |
| :--- | :--- |
| IDs | Nutzer, Events, Posts, Kategorien: Text/UUID. Methoden: stabile **Integer**-IDs (Favoriten und Related Methods). |
| Bilingual | `_de` / `_en` Spalten, keine Locale-Tabellen. |
| Status | Inhalte: `draft` \| `scheduled` \| `published` \| `rejected` (Events ohne `scheduled`). |
| Trust | `draft` (Review), `approved` (kann direkt publizieren), `verified`. Unabhängig von `can_*`. |
| Related Methods | `blog_posts.related_method_ids` ist eine serialisierte ID-Liste, kein Join-Table. |
| Kategorien | Löschen blockiert, solange Posts oder Methoden referenzieren. |
| Auth-Tokens | Magic-Link speichert **Hash**, nicht Klartext; Einmalnutzung über `used_at`. |

Nützliche Index-Familien: `(status, post_date)` und `user_id` / `category_id` auf Posts und Methoden; Slugs und `(status, publish_at)` auf Methoden.

## 5. Sicherheitsarchitektur (Ist)

```mermaid
flowchart TB
  Req[Incoming_Request] --> CSRF[SameOrigin_CSRF]
  CSRF --> Session[D1_Session_Cookie]
  Session --> RBAC[Admin_Owner_canFlags]
  RBAC --> Handler[Route_Contract]
  Handler --> Sanitize[RichText_and_Field_Limits]
  Handler --> RL[KV_RateLimit]
```

| Ebene | Vertrag |
| :--- | :--- |
| CSRF | Mutierende Routen außer dokumentierten Public-Sonderfällen |
| Session | Cookie `op_session`, Validierung gegen D1, nicht KV |
| XSS | Server-seitige Rich-Text-Allowlist; enge Längenlimits |
| HTTP | CSP, `X-Frame-Options: DENY`, `nosniff`, JSON `Cache-Control: no-store` |
| Rate-Limit | Magic Link (E-Mail, fail-open ohne KV); Kontakt (IP, fail-closed in Production ohne KV) |
| Enumeration | Unbekannte Magic-Link-E-Mail → gleiches `ok` wie bekannte |
| Kontakt | Kein D1-Insert; Honeypot-Feld wird still mit `ok` beantwortet |

Bewusst offen: kein globales API-Throttling; CSP wird manuell gepflegt.

## 6. Qualitätsgatter (ohne Pipeline-Code)

Typische Kette vor einem Production-Deploy: Dependency-Audit (high) → Unit- und Security-Coverage → Worker-Build → Bundle-/Cron-Smoke → E2E. Remote-D1-Schreiben ist betrieblich gegated (Bestätigung), getrennt von diesem Showcase.

## 7. API-Verweis

Vollständige HTTP-Verträge: [../openapi.yaml](../openapi.yaml).
