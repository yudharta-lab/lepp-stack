# LEPP Docker Compose Stack

Docker Compose stack untuk menjalankan aplikasi Laravel 13 dengan PostgreSQL, Nginx, Redis, dan PHP 8.4 FPM dalam environment terisolasi.

## 📋 Overview

Stack ini menyediakan:
- **PHP 8.4 FPM** - Application server dengan Supervisor untuk job queue
- **Nginx 1.29.3** - Web server dengan SSL support
- **PostgreSQL 16** - Database dengan optimized configuration untuk high-load
- **Redis 7.0.4** - In-memory cache dengan persistence
- **Supervisor** - Process manager untuk background jobs

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Host Machine                          │
│                   (lepp-stack)                        │
└─────────────────────────────────────────────────────────┘
         │
         ├─ Nginx (Port 8884/4434)
         │   └─ :443 SSL
         │
         ├─ PHP 8.4 FPM
         │   ├─ Supervisor (Job Queue)
         │   └─ Worker Processes
         │
         ├─ PostgreSQL (Port 54321)
         │   └─ app_db
         │
         └─ Redis Cache (Port 6379)
             └─ Persistence enabled
```

## 🚀 Quick Start

### 1. Prerequisites

- Docker & Docker Compose
- Git
- Minimal 8GB RAM (recommended 16GB)
- Port availability: 8884, 4434, 54321

### 2. Setup

#### Clone/Download project
```bash
cd /home/user/Docker/lepp
```

#### Configure environment
File `.env` sudah dikonfigurasi dengan default value. Edit jika diperlukan:

```env
PROJECT_STACK_NAME=lepp-stack
APP_PATH=/home/user/Projects/app

# PostgreSQL
POSTGRESQL_USER=app
POSTGRESQL_PASSWORD=app-db-pass
POSTGRESQL_DATABASE=app_db
POSTGRESQL_PORT=5432

# Resource Limits
APP_LIMITS_CPU=0.50
APP_LIMITS_MEMORY=4096M
WEB_SERVER_LIMITS_CPU=0.50
WEB_SERVER_LIMITS_MEMORY=4096M
POSTGRESQL_LIMITS_CPU=0.50
POSTGRESQL_LIMITS_MEMORY=4096M
```

#### Build & Run
```bash
# Build images
docker compose build

# Start all services
docker compose up -d

# Check status
docker compose ps
```

## 📦 Services

### PHP Application (`lepp-stack-php8`)
- **Image**: Built from `./bin/php8/Dockerfile`
- **Port**: 9000 (internal FPM)
- **Volume**: `/home/user/Projects/lepp` → `/var/www`
- **Features**:
  - PHP 8.4 dengan extensions: pdo_pgsql, redis, gd, intl, dll
  - Node.js 24.x LTS
  - Composer & npm
  - Supervisor untuk background jobs
  - Cron support

### Nginx Web Server (`lepp-stack-nginx`)
- **Image**: `nginx:1.29.3-alpine`
- **Ports**: 
  - 8884 (HTTP)
  - 4434 (HTTPS)
- **Volume**: Laravel app root
- **Config**: `./config/nginx/nginx.conf`

### PostgreSQL Database (`lepp-stack-postgresql`)
- **Image**: `postgres:16-alpine`
- **Port**: 54321
- **Database**: `app_db`
- **User**: `app`
- **Features**:
  - Optimized untuk high-load queries
  - Persistent volume: `postgresql_data`
  - Shared buffers: 512MB
  - Max connections: 800
  - Auto vacuum enabled
  - Checkpoint & WAL optimization
  - Slow query logging (> 5s)
- **Config**: 
  - `./config/postgresql/postgresql.conf`
  - `./config/postgresql/pg_hba.conf`

### Redis Cache (`lepp-stack-redis-cache`)
- **Image**: `redis:7.0.4-alpine3.16`
- **Port**: 6379 (internal)
- **Password**: `redis_pass`
- **Features**:
  - Max memory: 2048MB
  - Policy: `allkeys-lru`
  - Persistence: `appendonly yes`
  - Fsync: `everysec`

## 🔧 Configuration

### PHP Configuration
Edit `./config/php/php.ini` untuk mengubah PHP settings:
```ini
memory_limit = 512M
max_execution_time = 300
upload_max_filesize = 100M
post_max_size = 100M
```

### Nginx Configuration
Edit `./config/nginx/nginx.conf` untuk web server settings.

### PostgreSQL
- **Connection**: `./config/postgresql/postgresql.conf`
- **Authentication**: `./config/postgresql/pg_hba.conf`

Setiap perubahan config memerlukan container restart:
```bash
docker compose restart postgresql
```

## 📝 Common Commands

### Container Management
```bash
# Start stack
docker compose up -d

# Stop stack
docker compose down

# Stop dan remove volumes
docker compose down -v

# View logs
docker compose logs -f app
docker compose logs -f postgresql
docker compose logs -f web_server

# Rebuild images
docker compose build --no-cache
```

### Database Commands
```bash
# Connect ke PostgreSQL
psql -h localhost -p 54321 -U app -d app_db

# Backup database
docker compose exec postgresql pg_dump -U app app_db > backup.sql

# Restore database
docker compose exec -T postgresql psql -U app app_db < backup.sql
```

### Laravel Commands
```bash
# SSH ke PHP container
docker compose exec app bash

# Run artisan commands
docker compose exec app php artisan migrate
docker compose exec app php artisan seed
docker compose exec app php artisan cache:clear
docker compose exec app php artisan config:cache
```

### Node/npm Commands
```bash
# Install dependencies
docker compose exec app npm install

# Build assets
docker compose exec app npm run build
docker compose exec app npm run dev
```

## 🔍 Troubleshooting

### PostgreSQL Connection Error
```
ERROR: could not connect to database

Solusi:
1. Check PostgreSQL running: docker compose ps postgresql
2. Verify credentials di .env
3. Check logs: docker compose logs postgresql
4. Restart: docker compose restart postgresql
```

### PHP Container Not Starting
```
Solusi:
1. Check build logs: docker compose build --no-cache
2. Verify Dockerfile syntax
3. Check disk space: docker system df
4. Clear unused images: docker system prune -a
```

### Port Already in Use
```
docker: Error response from daemon: Ports are not available

Solusi:
Ubah port di .env atau stop service yang menggunakan port tersebut:
sudo lsof -i :8884
sudo kill -9 <PID>
```

### Performance Issues
- Monitor resources: `docker stats`
- Check memory limits di `.env`
- Review PostgreSQL logs: `./logs/postgresql/`
- Check PHP slow logs: `./logs/php/`

## 📂 Directory Structure

```
lepp/
├── README.md                      # Dokumentasi ini
├── .env                           # Environment variables
├── docker-compose.yml             # Main compose file
├── .gitignore                     # Git ignore rules
│
├── bin/
│   └── php8/
│       └── Dockerfile            # PHP 8.4 image definition
│
├── config/
│   ├── php/
│   │   └── php.ini               # PHP configuration
│   ├── nginx/
│   │   └── nginx.conf            # Nginx configuration
│   ├── postgresql/
│   │   ├── postgresql.conf       # PostgreSQL settings
│   │   └── pg_hba.conf           # Authentication rules
│   ├── supervisor/
│   │   └── supervisord.conf      # Supervisor config
│   └── ssl/
│       └── (SSL certificates)
│
├── logs/
│   ├── php/                      # PHP & FPM logs
│   ├── nginx/                    # Nginx access/error logs
│   ├── postgresql/               # PostgreSQL logs
│   ├── redis/                    # Redis logs
│   └── supervisor/               # Supervisor logs
│
└── initdb/
    └── (Database init scripts)
```

## 🔐 Security Notes

⚠️ **Untuk Production:**
1. Ubah default passwords di `.env`
2. Implement SSL certificates di `./config/ssl/`
3. Configure firewall untuk restrict ports
4. Setup backup automation untuk database
5. Monitor logs regularly
6. Update images: `docker compose pull`
7. Use strong Redis password
8. Restrict PostgreSQL access via `pg_hba.conf`

## 📊 Resource Monitoring

Check container resource usage:
```bash
# Real-time monitoring
docker stats

# View container details
docker compose ps -a

# Check disk usage
docker system df
```

Adjust limits di `.env`:
```env
APP_LIMITS_CPU=0.50          # CPU cores
APP_LIMITS_MEMORY=4096M      # Memory allocation
```

## 🐛 Debugging

### Enable Query Logging
PostgreSQL (postgresql.conf):
```ini
log_min_duration_statement = 1000  # Log queries > 1s
```

### Enable PHP Debug
PHP (php.ini):
```ini
display_errors = On
error_reporting = E_ALL
xdebug.mode = develop,debug
```

### View Supervisor Status
```bash
docker compose exec app supervisorctl status
```

## 📚 Useful Resources

- [Laravel Documentation](https://laravel.com/docs)
- [PostgreSQL Docs](https://www.postgresql.org/docs/)
- [Docker Compose Reference](https://docs.docker.com/compose/reference/)
- [Nginx Configuration](https://nginx.org/en/docs/)
- [PHP Configuration](https://www.php.net/manual/en/ini.list.php)

## ✨ Next Steps

1. Copy `APP_PATH` dari `.env` ke project folder:
   ```bash
   cp -r /home/user/Projects/lepp /path/to/laravel/app
   ```

2. Configure database connection di Laravel `.env`:
   ```env
   DB_CONNECTION=pgsql
   DB_HOST=postgresql
   DB_PORT=5432
   DB_DATABASE=app_db
   DB_USERNAME=app
   DB_PASSWORD=app_pass
   
   REDIS_HOST=redis_cache
   REDIS_PASSWORD=redis_pass
   ```

3. Run migrations:
   ```bash
   docker compose exec app php artisan migrate
   ```

4. Build frontend assets:
   ```bash
   docker compose exec app npm install
   docker compose exec app npm run build
   ```

---

**Created**: May 12, 2026  
**Last Updated**: May 12, 2026
