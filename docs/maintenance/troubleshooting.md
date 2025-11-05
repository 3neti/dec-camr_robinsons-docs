# Troubleshooting

This guide provides comprehensive troubleshooting procedures for common issues encountered in the CAMR (Centralized Automated Meter Reading) Robinsons system. Issues are categorized by component for easier navigation.

---

## 🔍 Diagnostic Starting Points

### Quick Diagnostics

Before diving into specific issues, run these quick checks:

```bash
# Check application status
php artisan about

# Check Laravel logs
tail -f storage/logs/laravel.log

# Check web server logs
# Apache:
tail -f /var/log/apache2/error.log
# Nginx:
tail -f /var/log/nginx/error.log

# Check MySQL status
sudo systemctl status mysql

# Test database connection
php artisan tinker
>>> DB::connection()->getPdo();
```

### Enable Debug Mode (Development Only)

```ini
# .env
APP_DEBUG=true
APP_ENV=local
```

**⚠️ Warning:** Never enable `APP_DEBUG=true` in production as it exposes sensitive information.

---

## 🗄️ Database Issues

### Issue 1: Database Connection Failed

**Symptoms:**
- "SQLSTATE[HY000] [2002] Connection refused"
- "SQLSTATE[HY000] [1045] Access denied for user"
- Application shows blank page or 500 error

**Diagnosis:**

```bash
# Check MySQL is running
sudo systemctl status mysql

# Test connection manually
mysql -u root -p -e "SHOW DATABASES;"

# Check Laravel can connect
php artisan tinker
>>> DB::connection()->getPdo();
```

**Solutions:**

**1. MySQL not running:**
```bash
sudo systemctl start mysql
sudo systemctl enable mysql  # Auto-start on boot
```

**2. Wrong credentials:**
```ini
# .env - Verify these settings
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=meter_reading_robinsons
DB_USERNAME=your_username
DB_PASSWORD=your_password
```

**3. Database doesn't exist:**
```bash
mysql -u root -p -e "CREATE DATABASE meter_reading_robinsons CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
```

**4. User lacks privileges:**
```sql
GRANT ALL PRIVILEGES ON meter_reading_robinsons.* TO 'camr_user'@'localhost';
FLUSH PRIVILEGES;
```

### Issue 2: Migration Failures

**Symptoms:**
- "SQLSTATE[42S01]: Base table or view already exists"
- "SQLSTATE[42S02]: Base table or view not found"
- Migrations hang or timeout

**Solutions:**

**1. Table already exists:**
```bash
# Check which migrations have run
php artisan migrate:status

# Rollback last batch
php artisan migrate:rollback

# Or start fresh (CAUTION: destroys data)
php artisan migrate:fresh
```

**2. Migration timeout:**
```ini
# .env - Increase timeout
DB_TIMEOUT=60
```

```php
// In migration file
Schema::connection()->getDoctrineSchemaManager()->getDatabasePlatform()->registerDoctrineTypeMapping('enum', 'string');
```

**3. Foreign key constraint errors:**
```bash
# Disable foreign key checks temporarily
mysql -u root -p meter_reading_robinsons -e "SET FOREIGN_KEY_CHECKS=0;"

# Run migrations
php artisan migrate

# Re-enable checks
mysql -u root -p meter_reading_robinsons -e "SET FOREIGN_KEY_CHECKS=1;"
```

### Issue 3: Query Performance Issues

**Symptoms:**
- Slow page loads
- Timeouts on report generation
- Gateway/meter lists take long to load

**Diagnosis:**

```bash
# Enable query logging
php artisan tinker
>>> DB::enableQueryLog();
>>> // Run your query
>>> dd(DB::getQueryLog());
```

**Solutions:**

**1. Missing indexes:**
```sql
-- Add indexes to frequently queried columns
CREATE INDEX idx_meter_gateway ON meter_table(meter_gateway_id);
CREATE INDEX idx_gateway_site ON meter_gateway_table(gateway_site_id);
CREATE INDEX idx_site_code ON meter_site(site_code);
```

**2. Large result sets:**
```php
// Use chunk() for large datasets
DB::table('meter_table')->chunk(100, function ($meters) {
    foreach ($meters as $meter) {
        // Process meter
    }
});

// Or use cursor() for memory efficiency
foreach (DB::table('meter_table')->cursor() as $meter) {
    // Process meter
}
```

**3. N+1 query problem:**
```php
// Bad: N+1 queries
$gateways = Gateway::all();
foreach ($gateways as $gateway) {
    echo $gateway->site->site_name; // Queries site for each gateway
}

// Good: Eager loading
$gateways = Gateway::with('site')->get();
```

---

## 🌐 Web Server Issues

### Issue 4: 404 Not Found on All Routes

**Symptoms:**
- Homepage works but all other pages show 404
- "The requested URL was not found on this server"

**Cause:** `.htaccess` not working or `mod_rewrite` disabled

**Solutions:**

**Apache:**
```bash
# Enable mod_rewrite
sudo a2enmod rewrite
sudo systemctl restart apache2

# Check .htaccess exists
ls -la public/.htaccess

# Verify AllowOverride is set
# In /etc/apache2/sites-available/camr.conf
<Directory /var/www/camr/public>
    AllowOverride All
</Directory>
```

**Nginx:**
```nginx
# Ensure try_files is configured
location / {
    try_files $uri $uri/ /index.php?$query_string;
}
```

### Issue 5: 500 Internal Server Error

**Symptoms:**
- Blank page or generic 500 error
- No detailed error message

**Diagnosis:**

```bash
# Check Laravel logs
tail -f storage/logs/laravel.log

# Check web server logs
# Apache
tail -f /var/log/apache2/error.log
# Nginx
tail -f /var/log/nginx/error.log

# Enable debug temporarily
# .env
APP_DEBUG=true
```

**Common Causes:**

**1. Permission issues:**
```bash
chmod -R 775 storage bootstrap/cache
chown -R www-data:www-data storage bootstrap/cache
```

**2. Missing .env file:**
```bash
cp .env.example .env
php artisan key:generate
```

**3. Composer dependencies:**
```bash
composer install --no-dev
```

**4. PHP memory limit:**
```ini
; php.ini
memory_limit = 256M
```

### Issue 6: CSS/JS Not Loading

**Symptoms:**
- Page loads but no styling
- JavaScript functionality broken
- Console shows 404 for asset files

**Solutions:**

**1. Assets not compiled:**
```bash
npm install
npm run prod  # For production
# or
npm run dev   # For development
```

**2. Wrong asset URL:**
```ini
# .env
ASSET_URL=http://your-domain.com
```

**3. Symlink missing:**
```bash
php artisan storage:link
```

**4. Cache issue:**
```bash
php artisan cache:clear
php artisan config:clear
php artisan view:clear
```

---

## 🔐 Authentication Issues

### Issue 7: Cannot Login (Invalid Credentials)

**Symptoms:**
- Correct password doesn't work
- "Invalid username or password" error
- Password reset doesn't help

**Diagnosis:**

```bash
# Check user exists
php artisan tinker
>>> User::where('user_name', 'admin')->first();

# Check password hash
>>> $user = User::where('user_name', 'admin')->first();
>>> Hash::check('your-password', $user->user_password);
```

**Solutions:**

**1. Reset password manually:**
```bash
php artisan tinker
>>> $user = User::where('user_name', 'admin')->first();
>>> $user->user_password = Hash::make('newpassword123');
>>> $user->save();
```

**2. Check session configuration:**
```ini
# .env
SESSION_DRIVER=file
SESSION_LIFETIME=120
```

**3. Clear sessions:**
```bash
php artisan session:table
php artisan migrate
rm -rf storage/framework/sessions/*
```

### Issue 8: Logged Out Immediately After Login

**Symptoms:**
- Login succeeds but redirects back to login
- Session not persisting

**Solutions:**

**1. Session permissions:**
```bash
chmod -R 775 storage/framework/sessions
chown -R www-data:www-data storage/framework/sessions
```

**2. Session configuration:**
```ini
# .env
SESSION_DRIVER=database  # Try database instead of file
SESSION_DOMAIN=.yourdomain.com
SESSION_SECURE_COOKIE=false  # Set to true if using HTTPS
```

```bash
# Create sessions table
php artisan session:table
php artisan migrate
```

**3. Cookie domain mismatch:**
```php
// config/session.php
'domain' => env('SESSION_DOMAIN', null),
'secure' => env('SESSION_SECURE_COOKIE', false),
'same_site' => 'lax',
```

### Issue 9: Password Reset Email Not Sending

**Symptoms:**
- "Email sent" message but no email received
- SMTP errors in logs

**See:** [Email Configuration Troubleshooting](../configuration/email.md#troubleshooting)

---

## 📡 Gateway & Device Issues

### Issue 10: Gateway Not Connecting

**Symptoms:**
- Gateway shows as offline
- No data received from gateway
- HTTP POST requests not arriving

**Diagnosis:**

```bash
# Check gateway API logs
tail -f storage/logs/laravel.log | grep "check_time\|csv_update\|load_profile"

# Test gateway connectivity
curl http://your-server.com/check_time.php

# Check firewall
sudo ufw status
```

**Solutions:**

**1. Firewall blocking:**
```bash
# Allow gateway IP
sudo ufw allow from <gateway-ip> to any port 80
sudo ufw allow from <gateway-ip> to any port 443
```

**2. Wrong gateway configuration:**
```bash
# Verify gateway settings in database
php artisan tinker
>>> Gateway::where('gateway_mac_address', 'XX:XX:XX:XX:XX:XX')->first();
```

**3. Route not accessible:**
```bash
# Test from gateway's perspective
curl -X GET http://your-server.com/check_time.php
curl -X GET http://your-server.com/rtu/index.php/rtu/rtu_check_update/<MAC>/get_update_csv
```

### Issue 11: Load Profile Data Not Uploading

**Symptoms:**
- Gateway connects but load profile files not processed
- Files not appearing in storage
- Database not updated with meter readings

**Diagnosis:**

```bash
# Check load profile logs
tail -f storage/logs/laravel.log | grep "LoadProfile"

# Check storage permissions
ls -la storage/app/load_profiles/

# Check file uploads in PHP
php -i | grep upload_max_filesize
```

**Solutions:**

**1. Storage permissions:**
```bash
mkdir -p storage/app/load_profiles
chmod -R 775 storage/app
chown -R www-data:www-data storage/app
```

**2. PHP upload limits:**
```ini
; php.ini
upload_max_filesize = 100M
post_max_size = 100M
max_execution_time = 300
```

```bash
sudo systemctl restart apache2  # or nginx
```

**3. Route not handling POST:**
```bash
# Check route exists
php artisan route:list | grep receive_file
```

### Issue 12: CSV Configuration Update Failing

**Symptoms:**
- Gateway requests CSV but receives error
- "CSV update not available" from gateway
- Flag not resetting after download

**Solutions:**

**1. Check CSV update flag:**
```bash
php artisan tinker
>>> $gateway = Gateway::where('gateway_mac_address', 'MAC')->first();
>>> $gateway->gateway_csv_update; // Should be 1 for update pending
```

**2. Verify CSV generation:**
```php
// Test CSV content generation
$gateway = Gateway::find($id);
$meters = Meter::where('meter_gateway_id', $gateway->gateway_id)->get();
// Check meters exist
```

**3. Reset update flag manually:**
```bash
php artisan tinker
>>> Gateway::where('gateway_mac_address', 'MAC')->update(['gateway_csv_update' => 0]);
```

---

## 📊 Report Generation Issues

### Issue 13: Report Generation Timeout

**Symptoms:**
- "Maximum execution time exceeded"
- Report page hangs
- Partial data in exported Excel

**Solutions:**

**1. Increase PHP timeout:**
```ini
; php.ini
max_execution_time = 600
max_input_time = 600
```

**2. Optimize query:**
```php
// Use date range limits
$startDate = Carbon::now()->subDays(30);
$endDate = Carbon::now();

// Use chunk for large datasets
DB::table('meter_readings')->whereBetween('timestamp', [$startDate, $endDate])
    ->chunk(1000, function($readings) {
        // Process
    });
```

**3. Use queue for large reports:**
```bash
# Configure queue
php artisan queue:table
php artisan migrate

# Process queue
php artisan queue:work
```

### Issue 14: Excel Export Empty or Corrupted

**Symptoms:**
- Downloaded Excel file is empty
- File won't open in Excel
- "File format is not valid" error

**Solutions:**

**1. Check maatwebsite/excel installation:**
```bash
composer require maatwebsite/excel
php artisan vendor:publish --provider="Maatwebsite\Excel\ExcelServiceProvider"
```

**2. Memory limit:**
```ini
; php.ini
memory_limit = 512M
```

**3. Check data exists:**
```php
// Verify query returns data
$data = ReportController::getReportData($filters);
dd($data); // Should show records
```

### Issue 15: Missing Data in SAP/RAW Reports

**Symptoms:**
- Report generates but some meters missing
- Date range shows no data
- Meters exist but readings don't appear

**Solutions:**

**1. Verify meter has readings:**
```bash
php artisan tinker
>>> $meter = Meter::where('meter_serial_number', 'SERIAL')->first();
>>> DB::table('meter_load_profile_table')->where('meter_id', $meter->meter_id)->count();
```

**2. Check date range:**
```php
// Ensure date format matches database
$startDate = Carbon::parse($request->start_date)->format('Y-m-d H:i:s');
```

**3. Configuration file assignment:**
```bash
# Verify meter has config file assigned
php artisan tinker
>>> Meter::whereNull('meter_configuration_file_id')->count(); // Should be 0
```

---

## 💾 Performance Issues

### Issue 16: Slow Page Load Times

**Symptoms:**
- Pages take >5 seconds to load
- DataTables very slow
- High server load

**Diagnosis:**

```bash
# Check server resources
top
htop
df -h  # Disk space
free -h  # Memory

# Enable query log to find slow queries
php artisan debugbar:publish  # If using debugbar
```

**Solutions:**

**1. Enable caching:**
```bash
php artisan config:cache
php artisan route:cache
php artisan view:cache
```

**2. Optimize database:**
```sql
-- Analyze tables
ANALYZE TABLE meter_table, meter_gateway_table, meter_site;

-- Optimize tables
OPTIMIZE TABLE meter_table, meter_gateway_table;
```

**3. Use Redis for caching:**
```ini
# .env
CACHE_DRIVER=redis
SESSION_DRIVER=redis
REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379
```

```bash
# Install Redis
sudo apt-get install redis-server
composer require predis/predis
```

### Issue 17: Out of Memory Errors

**Symptoms:**
- "Allowed memory size exhausted"
- Process killed by system
- Report generation fails

**Solutions:**

**1. Increase PHP memory:**
```ini
; php.ini
memory_limit = 512M  # or higher
```

**2. Use chunk/cursor:**
```php
// Instead of get() for large datasets
Meter::chunk(100, function($meters) {
    // Process
});
```

**3. Unset large variables:**
```php
$largeArray = processData();
// Use data
unset($largeArray); // Free memory
gc_collect_cycles(); // Force garbage collection
```

---

## 🔧 Maintenance Issues

### Issue 18: Composer Install Fails

**Symptoms:**
- "Your requirements could not be resolved"
- Version conflict errors
- Package not found

**Solutions:**

**1. Update composer:**
```bash
composer self-update
```

**2. Clear composer cache:**
```bash
composer clear-cache
composer install
```

**3. Delete vendor and reinstall:**
```bash
rm -rf vendor composer.lock
composer install
```

### Issue 19: NPM Build Errors

**Symptoms:**
- "npm ERR! code ELIFECYCLE"
- Assets not compiling
- Webpack errors

**Solutions:**

**1. Clear npm cache:**
```bash
rm -rf node_modules package-lock.json
npm cache clean --force
npm install
```

**2. Update Node:**
```bash
# Using nvm
nvm install --lts
nvm use --lts
```

**3. Fix permissions:**
```bash
sudo chown -R $USER:$GROUP ~/.npm
sudo chown -R $USER:$GROUP node_modules
```

### Issue 20: Activity Log Growing Too Large

**Symptoms:**
- Database size growing rapidly
- Slow queries on activity_log table
- Disk space issues

**Solutions:**

**1. Clean old activity logs:**
```bash
php artisan tinker
>>> Activity::where('created_at', '<', now()->subMonths(6))->delete();
```

**2. Create cleanup command:**
```php
// app/Console/Commands/CleanActivityLog.php
public function handle()
{
    $deleted = Activity::where('created_at', '<', now()->subMonths(3))->delete();
    $this->info("Deleted {$deleted} old activity log records");
}
```

**3. Schedule cleanup:**
```php
// app/Console/Kernel.php
protected function schedule(Schedule $schedule)
{
    $schedule->command('activity:clean')->monthly();
}
```

---

## 🔍 Diagnostic Commands

### Useful Artisan Commands

```bash
# Application info
php artisan about

# List all routes
php artisan route:list

# Test database connection
php artisan migrate:status

# Interactive shell
php artisan tinker

# Clear all caches
php artisan optimize:clear

# View Laravel logs in real-time
tail -f storage/logs/laravel.log
```

### Database Diagnostic Queries

```sql
-- Check table sizes
SELECT 
    table_name,
    ROUND(((data_length + index_length) / 1024 / 1024), 2) AS "Size (MB)"
FROM information_schema.TABLES
WHERE table_schema = "meter_reading_robinsons"
ORDER BY (data_length + index_length) DESC;

-- Count records per table
SELECT COUNT(*) FROM meter_table;
SELECT COUNT(*) FROM meter_gateway_table;
SELECT COUNT(*) FROM meter_load_profile_table;

-- Find offline gateways
SELECT gateway_serial_number, gateway_last_seen 
FROM meter_gateway_table 
WHERE gateway_last_seen < DATE_SUB(NOW(), INTERVAL 1 HOUR);

-- Check for orphaned meters (no gateway)
SELECT COUNT(*) FROM meter_table WHERE meter_gateway_id IS NULL;
```

---

## 📞 Getting Help

### Log Files to Check

1. **Laravel Application:** `storage/logs/laravel.log`
2. **Apache Access:** `/var/log/apache2/access.log`
3. **Apache Error:** `/var/log/apache2/error.log`
4. **Nginx Access:** `/var/log/nginx/access.log`
5. **Nginx Error:** `/var/log/nginx/error.log`
6. **MySQL Error:** `/var/log/mysql/error.log`
7. **PHP-FPM:** `/var/log/php8.0-fpm.log`

### Information to Gather

```bash
# System info
php -v
mysql --version
composer --version
node --version
npm --version

# Laravel info
php artisan --version
php artisan about

# Error details
tail -n 100 storage/logs/laravel.log

# Server resources
df -h
free -h
top -bn1 | head -20
```

---

## 🔗 Related Documentation

- [Installation Guide](../installation.md) - Initial setup and configuration
- [Known Issues](known-issues.md) - Documented bugs and limitations
- [Configuration](../configuration.md) - System configuration guides
- [Database Schema](../database-schema.md) - Database structure reference
- [Architecture](../architecture.md) - System architecture overview

---

## 📝 Summary

This troubleshooting guide covers the most common issues encountered in CAMR Robinsons deployments. For issues not covered here:

1. **Check logs first** - Most issues leave traces in Laravel or web server logs
2. **Enable debug mode temporarily** - Only in development environments
3. **Search GitHub issues** - Check if others have encountered similar problems
4. **Document your solution** - Help others by updating this guide

When reporting issues, always include:
- Laravel version (`php artisan --version`)
- PHP version (`php -v`)
- Error messages from logs
- Steps to reproduce
- Environment details (dev/staging/production)
