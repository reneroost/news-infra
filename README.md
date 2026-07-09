# news-infra

Production reverse-proxy configuration for [MostReadNews](https://mostreadnews.com) —
a news aggregator tracking article popularity over time via scraped homepage
positions. This repo is the Caddy layer only: it fronts three independent
services (`news-scraper`, `news-backend`, `news-frontend`) that live in their
own repositories.

## What's in here

- **`Caddyfile`** — the production reverse-proxy config, live on the VPS at
  `/etc/caddy/Caddyfile`.
- **`backend.env.example`** — the shape of the real `backend.env` (git-ignored,
  lives only on the host). Copy it to `backend.env` and fill in a real value.

## What this repo does *not* contain

- **No secrets.** `BACKEND_SECRET` and both `basic_auth` password files are
  git-ignored and exist only on the production host. See `.gitignore`.
- **No application code.** This is routing, auth-at-the-edge, and caching
  config — not the services themselves.

## Architecture this file implements

```
Browser ── same-origin /api/v1/* ──► Cloudflare Worker (frontend repo)
                                        │  attaches X-Backend-Secret
                                        ▼
                              backend.mostreadnews.com
                                        │  GET-only + secret-gated (fail-closed)
                                        │  in-request cache: Souin (RFC 7234),
                                        │    opt-in per-handler via Cache-Control
                                        ▼
                              news-backend on 127.0.0.1:8080 (loopback-bound)

Operator ── basic_auth ──► diagnostics / scraper / testscraper subdomains
                                        ▼
                              respective services, loopback-bound
```

Full trust-chain detail, invariants, and the reasoning behind the perimeter-only
auth model live in `news-backend`'s `SECURITY.md` — this repo is the
enforcement layer that document describes; that document is the design record.

## The cache layer

`backend.mostreadnews.com` runs a custom Caddy build
(`v2.11.4` + [`caddyserver/cache-handler`](https://github.com/caddyserver/cache-handler),
in-memory storage) rather than the stock package binary. Caching is fail-closed:
nothing is cached unless the upstream Spring Boot handler explicitly sets
`Cache-Control`. See `news-backend`'s docs for which endpoints opt in.

Cache TTLs are **boundary-aligned**: the backend sets each cacheable response's
`max-age` to expire at the hourly data-refresh boundary (~:15, once the scraper
and scoring jobs have settled), not a rolling hour from when it was cached. A
side effect is that every key expires at the *same instant*, which makes the
global `stale 1h` directive load-bearing — it serves stale-while-revalidate so
that synchronized expiry becomes one background revalidation per key rather than
a thundering herd on Postgres at :15 each hour. Don't drop `stale` for
"freshness" without accounting for that.

**Operational note:** because this is a manually-swapped custom binary, a
routine `dnf update` on the host will silently reinstall the stock Caddy
package and drop the cache module (fails safe — caching just stops, nothing
opens up). Either `dnf versionlock caddy`, or rebuild via `xcaddy` after any
Caddy version bump:

```bash
xcaddy build v2.11.4 --with github.com/caddyserver/cache-handler
```

## Deploying a Caddyfile change

There's no CI/CD here — this is a single Caddyfile on a single host. The flow is:

1. Edit `/etc/caddy/Caddyfile` on the VPS directly (this repo is version
   control for change history and rollback, not a deploy pipeline).
2. Validate before reloading:
   ```bash
   caddy validate --config /etc/caddy/Caddyfile
   ```
3. Reload:
   ```bash
   sudo systemctl reload caddy
   ```
4. Commit and push from the same directory once the change is confirmed live.

## Secrets rotation

If either `BACKEND_SECRET` or a `basic_auth` credential is ever rotated
(recommended periodically, and any time a compromise is suspected — see
`news-backend`'s `SECURITY.md` residual-risks section): update the relevant
file on the host and the corresponding value in the Cloudflare Worker/dashboard.
No change is needed in this repo, since neither value is tracked here.
