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
- **`systemd/caddy.service.d/override.conf`** — the unit drop-in, live at
  `/etc/systemd/system/caddy.service.d/`. See [Crash resilience](#crash-resilience).
- **`patches/`** — source patches the custom build applies. See
  [The Souin patch](#the-souin-patch).

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

**That cap bounds the cache and nothing beside it, so surrogate keys are off.**
Souin also indexes every stored response by surrogate-key tag, for its purge API.
The backend sends no `Surrogate-Key` header, so every response lands in one
global tag plus one per path, and with a storage that has no native sets (Otter
has none) each tag is a single comma-joined string: every store reads it, scans
it, appends one key and writes it back, under one global mutex. Evicting the
entry does not remove its key, and every append renews the tag's TTL, so the
string grows with every distinct URL ever cached. Measured locally on
2026-09-30 with the production binary caching unique URLs: 18,834 goroutines
queued on that mutex and 1.3 GB within seconds, still growing after the load
stopped, against ~200 MB and ~85 goroutines with `disable_surrogate_key`.
Nothing here purges by tag, so the index had no consumer. Re-enable it only
together with a storage that implements `core.SetStorer`.

**Souin has its own upstream timeout, 10 s by default** (`timeout.backend`, not
set here). A request past it gets Souin's `504` with the body
`Internal server error`, whatever the backend would have answered. A cold
category feed took 7.4 s on 2026-09-30, so that ceiling is closer than it looks;
raise it in the `cache` block before a slow endpoint meets it, not after.

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

## Crash resilience

On 2026-09-30 at 01:41 UTC Caddy exited with a Go runtime
`fatal error: concurrent map iteration and map write` and stayed down for almost
three hours: the unit had `Restart=no`, and a fatal error, unlike a panic, cannot
be recovered inside the process. The cause was a Souin bug, now patched in the
build (see [The Souin patch](#the-souin-patch)). The unit no longer depends on
that being the last one.

`systemd/caddy.service.d/override.conf` is the drop-in, live at
`/etc/systemd/system/caddy.service.d/override.conf`:

- **`Restart=on-failure` with no start limit.** The default limit (5 starts per
  interval) turns a crash that can be triggered from outside into a unit that
  stays stopped once it is triggered five times. A binary or Caddyfile that
  cannot start (the `dnf update` case above) now retries every 15 seconds
  instead, which costs nothing. The delay backs off 2 s → 4 s → 8 s → 15 s and
  stays at 15 s until the next manual start resets the count.
- **`GOTRACEBACK=all`.** A concurrent-map fatal error prints only the goroutine
  that detected it. `all` prints every goroutine, including the one that caused
  it, which is what an upstream report needs.

Install or update it:

```bash
scp systemd/caddy.service.d/override.conf hetzner-news:caddy-override.conf
ssh -t hetzner-news 'sudo install -m 0644 ~/caddy-override.conf /etc/systemd/system/caddy.service.d/override.conf && sudo systemctl daemon-reload && rm ~/caddy-override.conf'
```

`daemon-reload` applies the restart policy to the running process; the
environment variable only reaches the next start.

**An automatic restart hides the outage, not the crash.** Each one leaves the
full trace in the journal, and systemd counts them:

```bash
systemctl show caddy -p NRestarts               # since the last manual start
journalctl -u caddy | grep -E 'fatal error|panic:|Scheduled restart job'
```

The journal is persistent since 2026-09-30 (`/var/log/journal`). Before that it
lived in `/run`, capped at about 70 MB: roughly five days of history, lost on
every reboot, which is why nothing earlier than 2026-09-25 survives.

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
| `darkweak/souin` | v1.7.9 **+ `patches/`** | replaced with a patched checkout, see below |
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

# Souin v1.7.9 with the patch applied (path to this repo's patches/ directory):
git clone -q --branch v1.7.9 https://github.com/darkweak/souin souin
git -C souin am <news-infra>/patches/souin-v1.7.9-wait-for-upstream-on-cancel.patch
(cd souin && go test -race -count=1 ./pkg/middleware/)

xcaddy build v2.11.4 \
  --with github.com/caddyserver/cache-handler@v0.17.0 \
  --with github.com/darkweak/storages/otter/caddy@v0.0.20 \
  --replace github.com/darkweak/souin=$PWD/souin \
  --output ./caddy
./caddy list-modules --versions | grep -E 'cache|otter'   # expect cache v0.17.0, storages.cache.otter v0.0.20
./caddy build-info | grep -A1 'darkweak/souin'            # expect the `=> .../souin (devel)` replace line
```

Compare `build-info` with the running binary's before swapping: the only
differences should be the Souin line and its `=>` replace.

### The Souin patch

`patches/souin-v1.7.9-wait-for-upstream-on-cancel.patch` fixes the crash in
[Crash resilience](#crash-resilience). Souin runs each cache miss's upstream
request in a goroutine and returns as soon as the request context ends, whether
the client disconnected or Souin's own backend timeout fired. It returns while
that goroutine may still hold the live response header map, which
`CustomWriter.Header()` handed out before the context ended. Caddy then writes
the error status, which iterates the same map, and `reverse_proxy` copying the
upstream's headers into it is a concurrent map write. That is a fatal error for
the whole process. With the patch, `ServeHTTP` waits for the goroutine first; it
observes the same context, so the wait is short.

This is not a timing rarity. Through the frontend Worker any reader can cancel
requests, and the unpatched binary fell over within about a second under a local
HTTP/2 stress run that aborted requests around the moment the upstream answered.
The patched build ran 750,000 such requests, including 94,000 cancellations,
flat at ~200 MB. The patch carries two regression tests, and without the fix the
first reproduces the production fatal error itself.

**Drop the patch** once a Souin release contains an equivalent fix: then build
without `--replace`, and delete the file.

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
