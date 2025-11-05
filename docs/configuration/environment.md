# ⚙️ Environment Setup

The CAMR system uses Laravel's **environment configuration** to manage application settings, database connections, mail servers, and other environment-specific parameters. All sensitive configuration is stored in the `.env` file, which is excluded from version control.

---

## 📋 Overview

**Configuration File:** `.env` (root directory)  
**Template:** `.env.example`  
**Framework:** Laravel 8  
**PHP Version:** 7.3+ / 8.0+

**Purpose:** Centralize all environment-specific settings in a single file that can be customized per deployment environment (local, staging, production).

---

## 🔑 Required Environment Variables

### Application Settings

```bash
APP_NAME=AMR
APP_ENV=local
APP_KEY=base64:aX5Yg7cTEia7Y/egSkcb/f0ofcATkG4NeTHMuTj5Kkg=
APP_DEBUG=true
APP_URL=http://localhost
```

| Variable | Description | Values | Production Recommendation |
|----------|-------------|--------|---------------------------|
| `APP_NAME` | Application name | AMR / Centralized AMR | AMR |
| `APP_ENV` | Environment type | local, staging, production | production |
| `APP_KEY` | Encryption key | base64:... (32 chars) | Generate unique key |
| `APP_DEBUG` | Debug mode | true / false | **false** (security risk if true) |
| `APP_URL` | Base URL | http://localhost | https://camr.yourdomain.com |

**Generate APP_KEY:**
```bash
php artisan key:generate
```

### Database Configuration

```bash path=/Users/rli/Documents/DEC/camr_robinsons-main/camr_robinsons-main/.env start=9
DB_CONNECTION=mysql
DB_HOST=localhost
DB_PORT=3380
DB_DATABASE=meter_reading_robinsons
DB_USERNAME=root
DB_PASSWORD=
```

| Variable | Description | Default | Production Example |
|----------|-------------|---------|--------------------|
| `DB_CONNECTION` | Database driver | mysql | mysql |
| `DB_HOST` | Database server | localhost | 192.168.1.100 |
| `DB_PORT` | Database port | 3380 | 3306 |
| `DB_DATABASE` | Database name | meter_reading_robinsons | meter_reading_robinsons |
| `DB_USERNAME` | Database user | root | camr_user |
| `DB_PASSWORD` | Database password | (empty) | **Strong password required** |

**Security Note:** Never use `root` user in production. Create dedicated database user with minimal required privileges.

### Session & Cache Configuration

```bash path=/Users/rli/Documents/DEC/camr_robinsons-main/camr_robinsons-main/.env start=17
BROADCAST_DRIVER=log
CACHE_DRIVER=file
QUEUE_CONNECTION=sync
SESSION_DRIVER=database
SESSION_LIFETIME=120
```

| Variable | Description | Values | Recommendation |
|----------|-------------|--------|----------------|
| `CACHE_DRIVER` | Cache storage | file, database, redis | redis (for production) |
| `SESSION_DRIVER` | Session storage | file, database | database (for multi-server) |
| `SESSION_LIFETIME` | Session timeout (minutes) | 120 | 120-180 |
| `QUEUE_CONNECTION` | Queue driver | sync, database | database (for async jobs) |

**Session Driver:** Uses `database` to store sessions in `sessions` table (better for load-balanced environments).

### Mail Configuration

```bash path=/Users/rli/Documents/DEC/camr_robinsons-main/camr_robinsons-main/.env start=27
MAIL_MAILER=smtp
MAIL_HOST=smtp.gmail.com
MAIL_PORT=587
MAIL_USERNAME=
MAIL_PASSWORD=
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS=
MAIL_FROM_NAME="Centralized Automated Meter Reading"
```

**Detailed in:** [Email Configuration](email.md)

### Logging

```bash
LOG_CHANNEL=stack
```

**Log Location:** `storage/logs/laravel.log`  
**Log Levels:** emergency, alert, critical, error, warning, notice, info, debug

---

## 🖥️ Server Requirements

### PHP Requirements

**Minimum PHP Version:** 7.3  
**Recommended:** PHP 8.0+

**Required PHP Extensions:**
```bash
- BCMath PHP Extension
- Ctype PHP Extension
- Fileinfo PHP Extension
- JSON PHP Extension
- Mbstring PHP Extension
- OpenSSL PHP Extension
- PDO PHP Extension
- Tokenizer PHP Extension
- XML PHP Extension
```

**Check PHP Version:**
```bash
php -v
```

**Check Extensions:**
```bash
php -m | grep -E "(bcmath|ctype|fileinfo|json|mbstring|openssl|pdo|tokenizer|xml)"
```

### Web Server

**Supported:**
- Apache 2.4+ (with mod_rewrite)
- Nginx 1.18+

**Document Root:** `/path/to/camr_robinsons/public`

**Apache .htaccess (included):**
```apache
<IfModule mod_rewrite.c>
    RewriteEngine On
    RewriteRule ^(.*)$ public/$1 [L]
</IfModule>
```

**Nginx Configuration:**
```nginx
server {
    listen 80;
    server_name camr.yourdomain.com;
    root /var/www/camr_robinsons/public;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";

    index index.php;

    charset utf-8;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    error_page 404 /index.php;

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.0-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

### Database

**MySQL Version:** 5.7+ or 8.0+  
**Database:** `meter_reading_robinsons`

**Create Database:**
```sql
CREATE DATABASE meter_reading_robinsons 
CHARACTER SET utf8mb4 
COLLATE utf8mb4_unicode_ci;

CREATE USER 'camr_user'@'localhost' IDENTIFIED BY 'secure_password';
GRANT ALL PRIVILEGES ON meter_reading_robinsons.* TO 'camr_user'@'localhost';
FLUSH PRIVILEGES;
```

---

## 🚀 Installation Steps

### 1. Clone Repository

```bash
git clone https://github.com/yourusername/camr_robinsons.git
cd camr_robinsons
```

### 2. Install Dependencies

```bash
composer install
npm install
```

### 3. Environment Configuration

```bash
# Copy environment template
cp .env.example .env

# Generate application key
php artisan key:generate
```

### 4. Configure .env File

Edit `.env` and update:
- Database credentials
- Mail server settings
- Application URL
- Debug mode (false for production)

### 5. Run Migrations

```bash
# Run database migrations
php artisan migrate

# Seed initial data (if available)
php artisan db:seed
```

### 6. Set Permissions

```bash
# Storage and cache directories
chmod -R 775 storage bootstrap/cache
chown -R www-data:www-data storage bootstrap/cache
```

### 7. Compile Assets

```bash
# Development
npm run dev

# Production (minified)
npm run production
```

### 8. Configure Web Server

- Point document root to `/path/to/camr_robinsons/public`
- Enable `.htaccess` (Apache) or configure Nginx
- Restart web server

### 9. Test Installation

Navigate to `http://your-domain.com` and verify login page appears.

---

## 🔧 Environment-Specific Configurations

### Local Development

```bash
APP_ENV=local
APP_DEBUG=true
APP_URL=http://localhost

DB_HOST=localhost
DB_PORT=3306

MAIL_MAILER=log  # Log emails instead of sending
```

### Staging Environment

```bash
APP_ENV=staging
APP_DEBUG=true
APP_URL=https://staging.camr.yourdomain.com

DB_HOST=staging-db.internal
DB_PORT=3306

MAIL_MAILER=smtp
MAIL_HOST=smtp.gmail.com
```

### Production Environment

```bash
APP_ENV=production
APP_DEBUG=false  # CRITICAL: Must be false
APP_URL=https://camr.yourdomain.com

DB_HOST=production-db.internal
DB_PORT=3306

CACHE_DRIVER=redis
SESSION_DRIVER=redis
QUEUE_CONNECTION=database

MAIL_MAILER=smtp
MAIL_HOST=smtp.company.com
```

---

## 🛡️ Security Considerations

### Production Checklist

✅ **Set APP_DEBUG=false**
```bash
APP_DEBUG=false
```

✅ **Use HTTPS**
```bash
APP_URL=https://camr.yourdomain.com
```

✅ **Strong Database Credentials**
```bash
DB_USERNAME=camr_user  # Not root!
DB_PASSWORD=Str0ng!P@ssw0rd123
```

✅ **Restrict File Permissions**
```bash
chmod 644 .env
chmod 755 storage
chown -R www-data:www-data storage
```

✅ **Enable HTTPS Redirection**
```php
// In App\Providers\AppServiceProvider::boot()
if (config('app.env') === 'production') {
    URL::forceScheme('https');
}
```

✅ **Disable Directory Listing**
```apache
# Apache
Options -Indexes
```

```nginx
# Nginx
autoindex off;
```

✅ **Configure Firewall**
```bash
# Only allow HTTP/HTTPS and SSH
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw enable
```

---

## 🛠️ Troubleshooting

### Issue: "No application encryption key"

**Error:** `RuntimeException: No application encryption key has been specified.`

**Solution:**
```bash
php artisan key:generate
```

### Issue: Permission Denied

**Error:** `The stream or file "storage/logs/laravel.log" could not be opened`

**Solution:**
```bash
chmod -R 775 storage bootstrap/cache
chown -R www-data:www-data storage
```

### Issue: Database Connection Failed

**Error:** `SQLSTATE[HY000] [2002] Connection refused`

**Solution:**
```bash
# Check database is running
sudo systemctl status mysql

# Verify credentials in .env
# Test connection
php artisan tinker
>>> DB::connection()->getPdo();
```

### Issue: 500 Internal Server Error

**Causes:**
- `.htaccess` not working (Apache mod_rewrite disabled)
- File permissions incorrect
- APP_DEBUG=false hiding errors

**Solution:**
```bash
# Enable debug temporarily
APP_DEBUG=true

# Check Laravel logs
tail -f storage/logs/laravel.log

# Enable Apache mod_rewrite
sudo a2enmod rewrite
sudo systemctl restart apache2
```

### Issue: Session Not Persisting

**Cause:** Session driver misconfigured or permissions issue

**Solution:**
```bash
# If using database sessions
php artisan session:table
php artisan migrate

# If using file sessions
chmod -R 775 storage/framework/sessions
```

---

## 📊 Performance Optimization

### Enable OpCache (Production)

```ini
; php.ini
opcache.enable=1
opcache.memory_consumption=128
opcache.max_accelerated_files=10000
opcache.revalidate_freq=60
```

### Cache Configuration

```bash
# Cache routes, config, views
php artisan config:cache
php artisan route:cache
php artisan view:cache

# Clear caches (after deployment)
php artisan cache:clear
php artisan config:clear
php artisan route:clear
php artisan view:clear
```

### Use Redis (Production)

```bash
CACHE_DRIVER=redis
SESSION_DRIVER=redis
QUEUE_CONNECTION=redis

REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379
```

---

## 🔗 Related Documentation

- **[Email Configuration](email.md)** - Mail server setup
- **[Database Schema](../database-schema.md)** - Database structure
- **[Installation Guide](../installation.md)** - Complete setup guide
- **[Configuration](../configuration.md)** - Application configuration overview

---

**Last Updated:** 2024-03-15  
**Document Version:** 1.0  
**Maintainer:** CAMR Development Team
