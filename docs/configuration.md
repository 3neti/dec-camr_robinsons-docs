# Configuration

## 🔧 Overview

CAMR configuration is managed through environment files, database settings, and administrative interfaces.

## 📁 Configuration Files

### .env File

**Location:** Project root

**Key Settings:**
```bash
APP_NAME="CAMR Robinsons"
APP_ENV=production
APP_DEBUG=false
APP_URL=http://your-domain.com

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=meter_reading_robinsons
DB_USERNAME=camr_user
DB_PASSWORD=your_password

MAIL_MAILER=smtp
MAIL_HOST=smtp.gmail.com
MAIL_PORT=587
MAIL_USERNAME=your-email@gmail.com
MAIL_PASSWORD=your-app-password
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS=noreply@camr.com
MAIL_FROM_NAME="${APP_NAME}"

SESSION_LIFETIME=120
SESSION_DRIVER=file
```

## 🏢 Company & Division

**Tables:**
- `meter_company_table` - Companies/tenants
- `meter_division_table` - Property divisions

**Management:** Via admin interface

**Usage:**
- Site organization
- Report grouping
- Access control

## 📧 Email Configuration

**Purpose:**
- New user credentials
- Password reset
- System notifications

**Setup:**
1. Configure SMTP in `.env`
2. Test with `php artisan tinker`
3. Verify email delivery

**Gmail Example:**
```bash
MAIL_MAILER=smtp
MAIL_HOST=smtp.gmail.com
MAIL_PORT=587
MAIL_USERNAME=your-email@gmail.com
MAIL_PASSWORD=app-specific-password
MAIL_ENCRYPTION=tls
```

## 🔧 Configuration Files (Meter)

**Table:** `meter_configuration_file`

**Purpose:** Define meter communication protocols

**Common Files:**
- `default.cfg` - Standard ModBus RTU
- `schneider_pm5560.cfg` - Schneider PM5560
- `abb_m2m.cfg` - ABB M2M
- `socomec_diris.cfg` - Socomec Diris A

**Storage:** Gateway devices

## 🌐 Web Settings

**Table:** `web_page_settings`

**Settings:**
- Application name
- Logo
- Theme colors
- Footer text
- Contact information

## 🔐 Security Settings

**Session:**
- Timeout: 2 hours (configurable)
- Driver: file/database/redis

**CSRF:**
- Enabled by default
- Laravel middleware

**Password:**
- Min: 6 characters
- Max: 20 characters
- Hashing: bcrypt

## Related

- [Installation](installation.md)
- [Environment Setup](configuration/environment.md)
- [User Management](user-management.md)
