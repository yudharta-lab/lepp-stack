# LEPP Docker Compose Stack

Docker Compose stack untuk menjalankan aplikasi Laravel 13 dengan PostgreSQL, Redis, dan **Laravel Octane + FrankenPHP** dalam environment terisolasi.

> **Branch note**: branch ini (`octane-production`) menggantikan Nginx + PHP-FPM dengan satu container **FrankenPHP** yang menjalankan Laravel Octane (worker mode, persistent). Branch `main` masih memakai setup lama (Nginx + PHP-FPM) untuk development. Sinkronkan fix yang relevan dari `main` ke branch ini sebelum deploy ke production.

## 📋 Overview

Stack ini menyediakan:
- **FrankenPHP (PHP 8.4) + Laravel Octane** - Application server + web server dalam satu proses, worker persistent (tidak boot ulang Laravel setiap request)
- **PostgreSQL 16** - Database dengan optimized configuration untuk high-load
- **Redis 7.0.4** - In-memory cache dengan persistence
- **Supervisor** - Process manager untuk Octane worker & (opsional) job queue/scheduler

### Kenapa FrankenPHP + Octane, bukan Nginx + PHP-FPM?

FrankenPHP adalah web server (berbasis Caddy) yang meng-embed PHP secara langsung. Dikombinasikan dengan Laravel Octane, aplikasi di-boot **sekali** lalu request-request berikutnya diproses oleh worker yang sama di memori — jauh lebih cepat dibanding PHP-FPM yang boot ulang framework setiap request. Konsekuensinya:
- Nginx tidak dibutuhkan lagi — FrankenPHP sudah jadi web server sekaligus (TLS, gzip, serve static file, reverse ke PHP).
- State aplikasi (static property, singleton, dsb) **persist antar request** dalam satu worker. Kode yang menyimpan state di static/global variable tanpa reset bisa menyebabkan memory leak atau data nyasar ke request lain — perlu perhatian ekstra dibanding PHP-FPM.
- Operasi berat & sinkron (generate PDF/Excel besar, loop tak terbatas) menahan satu worker lebih lama — pertimbangkan pindah ke queue job (Redis sudah tersedia) untuk operasi semacam ini.

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Host Machine                          │
│                   (sim-bau-stack)                        │
└─────────────────────────────────────────────────────────┘
         │
         ├─ FrankenPHP + Octane (Port 8884/4434 → container 8000/8443)
         │   ├─ :8000  → redirect ke HTTPS
         │   ├─ :8443  → TLS manual (./config/ssl) + Laravel app
         │   └─ Supervisor → proses "octane" (php artisan octane:frankenphp)
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

# Octane / FrankenPHP
OCTANE_WORKERS=auto       # jumlah worker; "auto" = seukuran jumlah core yang di-alokasikan
OCTANE_MAX_REQUESTS=500   # worker di-recycle setelah N request (mitigasi memory leak)
```

Di sisi aplikasi Laravel, pastikan `.env` proyek juga berisi:
```env
OCTANE_SERVER=frankenphp
OCTANE_HTTPS=true   # set false kalau APP_URL masih http
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

### FrankenPHP + Octane (`sim-bau-stack-frankenphp`)
- **Image**: Built from `./bin/frankenphp/Dockerfile` (base: `dunglas/frankenphp:1-php8.4-bookworm`)
- **Ports**:
  - `8884` (host) → `8000` (container, HTTP — redirect ke HTTPS saja)
  - `4434` (host) → `8443` (container, HTTPS — serve aplikasi)
- **Volume**: `${APP_PATH}` → `/var/www`
- **Jalan sebagai**: `www-data` non-root (uid/gid mengikuti `USER_UID`/`USER_GID`), sama seperti setup lama — tidak butuh privilege root walau serve HTTPS
- **Features**:
  - PHP 8.4 dengan extensions: pdo_pgsql, redis, gd, intl, bcmath, sockets, dll (lihat `bin/frankenphp/Dockerfile`)
  - Node.js 24.x LTS, Composer & npm
  - Supervisor menjalankan proses `octane` (`php artisan octane:frankenphp`), bukan lagi php-fpm
  - Caddyfile custom (`./config/frankenphp/Caddyfile`) untuk TLS manual, gzip, worker mode
  - opcache aktif (`opcache.enable=1`, `validate_timestamps=0`) — **perlu restart container setelah deploy kode baru** karena worker persistent + opcache tidak lagi mengecek perubahan file

### PostgreSQL Database (`sim-bau-stack-postgresql`)
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

### Redis Cache (`sim-bau-stack-redis-cache`)
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

; Opcache — worker Octane butuh ini aktif untuk mendapat manfaat bytecode cache
opcache.enable = 1
opcache.validate_timestamps = 0   ; restart container setelah deploy kode baru
```

### Caddyfile (FrankenPHP)
Edit `./config/frankenphp/Caddyfile` untuk mengubah routing/TLS/gzip.

Site block HTTPS sengaja **catch-all** (`:8443`, tanpa nama domain spesifik), bukan `sim-bau.dev:8443, ... {`. Ini karena Caddy mencocokkan TLS berdasarkan SNI secara strict — kalau di depan stack ini ada reverse proxy lain (misalnya Nginx Proxy Manager) yang mem-forward pakai nama container sebagai SNI (bukan domain publik), request akan gagal TLS handshake (502) kalau site block dibatasi per-hostname. Kalau nanti butuh banyak vhost di satu FrankenPHP, baru pisahkan lagi per domain.

### Supervisor
Edit `./config/frankenphp/supervisord.conf`:
- Program `octane` menjalankan `php artisan octane:frankenphp` dengan `--workers` dan `--max-requests` dari env `OCTANE_WORKERS` / `OCTANE_MAX_REQUESTS`.
- Program `laravel-scheduler` dan `laravel-worker` masih di-comment (belum aktif) — uncomment kalau butuh cron scheduler / queue worker berjalan di container yang sama.

### PostgreSQL
- **Connection**: `./config/postgresql/postgresql.conf`
- **Authentication**: `./config/postgresql/pg_hba.conf`

Setiap perubahan config memerlukan container restart:
```bash
docker compose restart app          # FrankenPHP/Octane, Caddyfile & php.ini di-mount (bind mount) → cukup restart, tanpa rebuild
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
# SSH ke FrankenPHP container
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

### Octane Commands
```bash
# Status worker Octane (jumlah worker, PID, dsb)
docker compose exec app php artisan octane:status

# Reload worker tanpa restart container (ambil kode baru tanpa downtime penuh)
docker compose exec app php artisan octane:reload

# Stop Octane (supervisor akan otomatis restart proses karena autorestart=true)
docker compose exec app php artisan octane:stop
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

### FrankenPHP/Octane Container Not Starting
```
Solusi:
1. Check build logs: docker compose build --no-cache
2. Verify Dockerfile syntax
3. Check disk space: docker system df
4. Clear unused images: docker system prune -a
5. Cek log supervisor: docker compose logs app
```

### 502 Bad Gateway di Belakang Reverse Proxy (mis. Nginx Proxy Manager)
```
Kemungkinan penyebab: TLS handshake gagal karena SNI yang dikirim reverse proxy
(sering kali nama container, bukan domain publik) tidak cocok dengan site block
Caddy.

Solusi:
1. Pastikan site block HTTPS di ./config/frankenphp/Caddyfile catch-all (":8443",
   bukan dibatasi per-hostname) — lihat bagian Configuration di atas.
2. Test manual dari dalam container reverse proxy:
   curl -vk https://<nama-container-app>:8443/
3. Kalau forward_host di reverse proxy diisi domain publik, gunakan --resolve
   atau pastikan DNS/hosts di reverse proxy mengarah ke container yang benar.
```

### Kode Baru Tidak Ter-apply Setelah Deploy
```
Penyebab: opcache.validate_timestamps=0 + worker Octane persistent, jadi
perubahan file PHP tidak otomatis kepakai sampai worker di-restart.

Solusi:
docker compose exec app php artisan octane:reload
# atau kalau perlu restart penuh:
docker compose restart app
```

### Port Already in Use
```
docker: Error response from daemon: Ports are not available

Solusi:
Ubah port di .env atau stop service yang menggunakan port tersebut:
sudo lsof -i :8884
sudo kill -9 <PID>
```

### Performance Issues / Memory Leak
- Monitor resources: `docker stats`
- Pantau RSS memory container dari waktu ke waktu — kenaikan terus-menerus tanpa turun adalah indikasi memory leak di worker (state static/singleton yang tidak di-reset antar request)
- `OCTANE_MAX_REQUESTS` membatasi umur worker sebagai mitigasi sementara, bukan solusi permanen
- Check memory limits di `.env`
- Review PostgreSQL logs: `./logs/postgresql/`
- Check Caddy access log: `./logs/caddy/`

## 📂 Directory Structure

```
lepp/
├── README.md                      # Dokumentasi ini
├── .env                           # Environment variables
├── docker-compose.yml             # Main compose file
├── .gitignore                     # Git ignore rules
│
├── bin/
│   ├── frankenphp/
│   │   └── Dockerfile            # FrankenPHP + PHP 8.4 image definition (aktif)
│   └── php8/
│       └── Dockerfile            # PHP 8.4 FPM image lama (dipakai di branch main)
│
├── config/
│   ├── php/
│   │   └── php.ini               # PHP configuration (opcache aktif untuk Octane)
│   ├── frankenphp/
│   │   ├── Caddyfile             # Konfigurasi web server FrankenPHP (TLS, gzip, worker)
│   │   └── supervisord.conf      # Supervisor config (proses "octane")
│   ├── nginx/
│   │   └── nginx.conf            # Nginx config lama (tidak dipakai di branch ini)
│   ├── postgresql/
│   │   ├── postgresql.conf       # PostgreSQL settings
│   │   └── pg_hba.conf           # Authentication rules
│   ├── supervisor/
│   │   └── supervisord.conf      # Supervisor config lama (php-fpm, branch main)
│   └── ssl/
│       └── (SSL certificates)
│
├── logs/
│   ├── php/                      # PHP error logs
│   ├── caddy/                    # Caddy/FrankenPHP access logs
│   ├── nginx/                    # Nginx access/error logs (branch main)
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
9. FrankenPHP/Octane jalan non-root (`www-data`) — jangan tambahkan `user: root` di compose kecuali benar-benar perlu
10. Ingat: `opcache.validate_timestamps=0` — restart/reload Octane wajib setiap deploy kode baru

## 📊 Resource Monitoring

Check container resource usage:
```bash
# Real-time monitoring
docker stats

# View container details
docker compose ps -a

# Check disk usage
docker system df

# Status worker Octane
docker compose exec app php artisan octane:status
```

Adjust limits di `.env`:
```env
APP_LIMITS_CPU=0.50          # CPU cores
APP_LIMITS_MEMORY=4096M      # Memory allocation
OCTANE_WORKERS=auto          # atau angka spesifik, mis. 4
OCTANE_MAX_REQUESTS=500      # recycle worker setelah N request
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

> Catatan: `display_errors = On` di worker Octane akan menampilkan error ke response HTML seperti biasa, tapi pastikan di-nonaktifkan lagi sebelum production karena worker persistent membuat kebocoran informasi lebih terasa (semua request lewat worker yang sama).

### View Supervisor Status
```bash
docker compose exec app supervisorctl status
```

## 📚 Useful Resources

- [Laravel Documentation](https://laravel.com/docs)
- [Laravel Octane Documentation](https://laravel.com/docs/octane)
- [FrankenPHP Documentation](https://frankenphp.dev/docs/)
- [PostgreSQL Docs](https://www.postgresql.org/docs/)
- [Docker Compose Reference](https://docs.docker.com/compose/reference/)
- [PHP Configuration](https://www.php.net/manual/en/ini.list.php)

## ✨ Next Steps

1. Copy `APP_PATH` dari `.env` ke project folder:
   ```bash
   cp -r /home/user/Projects/lepp /path/to/laravel/app
   ```

2. Configure database connection & Octane di Laravel `.env`:
   ```env
   DB_CONNECTION=pgsql
   DB_HOST=postgresql
   DB_PORT=5432
   DB_DATABASE=app_db
   DB_USERNAME=app
   DB_PASSWORD=app_pass
   
   REDIS_HOST=redis_cache
   REDIS_PASSWORD=redis_pass

   OCTANE_SERVER=frankenphp
   OCTANE_HTTPS=true
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

5. Pastikan `composer.json` project berisi `laravel/octane` dan `config/octane.php` sudah ter-publish (`php artisan vendor:publish --tag=octane-config`).

---

**Created**: May 12, 2026  
**Last Updated**: July 14, 2026 — migrasi ke Laravel Octane + FrankenPHP (branch `octane-production`)
