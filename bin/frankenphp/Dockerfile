FROM dunglas/frankenphp:1-php8.4-bookworm

ARG PUID=1000
ARG PGID=1000

WORKDIR /var/www/

# ─────────────────────────────────────────────
# 1. Install system dependencies
# ─────────────────────────────────────────────
RUN apt-get update && apt-get install -y \
    locales zip unzip git curl vim cron supervisor ca-certificates postgresql-client \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# ─────────────────────────────────────────────
# 2. Install PHP extensions (Laravel 13)
#    install-php-extensions handles configure flags
#    (gd, intl, etc.) automatically.
# ─────────────────────────────────────────────
RUN install-php-extensions \
    pdo_pgsql \
    mbstring \
    zip \
    exif \
    pcntl \
    gettext \
    opcache \
    bcmath \
    sockets \
    intl \
    gd \
    redis

# ─────────────────────────────────────────────
# 3. Install Composer
# ─────────────────────────────────────────────
COPY --from=composer:latest /usr/bin/composer /usr/local/bin/composer

# ─────────────────────────────────────────────
# 4. Install Node.js 24.x LTS
# ─────────────────────────────────────────────
RUN curl -fsSL https://deb.nodesource.com/setup_24.x | bash - \
    && apt-get install -y nodejs \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# ─────────────────────────────────────────────
# 5. Set UID & GID dari build args
#    (image sudah menyediakan user "www-data")
# ─────────────────────────────────────────────
RUN groupmod -g ${PGID} www-data \
    && usermod -u ${PUID} www-data

# ─────────────────────────────────────────────
# 6. Buat direktori log & set permissions
# ─────────────────────────────────────────────
RUN mkdir -p /var/logs/supervisor /var/log/caddy /data/caddy /config/caddy \
    && chown -R www-data:www-data /var/www /var/logs/supervisor /var/log/caddy /data/caddy /config/caddy

# ─────────────────────────────────────────────
# 7. Copy supervisor config
#    (Caddyfile & SSL certs di-mount via docker-compose volumes)
# ─────────────────────────────────────────────
COPY ./config/frankenphp/supervisord.conf /etc/supervisord.conf

# FrankenPHP HTTP (redirect) & HTTPS (app) ports - unprivileged, non-root
EXPOSE 8000 8443

CMD ["/usr/bin/supervisord", "-n", "-c", "/etc/supervisord.conf"]

USER www-data
