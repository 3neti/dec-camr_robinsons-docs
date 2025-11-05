# Installation Guide

## 📥 Prerequisites

Before installing CAMR Robinsons, ensure your system meets the following requirements:

### Server Requirements
- **PHP:** 7.3, 8.0, 8.1, or 8.2
- **MySQL:** 5.7+ or 8.0+
- **Web Server:** Apache 2.4+ or Nginx 1.18+
- **Composer:** 2.x
- **Node.js:** 14+ (for asset compilation)
- **NPM:** 6+

### PHP Extensions
Ensure the following PHP extensions are installed:
```bash
php -m | grep -E '(BCMath|Ctype|JSON|Mbstring|OpenSSL|PDO|Tokenizer|XML|cURL|GD)'
```

Required extensions:
- BCMath
- Ctype
- JSON
- Mbstring
- OpenSSL
- PDO
- PDO MySQL
- Tokenizer
- XML
- cURL
- GD or Imagick

## 🚀 Installation Steps

### Step 1: Clone or Extract the Repository

```bash
# If using Git
git clone <repository-url> camr-robinsons
cd camr-robinsons

# Or extract from archive
unzip camr-robinsons.zip
cd camr-robinsons
```

### Step 2: Install PHP Dependencies

```bash
composer install
```

This will install all required Laravel packages:
- Laravel Framework 8.x
- maatwebsite/excel (Excel import/export)
- yajra/laravel-datatables-oracle (DataTables)
- spatie/laravel-activitylog (Activity logging)
- phpoffice/phpspreadsheet (Spreadsheet processing)

### Step 3: Install JavaScript Dependencies

```bash
npm install
```

### Step 4: Configure Environment File

Copy the example environment file:

```bash
cp .env.example .env
```

Edit `.env` with your configuration:

```ini
# Application Settings
APP_NAME=AMR
APP_ENV=production  # Change to 'production' for live deployment
APP_DEBUG=false     # Set to false in production
APP_URL=http://your-domain.com

# Database Configuration
DB_CONNECTION=mysql
DB_HOST=localhost
DB_PORT=3306
DB_DATABASE=meter_reading_robinsons
DB_USERNAME=your_database_user
DB_PASSWORD=your_database_password

# Session Configuration
SESSION_DRIVER=database
SESSION_LIFETIME=120  # Session timeout in minutes

# Mail Configuration (for password reset)
MAIL_MAILER=smtp
MAIL_HOST=smtp.gmail.com
MAIL_PORT=587
MAIL_USERNAME=your-email@gmail.com
MAIL_PASSWORD=your-app-password
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS=your-email@gmail.com
MAIL_FROM_NAME="Centralized Automated Meter Reading"

# Cache & Queue (optional)
CACHE_DRIVER=file
QUEUE_CONNECTION=sync
```

**Important Environment Variables:**

| Variable | Description | Example |
|----------|-------------|--------|
| `APP_ENV` | Environment (local/production) | `production` |
| `APP_DEBUG` | Debug mode (true/false) | `false` |
| `DB_HOST` | Database server hostname | `localhost` |
| `DB_PORT` | MySQL port | `3306` |
| `DB_DATABASE` | Database name | `meter_reading_robinsons` |
| `SESSION_DRIVER` | Session storage | `database` |
| `MAIL_HOST` | SMTP server | `smtp.gmail.com` |

### Step 5: Generate Application Key

```bash
php artisan key:generate
```

This creates a unique encryption key for your application.

### Step 6: Create Database

Create the MySQL database:

```bash
mysql -u root -p
```

```sql
CREATE DATABASE meter_reading_robinsons CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
GRANT ALL PRIVILEGES ON meter_reading_robinsons.* TO 'your_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### Step 7: Run Database Migrations

Create all database tables:

```bash
php artisan migrate
```

This will create 16 tables:
- `user_access_group`
- `meter_company_table`
- `user_tb`
- `meter_building_table`
- `live_meter_data`
- `meter_configuration_file`
- `meter_location_table`
- `activity_log`
- `meter_site`
- `meter_details`
- `load_profile`
- `meter_rtu`
- `meter_division_table`
- `meter_data`
- `web_page_settings`
- `sessions`

### Step 8: Create Initial Admin User

Create the first admin user directly in the database:

```bash
php artisan tinker
```

```php
$user = new \App\Models\User();
$user->user_name = 'admin';
$user->user_real_name = 'System Administrator';
$user->user_password = bcrypt('your-secure-password');
$user->user_type = 'Admin';
$user->user_access = 'All';
$user->user_email_address = 'admin@example.com';
$user->user_expiration = '9999-12-31';
$user->created_by_user_idx = 1;
$user->save();
exit;
```

**Or using SQL:**

```sql
INSERT INTO user_tb (
    user_name, 
    user_real_name, 
    user_password, 
    user_type, 
    user_access, 
    user_email_address,
    user_expiration,
    created_by_user_idx,
    created_at
) VALUES (
    'admin',
    'System Administrator',
    '$2y$10$92IXUNpkjO0rOQ5byMi.Ye4oKoEa3Ro9llC/.og/at2.uheWG/igi',  -- 'password'
    'Admin',
    'All',
    'admin@example.com',
    '9999-12-31',
    1,
    NOW()
);
```

> **Note:** The default password hash above is for 'password'. Change it immediately after first login.

### Step 9: Compile Assets

Compile CSS and JavaScript assets:

```bash
# For development
npm run dev

# For production (minified)
npm run prod
```

### Step 10: Set Directory Permissions

Ensure Laravel can write to storage and cache directories:

```bash
chmod -R 775 storage bootstrap/cache
chown -R www-data:www-data storage bootstrap/cache
```

For development:
```bash
chmod -R 777 storage bootstrap/cache
```

### Step 11: Create Storage Link

Create symbolic link for public file storage:

```bash
php artisan storage:link
```

## 🌐 Web Server Configuration

### Apache Configuration

Create a virtual host configuration:

```apache
<VirtualHost *:80>
    ServerName camr.yourdomain.com
    DocumentRoot /var/www/camr-robinsons/public

    <Directory /var/www/camr-robinsons/public>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/camr-error.log
    CustomLog ${APACHE_LOG_DIR}/camr-access.log combined
</VirtualHost>
```

Enable required Apache modules:
```bash
sudo a2enmod rewrite
sudo a2enmod headers
sudo systemctl restart apache2
```

### Nginx Configuration

```nginx
server {
    listen 80;
    server_name camr.yourdomain.com;
    root /var/www/camr-robinsons/public;

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

Test and reload Nginx:
```bash
sudo nginx -t
sudo systemctl reload nginx
```

## 🔒 Security Configuration

### SSL/HTTPS Setup (Recommended for Production)

Using Let's Encrypt with Certbot:

```bash
sudo apt install certbot python3-certbot-apache  # For Apache
# OR
sudo apt install certbot python3-certbot-nginx   # For Nginx

sudo certbot --apache -d camr.yourdomain.com     # Apache
# OR
sudo certbot --nginx -d camr.yourdomain.com      # Nginx
```

### Firewall Configuration

```bash
sudo ufw allow 'Apache Full'  # For Apache
# OR
sudo ufw allow 'Nginx Full'   # For Nginx

sudo ufw allow 3306/tcp       # MySQL (only if remote access needed)
sudo ufw enable
```

## 📊 Post-Installation Setup

### 1. Login to the System

Navigate to: `http://your-domain.com`

Login with:
- Username: `admin`
- Password: (the password you set)

### 2. Configure Company and Division

Navigate to:
- **Company Management** - Add your company
- **Division Management** - Add divisions

### 3. Create First Site

1. Go to **Site Management**
2. Click **Create Site**
3. Fill in:
   - Site Code
   - Company
   - Division
   - Building information

### 4. Register Gateways

1. Go to **Gateway Management**
2. Click **Add Gateway**
3. Enter:
   - Serial Number
   - MAC Address
   - IP Address
   - Site assignment

### 5. Configure Meters

1. Go to **Meter Management**
2. Add meters individually or import via CSV
3. Assign:
   - Gateway
   - Building
   - Meter Location (EE Room)
   - Configuration file

## 🔧 Maintenance Commands

### Clear Application Cache

```bash
php artisan cache:clear
php artisan config:clear
php artisan route:clear
php artisan view:clear
```

### Optimize for Production

```bash
php artisan config:cache
php artisan route:cache
php artisan view:cache
composer install --optimize-autoloader --no-dev
```

### Database Backup

```bash
mysqldump -u root -p meter_reading_robinsons > camr_backup_$(date +%Y%m%d).sql
```

### Update Application

```bash
git pull origin main
composer install --no-dev
npm install
npm run prod
php artisan migrate
php artisan config:cache
php artisan route:cache
```

## 🐛 Troubleshooting

### Issue: "No application encryption key has been specified"

**Solution:**
```bash
php artisan key:generate
```

### Issue: Permission denied on storage/logs

**Solution:**
```bash
chmod -R 775 storage bootstrap/cache
chown -R www-data:www-data storage bootstrap/cache
```

### Issue: Database connection error

**Check:**
1. MySQL is running: `sudo systemctl status mysql`
2. Database credentials in `.env` are correct
3. Database exists: `mysql -u root -p -e "SHOW DATABASES;"`

### Issue: 500 Internal Server Error

**Check:**
1. Enable debug in `.env`: `APP_DEBUG=true`
2. Check Laravel logs: `tail -f storage/logs/laravel.log`
3. Check web server logs:
   - Apache: `/var/log/apache2/error.log`
   - Nginx: `/var/log/nginx/error.log`

## 📝 Environment-Specific Notes

### Development Environment

```ini
APP_ENV=local
APP_DEBUG=true
DB_HOST=localhost
```

### Production Environment

```ini
APP_ENV=production
APP_DEBUG=false
DB_HOST=your-production-db-host
```

**Additional Production Steps:**
1. Configure automated backups
2. Set up monitoring (logs, uptime)
3. Configure email properly for password resets
4. Set up SSL/HTTPS
5. Implement firewall rules
6. Regular security updates

## ✅ Verification Checklist

- [ ] PHP version 7.3+ installed
- [ ] MySQL 5.7+ installed and running
- [ ] Composer installed
- [ ] Node.js and NPM installed
- [ ] `.env` file configured
- [ ] Application key generated
- [ ] Database created
- [ ] Migrations run successfully
- [ ] Admin user created
- [ ] Assets compiled
- [ ] Storage permissions set
- [ ] Web server configured
- [ ] Can access application in browser
- [ ] Can login with admin credentials
- [ ] Email configuration tested (password reset)

---

**Installation Complete!** 

You can now proceed to configure your first site, gateways, and meters.

**Next Steps:**
- Read [Site Management](modules/site-management.md)
- Read [Gateway Management](modules/gateway-management.md)
- Read [Meter Management](modules/meter-management.md)
