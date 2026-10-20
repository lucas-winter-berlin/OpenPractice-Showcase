# Structure

How the public site, studio, and workers are laid out. No application source in this repo.

## Workers

| Worker | Role |
| :--- | :--- |
| App | Astro SSR on Cloudflare Workers. HTML pages and `/api/*`. Custom domains only. |
| Scheduled | Separate worker on the same D1. Hourly publish. Daily event draft + session/token cleanup. Not part of the app deploy. |

Bindings used by the app (names only):

| Binding | Type | Use |
| :--- | :--- | :--- |
| `op_db` | D1 | Content, users, sessions, magic-link tokens |
| `SESSION` | KV | Rate limits only (magic-link, contact, studio translate) |
| `AI` | Workers AI | Studio translate |
| `CONTENT_EVENTS` | Analytics Engine | Telemetry dataset |
| Images / ASSETS | Media and client bundle | Public pages |

Secrets stay off this repo. Typical names: mail (Resend), notify webhook, contact recipient, content-read token, analytics read token, Cloudflare account id.

## Public pages

Language mode is owned by a route registry. Page files stay thin.

| Mode | Paths | Language switch |
| :--- | :--- | :--- |
| Toggle | `/`, `/blog`, `/events` | Same URL. UI locale and cookie change. |
| Navigate | `/favoriten` ↔ `/favorites`, `/ueber-uns` ↔ `/about`, `/datenschutz` ↔ `/privacy`, `/impressum` ↔ `/imprint`, `/kontakt` ↔ `/contact` | Browser goes to the partner URL. |
| Detail | `/methoden/{slug}` ↔ `/methods/{slug}`, `/blog/{slug}`, `/events/{slug}` | Partner URL from the resource. |

Other public routes: `/login`, `/sitemap.xml`.

Legacy `/praxis` and `/practice` 301 to the saved-items pages.

Site pages (about, privacy, imprint, contact) are Markdown pairs in the app repo, not D1, and have no studio editor.

## Studio pages

Session required. Missing session redirects to login. Missing grant redirects to `/studio` or the matching tab.

| Path | Role |
| :--- | :--- |
| `/studio` | List and tabs (methods, posts, events, categories, users, analytics) |
| `/studio/methods/new`, `/studio/methods/:id` | Create / edit a method |
| `/studio/posts/new`, `/studio/posts/:id` | Create / edit a post |
| `/studio/events/new`, `/studio/events/:id` | Create / edit an event |

## HTTP surfaces

| Path family | Auth | Notes |
| :--- | :--- | :--- |
| `/api/studio.json` | Session | Aggregate for the studio UI. Empty grants return empty lists, not `403`. |
| `/api/posts`, `/api/methods`, `/api/events` | Session + CSRF | Writes. Owner or admin. Show-tab plus action flag. |
| `/api/categories` | Session; CSRF on writes | List also open when the posts or methods tab is visible. |
| `/api/users` | Admin + CSRF | Grant columns live here. |
| `/api/studio/translate` | Session + CSRF | Workers AI. Rate-limited in KV. |
| `/api/studio/analytics` | Session; admin or `can_analytics` | Read-only report. |
| `/api/auth/magic/*`, `/api/auth/logout` | Public request / session | Password login is `410`. |
| `/api/contact` | Public + CSRF | Mail only. |
| `/api/events-subscribe.ics` | Public | Calendar. |
| `/api/telemetry` | Same-origin CSRF | `204` even when the body is discarded. |
| `/api/content/{posts,methods,events}` | Bearer token | Published JSON for outside callers. |

Full contracts: [../openapi.yaml](../openapi.yaml). Edge and data model: [architecture.md](architecture.md).

## Client-only state

| Key | Store | Value |
| :--- | :--- | :--- |
| `openPathFavorites` | `localStorage` | Method ids (integers) |
| `openPathBlogFavorites` | `localStorage` | Blog ids |
| `openPathEventFavorites` | `localStorage` | `{ id, title }` |
| `openPathFavoritesReturn` | `sessionStorage` | URL after leaving the saved page |
| `openPathTheme` | `localStorage` | `system` \| `light` \| `dark` |
