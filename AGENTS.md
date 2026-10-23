# AGENTS.md

Context for agents (Claude Code, etc.) working in this repo.

## Project goal

Build a custom PHP base image, pushed to GHCR (`ghcr.io/farhan15r/php-base`):

- Sources: **Remi** (Rocky Linux 10) and **Ondřej Surý** (Debian trixie, Ubuntu noble)
- Many PHP versions (7.4, 8.1, 8.3, 8.4, …)
- Variants: **fpm** and **apache**

**Why this project exists**: it targets PHP versions that are no longer supported — either by PHP itself (EOL) or by the distro's own package repositories — so it depends on third-party backport repos (Remi for Rocky Linux, Ondřej Surý for Debian/Ubuntu) to keep getting these versions with security patches. Keep this in mind when making design choices: these are backport/best-effort sources, not vendor-supported ones, which is why things like "don't pin package releases" and "review Sury's telemetry patches" below matter.

Full background and design rationale are in the sections below.

## Folder structure (target)

```
.
├── remi/rocky/Dockerfile        # ARG ROCKY_VERSION, PHP_VERSION
├── ondrej/debian/Dockerfile     # ARG DEBIAN_CODENAME, PHP_VERSION
├── ondrej/ubuntu/Dockerfile     # ARG UBUNTU_CODENAME, PHP_VERSION
├── .github/workflows/build.yml  # matrix: variant × cfg
├── .dockerignore
└── README.md
```

Current status: `remi/rocky/Dockerfile` (PHP 7.4, Rocky 10), `ondrej/debian/Dockerfile` (PHP 7.4, Debian trixie), `.github/workflows/build.yml`, and `README.md` exist. `ondrej/ubuntu` and `.dockerignore` have not been created yet.

One Dockerfile per source/distro, PHP version passed via `ARG` (never hardcode the version in a folder name or in the Dockerfile body). Multi-stage: `base` → `fpm` | `apache`, selected with `docker build --target`.

## Design decisions (must be followed)

- **Never pin package release numbers** (`-29`, `-31`, etc). Remi & Ondřej are rolling repos: only the latest patch/release per minor branch is available; pinning an old version will break on rebuild.
- **Consistency is enforced at the image level**: immutable tags + digest pinning in consumer images (not package pinning in the Dockerfile).
- **Remi uses module streams** (`dnf module enable php:remi-X.Y`, `php-*` packages, `/usr/bin/php` binary), not the `php74-*` SCL packages from remi-safe — this keeps it consistent with Ondřej.
- **Apache = mod_php + MPM prefork** (one process per container, not event/worker). FPM listens on TCP 9000; logs go to stderr/stdout (not log files).
- Extensions: `PHP_EXTENSIONS` (default: `cli mbstring opcache xml curl zip intl bcmath pdo`) + `PHP_EXTRA_EXTENSIONS` per CI matrix row for anything app-specific (e.g. a DB driver).
  - This default set is chosen to cover what's near-universal across frameworks/CMSs (Composer needs `zip`; Laravel/Symfony commonly need `intl`/`bcmath`/`pdo`; almost everything needs `curl`), not just the bare minimum — a generic base image should spare every consuming app from re-declaring these.
  - `opcache` is in the **default** set — it benefits virtually every PHP workload, so it shouldn't be opt-in. It's built into core since PHP 8.5; for older versions check whether it's still a separate package before porting this default to another Dockerfile.
  - `json` is only a separate package through PHP 7.4; built-in since 8.0 (it will still show up as an auto-pulled dependency on 7.4 — that's expected, no need to list it explicitly).
  - Some extensions requested by name may not appear as their own RPM in `/etc/php-packages.txt` because they're already compiled into `php-common`/`php-cli` for this build (observed for `curl`, `fileinfo`, `ctype`, `tokenizer` on Remi's PHP 7.4) — verify with `php -m` rather than assuming a missing package means a missing extension.
  - DB drivers (e.g. Remi's `mysqlnd`, Ondřej's `mysql`) are **not** in the default set — they're app-specific, add them via `PHP_EXTRA_EXTENSIONS` per matrix row instead. Package names differ across sources: Ondřej's `mysql` vs. Remi's `mysqlnd`.
- Record the installed package list in the image at `/etc/php-packages.txt` (`rpm -qa` for Remi/rpm-based, `dpkg-query` for Ondřej/deb-based); optionally a changelog at `/etc/php-changelog.txt`.
- Always use non-interactive flags (`-y` on every `dnf`/`apt-get` command) — the GPG key prompt defaults to "N" and will fail the build with "GPG check FAILED".

### Production hardening

Beyond the header-leak hardening already in place (`expose_php = Off`, `ServerTokens Prod`, `ServerSignature Off` — see `/etc/php.d/99-hardening.ini` and `/etc/httpd/conf.d/00-hardening.conf`), the following are also applied and should be ported to any new Dockerfile built on this pattern:

- **PHP ini** (`/etc/php.d/99-hardening.ini`): `display_errors = Off` + `display_startup_errors = Off` (errors must never leak into the HTTP response/stdout) paired with `log_errors = On` (so they still reach `docker logs` via the existing stderr/stdout redirection). `opcache.validate_timestamps = 0` plus explicit `opcache.memory_consumption`/`opcache.interned_strings_buffer`/`opcache.max_accelerated_files` — `validate_timestamps=0` is safe specifically because this is an **immutable** image (code is baked in at build time); skipping the per-request filesystem stat is a real perf win here, but it means a code change requires a rebuild, not a hot-reload. `date.timezone` is intentionally left unset so the `TZ` env var governs it at runtime. `memory_limit = 128M` is also set explicitly — Remi ships one shared `/etc/php.ini` for every SAPI (so CLI already got 128M from the distro default), but Ondřej/Debian splits ini per SAPI and its CLI template (`php.ini-production.cli`) defaults to `-1` (unlimited) while apache2/fpm default to `128M` — without this explicit line, `php -i`/`php -m` on the Debian image would show a different `memory_limit` than Rocky's for no reason other than packaging-template drift. Set it in the shared hardening ini (applies to every SAPI identically) rather than relying on either distro's per-SAPI default.
- **FPM** (`/etc/php-fpm.d/www.conf`): `clear_env = no` — FPM's default wipes the environment before PHP ever sees it, which breaks the common container pattern of passing config via `docker run -e`. Also `catch_workers_output = yes` (worker stdout/stderr actually reaches `docker logs`) and `security.limit_extensions = .php` (FPM refuses to execute anything but `.php`, mitigating upload-then-execute style attacks).
- **Apache** (`/etc/httpd/conf.d/00-hardening.conf` + `conf.modules.d`): `TraceEnable Off` and `Options -Indexes` on `/var/www/html` (no directory listing). Unused modules are disabled by commenting their `LoadModule` line — `autoindex`, `userdir`, `dav`/`dav_fs`/`dav_lock`, `cgi`, `status` — to shrink attack surface; `proxy_fcgi`/`proxy_scgi` are left alone (unrelated to mod_php). **Gotcha**: disabling `mod_autoindex` requires also deleting `/etc/httpd/conf.d/autoindex.conf` — its `IndexOptions`/`AddIcon*` directives are provided by that module and aren't `<IfModule>`-guarded, so leaving the file in place crashes httpd at startup ("Invalid command") once the module is gone. `userdir.conf` is safe to leave (its directives are `<IfModule>`-guarded). The default Rocky "welcome" test page is also removed (`rm -f /etc/httpd/conf.d/welcome.conf`).
- **`.htaccess` support**: both distros' own upstream default is `AllowOverride None` on `/var/www/html` (Rocky's `httpd.conf`, Debian's `apache2.conf`) — this silently ignores `.htaccess` entirely rather than erroring, so `httpd -t`/`apachectl configtest` passes either way and the breakage only shows up as "my rewrite rules don't do anything." Fixed by adding `AllowOverride All` to the `<Directory "/var/www/html">` block in the hardening conf (`zz-hardening.conf` on both) — that block is as/more specific than the distro's own default block for the same path, so it wins regardless of file-load order. Debian additionally needs `a2enmod rewrite` explicitly in the apache `RUN` step (bundled with the existing `a2enmod mpm_prefork php${PHP_VERSION}` call) — `mod_rewrite` ships in the `apache2` package but isn't enabled by default, unlike Rocky's `httpd`, which loads `rewrite_module` out of the box via `00-base.conf`.
- When adding similar hardening to a new Dockerfile, verify with: `httpd -t` (config syntax) + actually starting the container (catches the autoindex.conf-style crash), `php -i | grep <setting>` for ini values, `docker run -e FOO=bar <image> php -r 'echo getenv("FOO");'` for `clear_env`, `curl` against a directory with no index file / with `-X TRACE` to confirm the Apache behaviors land, and a real `.htaccess` + `RewriteRule` test (write one into `/var/www/html`, request the rewritten path, confirm it resolves) since `-t`/`configtest` cannot catch an `AllowOverride None`/disabled-`mod_rewrite` regression.

## Tag scheme

| Tag       | Example                          | Behavior                                 |
| --------- | --------------------------------- | ----------------------------------------- |
| immutable | `7.4.33-apache-trixie-20261022`  | never overwritten, used in production     |
| patch     | `7.4.33-apache-trixie`           | moving                                    |
| minor     | `7.4-apache-trixie`              | moving                                    |

The full version is read from the built image (`php -r 'echo PHP_VERSION;'`), not from the build-time ARG. Add a short SHA to the tag if more than one build per day is possible.

## Technical notes per source

### Remi (Rocky 10) — `remi/rocky/Dockerfile`

- Repo setup: `epel-release` + `remi-release-10.rpm`. `remi-modular` & `remi-safe` are enabled by default.
- CRB is required (EPEL depends on it): use `--enablerepo=crb` on `dnf install` (no need for `dnf-plugins-core`).
- `dnf module` still works on Rocky 10 (DNF4), with a deprecation warning — safe to ignore.
- Stream packages are hidden from `dnf search` until the stream is enabled (`dnf module enable php:remi-X.Y`).
- Apache: the `php` package is mod_php; it's only active if the MPM is prefork (edit `/etc/httpd/conf.modules.d/00-mpm.conf`, disable `mpm_event`, enable `mpm_prefork`).
- CVE backports for EOL branches (e.g. 7.4) show up in the RPM version/release, not in `phpinfo()`:
  `rpm -q php-common`, `rpm -q --changelog php-common`.
- EPEL/Remi mirrors occasionally rate-limit or hiccup on GitHub-hosted runner IP ranges — `/etc/dnf/dnf.conf` gets `retries=10`/`timeout=30` appended at the very start of the `base` stage so transient failures there retry instead of hard-failing the whole CI job.
- The hardening conf is named `/etc/httpd/conf.d/zz-hardening.conf` (not `00-`) — matches `ondrej/debian/Dockerfile` for consistency, and guarantees it loads last regardless of what else ever lands in `conf.d/`. Confirmed empirically (no current Rocky/EPEL package ships a conflicting `ServerTokens`/`ServerSignature` there, unlike Debian's `security.conf`), but the naming itself costs nothing and removes the "did a 00-prefixed file quietly start losing a conflict" question for good.

### Ondřej (Debian) — `ondrej/debian/Dockerfile`

- Repo setup: `ADD https://packages.sury.org/debsuryorg-archive-keyring.deb` (fetched by the build engine itself, so no `curl`/`ca-certificates` needed in the image just for this step — `ca-certificates` is still installed separately because the container's own `apt-get update` over https needs it), then `apt-get install -y /path/to/keyring.deb` and a deb822 source at `/etc/apt/sources.list.d/php.sources` pointing to `https://packages.sury.org/php/`, `Signed-By: /usr/share/keyrings/deb.sury.org-php.gpg` (the path the keyring package actually installs to — verified by extracting the `.deb`, not guessed).
- Packages are `phpX.Y-*`; FPM binary/service is `phpX.Y-fpm`; Apache module is `libapache2-mod-phpX.Y`.
- **`pdo` has no real `phpX.Y-pdo` package** — it's a virtual package `Provides`d by `phpX.Y-common` (confirmed via the `Packages` index and the `.deb`'s control file). `apt-get install phpX.Y-pdo` still resolves correctly since `phpX.Y-common` is already pulled in as a dependency — so the same `PHP_EXTENSIONS`-to-package-name mapping used for Remi works unchanged for Debian, no special-casing needed.
- PHP config is split per-SAPI (`/etc/php/X.Y/{cli,fpm,apache2}/conf.d/`), unlike Remi's single shared `/etc/php.d/`. The hardening ini is written once to `/etc/php/X.Y/mods-available/99-hardening.ini` and symlinked (absolute path, matching the distro's own `php_invoke` convention) into each SAPI's `conf.d/` individually.
- Apache: installing `libapache2-mod-phpX.Y` **auto-switches the MPM to prefork** via its postinst (mod_php isn't thread-safe) — confirmed by inspecting the package's postinst script and by a real build. `a2dismod`/`a2enmod` are run explicitly afterward anyway, to keep this deterministic rather than depending on dpkg trigger/install ordering.
- Default Debian `mods-enabled` includes `autoindex` and `status` (disabled with `a2dismod -f` — `-f` is required non-interactively for `autoindex` since apache2 flags it as "essential"); `serve-cgi-bin` is disabled via `a2disconf`. `userdir` and `dav` are not enabled by default on Debian, so no action needed for those (unlike Rocky/httpd, which ships them enabled).
- The stock `/var/www/html/index.html` (Debian/Apache's own "It works" page) is removed, mirroring Rocky's `welcome.conf` removal.
- **Gotcha**: Debian's own `conf-available/security.conf` is enabled by default and also sets `ServerTokens`/`ServerSignature` (to `OS`/`On`) — since `conf-enabled/*.conf` loads in alphabetical order with last-directive-wins, a hardening conf named `00-*` gets silently overridden back by `security.conf`. Ours is named `zz-hardening.conf` so it always loads last. Verify with a real `curl -I` against a running container (not just `apachectl configtest`, which doesn't catch this — the config is syntactically valid either way).
- **Gotcha**: don't use `CMD ["apachectl", "-DFOREGROUND"]` — `apachectl`'s catch-all case runs `$HTTPD "$@"` without `exec`, so as PID 1 it never forwards `SIGTERM`/`SIGINT` to the real `apache2` process underneath it (`docker stop`/Ctrl+C just hang until the grace period forces a `SIGKILL`). `CMD` instead runs the real `apache2` binary directly, same pattern as Rocky's `CMD ["httpd", "-DFOREGROUND"]`. Since that skips `apachectl`/`envvars`, the `${APACHE_RUN_USER,RUN_GROUP,PID_FILE,RUN_DIR,LOCK_DIR,LOG_DIR}` vars `apache2.conf` depends on are baked in as Docker `ENV` instead, and their runtime dirs are pre-created/chowned at build time. Verify with `docker exec <c> ps -ef` (apache2 must be PID 1, not a child of a shell) and timing a real `docker stop -t 5` (should return in well under 5s with exit code 0, not time out to exit 137).
- **Caution**: php.net has warned that Sury's packages include telemetry patches in the Apache 2 and PHP-FPM modules — review the patched source before using in production, and consider disabling/removing it in the Dockerfile.
- **Gotcha**: unlike Remi (where the backport patch/release only shows up in the RPM version, not in `phpinfo()`/`PHP_VERSION`), Sury patches the `PHP_VERSION` **constant itself** on EOL branches to embed the full Debian package version, e.g. `php -r 'echo PHP_VERSION;'` on 7.0 returns `7.0.33-93+0~20261022.119+debian13~1.gbp952195`, not just `7.0.33`. `+` and `~` aren't valid Docker tag characters, so `build.yml`'s version-detection step must strip this down to the leading `major.minor.patch` (`grep -oE '^[0-9]+\.[0-9]+\.[0-9]+'`) before using it in any tag — using the raw value fails the push with `invalid reference format`. Confirmed by actually running `php -r 'echo PHP_VERSION;'` in a built container, not assumed from the RPM-based Remi behavior.
- Changelog: `zcat /usr/share/doc/phpX.Y-common/changelog.Debian.gz`.
- Verified locally with real `docker build --target fpm|apache` + smoke tests (`php -m`, `php -i`, FPM pool config, `clear_env` via `-e`, live `curl` against Apache including a `TRACE` request returning 405 and `/` returning 404 with no directory listing).

### Ondřej (Ubuntu) — not built yet

- PPA `ppa.launchpadcontent.net/ondrej/php/ubuntu/`, key from a keyserver (fingerprint `14AA40EC0831756756D7F66C4F4EA0AAE5267A6C` — verify against Launchpad), `chmod 644` on the `.asc` file. Only active Ubuntu LTS releases are targeted.
- Package naming and the `pdo`-is-virtual, split-SAPI-conf.d, and MPM-auto-switch notes above for Debian should carry over unchanged (same Debian-derived packaging), but verify against the actual Ubuntu/Sury archive before relying on it.

## GitHub Actions workflow — `.github/workflows/build.yml`

- Triggers: push to `main` (path-filtered to `remi/**`, `ondrej/**`, the workflow file itself), a monthly schedule (1st of the month — pick up the latest backports; rolling repos keep no old releases), `workflow_dispatch`.
- `permissions: packages: write`, logs in to GHCR with `GITHUB_TOKEN` (not a PAT).
- Matrix: `variant: [fpm, apache]` × `cfg: [{src, distro, context, tagos, php, rocky_version, codename, extra}]`, with `fail-fast: false` so one failing combination doesn't stop the others. **Currently five `cfg` entries exist**: `remi/rocky` on Rocky 10 and Rocky 9 (both PHP 7.4), and `ondrej/debian` on trixie at PHP 7.4, 7.2, and 7.0 (`tagos: trixie` is shared across all three since they're the same distro/codename — the `php` field is what keeps their tags distinct). Add a `cfg` entry per source/distro/php-version combination as `ondrej/ubuntu` and other PHP versions land, following the same field shape.
- Build with `load: true` first → read the PHP version out of the image (`docker run ... php -r 'echo PHP_VERSION;'`) → compute immutable/patch/minor tags → **fail if the immutable tag already exists** (`docker manifest inspect`, see GHCR mitigations below) → push with all three tags. Cache with `type=gha`, scoped per `src`-`distro`-`php` combination — **deliberately not including `variant`**: the `fpm` and `apache` matrix legs share an identical `base` stage (same `dnf` install of PHP/extensions), so splitting the cache scope by variant would make both legs redownload/reinstall that shared layer independently on every run instead of reusing one cached copy. (On the very first run ever, cache is still empty either way — this only pays off from the second run onward.)
- **Multi-arch (`linux/amd64,linux/arm64`)**: `docker/setup-qemu-action` provides arm64 emulation on the (amd64) GitHub-hosted runner. The version-inspection build stays **single-platform** on purpose — `--load` can't load a multi-platform result into the local daemon, so only the final `push: true` build sets `platforms: linux/amd64,linux/arm64`. This relies on upstream `rockylinux/rockylinux` publishing an arm64 manifest (it does). Expect arm64 legs to be noticeably slower in CI since `dnf`-heavy RUN steps run under QEMU emulation, not native silicon.
- Every Dockerfile's codename/version build-args (`ROCKY_VERSION`, `DEBIAN_CODENAME`, `UBUNTU_CODENAME`) are passed on every build regardless of which Dockerfile is being built; an undeclared `ARG` is silently ignored by Docker (just a harmless warning), so this is safe.
- Adds the standard OCI annotation labels (`org.opencontainers.image.source`, `url`, `title`, `description`, `version`, `revision`, `created`) so the GHCR package links back to the repo and each image is traceable to the commit/time it was built from. `version` reuses the same `<php>-<variant>-<tagos>` string as the patch tag (without the registry prefix); `created`/`revision` come from the build job itself (`date -u`, `github.sha`). `licenses` is intentionally omitted — the repo has no `LICENSE` file yet, so there's nothing accurate to point to.
- Not yet verified end-to-end (needs a real push to `main` with GHCR credentials) — only validated locally for YAML syntax so far.

## GHCR

- The image name can differ from the GitHub repo name; must be lowercase; visibility is set per package (not inherited from the repo).
- **GHCR does not support immutable tags yet.** Chosen mitigations:
  1. Fail the workflow if the immutable tag already exists (check with `docker manifest inspect` before pushing).
  2. Restrict package Write access to this repo only.
  3. Consumer images pin with the `@sha256:` digest, not a tag — use Renovate to update the digest.
  4. Optional: `actions/attest-build-provenance` or cosign for provenance/signing.
- If true immutable tags are needed, alternatives are GitLab Container Registry (check version/tier) or Docker Hub — not yet decided.

## Status & next steps

Done:
- [x] `remi/rocky/Dockerfile` (PHP 7.4, Rocky 10) — multi-stage `base` → `fpm`/`apache`, verified with a local build + smoke test.
- [x] Extension defaults widened (`curl zip intl bcmath pdo` added) and production hardening applied (PHP ini, FPM, Apache) — see "Production hardening" above.
- [x] `README.md`.
- [x] `.github/workflows/build.yml` — matrix has Remi/Rocky (10, 9, PHP 7.4) and Ondřej/Debian (trixie, PHP 7.4/7.2/7.0) `cfg` entries; immutable/patch/minor tagging and the immutable-tag-exists check are wired up, but **not yet verified end-to-end** against real GHCR (needs a push to `main` with Actions enabled).
- [x] `ondrej/debian/Dockerfile` (PHP 7.4, Debian trixie) — multi-stage `base` → `fpm`/`apache`, same extension defaults and production hardening as Remi/Rocky, verified with a local build + smoke test, including multi-arch (`linux/amd64`+`linux/arm64` via buildx+QEMU), real `docker stop`/signal-handling checks, and cross-SAPI (cli/fpm/apache) consistency of both the extension set and every hardening ini value. Two Debian-specific bugs found and fixed in review: `security.conf` silently overriding the hardening conf's `ServerTokens`/`ServerSignature`, and `apachectl` not forwarding `SIGTERM`/Ctrl+C to the real server process — see the two "Gotcha" notes under "Ondřej (Debian)" below. Wired into `build.yml`'s matrix (`tagos: trixie`).
- [x] `ondrej/debian/Dockerfile` also verified at PHP 7.2 and PHP 7.0 on Debian trixie (`apache` target) — Sury does publish these EOL PHP versions for trixie, confirmed with a real `docker build` + running container + `phpinfo()` served over HTTP for each, not just package-availability checks. Both wired into `build.yml`'s matrix as additional `cfg` entries alongside the existing PHP 7.4/trixie one.

Not done yet:
1. `ondrej/ubuntu/Dockerfile` — follow the same pattern as `ondrej/debian/Dockerfile`.
2. Add other PHP versions (8.1, 8.3, 8.4, …) to the matrix (both the Dockerfile build-args and a new `cfg` entry per version in `build.yml`, for every source/distro combination that should carry that version).
3. `.dockerignore`.
4. Verify the workflow actually runs successfully on GHCR (first real push/dispatch) — check tag correctness and that the immutable-tag-exists guard behaves as expected on a second run.
5. Verify version availability per source: `dnf module list php` (Remi), `apt-cache policy phpX.Y-cli` (Ondřej).
6. Review Sury's telemetry patches for the Ondřej variants (Apache/FPM) before using in production.

Update this list when a step is completed.

## Expected agent workflow

- Never add package release pinning (`-29`, `-31`, etc.) to any Dockerfile — this violates the design decisions above.
- When adding a new Dockerfile (e.g. Ondřej/Ubuntu), follow the same pattern as `remi/rocky/Dockerfile` and `ondrej/debian/Dockerfile`: multi-stage `base` → `fpm`/`apache`, version via `ARG`, no hardcoded version in the path.
- Verify Dockerfile changes with a local build (`docker build --target fpm|apache --build-arg ...`) and a smoke test (`php -v`, check the relevant Apache module) before considering the work done.
