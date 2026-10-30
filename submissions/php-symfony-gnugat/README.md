# PHP + Symfony (FrankenPHP worker mode)

A regular, idiomatic Symfony application, the kind you'd write at work, tuned only through configuration and standard
Symfony components. No custom HTTP server, no hand-written JSON, no group commit.

| | |
|---|---|
| Language | PHP 8.5.11 (ZTS), built into [FrankenPHP's release binary](https://github.com/php/frankenphp/releases/tag/v1.13.0) |
| Framework | Symfony 8.1 (FrameworkBundle 8.1.8, SecurityBundle, JsonStreamer, Runtime for the console) |
| Server | FrankenPHP 1.13.0 (Caddy 2.11.7) in worker mode, with the worker loop of FrankenPHP's docs in `public/index.php` |
| SQLite driver | PDO SQLite (SQLite 3.45.2) via Doctrine DBAL 4.5 (DoctrineBundle 3.3) |
| **Nginx or direct** | **Direct**: FrankenPHP serves `HOST:PORT` itself (`0.0.0.0:80` on the droplet) |

FrankenPHP and Composer are pinned and checksum-verified in `install.sh`, PHP dependencies in `composer.lock`.

## Running it

```bash
sudo bash install.sh   # FrankenPHP's release binary (PHP built in) and Composer, pinned and checksum-verified
bash build.sh          # composer install --no-dev --classmap-authoritative, dump-env prod, cache:warmup
SQLITE_PATH=... JWT_SECRET=... HOST=0.0.0.0 PORT=80 bash start.sh
```

## The application

- **Routing, controllers**: attribute routes on one `AbstractController`, a `#[ValueResolver]` for post ids (so
  `/posts/abc` is a 400, not a 404), a `kernel.exception` listener for the `{"error":...}` bodies.
- **Auth**: SecurityBundle, a stateless firewall with the `access_token` authenticator, matching only `POST ^/posts`:
  reads never go through it. The JWT is checked on every request by an `AccessTokenHandlerInterface`: HS256 signature
  (`hash_hmac` + `hash_equals`), `exp` and, when present, `nbf`. It's hand-written because the challenge's 23-byte
  secret is shorter than RFC 7518's 256-bit minimum for HS256, which the maintained JWT libraries enforce.
- **Input**: `CreatePostPayload::fromRequest()` decodes and checks `POST /posts`'s body (`json_decode()`, `trim()`,
  `mb_strlen()`), a 400 `BadRequestHttpException` with the spec's message otherwise.
- **Reads**: a read model. `GET /feed` and `GET /posts/{id}` are one SQL query each, through DBAL, and SQLite writes
  the whole response body (`json_object()`, `json_group_array()`), which is returned as is.
- **Writes**: one SQL statement each, through DBAL, autocommitted before the response. A like is
  `INSERT ... SELECT ... ON CONFLICT DO NOTHING` (a second query only tells 404 from 200), a post is
  `INSERT ... RETURNING`. Their responses are written by JsonStreamer from `#[JsonStreamable]` output objects.
  No ORM: the repositories get DBAL's `Connection`.
- **SQLite**: WAL (set in the database file), and a DBAL driver middleware sets `synchronous=NORMAL` and
  `busy_timeout` on each connection. Connections are persistent (`PDO::ATTR_PERSISTENT`), one per PHP thread.
- **Prepared statements are reused** (rule 5: they cache the query plan, not data) by another DBAL driver middleware,
  pooled per SQL string for the life of the driver connection. A statement is checked out while in use, so two users
  of the same SQL never share one, and goes back only once its results are gone, its cursor reset: an SQLite statement
  left mid-result keeps the connection's read snapshot open, and other queries would read stale data.
- **No data is cached across requests**: no Doctrine result cache, and Symfony resets its services between requests.
  What persists across requests is SQLite's page cache and schema, and prepared statements.

## Optimizations, and why

Each change was measured on its own, in both FrankenPHP modes (classic: Symfony boots on every request, like with
PHP-FPM; worker: once per worker), with one commit per change. The method, every run and a droplet estimate are in
[`local-bench/LADDER.md`](local-bench/LADDER.md); profiles of the main steps are in `local-bench/PROFILE-p*.md`.

The local stand-in pins the server to one core of an Apple silicon Mac with 2 GB (no swap) in Docker, with k6 on
other cores. **Its user counts are not droplet scores**: one core there is much faster than the droplet's vCPU. They
only compare with each other.

| Phase | Change | Classic | Worker | Helps |
|---|---|--:|--:|---|
| 0 | Idiomatic Symfony 8.1, FrankenPHP and PHP defaults | 8,500 | 23,000 | |
| 1 | php.ini production settings (OPcache, realpath) | 8,500 | 23,000 | none |
| 2 | Preloading | 9,750 | 23,500 | classic |
| 3 | `classmap-authoritative` | 9,750 | 23,500 | none |
| 4 | JIT (tracing) | 11,000 | 33,000 | both |
| 5 | DQL read model | 13,500 | 52,000 | both |
| 6 | JsonStreamer | 15,750 | 65,500 | both |
| 7 | DBAL read model | 15,250 | 72,750 | both |
| 8 | Firewall on `POST /posts` only | 17,250 | 73,250 | classic |
| 9 | API-only routing in the Caddyfile | 17,250 | 75,750 | worker |
| 10 | Prepared-statement reuse | 17,000 | 76,750 | worker |
| 11 | Worker mode by default | 17,250 | 76,750 | |
| 12 | SQLite writes the reads' JSON | 17,500 | 77,500 | both, a little |
| 13 | Persistent SQLite connections | 29,500 | 77,500 | classic |
| 14 | DBAL writes | 29,500 | 79,000 | worker (less memory) |
| 15 | `max_threads 8` | 30,250 | 79,500 | both, within noise |

Worker rows from phase 6 are measured with 200,000 open files: the rules' 65,535 stop worker mode near 65,500 users
(one keep-alive connection per user), whatever its CPU.

1. **Worker mode** (phase 11): Symfony boots once per worker instead of once per request, ×2.6 users at phase 15.
   FrankenPHP's defaults are kept (2 workers, restarted every 500 requests): 1 worker, 4 workers
   and never restarting were within noise or worse.
2. **The JIT** (phase 4): worker mode's long-lived workers keep the traces hot, +40% users there.
3. **A read model instead of serializing entities** (phases 5 and 7): the `Post` entities' `like_count` was an
   extra-lazy collection, so serializing the feed ran a COUNT query per post (37% of the feed's CPU), and hydrating
   posts and authors another 16%. A DQL `NEW` expression (phase 5), then plain SQL through DBAL (phase 7), return the
   API's posts in one query.
4. **JsonStreamer instead of the Serializer** (phase 6): the Serializer was 64% of the feed's CPU after phase 5.
   JsonStreamer generates the PHP code that writes each output class at cache warmup; writing the feed's JSON cost
   81% less. Phase 12 then has SQLite write the reads' JSON itself.
5. **Persistent SQLite connections** (phase 13), mostly for classic mode (+69% users): every request opened the
   database again, the pragmas read the schema, and closing the last connection made SQLite checkpoint and delete
   the WAL, so each write started over. A CPU profile didn't show it, a wall-clock one did.
6. **DBAL writes** (phase 14): one statement per write instead of the ORM's `find()`, `findOneBy()` and `flush()`
   (BEGIN, INSERT, COMMIT); less memory in worker mode.
7. Smaller ones: preloading (classic), API-only routing (`rewrite * /index.php` + `php` instead of `php_server`,
   no `try_files` filesystem checks), the firewall only on `POST /posts`, prepared-statement reuse.
8. Direct instead of Nginx: on one vCPU, Nginx's CPU would come out of the application's budget.

Tried and not kept: `max_threads 8` (phase 15, removed after the leaderboard run, see below), 8 PHP threads all the
time in classic mode (fewer users), `zend.enable_gc=0` (one step or two, and a leaking cycle would grow a worker until
its restart), `GOMEMLIMIT=1100MiB` (fewer users: FrankenPHP already sets it to 90% of the container's memory when it's
unset), `GOGC=200` (no gain, more memory), `GODEBUG=cgocheck=0` (PR 40 sets it; no gain, +0.5% over three
throttled rounds).

Where it stops, locally: worker mode on memory from phase 6 on, about 31 KiB per open keep-alive connection, all of
it Caddy's and Go's (`net/http`'s buffers, goroutine stacks, GC headroom); PHP is about 40 MiB of it at any load.
Classic mode on CPU.

## On the leaderboard

The leaderboard's 1 vCPU scored phase 15 at 15,500 users, CPU-bound (599 µs of CPU per request), and PR 40, the
other Symfony + FrankenPHP entry, at 21,000 (428 µs). Locally, at a full core, the server's CPU sits near 50% at any
load, which hides the gap; throttled to a quarter of the core (`local-bench/compare-pr40.sh`), both saturate and PR
40 serves 35% more requests, as on the box. Since then, in the same throttled setup (requests served, against the
step before, over two rounds):

- FrankenPHP's release binary instead of the deb packages, as PR 40 (the same FrankenPHP and PHP versions), and no
  `max_threads 8` (3 PHP threads: the 2 workers, 1 for other scripts): +4%;
- phase 16, no ORM: the repositories get DBAL's `Connection`, without `doctrine/orm` and the entities: +0.5% to +2.9%;
- phase 17, the worker loop in `public/index.php` instead of `symfony/runtime`'s runner: with phase 16, +5.2% to
  +12.5% over four rounds, against +2.2% to +6.9% for each alone;
- phase 18, `CreatePostPayload::fromRequest()` instead of `#[MapRequestPayload]`, without the Serializer and the
  Validator: +2.8% to +7.8% over three rounds.

Not kept: PR 40's Caddyfile settings (`php_server` with the worker on `match *`, log level `WARN`, `expose_php 0`),
-1.9% and -5.3%.

Phases 10 and 12, within noise at a full core, were checked again this way and kept: without prepared-statement
reuse, -3.8% and -9.6%; with PHP writing the reads' JSON, -1.8% and -2.4%.

## License

MIT, under the repo's [license](../../LICENSE).
