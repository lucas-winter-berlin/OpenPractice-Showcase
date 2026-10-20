# Features

What the live product does. This is the public map of the private app, without source or production data.

## Public site

| Feature | Behavior |
| :--- | :--- |
| Methods catalog | Published methods on `/`. Cards can be filtered by quick-help tags: `goal` (`pressure`, `clarity`, `energy`), `context` (`people`, `home`, `transit`), `time` (`1min`, `5-10min`, `open`). |
| Method detail | `/methoden/{slug}` and `/methods/{slug}`. Description, steps, optional note and source. |
| Blog | Listing on `/blog`, detail on `/blog/{slug}`. Related method IDs may appear on a post. |
| Events | Listing on `/events`, detail on `/events/{slug}`. Date, time, city, location, price, spoken languages, optional external link. |
| Calendar feed | `GET /api/events-subscribe.ics` — published events as `VCALENDAR`. Optional filters: `lang`, `scope`, `price`, `city`, `language`, `dateFrom`, `dateTo`, `q`. |
| Favorites | Hearts on listing cards (home, blog, events) and on the saved page. Stored in the browser only. No account. No count badge on the nav heart. Not on detail pages or modals. |
| Contact | Public form (`/kontakt` / `/contact`). Emailed, not stored in D1. Honeypot field still returns `ok`. |
| Legal and about | About, privacy, imprint, contact are Markdown in the repo, not D1 rows. |
| Language | German and English. No `/de` or `/en` prefix. Toggle, navigate, or detail partner URL. |
| Theme | Light, dark, or system. Stored per browser. Default follows `prefers-color-scheme`. |
| Sitemap | `/sitemap.xml` mixes registry URLs with published D1 entries. Unknown GET paths 302 to `/`. |

## Studio (signed-in)

| Feature | Behavior |
| :--- | :--- |
| Magic-link login | Password login is gone (`410`). Email link, 15-minute hashed token, HttpOnly session cookie. |
| Editor workspace | `/studio` plus per-record pages for methods, posts, and events. |
| Grants | Admins set show-tab plus create / edit / publish per feature. Analytics is view-only. `is_admin` bypasses flags. |
| Publish | Posts and methods: now, or `publish_at` (Berlin wall time) → `scheduled`. Events: draft, published, or rejected. No scheduled events. |
| Review | Without publish permission the row is stored as `draft`. Admins get review mail. Only an admin may reject. |
| Categories | Shared by posts and methods. Delete is blocked while either still references the category. |
| Users | Admin-only create / replace / delete. Saving an admin stores every grant as 0. |
| Translate | Studio action fills empty fields in the other language via Workers AI. Does not write D1; the editor still saves. |
| Analytics | Studio report from Cloudflare Analytics Engine. Windows `24h` / `7d` / `30d`. Surfaces: methods, blog, events, page. |
| Notifications | Resend, else optional webhook `{ kind, payload }`. Posts and events notify; methods do not. |

## Background jobs

A second worker shares the same D1 database.

| Schedule | Work |
| :--- | :--- |
| Hourly | Publish due scheduled posts and methods. Posts send `post_published`. Methods do not mail. |
| Daily 03:00 UTC | Events older than five UTC days become `draft` (rows stay). Expired sessions and spent or expired magic-link tokens are deleted. |

## Integration surfaces

| Surface | Who |
| :--- | :--- |
| Same-origin HTML and `/api/*` | Browsers on the site. Session cookie plus CSRF on mutations. |
| Content-read JSON | Outside callers with a Bearer token. Published posts, methods, and events only. No session, no CORS, no drafts. |
| ICS | Anyone. Published events only. |
| Telemetry | Same-origin `POST /api/telemetry`. No visitor id. Verified bots are not written. |

HTTP contracts: [../openapi.yaml](../openapi.yaml).
