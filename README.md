# php-base

Custom PHP base Docker images, built from upstream PHP repositories and pushed to GHCR as `ghcr.io/farhan15r/php-base`.

- **Sources**: [Remi](https://rpms.remirepo.net/) (Rocky Linux 10, 9) and [Ondřej Surý](https://deb.sury.org/) (Debian trixie, Ubuntu noble)
- **PHP versions**: 7.4 (Rocky); 7.0, 7.2, 7.4 (Debian trixie); more planned (8.1, 8.3, 8.4, …) — one Dockerfile per source/distro, version selected via build-arg
- **Variants**: `fpm` and `apache`, built from the same multi-stage Dockerfile via `--target`

> **Why this exists**: this project targets PHP versions that are no longer supported — either by PHP itself (EOL) or by the distro's own repositories — so it relies on third-party backport repos (Remi for Rocky Linux, Ondřej Surý for Debian/Ubuntu) to still get these versions with security patches.

## Status

| Source | Distro | Dockerfile | Status |
| --- | --- | --- | --- |
| Remi | Rocky Linux 10, 9 | [`remi/rocky/Dockerfile`](remi/rocky/Dockerfile) | ✅ built & tested (PHP 7.4) |
| Ondřej | Debian trixie | [`ondrej/debian/Dockerfile`](ondrej/debian/Dockerfile) | ✅ built & tested (PHP 7.0, 7.2, 7.4) |
| Ondřej | Ubuntu noble | `ondrej/ubuntu/Dockerfile` | not built yet |

CI (`.github/workflows/build.yml`) builds and pushes to GHCR on push to `main`, monthly (to pick up the latest backports), or on demand — matrix currently covers the five source/distro/PHP-version combinations above; not yet verified against a real GHCR push. See [AGENTS.md](AGENTS.md) for the full design rationale, tag scheme, and remaining steps.

## Building locally

Each Dockerfile is multi-stage (`base` → `fpm` | `apache`), so pick a variant with `--target`:

```sh
# FPM variant
docker build --target fpm \
  --build-arg ROCKY_VERSION=10 \
  --build-arg PHP_VERSION=7.4 \
  -t php-base:7.4-fpm-rocky remi/rocky

# Apache variant
docker build --target apache \
  --build-arg ROCKY_VERSION=10 \
  --build-arg PHP_VERSION=7.4 \
  -t php-base:7.4-apache-rocky remi/rocky

# Ondřej/Debian, Apache variant
docker build --target apache \
  --build-arg DEBIAN_CODENAME=trixie \
  --build-arg PHP_VERSION=7.4 \
  -t php-base:7.4-apache-trixie ondrej/debian
```

Smoke test:

```sh
docker run --rm php-base:7.4-fpm-rocky php -v
docker run --rm php-base:7.4-apache-rocky httpd -M | grep -i php
docker run --rm php-base:7.4-apache-trixie apache2ctl -M | grep -i php
```

Every image records its installed PHP-related packages at `/etc/php-packages.txt`.

## Extensions

Each Dockerfile accepts two build-args to control installed extensions:

- `PHP_EXTENSIONS` — default set: `cli mbstring opcache xml curl zip intl bcmath pdo` (covers what's near-universal across frameworks/CMSs — Composer, Laravel, Symfony, WordPress-class apps)
- `PHP_EXTRA_EXTENSIONS` — additional, app-specific extensions (e.g. a DB driver: Remi's `mysqlnd`, Ondřej's `mysql`)

```sh
docker build --target fpm \
  --build-arg PHP_EXTRA_EXTENSIONS="redis gd" \
  -t php-base:7.4-fpm-rocky-extra remi/rocky
```

## Production hardening

Applied by default in both `remi/rocky/Dockerfile` and `ondrej/debian/Dockerfile` — no build-arg needed, no extra flags to remember:

| Area | Setting | Effect |
| --- | --- | --- |
| PHP | `expose_php = Off` | No PHP version in the `X-Powered-By` header |
| PHP | `display_errors = Off` / `display_startup_errors = Off` | Errors never leak into the response body or stdout |
| PHP | `log_errors = On` | Errors still land in `docker logs` (via stderr/stdout redirection) |
| PHP | `opcache.validate_timestamps = 0` | No per-request filesystem stat (safe since the image is immutable — a code change needs a rebuild, not a hot-reload) |
| PHP | `opcache.memory_consumption` / `interned_strings_buffer` / `max_accelerated_files` | Explicit OPcache sizing instead of distro defaults |
| FPM | `clear_env = no` | Env vars passed via `docker run -e` actually reach `getenv()` (FPM's default wipes them) |
| FPM | `catch_workers_output = yes` | Worker stdout/stderr reaches `docker logs` |
| FPM | `security.limit_extensions = .php` | FPM refuses to execute anything but `.php` |
| Apache | `ServerTokens Prod` / `ServerSignature Off` | `Server` header reduced to just `Apache` (no version, OS, or PHP) |
| Apache | `TraceEnable Off` | `TRACE` requests rejected (405) |
| Apache | `Options -Indexes` | No directory listing |
| Apache | `autoindex`/`userdir`/`dav*`/`cgi`/`status` modules disabled | Smaller attack surface; default welcome/test page removed |

> **Note**: `opcache.validate_timestamps = 0` means PHP stops checking file mtimes on every request — it caches compiled opcode and never looks at the source file again until the process restarts. This makes the image effectively **immutable at runtime**: if you mount live code over `/var/www/html` for local dev, edits won't be picked up without restarting the container. It's safe here because this is a build-time-baked image (a code change means rebuilding the image), but don't rely on this setting for a hot-reload dev workflow.

See [AGENTS.md](AGENTS.md#production-hardening) for the rationale and gotchas behind each of these (e.g. why removing `autoindex.conf` matters once `mod_autoindex` is disabled).

### Apache: `.htaccess` works out of the box

Both Apache variants ship `mod_rewrite` enabled and `AllowOverride All` on `/var/www/html` — the Rocky/httpd and Debian/apache2 upstream defaults are `AllowOverride None` (and Debian also disables `mod_rewrite` by default), which silently ignores `.htaccess` entirely with no config-syntax error to reveal it. This is pre-configured so framework/app `.htaccess` rules (Laravel, WordPress, etc.) just work without extra setup.

## Tags (planned)

| Tag | Example | Behavior |
| --- | --- | --- |
| immutable | `7.4.33-apache-trixie-20261022` | never overwritten, used in production |
| patch | `7.4.33-apache-trixie` | moving |
| minor | `7.4-apache-trixie` | moving |

GHCR does not support true immutable tags, so consumers should pin images with the `@sha256:` digest rather than relying on the immutable tag alone — see [AGENTS.md](AGENTS.md#ghcr) for the mitigations in place.

## Contributing / working on this repo

See [AGENTS.md](AGENTS.md) for design decisions, per-source technical notes, and the current list of remaining steps.
