# syntax=docker/dockerfile:1
ARG ROCKY_VERSION=10

############################ base ############################
FROM rockylinux/rockylinux:${ROCKY_VERSION} AS base
ARG ROCKY_VERSION
ARG PHP_VERSION=7.4
ARG PHP_EXTENSIONS="cli mbstring opcache xml curl zip intl bcmath pdo"
ARG PHP_EXTRA_EXTENSIONS=""
ENV PHP_VERSION=${PHP_VERSION}

# Override labels inherited from the rockylinux/rockylinux base image (name, vendor,
# version, summary, license, io.buildah.version, org.opencontainers.image.authors/
# vendor/licenses) — otherwise they leak Rocky's own identity/metadata into this image.
LABEL name="php-base" \
      vendor="" \
      version="${PHP_VERSION}" \
      summary="PHP ${PHP_VERSION} base image (Remi/Rocky)" \
      license="" \
      io.buildah.version="" \
      org.opencontainers.image.authors="" \
      org.opencontainers.image.vendor="" \
      org.opencontainers.image.licenses=""

RUN echo -e 'retries=10\ntimeout=30' >> /etc/dnf/dnf.conf && \
    dnf upgrade -y && \
    dnf install -y epel-release && \
    dnf install -y https://rpms.remirepo.net/enterprise/remi-release-${ROCKY_VERSION}.rpm && \
    dnf module reset -y php && \
    dnf module enable -y php:remi-${PHP_VERSION} && \
    dnf install -y --enablerepo=crb --setopt install_weak_deps=false --nodocs \
      $(for e in $PHP_EXTENSIONS $PHP_EXTRA_EXTENSIONS; do echo "php-$e"; done) && \
    dnf clean all

RUN echo 'expose_php = Off' > /etc/php.d/99-hardening.ini && \
    echo 'display_errors = Off' >> /etc/php.d/99-hardening.ini && \
    echo 'display_startup_errors = Off' >> /etc/php.d/99-hardening.ini && \
    echo 'log_errors = On' >> /etc/php.d/99-hardening.ini && \
    echo 'memory_limit = 128M' >> /etc/php.d/99-hardening.ini && \
    echo 'opcache.validate_timestamps = 0' >> /etc/php.d/99-hardening.ini && \
    echo 'opcache.memory_consumption = 128' >> /etc/php.d/99-hardening.ini && \
    echo 'opcache.interned_strings_buffer = 16' >> /etc/php.d/99-hardening.ini && \
    echo 'opcache.max_accelerated_files = 10000' >> /etc/php.d/99-hardening.ini

WORKDIR /var/www/html

############################ fpm #############################
FROM base AS fpm
RUN dnf install -y --enablerepo=crb --setopt install_weak_deps=false --nodocs php-fpm && \
    dnf clean all && \
    sed -i \
      -e 's|^listen = .*|listen = 9000|' \
      -e 's|^listen.allowed_clients|;listen.allowed_clients|' \
      -e 's|^;catch_workers_output = yes|catch_workers_output = yes|' \
      -e 's|^;clear_env = no|clear_env = no|' \
      -e 's|^;security.limit_extensions.*|security.limit_extensions = .php|' \
      /etc/php-fpm.d/www.conf && \
    sed -i 's|^error_log = .*|error_log = /proc/self/fd/2|' /etc/php-fpm.conf && \
    mkdir -p /run/php-fpm && \
    rpm -qa 'php*' | sort > /etc/php-packages.txt
EXPOSE 9000
CMD ["php-fpm", "-F"]

############################ apache ##########################
FROM base AS apache
# paket "php" = mod_php; aktif hanya kalau MPM prefork
RUN dnf install -y --enablerepo=crb --setopt install_weak_deps=false --nodocs httpd php && \
    dnf clean all && \
    sed -i \
      -e 's|^LoadModule mpm_event_module|#LoadModule mpm_event_module|' \
      -e 's|^#LoadModule mpm_prefork_module|LoadModule mpm_prefork_module|' \
      /etc/httpd/conf.modules.d/00-mpm.conf && \
    sed -i \
      -e 's|^ErrorLog .*|ErrorLog /dev/stderr|' \
      -e 's|^\s*CustomLog .*|    CustomLog /dev/stdout combined|' \
      /etc/httpd/conf/httpd.conf && \
    sed -i -E 's,^(LoadModule (autoindex|status|userdir)_module.*),#\1,' /etc/httpd/conf.modules.d/00-base.conf && \
    sed -i -E 's,^(LoadModule dav(_fs|_lock)?_module.*),#\1,' /etc/httpd/conf.modules.d/00-dav.conf && \
    sed -i 's|LoadModule cgi_module|#LoadModule cgi_module|' /etc/httpd/conf.modules.d/01-cgi.conf && \
    rm -f /etc/httpd/conf.d/autoindex.conf /etc/httpd/conf.d/welcome.conf && \
    rpm -qa 'php*' 'httpd*' | sort > /etc/php-packages.txt

RUN echo 'ServerTokens Prod' > /etc/httpd/conf.d/zz-hardening.conf && \
    echo 'ServerSignature Off' >> /etc/httpd/conf.d/zz-hardening.conf && \
    echo 'TraceEnable Off' >> /etc/httpd/conf.d/zz-hardening.conf && \
    echo '<Directory "/var/www/html">' >> /etc/httpd/conf.d/zz-hardening.conf && \
    echo '    Options -Indexes' >> /etc/httpd/conf.d/zz-hardening.conf && \
    echo '    AllowOverride All' >> /etc/httpd/conf.d/zz-hardening.conf && \
    echo '</Directory>' >> /etc/httpd/conf.d/zz-hardening.conf

EXPOSE 80
CMD ["httpd", "-DFOREGROUND"]