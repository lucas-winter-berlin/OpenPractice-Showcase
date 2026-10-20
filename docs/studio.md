# Studio

Editorial model as built in the private app. No handler source here.

## Access

1. Magic-link login creates an HttpOnly `op_session` cookie (7-day TTL), validated against D1.
2. `/studio` and the record pages require that session.
3. Tabs and write routes use grant flags on `users`. Admin (`is_admin`) bypasses flags and owner checks.
4. Non-admins only load and mutate their own rows.

Show-tab is the master switch. A tab is visible when the show flag is on and at least one action flag is on. Analytics is view-only (`can_analytics`).

| Column | Meaning |
| :--- | :--- |
| `can_events` / `_create` / `_edit` / `_publish` | Events tab and actions. Events have no `scheduled` status. Publish also covers going live now. |
| `can_posts` / `_create` / `_edit` / `_publish` | Posts tab and actions. Publish also allows `publish_at`. |
| `can_methods` / `_create` / `_edit` / `_publish` | Methods tab and actions. Same schedule rule as posts. |
| `can_categories` / `_create` / `_edit` | Categories. No publish flag. |
| `can_analytics` | Analytics tab. |

Saving a show-tab as off stores that feature's action flags as 0. Saving an admin stores every flag as 0.

Category list access is also open when the posts or methods tab is visible. The published-methods catalog (related-method picker) is open to posts create or posts edit.

## Status

| Kind | Statuses |
| :--- | :--- |
| Posts, methods | `draft` \| `scheduled` \| `published` \| `rejected` |
| Events | `draft` \| `published` \| `rejected` |

Rules:

- Admin keeps the requested status. Only an admin may set `rejected`.
- With the publish flag: posts/methods may be `published` or `scheduled`; events may be `published`. Voluntary `draft` is still allowed.
- Without the publish flag: the API stores `draft`. That is not a `403`.
- `publishAt` is Berlin `YYYY-MM-DDTHH:MM`. It is ignored unless the caller may schedule.
- The hourly cron publishes due posts and methods. It does not publish events.

Trust level (`draft` / `approved` / `verified`) is independent of `can_*`.

## Notifications

Delivery: Resend, else optional webhook, else console (redacted). Post and event API mail is non-blocking: a send error is logged and the write still returns `200`. Methods have no notification kind.

| Kind | When | Recipients |
| :--- | :--- | :--- |
| `magic_login_link` | Magic-link request | The address on the user row |
| `contact_form` | Public contact | Contact inbox |
| `post_review_needed` / `event_review_needed` | Create stored as draft (no publish flag) | Admins |
| `post_resubmitted` / `event_resubmitted` | Non-admin update still draft | Admins |
| `post_published` / `event_published` | Published now, or hourly post publish | Admins and owner |
| `post_rejected` / `event_rejected` | Admin reject | Owner |

## Translate

`POST /api/studio/translate` sends only the empty target fields. Resource is `methods`, `posts`, or `events`. The viewer needs create or edit on that resource. Rate limit: 10 requests / 10 minutes / user in KV. Missing AI or KV → `503`. The route does not write D1.

## Aggregate

`GET /api/studio.json` returns `viewer`, `isAdmin`, `users`, `events`, `posts`, `categories`, `methods`, `methodsCatalog`.

| Field | Admin | Not admin |
| :--- | :--- | :--- |
| `users` | All | `[]` |
| `events` / `posts` / `methods` | All | Own rows when that tab is visible, else `[]` |
| `methodsCatalog` | Published methods | Only with posts create or edit, else `[]` |
| `categories` | Full list | When posts, methods, or categories tab is visible, else `[]` |

Empty flags: `200` with empty lists, not `403`.
