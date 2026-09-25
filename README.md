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
                                        │  GET + /api/v1/* + secret, else 403 (fail-closed,
                                        │    including when the secret is unset)
                                        │  in-request cache: Souin (RFC 7234), bounded Otter
                                        │    store, opt-in per-handler via Cache-Control
                                        ▼
                              news-backend on 127.0.0.1:8080 (loopback-bound)

Operator ── basic_auth ──► diagnostics / scraper / testscraper / my-bookmarks
                                        ▼
                              respective services on 127.0.0.1, credential stripped
```

Every hostname resolves to Cloudflare, so Caddy only ever sees Cloudflare as the
client. `robots.txt` is answered on every site without auth.

Full trust-chain detail, invariants, and the reasoning behind the perimeter-only
auth model live in `news-backend`'s `SECURITY.md` — this repo is the
enforcement layer that document describes; that document is the design record.

## The cache layer

`backend.mostreadnews.com` runs a custom Caddy build
([`caddyserver/cache-handler`](https://github.com/caddyserver/cache-handler)
+ the [Otter](https://github.com/darkweak/storages/tree/main/otter) storage module)
rather than the stock package binary. Caching is fail-closed: nothing is cached
unless the upstream Spring Boot handler explicitly sets `Cache-Control`. See
`news-backend`'s docs for which endpoints opt in.

**The storage must be bounded, which is what the Otter module is for.** The cache
key is the method, host, path and *raw query string*
(`GET-https-backend.mostreadnews.com-/api/v1/tags/top?range=this-week`), and the
frontend Worker forwards any query string it receives. Without a storage module,
cache-handler falls back to Souin's plain in-process map, which never evicts: an
expired entry stops being served but stays in memory until Caddy restarts. So
every distinct URL ever cached (each hourly time-window URL, and any `?x=1` a
caller appends) was held for the life of the process, on a host with no swap.
Otter caps the store at `size` entries (two per cached response) and evicts
expired and least-used entries first.

Cache TTLs are **boundary-aligned**: the backend sets each cacheable response's
`max-age` to expire at the hourly data-refresh boundary (~:15, once the scraper
and scoring jobs have settled), not a rolling hour from when it was cached. A
side effect is that every key expires at the *same instant*, which makes the
global `stale 1h` directive load-bearing — it serves stale-while-revalidate so
that synchronized expiry becomes one background revalidation per key rather than
a thundering herd on Postgres at :15 each hour. Don't drop `stale` for
"freshness" without accounting for that.

Considered and not enabled: Souin's `key { sort_query }` (v1.7.9+). It would fold
parameter-order variants into one key, but it sorts the *values* of a repeated
parameter too, which is only correct while every multi-valued API parameter is a
set. That holds today (`sourceId`, `excludeSource`); a bounded store makes the
remaining benefit too small to take on that constraint.

**Operational note: a `dnf update` takes every domain down.** The custom binary
overwrites the one the `caddy` RPM owns, so a routine `dnf update` reinstalls the
stock binary, and a stock Caddy **rejects this Caddyfile outright**
(`unrecognized global option: cache`, verified 2026-09-25). On the next restart
Caddy does not come up, and all five domains go down with it. This is not a
fail-safe loss of caching. Keep the package out of updates:

```bash
echo 'exclude=caddy' | sudo tee -a /etc/dnf/dnf.conf
```

and bump Caddy only by rebuilding, as below.

## The admin API

The admin API listens on the Unix socket `/var/lib/caddy/admin.sock`, not on TCP
`127.0.0.1:2019`. On TCP, any process on the host could `POST /load` a new
configuration without auth, dropping the secret check and every `basic_auth`
with it. The socket sits in the `caddy` user's own home (`0750`), so only that
user and root can reach it. `sudo systemctl reload caddy` is unaffected, because
`caddy reload` takes the admin address from the Caddyfile it loads. Caddy logs
`admin endpoint on open interface; host checking disabled` at startup; that is
expected for a socket, where host checking does not apply. For a manual query:

```bash
sudo -u caddy curl -s --unix-socket /var/lib/caddy/admin.sock http://localhost/config/
```

## Rebuilding the Caddy binary

Pinned so the build is reproducible. Latest of each as of 2026-09-25:

| Component | Version | Notes |
|---|---|---|
| Caddy | v2.11.4 | latest release |
| `caddyserver/cache-handler` | v0.17.0 | brings Souin v1.7.9 and `storages/core` v0.0.20 |
| `darkweak/storages/otter/caddy` | v0.0.20 | resolves `storages/otter` v0.0.19, identical code to v0.0.20 |
| Go | 1.27.1 | |
| xcaddy | v0.4.7 | |

The storage module must match cache-handler's `storages/core` line: pick the
`otter/caddy` release published alongside the cache-handler you build.

**Build off the VPS.** A Caddy build needs 1–2 GB of RAM, and the host has a few
hundred MB free next to three JVMs, with no swap. Build on a workstation with a
checksum-verified Go toolchain unpacked into the build folder (no system install).
This produces a static `linux/amd64` binary with xcaddy's default tags
`nobadger,nomysql,nopgx`, the same shape as the one in production. Docker Desktop's
VM crashed twice mid-build on 2026-09-25, which is why this is not a
`docker run golang` recipe.

```bash
mkdir -p ~/caddy-build && cd ~/caddy-build
curl -sSLO https://go.dev/dl/go1.27.1.linux-amd64.tar.gz
sha256sum go1.27.1.linux-amd64.tar.gz   # compare with https://go.dev/dl/
tar -xzf go1.27.1.linux-amd64.tar.gz && rm go1.27.1.linux-amd64.tar.gz
export GOROOT=$PWD/go GOPATH=$PWD/gopath GOCACHE=$PWD/gocache CGO_ENABLED=0 GOFLAGS=-p=4
export PATH=$GOROOT/bin:$GOPATH/bin:$PATH
go install github.com/caddyserver/xcaddy/cmd/xcaddy@v0.4.7
xcaddy build v2.11.4 \
  --with github.com/caddyserver/cache-handler@v0.17.0 \
  --with github.com/darkweak/storages/otter/caddy@v0.0.20 \
  --output ./caddy
./caddy list-modules --versions | grep -E 'cache|otter'   # expect cache v0.17.0, storages.cache.otter v0.0.20
```

## Swapping the binary in

A new binary needs a `restart`, not a `reload`. That empties the in-memory cache
and briefly drops all five domains, so do it inside the backend's deploy window
(XX:15–XX:50) and not while a backend deploy is running: its drain step calls
`diagnostics.mostreadnews.com`.

**The first rollout of this Caddyfile must be a restart as well**, and the binary
and the Caddyfile go in together. The running Caddy listens for admin requests on
TCP 2019, while `reload` would look for the new socket, and the new `otter` block
needs the new binary. Validate the pair before touching either:

```bash
scp ~/caddy-build/caddy hetzner-news:caddy.new
scp Caddyfile hetzner-news:Caddyfile.new
ssh hetzner-news
sudo ~/caddy.new validate --config ~/Caddyfile.new   # sudo: the basic_auth imports
sudo cp /usr/bin/caddy ~/caddy.prev.bak && sudo cp /etc/caddy/Caddyfile ~/Caddyfile.prev.bak
sudo install -m 0755 ~/caddy.new /usr/bin/caddy
sudo install -m 0644 ~/Caddyfile.new /etc/caddy/Caddyfile
sudo systemctl restart caddy && systemctl is-active caddy
sudo journalctl -u caddy --since '2 min ago' | grep -E 'otter.storage.size|admin endpoint started'
```

Then confirm, from anywhere:

```bash
curl -sI https://mostreadnews.com/api/v1/time-range-options | grep -i cache-status   # 200, Souin
curl -s -o /dev/null -w '%{http_code}\n' https://diagnostics.mostreadnews.com/robots.txt  # 200, was 401
curl -s -o /dev/null -w '%{http_code}\n' https://diagnostics.mostreadnews.com/            # 401
```

Rollback: reinstall `~/caddy.prev.bak` and `~/Caddyfile.prev.bak` the same way
and `sudo systemctl restart caddy`.

Once it is confirmed live, push this repo, then bring the `/etc/caddy` checkout
level with it. Until this rollout that checkout carried an uncommitted
`caddy fmt` whitespace diff. The new Caddyfile is already `fmt`-clean and
byte-identical to `origin/main`, so the reset changes nothing Caddy reads:

```bash
sudo git -C /etc/caddy fetch origin
sudo git -C /etc/caddy diff --stat origin/main -- Caddyfile   # expect no output
sudo git -C /etc/caddy reset --hard origin/main
```

While on the host, it's worth tightening the password files. They hold bcrypt
hashes and are world-readable today; Caddy only needs its own group:

```bash
sudo chown root:caddy /etc/caddy/.monitor-password /etc/caddy/.my-bookmarks-password
sudo chmod 0640 /etc/caddy/.monitor-password /etc/caddy/.my-bookmarks-password
```

## Deploying a Caddyfile change

There's no CI/CD here — this is a single Caddyfile on a single host. The flow is:

1. Edit `/etc/caddy/Caddyfile` on the VPS directly (this repo is version
   control for change history and rollback, not a deploy pipeline).
2. Format and validate before reloading (`sudo`, because once the password files
   are `0640 root:caddy` only root and Caddy can read the `basic_auth` imports):
   ```bash
   sudo caddy fmt --overwrite /etc/caddy/Caddyfile
   sudo caddy validate --config /etc/caddy/Caddyfile
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
