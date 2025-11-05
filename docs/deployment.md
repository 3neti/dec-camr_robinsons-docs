# DevOps & Deployment

**Status:** ❌ Not documented - See [TODO-SECTIONS.md](TODO-SECTIONS.md)

This document will cover hosting, deployment, queue workers, scheduler setup, backups, and CI/CD pipelines.

## Server Requirements

### Minimum Specs
- **PHP:** 8.2+
- **PostgreSQL:** 14+
- **Redis:** 6+
- **Node.js:** 18+ (for asset compilation)
- **Supervisor:** For queue workers

### PHP Extensions
- BCMath
- Ctype
- JSON
- Mbstring
- OpenSSL
- PDO (pgsql)
- Tokenizer
- XML

---

## Hosting Options (TODO)

### Laravel Forge
- One-click Laravel deployment
- Automatic server provisioning
- Queue worker management
- SSL certificates

### Laravel Vapor
- Serverless deployment on AWS
- Auto-scaling
- Database and cache management

### DigitalOcean / Linode / Vultr
- Manual server setup
- Cost-effective
- Full control

---

## Deployment Workflow (TODO)

### Zero-Downtime Deployment

```bash
# 1. Git pull
git pull origin main

# 2. Install dependencies
composer install --no-dev --optimize-autoloader
npm ci && npm run build

# 3. Run migrations
php artisan migrate --force

# 4. Clear/cache config
php artisan config:cache
php artisan route:cache
php artisan view:cache

# 5. Restart queue workers
php artisan queue:restart

# 6. Reload PHP-FPM (if needed)
sudo service php8.2-fpm reload
```

### Deployment Script

```bash
#!/bin/bash
set -e

echo "Deploying Text Commander..."

cd /var/www/txtcmdr

# Maintenance mode
php artisan down

# Pull latest
git pull origin main

# Dependencies
composer install --no-dev --optimize-autoloader
npm ci && npm run build

# Migrate
php artisan migrate --force

# Optimize
php artisan optimize
php artisan config:cache
php artisan route:cache
php artisan view:cache

# Restart workers
php artisan queue:restart

# Exit maintenance
php artisan up

echo "Deployment complete!"
```

---

## Queue Workers (TODO)

### Supervisor Configuration

```ini
[program:txtcmdr-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/txtcmdr/artisan queue:work redis --sleep=3 --tries=3 --max-time=3600
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=www-data
numprocs=8
redirect_stderr=true
stdout_logfile=/var/www/txtcmdr/storage/logs/worker.log
stopwaitsecs=3600
```

```bash
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start txtcmdr-worker:*
```

**See:** [Jobs & Commands](jobs-commands.md)

---

## Scheduler Setup (TODO)

### Cron Entry

```cron
* * * * * cd /var/www/txtcmdr && php artisan schedule:run >> /dev/null 2>&1
```

### Scheduled Tasks

```php
// app/Console/Kernel.php
protected function schedule(Schedule $schedule)
{
    $schedule->command('sms:process-scheduled')->everyMinute();
    $schedule->command('backup:run')->daily();
}
```

---

## Backup Strategy (TODO)

### Database Backups

```bash
# Daily backup script
#!/bin/bash
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
pg_dump txtcmdr > /backups/txtcmdr_$TIMESTAMP.sql
gzip /backups/txtcmdr_$TIMESTAMP.sql

# Keep only last 30 days
find /backups -name "txtcmdr_*.sql.gz" -mtime +30 -delete
```

### File Backups

- Storage directory (`storage/app`)
- User uploads
- Logs (optional, for audit)

---

## CI/CD Pipeline (TODO)

### GitHub Actions Example

```yaml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.2'
      
      - name: Install dependencies
        run: composer install --no-dev --optimize-autoloader
      
      - name: Run tests
        run: php artisan test
      
      - name: Deploy to server
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.HOST }}
          username: ${{ secrets.USERNAME }}
          key: ${{ secrets.SSH_KEY }}
          script: |
            cd /var/www/txtcmdr
            ./deploy.sh
```

---

## Environment Configuration (TODO)

### .env.production Example

```env
APP_NAME="Text Commander"
APP_ENV=production
APP_DEBUG=false
APP_URL=https://txtcmdr.example.com

DB_CONNECTION=pgsql
DB_HOST={{DB_HOST}}
DB_DATABASE={{DB_NAME}}
DB_USERNAME={{DB_USER}}
DB_PASSWORD={{DB_PASSWORD}}

CACHE_DRIVER=redis
QUEUE_CONNECTION=redis
SESSION_DRIVER=redis

REDIS_HOST={{REDIS_HOST}}
REDIS_PASSWORD={{REDIS_PASSWORD}}

ENGAGESPARK_API_KEY={{API_KEY}}
ENGAGESPARK_ORG_ID={{ORG_ID}}

MAIL_MAILER=smtp
MAIL_HOST={{MAIL_HOST}}
MAIL_PORT=587
MAIL_USERNAME={{MAIL_USERNAME}}
MAIL_PASSWORD={{MAIL_PASSWORD}}
```

---

## Rollback Procedure (TODO)

```bash
#!/bin/bash
# Rollback to previous release

cd /var/www/txtcmdr

# Put in maintenance mode
php artisan down

# Checkout previous commit
git reset --hard HEAD~1

# Restore dependencies
composer install --no-dev --optimize-autoloader

# Rollback migrations (if needed)
php artisan migrate:rollback --step=1

# Optimize
php artisan optimize

# Restart workers
php artisan queue:restart

# Exit maintenance
php artisan up
```

---

## Related Documentation

- [Jobs & Commands](jobs-commands.md)
- [Logging & Monitoring](logging-monitoring.md)
- [Security Guide](security.md)
- [TODO-SECTIONS.md](TODO-SECTIONS.md)
