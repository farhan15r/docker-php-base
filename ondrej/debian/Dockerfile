# syntax=docker/dockerfile:1
ARG DEBIAN_CODENAME=trixie

############################ base ############################
FROM debian:${DEBIAN_CODENAME} AS base
ARG DEBIAN_CODENAME
ARG PHP_VERSION=7.4
ARG PHP_EXTENSIONS="cli mbstring opcache xml curl zip intl bcmath pdo"
ARG PHP_EXTRA_EXTENSIONS=""
ENV PHP_VERSION=${PHP_VERSION}
ENV DEBIAN_FRONTEND=noninteractive

LABEL org.opencontainers.image.title="php-base" \
      org.opencontainers.image.description="PHP ${PHP_VERSION} base image (Ondrej/Debian)"

# Mirror retries/timeouts (same rationale as Remi/Rocky: avoid transient CI
# mirror hiccups hard-failing the whole job) + skip docs/man to keep layers lean.
RUN echo 'Acquire::Retries "10";' > /etc/apt/apt.conf.d/80-retries && \
    echo 'path-exclude /usr/share/doc/*' > /etc/dpkg/dpkg.cfg.d/01-nodoc && \
    echo 'path-exclude /usr/share/man/*' >> /etc/dpkg/dpkg.cfg.d/01-nodoc

# Sury's archive key ships as a .deb (ADD fetches it with the build engine's own
# TLS, so no ca-certificates/curl needed in the image just for this step).
ADD https://packages.sury.org/debsuryorg-archive-keyring.deb /tmp/debsuryorg-archive-keyring.deb

RUN apt-get update && \
    apt-get install -y --no-install-recommends ca-certificates && \
    apt-get install -y /tmp/debsuryorg-archive-keyring.deb && \
    rm -f /tmp/debsuryorg-archive-keyring.deb && \
    echo 'Types: deb' > /etc/apt/sources.list.d/php.sources && \
    echo 'URIs: https://packages.sury.org/php/' >> /etc/apt/sources.list.d/php.sources && \
    echo "Suites: ${DEBIAN_CODENAME}" >> /etc/apt/sources.list.d/php.sources && \
    echo 'Components: main' >> /etc/apt/sources.list.d/php.sources && \
    echo 'Signed-By: /usr/share/keyrings/deb.sury.org-php.gpg' >> /etc/apt/sources.list.d/php.sources && \
    apt-get update && \
    apt-get install -y --no-install-recommends \
      $(for e in $PHP_EXTENSIONS $PHP_EXTRA_EXTENSIONS; do echo "php${PHP_VERSION}-$e"; done) && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

RUN echo 'expose_php = Off' > /etc/php/${PHP_VERSION}/mods-available/99-hardening.ini && \
    echo 'display_errors = Off' >> /etc/php/${PHP_VERSION}/mods-available/99-hardening.ini && \
    echo 'display_startup_errors = Off' >> /etc/php/${PHP_VERSION}/mods-available/99-hardening.ini && \
    echo 'log_errors = On' >> /etc/php/${PHP_VERSION}/mods-available/99-hardening.ini && \
    echo 'memory_limit = 128M' >> /etc/php/${PHP_VERSION}/mods-available/99-hardening.ini && \
    echo 'opcache.validate_timestamps = 0' >> /etc/php/${PHP_VERSION}/mods-available/99-hardening.ini && \
    echo 'opcache.memory_consumption = 128' >> /etc/php/${PHP_VERSION}/mods-available/99-hardening.ini && \
    echo 'opcache.interned_strings_buffer = 16' >> /etc/php/${PHP_VERSION}/mods-available/99-hardening.ini && \
    echo 'opcache.max_accelerated_files = 10000' >> /etc/php/${PHP_VERSION}/mods-available/99-hardening.ini && \
    ln -sf /etc/php/${PHP_VERSION}/mods-available/99-hardening.ini /etc/php/${PHP_VERSION}/cli/conf.d/99-hardening.ini

WORKDIR /var/www/html

############################ fpm #############################
FROM base AS fpm
RUN apt-get update && \
    apt-get install -y --no-install-recommends php${PHP_VERSION}-fpm && \
    apt-get clean && rm -rf /var/lib/apt/lists/* && \
    ln -sf /etc/php/${PHP_VERSION}/mods-available/99-hardening.ini /etc/php/${PHP_VERSION}/fpm/conf.d/99-hardening.ini && \
    sed -i \
      -e 's|^listen = .*|listen = 9000|' \
      -e 's|^;catch_workers_output = yes|catch_workers_output = yes|' \
      -e 's|^;clear_env = no|clear_env = no|' \
      -e 's|^;security.limit_extensions.*|security.limit_extensions = .php|' \
      /etc/php/${PHP_VERSION}/fpm/pool.d/www.conf && \
    sed -i 's|^error_log = .*|error_log = /proc/self/fd/2|' /etc/php/${PHP_VERSION}/fpm/php-fpm.conf && \
    mkdir -p /run/php && \
    ln -sf /usr/sbin/php-fpm${PHP_VERSION} /usr/sbin/php-fpm && \
    dpkg-query -W -f='${Package}\n' 'php*' | sort > /etc/php-packages.txt
EXPOSE 9000
CMD ["php-fpm", "-F"]

############################ apache ##########################
FROM base AS apache
# apache2.conf references these as ${APACHE_*} — normally sourced from
# /etc/apache2/envvars by the apachectl wrapper at runtime, but we run the
# apache2 binary directly (see CMD), so they're baked in as real env vars instead.
ENV APACHE_RUN_USER=www-data \
    APACHE_RUN_GROUP=www-data \
    APACHE_PID_FILE=/var/run/apache2/apache2.pid \
    APACHE_RUN_DIR=/var/run/apache2 \
    APACHE_LOCK_DIR=/var/lock/apache2 \
    APACHE_LOG_DIR=/var/log/apache2

# paket libapache2-mod-phpX.Y = mod_php; package postinst auto-switches the
# MPM to prefork (mod_php isn't thread-safe) — a2dismod/a2enmod below just
# make that explicit/deterministic instead of relying on the install-order trigger.
RUN apt-get update && \
    apt-get install -y --no-install-recommends apache2 libapache2-mod-php${PHP_VERSION} && \
    apt-get clean && rm -rf /var/lib/apt/lists/* && \
    mkdir -p ${APACHE_RUN_DIR} ${APACHE_LOCK_DIR} && \
    chown ${APACHE_RUN_USER}:${APACHE_RUN_GROUP} ${APACHE_RUN_DIR} ${APACHE_LOCK_DIR} && \
    ln -sf /etc/php/${PHP_VERSION}/mods-available/99-hardening.ini /etc/php/${PHP_VERSION}/apache2/conf.d/99-hardening.ini && \
    a2dismod -f mpm_event >/dev/null 2>&1 || true && \
    a2enmod mpm_prefork php${PHP_VERSION} && \
    a2dismod -f autoindex status && \
    a2disconf serve-cgi-bin && \
    sed -i 's|^ErrorLog .*|ErrorLog /dev/stderr|' /etc/apache2/apache2.conf && \
    sed -i \
      -e 's|^\tErrorLog .*|\tErrorLog /dev/stderr|' \
      -e 's|^\tCustomLog .*|\tCustomLog /dev/stdout combined|' \
      /etc/apache2/sites-available/000-default.conf && \
    rm -f /var/www/html/index.html && \
    dpkg-query -W -f='${Package}\n' 'php*' 'apache2*' | sort > /etc/php-packages.txt

RUN echo 'ServerTokens Prod' > /etc/apache2/conf-available/zz-hardening.conf && \
    echo 'ServerSignature Off' >> /etc/apache2/conf-available/zz-hardening.conf && \
    echo 'TraceEnable Off' >> /etc/apache2/conf-available/zz-hardening.conf && \
    echo '<Directory "/var/www/html">' >> /etc/apache2/conf-available/zz-hardening.conf && \
    echo '    Options -Indexes' >> /etc/apache2/conf-available/zz-hardening.conf && \
    echo '</Directory>' >> /etc/apache2/conf-available/zz-hardening.conf && \
    a2enconf zz-hardening

EXPOSE 80
CMD ["apache2", "-DFOREGROUND"]
