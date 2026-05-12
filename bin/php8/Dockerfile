FROM php:8.4-fpm

ARG DEBIAN_FRONTEND=noninteractive
ARG PUID=1000
ARG PGID=1000

WORKDIR /var/www/

# ─────────────────────────────────────────────
# 1. Install system dependencies
# ─────────────────────────────────────────────
RUN apt-get update && apt-get install -y \
    build-essential \
    libpng-dev \
    libjpeg62-turbo-dev \
    libfreetype6-dev \
    libonig-dev \
    libzip-dev \
    libcurl4-openssl-dev \
    libssl-dev \
    libicu-dev \
    libpq-dev \
    locales zip unzip git curl vim cron supervisor ca-certificates postgresql-client \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# ─────────────────────────────────────────────
# 2. Install PHP extensions (Laravel 13)
#    - Tanpa -j$(nproc): beberapa ekstensi tidak
#      aman dikompilasi paralel (race condition)
#    - GD diinstall terpisah setelah configure
# ─────────────────────────────────────────────
RUN docker-php-ext-install pdo_pgsql mbstring zip exif pcntl curl gettext opcache bcmath fileinfo sockets intl

RUN docker-php-ext-configure gd \
        --with-freetype=/usr/include/ \
        --with-jpeg=/usr/include/ \
    && docker-php-ext-install gd

# ─────────────────────────────────────────────
# 3. Install Redis PHP extension via PECL
# ─────────────────────────────────────────────
RUN pecl install redis \
    && docker-php-ext-enable redis \
    && rm -rf /tmp/pear

# ─────────────────────────────────────────────
# 4. Install Composer
# ─────────────────────────────────────────────
COPY --from=composer:latest /usr/bin/composer /usr/local/bin/composer

# ─────────────────────────────────────────────
# 5. Install Node.js 24.x LTS
# ─────────────────────────────────────────────
RUN curl -fsSL https://deb.nodesource.com/setup_24.x | bash - \
    && apt-get install -y nodejs \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# ─────────────────────────────────────────────
# 6. Set UID & GID dari build args
# ─────────────────────────────────────────────
RUN groupmod -g ${PGID} www-data \
    && usermod -u ${PUID} www-data

# ─────────────────────────────────────────────
# 7. Buat direktori log supervisor & set permissions
# ─────────────────────────────────────────────
RUN mkdir -p /var/logs/supervisor \
    && chown -R www-data:www-data /var/www \
    && chown -R www-data:www-data /var/logs/supervisor \
    && chmod -R 755 /var/logs/supervisor

# ─────────────────────────────────────────────
# 8. Copy supervisor config
# ─────────────────────────────────────────────
COPY ./config/supervisor/ /etc/

# Port php-fpm
EXPOSE 9000

# Supervisord berjalan sebagai root agar bisa me-manage proses worker
CMD ["/usr/bin/supervisord", "-n", "-c", "/etc/supervisord.conf"]

USER www-data