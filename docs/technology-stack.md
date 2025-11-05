# 🛠️ Technology Stack

The CAMR system is built using modern web technologies and follows industry best practices for enterprise-level PHP applications. This document provides a comprehensive overview of all technologies, frameworks, libraries, and tools used in the system.

---

## 📋 Stack Overview

| Layer | Technology | Version | Purpose |
|-------|------------|---------|----------|
| **Backend Framework** | Laravel | 8.x | PHP web application framework |
| **Language** | PHP | 7.3+ / 8.0+ | Server-side programming |
| **Database** | MySQL | 5.7+ / 8.0+ | Relational database |
| **Frontend** | Blade + Bootstrap | - | Templating + UI framework |
| **JavaScript** | jQuery + DataTables | - | Client-side interactions |
| **Web Server** | Apache / Nginx | 2.4+ / 1.18+ | HTTP server |
| **Gateway Scripts** | Python | 3.7+ | Field device communication |
| **Protocol** | Modbus RTU/TCP | - | Meter communication |

---

## 🔧 Backend Technologies

### PHP & Laravel

**PHP Version:** 7.3 / 7.4 / 8.0 (recommended 8.0+)

**Laravel Version:** 8.x

**Why Laravel:**
- **MVC Architecture** - Clean separation of concerns
- **Eloquent ORM** - Intuitive database interactions
- **Blade Templating** - Powerful yet simple view engine
- **Middleware** - Request filtering and authentication
- **Artisan CLI** - Command-line tools for development
- **Composer** - Dependency management
- **Built-in Security** - CSRF protection, password hashing

**Key Laravel Features Used:**
```php
// Eloquent ORM
$sites = SiteModel::with('building', 'division')->get();

// Blade Templating
@extends('layouts.app')
@section('content')
    <h1>{{ $title }}</h1>
@endsection

// Middleware
Route::get('/site', [SiteController::class, 'index'])
    ->middleware('isLoggedIn');

// Validation
$request->validate([
    'email' => 'required|email',
    'password' => 'required|min:6'
]);
```

---

## 📦 Core Dependencies

### PHP Packages (composer.json)

```json
{
    "require": {
        "php": "^7.3|^8.0",
        "laravel/framework": "^8.0",
        "maatwebsite/excel": "^3.1",
        "yajra/laravel-datatables-oracle": "^9.0",
        "spatie/laravel-activitylog": "^3.0",
        "phpoffice/phpspreadsheet": "^1.18"
    }
}
```

#### maatwebsite/excel

**Purpose:** Excel file import/export  
**Usage:** SAP, RAW, Consumption, Demand reports  
**Version:** 3.1+

**Features:**
- Excel file generation from arrays/collections
- Template-based export (uses pre-formatted .xlsx files)
- Large dataset handling with chunking
- Multiple sheet support

**Example:**
```php
use PhpOffice\PhpSpreadsheet\IOFactory;

$spreadSheet = IOFactory::load(public_path('/template/SAP_Report.xlsx'));
$spreadSheet->getActiveSheet()
    ->setCellValue('A2', $data->meter_name)
    ->setCellValue('B2', $data->consumption);
$writer = new Xlsx($spreadSheet);
$writer->save('php://output');
```

#### yajra/laravel-datatables-oracle

**Purpose:** Server-side DataTables processing  
**Usage:** All data tables (sites, gateways, meters, users)  
**Version:** 9.0+

**Features:**
- Server-side pagination
- Searching and sorting
- Custom column rendering
- AJAX request handling

**Example:**
```php
use DataTables;

public function getSiteList(Request $request)
{
    $data = SiteModel::select('site_id', 'site_code', 'building_description');
    
    return DataTables::of($data)
        ->addIndexColumn()
        ->addColumn('action', function($row){
            return '<button data-id="'.$row->site_id.'">Edit</button>';
        })
        ->rawColumns(['action'])
        ->make(true);
}
```

#### spatie/laravel-activitylog

**Purpose:** Activity logging and audit trail  
**Usage:** Track user actions on critical operations  
**Version:** 3.0+

**Features:**
- Log model changes (create, update, delete)
- Custom activity logging
- User association
- Properties storage (old/new values)

**Example:**
```php
activity()
    ->causedBy(auth()->user())
    ->performedOn($site)
    ->withProperties(['old' => $oldValues, 'new' => $newValues])
    ->log('Updated site information');
```

#### phpoffice/phpspreadsheet

**Purpose:** Advanced spreadsheet manipulation  
**Usage:** Excel report generation with formatting  
**Version:** 1.18+

**Features:**
- Cell formatting (borders, colors, alignment)
- Formula support
- Multiple worksheets
- Reading existing templates

---

## 🎨 Frontend Technologies

### HTML/CSS Framework

**Bootstrap:** 4.x / 5.x  
**Bootstrap Icons:** Latest

**Why Bootstrap:**
- Responsive grid system
- Pre-built UI components
- Cross-browser compatibility
- Extensive documentation

**Components Used:**
- Cards (dashboard panels)
- Modals (forms and dialogs)
- Tables (data display)
- Forms (input validation)
- Navbars (navigation)
- Buttons (actions)
- Alerts (notifications)

### JavaScript Libraries

#### jQuery

**Version:** 3.x  
**Purpose:** DOM manipulation and AJAX

**Usage:**
```javascript
// AJAX form submission
$('#createSiteForm').on('submit', function(e){
    e.preventDefault();
    $.ajax({
        url: '/create_site_post',
        type: 'POST',
        data: $(this).serialize(),
        success: function(response) {
            // Handle response
        }
    });
});
```

#### DataTables.js

**Version:** 1.10+  
**Purpose:** Interactive HTML tables

**Features:**
- Pagination
- Searching
- Sorting
- Column filtering
- Server-side processing
- Export (PDF, Excel, CSV)

**Implementation:**
```javascript
$('#siteTable').DataTable({
    processing: true,
    serverSide: true,
    ajax: '/site/list',
    columns: [
        { data: 'site_id' },
        { data: 'site_code' },
        { data: 'building_description' },
        { data: 'action', orderable: false }
    ]
});
```

#### Mermaid.js (Documentation)

**Version:** Latest  
**Purpose:** Diagram rendering in documentation

**Diagram Types Used:**
- Flowcharts (data flow, workflows)
- Entity-Relationship Diagrams (database schema)
- Sequence Diagrams (API interactions)

---

## 🗄️ Database

### MySQL

**Version:** 5.7+ or 8.0+  
**Database Name:** `meter_reading_robinsons`  
**Character Set:** utf8mb4  
**Collation:** utf8mb4_unicode_ci

**Storage Engines:**
- **InnoDB** - All tables (supports transactions, foreign keys)

**Key Features Used:**
- Foreign key constraints
- Indexes for performance
- Transactions (implicit via Laravel)
- Date/time functions (DATEDIFF, NOW)
- Aggregate functions (COUNT, SUM, AVG)

**Database Configuration:**
```php
// config/database.php
'mysql' => [
    'driver' => 'mysql',
    'host' => env('DB_HOST', 'localhost'),
    'port' => env('DB_PORT', '3306'),
    'database' => env('DB_DATABASE', 'meter_reading_robinsons'),
    'username' => env('DB_USERNAME', 'root'),
    'password' => env('DB_PASSWORD', ''),
    'charset' => 'utf8mb4',
    'collation' => 'utf8mb4_unicode_ci',
    'strict' => true,
    'engine' => 'InnoDB',
],
```

---

## 🌐 Web Server

### Apache

**Version:** 2.4+  
**Modules Required:**
- mod_rewrite (URL rewriting)
- mod_php (PHP processing)

**Configuration:**
```apache
<VirtualHost *:80>
    ServerName camr.yourdomain.com
    DocumentRoot /var/www/camr_robinsons/public
    
    <Directory /var/www/camr_robinsons/public>
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

### Nginx (Alternative)

**Version:** 1.18+  
**Advantages:**
- Better performance for static files
- Lower memory usage
- More efficient for high-traffic sites

**Configuration:**
```nginx
server {
    listen 80;
    server_name camr.yourdomain.com;
    root /var/www/camr_robinsons/public;
    index index.php;
    
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }
    
    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.0-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
```

---

## 📡 Field Devices

### Gateway/RTU Technology

**Hardware:** Industrial PCs / Embedded Linux devices  
**OS:** Linux (Debian/Ubuntu based)  
**Language:** Python 3.7+

**Python Libraries:**
```python
import requests        # HTTP communication with CAMR server
import pymodbus       # Modbus protocol implementation
import csv            # Load profile CSV generation
import schedule       # Periodic polling
import logging        # Error logging
```

**Gateway Responsibilities:**
1. Poll meters via Modbus (every 15 minutes)
2. Parse meter data according to config file
3. Generate CSV files (load profile format)
4. Upload to CAMR server via HTTP POST
5. Sync time from server
6. Update configuration when instructed

### Modbus Protocol

**Protocol:** Modbus RTU (serial) / Modbus TCP (ethernet)  
**Function Codes:**
- 0x03 - Read Holding Registers (most common)
- 0x04 - Read Input Registers

**Meter Communication:**
```python
from pymodbus.client.sync import ModbusTcpClient

client = ModbusTcpClient('192.168.1.100', port=502)
result = client.read_holding_registers(3000, 2, unit=1)  # Address 3000, 2 registers
energy = result.registers[0] * 1000  # Apply multiplier
```

---

## 📧 Email System

### SMTP Configuration

**Supported Providers:**
- Gmail (smtp.gmail.com:587)
- Microsoft 365 (smtp.office365.com:587)
- Custom SMTP servers

**Laravel Mail Configuration:**
```php
// config/mail.php
'mailers' => [
    'smtp' => [
        'transport' => 'smtp',
        'host' => env('MAIL_HOST', 'smtp.gmail.com'),
        'port' => env('MAIL_PORT', 587),
        'encryption' => env('MAIL_ENCRYPTION', 'tls'),
        'username' => env('MAIL_USERNAME'),
        'password' => env('MAIL_PASSWORD'),
    ],
],
```

**Mailable Class:**
```php
use Illuminate\Mail\Mailable;

class ResetPassword extends Mailable
{
    public function build()
    {
        return $this->subject('Password Reset')
                    ->view('emails.reset-password');
    }
}
```

---

## 🔒 Security Technologies

### Authentication

**Session Management:** Laravel sessions (database driver)  
**Password Hashing:** Bcrypt (Laravel Hash facade)  
**CSRF Protection:** Laravel VerifyCsrfToken middleware

```php
// Password hashing
$hashedPassword = Hash::make('password123');

// Password verification
if (Hash::check('password123', $hashedPassword)) {
    // Authenticated
}
```

### HTTPS/TLS

**Certificate:** Let's Encrypt (recommended)  
**Protocol:** TLS 1.2 / 1.3  
**Cipher Suites:** Modern, secure ciphers

---

## 🛠️ Development Tools

### Dependency Management

**Composer** (PHP packages)
```bash
composer install
composer update
composer require package/name
```

**NPM** (JavaScript packages)
```bash
npm install
npm run dev
npm run production
```

### Build Tools

**Laravel Mix** (Asset compilation)
```javascript
// webpack.mix.js
mix.js('resources/js/app.js', 'public/js')
   .sass('resources/sass/app.scss', 'public/css');
```

### Testing

**PHPUnit** (Unit and feature tests)
```bash
php artisan test
phpunit --filter testUserLogin
```

### Version Control

**Git**
```bash
git clone https://github.com/company/camr_robinsons.git
git add .
git commit -m "Add feature"
git push origin main
```

---

## 🚀 Production Stack

### Recommended Production Configuration

```yaml
Server:
  OS: Ubuntu 20.04 LTS / 22.04 LTS
  Web Server: Nginx 1.18+
  PHP: PHP 8.0 + PHP-FPM
  Database: MySQL 8.0
  Cache: Redis 6.0+
  Process Manager: Supervisor (for queues)
  SSL: Let's Encrypt (Certbot)
  Firewall: UFW
  Monitoring: Laravel Telescope (optional)
```

### Performance Stack

```yaml
Caching:
  Configuration: php artisan config:cache
  Routes: php artisan route:cache
  Views: Blade view caching
  Data: Redis for session/cache

Optimization:
  OpCache: Enabled (opcache.enable=1)
  Compression: Gzip enabled in Nginx
  CDN: For static assets (optional)
  Load Balancer: For multi-server (optional)
```

---

## 📊 Technology Comparison

### Why Laravel Over Other PHP Frameworks

| Feature | Laravel | CodeIgniter | Symfony |
|---------|---------|-------------|----------|
| Learning Curve | Moderate | Easy | Steep |
| ORM | Eloquent ✅ | QueryBuilder | Doctrine |
| Community | Large ✅ | Medium | Large |
| Documentation | Excellent ✅ | Good | Excellent |
| Package Ecosystem | Rich ✅ | Limited | Rich |
| Development Speed | Fast ✅ | Fast | Moderate |

### Why MySQL Over Other Databases

| Feature | MySQL | PostgreSQL | MongoDB |
|---------|-------|------------|----------|
| Maturity | High ✅ | High | Medium |
| Ease of Use | Easy ✅ | Moderate | Easy |
| Hosting Availability | Ubiquitous ✅ | Common | Common |
| Transactions | Yes ✅ | Yes | Limited |
| Data Structure | Relational ✅ | Relational | Document |
| Team Familiarity | High ✅ | Medium | Low |

---

## 🔗 Related Documentation

- **[Architecture](architecture.md)** - Application structure and design
- **[Data Flow](data-flow.md)** - How data moves through the system
- **[Installation](installation.md)** - Setup and deployment
- **[Environment Setup](configuration/environment.md)** - Server configuration
- **[Database Schema](database-schema.md)** - Database structure

---

**Last Updated:** 2024-03-15  
**Document Version:** 1.0  
**Maintainer:** CAMR Development Team
