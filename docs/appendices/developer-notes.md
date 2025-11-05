# Developer Notes

This document provides essential information for developers maintaining, extending, or troubleshooting the CAMR (Centralized Automated Meter Reading) Robinsons system. It includes architectural decisions, code patterns, gotchas, and best practices learned during development.

---

## 🎯 Purpose

This document serves as:

- **Onboarding guide** for new developers
- **Reference** for architectural decisions
- **Troubleshooting aid** with common pitfalls
- **Best practices guide** for code contributions
- **Historical context** for design choices

---

## 🏗️ Architecture Overview

### MVC Pattern

CAMR follows Laravel's MVC (Model-View-Controller) architecture:

```
app/
├── Models/              # Eloquent models (data layer)
├── Http/
│   ├── Controllers/    # Request handlers (business logic)
│   └── Middleware/     # Request filters (auth, validation)
└── Console/
    └── Commands/       # CLI tasks

resources/
└── views/              # Blade templates (presentation layer)

routes/
└── web.php             # Route definitions
```

### Key Design Patterns

**1. Repository Pattern (Not Fully Implemented)**

Direct Eloquent usage in controllers rather than repositories:

```php
// Current approach
public function index() {
    $sites = Site::all();
    return view('sites.index', compact('sites'));
}

// Repository pattern (not used)
// $sites = $this->siteRepository->all();
```

**Rationale:** For this application size, direct Eloquent is more maintainable. Consider repositories if app grows significantly.

**2. Service Layer (Partial Implementation)**

Complex business logic in controllers. Consider extracting to services for:
- Report generation
- CSV processing
- Load profile parsing

**3. Activity Logging (Implemented via Trait)**

Using Spatie's `LogsActivity` trait on models:

```php
use LogsActivity;

protected static $logName = 'Site Management';
protected static $logOnlyDirty = true;
```

---

## 📚 Code Patterns & Conventions

### Naming Conventions

**Controllers:**
```php
// Pattern: {Entity}{Action}Controller
CAMRSiteController
CAMRGatewayController
SAPReportController
```

**Models:**
```php
// Direct entity names
Site
Gateway  
Meter

// Or descriptive models
WebPageSettingsModel
ConfigurationFileModel
```

**Database Tables:**
```php
// Inconsistent naming (legacy):
'meter_site'                    // No 'table' suffix
'meter_gateway_table'           // Has 'table' suffix
'user_table'                    // Has 'table' suffix

// New tables should follow: {entity}_table pattern
```

**Routes:**
```php
// Pattern: descriptive names with underscores
Route::get('/site_details/{siteID}', [CAMRSiteController::class,'site_details_2']);
Route::post('/create_site_post', [CAMRSiteController::class,'create_site_post']);
```

### DataTables Pattern

Consistent pattern for server-side DataTables:

```php
public function getSiteForAdmin(Request $request)
{
    if ($request->ajax()) {
        $data = Site::select('*');
        return Datatables::of($data)
            ->addIndexColumn()
            ->addColumn('action', function($row){
                return '<button>Edit</button>';
            })
            ->rawColumns(['action'])
            ->make(true);
    }
}
```

**Key Points:**
- Always check `$request->ajax()`
- Use `addIndexColumn()` for row numbers
- Use `rawColumns()` for HTML in cells
- Return `->make(true)` for proper formatting

### Form Validation Pattern

```php
$request->validate([
    'site_code' => 'required|unique:meter_site,site_code',
    'site_name' => 'required|max:255',
    'site_company_id' => 'required|exists:meter_company_table,company_id'
], [
    'site_code.required' => 'Site Code is required',
    'site_code.unique' => 'Site Code already exists'
]);
```

**Best Practices:**
- Always provide custom error messages
- Validate foreign key existence with `exists:table,column`
- Use `unique` with table and column parameters

---

## ⚡ Performance Considerations

### N+1 Query Problem

**Problem:**
```php
// BAD: Generates N+1 queries
$gateways = Gateway::all();
foreach ($gateways as $gateway) {
    echo $gateway->site->site_name; // Query per gateway
}
```

**Solution:**
```php
// GOOD: Single query with join
$gateways = Gateway::with('site')->get();
foreach ($gateways as $gateway) {
    echo $gateway->site->site_name;
}
```

**Where to Watch:**
- Report generation with nested relationships
- DataTables with related data
- Dashboard widgets showing counts

### Large Dataset Handling

**Use Chunking:**
```php
// For processing large datasets
Meter::chunk(100, function($meters) {
    foreach ($meters as $meter) {
        // Process meter
    }
});

// Or use cursor for memory efficiency
foreach (Meter::cursor() as $meter) {
    // Process meter
}
```

**Excel Exports:**
```php
// Use queue for large exports
use Illuminate\Contracts\Queue\ShouldQueue;

class ReportExport implements FromQuery, ShouldQueue
{
    use Queueable;
    // ...
}
```

### Database Indexes

**Critical Indexes to Verify:**
```sql
-- Frequently queried foreign keys
CREATE INDEX idx_meter_gateway ON meter_table(meter_gateway_id);
CREATE INDEX idx_gateway_site ON meter_gateway_table(gateway_site_id);

-- Report filtering
CREATE INDEX idx_load_profile_timestamp ON meter_load_profile_table(timestamp);
CREATE INDEX idx_load_profile_meter ON meter_load_profile_table(meter_id);

-- User access control
CREATE INDEX idx_user_site_access ON user_site_access_table(user_id, site_id);
```

---

## 🐛 Common Gotchas & Pitfalls

### 1. Gateway API Routes

**Issue:** Legacy URL structure for gateway compatibility

```php
// These routes look unusual but are required for gateway devices:
Route::get('/check_time.php', [CAMRGatewayDeviceController::class,'check_time']);
Route::any('/lp/receive_file.php', [LoadProfileController::class,'LoadProfile']);
```

**⚠️ Don't Refactor:** Gateway firmware expects these exact URLs. Changing them breaks gateway communication.

### 2. WebPageSettingsModel Has No Primary Key

**Issue:** Unique model design for singleton pattern

```php
class WebPageSettingsModel extends Model
{
    protected $primaryKey = null; // No primary key!
    
    // Always use first() instead of find()
    public static function getSettings() {
        return self::first(); // Not find(1)
    }
}
```

**Why:** Single configuration row for entire application. No need for multiple records.

### 3. Session Authentication Implementation

**Issue:** Custom auth instead of Laravel's built-in Auth

```php
// Custom middleware checks session
if (!Session::has('loginID')) {
    return redirect('/')->with('fail', 'Please login first');
}

// NOT using:
// Auth::check()
// Auth::user()
```

**Impact:**
- Can't use `Auth` facade
- Must use `Session::get('loginID')` manually
- Activity logging requires custom causer ID setting

**Rationale:** Legacy decision. Refactoring to Laravel Auth would require extensive changes.

### 4. Database Table Naming Inconsistency

**Issue:** Mixed naming conventions

```php
'meter_site'              // No suffix
'meter_gateway_table'     // Has '_table' suffix
'user_table'              // Has '_table' suffix
```

**Impact:** Must remember exact table names. No consistent pattern.

**Best Practice:** When adding new tables, follow existing pattern of closest related table.

### 5. Load Profile File Processing

**Issue:** Synchronous processing can timeout

```php
// Current: Processes immediately in request
public function LoadProfile(Request $request) {
    $file = $request->file('load_profile');
    // Parse and insert thousands of readings
    // May timeout on large files
}
```

**Solution:** Consider queue-based processing for large files:

```php
ProcessLoadProfile::dispatch($filePath);
return response()->json(['status' => 'Processing']);
```

### 6. Excel Export Memory Issues

**Issue:** Large reports exhaust memory

**Solutions:**
```php
// 1. Increase memory limit temporarily
ini_set('memory_limit', '512M');

// 2. Use chunking in export class
public function query()
{
    return Meter::query()->chunk(1000);
}

// 3. Use queue
class ReportExport implements ShouldQueue { }
```

---

## 🔧 Development Workflow

### Local Development Setup

```bash
# 1. Clone repository
git clone <repository-url>
cd camr_robinsons

# 2. Install dependencies
composer install
npm install

# 3. Configure environment
cp .env.example .env
php artisan key:generate

# 4. Setup database
mysql -u root -p -e "CREATE DATABASE meter_reading_robinsons"
php artisan migrate

# 5. Seed admin user (if seeder exists)
php artisan db:seed

# 6. Compile assets
npm run dev

# 7. Start development server
php artisan serve
```

### Making Changes

**1. New Feature Development:**

```bash
# Create feature branch
git checkout -b feature/new-report-type

# Make changes
# - Add route to routes/web.php
# - Create controller
# - Create model if needed
# - Add migration if database changes
# - Create Blade view

# Test locally
php artisan migrate:fresh --seed
npm run dev

# Commit with descriptive message
git add .
git commit -m "Add consumption trend report

- Add ConsumptionTrendController
- Add route /reports/consumption-trend
- Add Blade view with chart.js
- Add migration for trend_cache table"
```

**2. Bug Fixes:**

```bash
git checkout -b bugfix/gateway-timeout

# Fix issue
# Add test case if possible

git commit -m "Fix gateway API timeout on large CSV

- Increase max_execution_time for CSV routes
- Add chunking for meter list generation  
- Fixes #123"
```

### Database Changes

**Creating Migrations:**

```bash
# New table
php artisan make:migration create_meter_alerts_table

# Modify existing table
php artisan make:migration add_threshold_to_meters_table
```

**Migration Best Practices:**

```php
public function up()
{
    Schema::create('meter_alerts_table', function (Blueprint $table) {
        $table->id();
        $table->unsignedBigInteger('meter_id');
        $table->string('alert_type');
        $table->timestamp('alert_time');
        $table->timestamps();
        
        // Always add foreign keys
        $table->foreign('meter_id')
              ->references('meter_id')
              ->on('meter_table')
              ->onDelete('cascade');
    });
}

public function down()
{
    // Always implement rollback
    Schema::dropIfExists('meter_alerts_table');
}
```

**⚠️ Important:**
- Never delete migrations that have run in production
- Always test both `up()` and `down()` methods
- Add foreign key constraints for data integrity

---

## 📝 Code Documentation Standards

### Controller Methods

```php
/**
 * Display list of sites for admin users
 *
 * @param Request $request
 * @return \Illuminate\Http\JsonResponse
 */
public function getSiteForAdmin(Request $request)
{
    if ($request->ajax()) {
        // Implementation
    }
}
```

### Model Relationships

```php
/**
 * Get the meters belonging to this gateway
 *
 * @return \Illuminate\Database\Eloquent\Relations\HasMany
 */
public function meters()
{
    return $this->hasMany(Meter::class, 'meter_gateway_id', 'gateway_id');
}
```

### Complex Business Logic

```php
// When logic is non-obvious, add explanatory comments
public function calculateDemand($readings)
{
    // Demand is calculated as maximum kW over 15-minute interval
    // Multiply by CT ratio (stored as multiplier)
    // SAP requires hourly aggregation (4 readings per hour)
    
    $hourlyDemand = [];
    // ...
}
```

---

## 🧪 Testing Strategies

### Manual Testing Checklist

**Before Each Release:**

- [ ] Login/Logout flow
- [ ] Password reset email
- [ ] Create site, gateway, meter
- [ ] Upload CSV configuration
- [ ] Generate each report type
- [ ] Test site access control (admin vs user)
- [ ] Check activity logs
- [ ] Test on mobile device
- [ ] Verify branding changes apply

### Gateway Simulation

Test gateway API endpoints:

```bash
# Check time
curl http://localhost:8000/check_time.php

# CSV update check
curl http://localhost:8000/rtu/index.php/rtu/rtu_check_update/AA:BB:CC:DD:EE:FF/get_update_csv

# Simulate load profile upload
curl -X POST -F "load_profile=@test_data.csv" \
  http://localhost:8000/lp/receive_file.php
```

### Database Testing

```php
// Use tinker for quick tests
php artisan tinker

// Test relationships
>>> $site = Site::first();
>>> $site->gateways()->count();
>>> $site->gateways->first()->meters()->count();

// Test queries
>>> Meter::with('gateway.site')->limit(10)->get();

// Test activity logging
>>> Activity::where('subject_type', 'Site')->latest()->first();
```

---

## 🚀 Deployment Guide

### Pre-Deployment Checklist

```bash
# 1. Run tests (if implemented)
php artisan test

# 2. Check for syntax errors
find app -name "*.php" -exec php -l {} \;

# 3. Clear and recache
php artisan config:clear
php artisan cache:clear
php artisan route:clear
php artisan view:clear

# 4. Optimize for production
php artisan config:cache
php artisan route:cache
php artisan view:cache

# 5. Compile assets for production
npm run prod

# 6. Run migrations (on staging first)
php artisan migrate --force
```

### Deployment Steps

```bash
# 1. Pull latest code
git pull origin main

# 2. Install dependencies (production only)
composer install --no-dev --optimize-autoloader

# 3. Run migrations
php artisan migrate --force

# 4. Clear old caches
php artisan optimize:clear

# 5. Cache config/routes
php artisan config:cache
php artisan route:cache

# 6. Compile assets
npm install --production
npm run prod

# 7. Set permissions
chmod -R 775 storage bootstrap/cache
chown -R www-data:www-data storage bootstrap/cache

# 8. Restart services
sudo systemctl restart php8.0-fpm
sudo systemctl restart nginx

# 9. Verify application
curl -I https://your-domain.com
```

### Zero-Downtime Deployment

For production with users:

```bash
# 1. Put application in maintenance mode
php artisan down --message="Upgrading system" --retry=60

# 2. Deploy (steps above)

# 3. Bring application back up
php artisan up
```

---

## 🛡️ Security Best Practices

### Input Validation

**Always validate user input:**

```php
// Whitelist allowed values
$request->validate([
    'status' => 'required|in:active,inactive',
    'email' => 'required|email|max:255',
    'site_id' => 'required|exists:meter_site,site_id'
]);
```

### SQL Injection Prevention

**Use Eloquent or query builder:**

```php
// GOOD: Parameterized
Site::where('site_code', $code)->first();
DB::table('meter_site')->where('site_id', $id)->update([...]);

// BAD: String concatenation
DB::select("SELECT * FROM meter_site WHERE site_code = '$code'");
```

### XSS Prevention

**Blade auto-escapes:**

```blade
{{-- SAFE: Auto-escaped --}}
{{ $site->site_name }}

{{-- UNSAFE: Raw output --}}
{!! $html_content !!}  {{-- Only if you trust the source --}}
```

### File Upload Security

```php
$request->validate([
    'csv_file' => 'required|file|mimes:csv,txt|max:10240', // 10MB
    'logo' => 'required|image|mimes:jpg,png|max:2048' // 2MB
]);

// Store with generated filename
$path = $request->file('csv_file')->store('uploads', 'local');
```

### Environment Variables

**Never commit `.env` file:**

```bash
# Check .gitignore includes
.env
.env.backup
.env.production
```

**Rotate secrets regularly:**
- APP_KEY
- Database passwords  
- API keys
- SMTP credentials

---

## 📊 Performance Monitoring

### Useful Debug Queries

```sql
-- Slow query log
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 2; -- Log queries > 2 seconds

-- Check table sizes
SELECT 
    table_name,
    ROUND(((data_length + index_length) / 1024 / 1024), 2) AS 'Size (MB)'
FROM information_schema.TABLES
WHERE table_schema = 'meter_reading_robinsons'
ORDER BY (data_length + index_length) DESC;

-- Find missing indexes
SELECT * FROM sys.schema_unused_indexes;
```

### Laravel Debugbar

Install for development:

```bash
composer require barryvdh/laravel-debugbar --dev
```

Shows:
- Queries executed (N+1 detection)
- Memory usage
- Execution time
- View rendering time

---

## 📚 Recommended Reading

### Laravel Resources

- [Laravel 8 Documentation](https://laravel.com/docs/8.x)
- [Laravel Best Practices](https://github.com/alexeymezenin/laravel-best-practices)
- [Eloquent Performance Patterns](https://eloquent-course.reinink.ca/)

### PHP

- [PHP The Right Way](https://phptherightway.com/)
- [PHP Standards Recommendations (PSR)](https://www.php-fig.org/psr/)

### JavaScript/Frontend

- [DataTables Documentation](https://datatables.net/manual/)
- [Bootstrap 5 Documentation](https://getbootstrap.com/docs/5.0/)
- [Chart.js Documentation](https://www.chartjs.org/docs/)

---

## 🤝 Contributing Guidelines

### Code Style

- Follow PSR-12 coding standard
- Use Laravel conventions (camelCase for methods, snake_case for variables)
- Keep methods focused (single responsibility)
- Limit controller methods to 50-75 lines

### Commit Messages

```
Add consumption trend analysis feature

- Create ConsumptionTrendController  
- Add trend calculation algorithm
- Implement Chart.js visualization
- Add unit tests for trend calculation

Fixes #142
```

**Format:**
- First line: Brief summary (50 chars max)
- Blank line
- Detailed description with bullet points
- Reference issue numbers

### Pull Request Process

1. Create feature branch from `main`
2. Make changes with tests
3. Update documentation
4. Submit PR with description
5. Address review comments
6. Squash commits if requested

---

## 🔗 Related Documentation

- [Architecture](../architecture.md) - System design overview
- [Database Schema](../database-schema.md) - Complete table reference
- [Troubleshooting](../maintenance/troubleshooting.md) - Common issues
- [Known Issues](../maintenance/known-issues.md) - Current limitations
- [Glossary](glossary.md) - Technical terminology

---

## 📝 Final Notes

### When in Doubt

1. **Check existing code** - Look for similar functionality
2. **Follow conventions** - Match existing patterns
3. **Ask questions** - Better to clarify than assume
4. **Document decisions** - Help future developers
5. **Test thoroughly** - Prevent regression

### Contact & Support

For questions or issues:
- Review this documentation first
- Check [Known Issues](../maintenance/known-issues.md)
- Search Laravel documentation  
- Consult [Troubleshooting Guide](../maintenance/troubleshooting.md)

---

**This is a living document.** As you discover new patterns, gotchas, or best practices, please update this file to help future developers.
