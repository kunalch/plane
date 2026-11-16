# Threat model: Plane (kunalch/plane)

_Status: drafted from the code. Facts marked **[user]** come from the user; everything else is inferred from the code and does not move finding priorities._

- **[user]** Plane is not hosted yet; the deployment model is undecided. Findings are judged as if it could be deployed in any supported way, so no attacker or surface is ruled out.

## System overview and trust boundaries
Plane is a project-management platform (work items, cycles, modules, pages, intake). Not deployed yet **[user]**; the supported deployment paths in the repo are `docker-compose.yml` / `deployments/` (cli, aio, swarm). Caddy (`apps/proxy/Caddyfile.ce`) is the single ingress and routes by path:

| Path | Upstream |
|---|---|
| `/` | web (React Router SPA, `apps/web`) |
| `/spaces/*` | space (public boards, `apps/space`) |
| `/god-mode/*` | admin (`apps/admin`) |
| `/api/*`, `/auth/*`, `/static/*` | api (Django, `apps/api`) |
| `/live/*` | live (Hocuspocus/Yjs WebSocket, `apps/live`) |
| `/{BUCKET_NAME}/*` | MinIO object storage |

Internal-only services: PostgreSQL (`plane-db`), Valkey (`plane-redis`), RabbitMQ, Celery workers (`apps/api/plane/bgtasks`). Trust boundaries: internet → Caddy; browser → API (session cookie + CSRF) or API client → `/api/v1` (X-Api-Key); live server → API (forwards user cookie); workers → external URLs (webhooks, link previews); API/browser → object storage.

## Components
| Component | Code |
|---|---|
| Web client | `apps/web`, `apps/space`, `apps/admin`, `packages/*` (editor, ui, services) |
| App API | `apps/api/plane/app`, `apps/api/plane/db`, `apps/api/plane/utils` |
| Public API | `apps/api/plane/api` (`/api/v1`, X-Api-Key) |
| Public Spaces | `apps/api/plane/space`, `apps/space` |
| Authentication | `apps/api/plane/authentication` (email/password, magic code, Google/GitHub/GitLab/Gitea OAuth, sessions) |
| Instance admin | `apps/api/plane/license` (God Mode instance config) + `apps/admin` |
| Live collaboration | `apps/live` (Hocuspocus, PDF export) |
| Background workers | `apps/api/plane/bgtasks` (webhooks, link previews, exports, emails, imports) |
| Infrastructure | `apps/proxy`, `docker-compose*.yml`, `deployments/` |

(Where Web client code is specific to Spaces or God Mode, the more specific component wins.)

## Entry points and who can reach them
- Unauthenticated: `/auth/*` sign-up/sign-in/magic code/password reset/OAuth callbacks; instance setup endpoints before the first admin exists; public Spaces (`/api/public/...` via `plane/space`) for projects published as deploy boards; MinIO bucket path (objects are `public-read`, `AWS_QUERYSTRING_AUTH=False`).
- Authenticated session (any user): workspace/project APIs under `plane/app`, gated by `ROLE` (Admin 20 / Member 15 / Guest 5) in `app/permissions/{workspace,project,page}.py`; restricted guests see only their own work items.
- API token holders: `/api/v1` (`plane/api`, `APIKeyAuthentication`).
- Live: WebSocket `/live` — `lib/auth.ts` takes cookie + userId from the client token and validates them against the API; only `project_page` documents.
- Instance admins: `/god-mode` and `plane/license` (`InstanceAdminPermission`), separate `admin-session-id` cookie.

## Assets and sensitivity
- Work item, page, comment and attachment content (confidential business data).
- Accounts, sessions (`plane.db.models.session`), API tokens, OAuth links, password-reset / magic codes.
- Instance secrets: `SECRET_KEY`, SMTP credentials, OAuth client secrets, S3/MinIO keys (env + instance configuration in DB).
- Uploaded files in object storage.
- Internal network reachable from api/workers (Postgres, Valkey, RabbitMQ, MinIO, cloud metadata).

## Attacker profiles (inferred; none ruled out by the user)
1. Unauthenticated internet user, including visitors to public Spaces.
2. Authenticated guest / low-privilege member escalating within a workspace (role bypass, restricted-guest bypass, IDOR across projects).
3. Member of one workspace reaching another workspace's data (cross-workspace IDOR via slug/UUID).
4. Holder of a stolen/low-scope API token.
Instance (God Mode) admins: trust level not confirmed by the user.

## Existing controls
- Session auth with HttpOnly cookies, Secure when served over https; CSRF with `CSRF_TRUSTED_ORIGINS`; separate admin session cookie with 1h age (`settings/common.py`).
- Startup refuses/warns on known insecure `SECRET_KEY` and placeholder MinIO secret.
- Role checks via permission classes (`app/permissions`), restricted guest handling.
- SSRF defenses: `utils/ip_address.py` (blocked IP ranges), `utils/url_security.py` (DNS-pinned fetch, redirect validation), `WEBHOOK_ALLOWED_HOSTS`; used by webhooks and `work_item_link_task.py`.
- Auth rate limiting (`authentication/rate_limit.py`), DRF anon throttling, API rate limit (`api/rate_limit.py`).
- Live server limits client-openable document types.
- Request body size limits (Caddy `FILE_SIZE_LIMIT`, `middleware/request_body_size.py`).

Notable defaults: `ALLOWED_HOSTS="*"`, CORS allows all origins when `CORS_ALLOWED_ORIGINS` is unset (with credentials), Caddy `trusted_proxies` defaults to `0.0.0.0/0` (client IP spoofable via X-Forwarded-For), uploads bucket is `public-read`.

## Out of scope / accepted risk
- Nothing confirmed by the user. `SECURITY.md` (upstream Plane disclosure policy) lists as out of scope: MITM or physical device access; content spoofing / text injection without HTML/CSS control; email spoofing; missing DNSSEC, CAA or CSP headers; missing Secure/HttpOnly flags on non-sensitive cookies.

## Open questions
- Planned deployment model (internet-facing vs internal-only vs multi-tenant)?
- Which attackers matter; are instance admins trusted?
- Most critical assets?
- Is the SECURITY.md out-of-scope list accepted?
