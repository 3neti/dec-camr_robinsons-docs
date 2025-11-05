# Known Issues

This document tracks known issues, limitations, and quirks in the CAMR (Centralized Automated Meter Reading) Robinsons system. These are documented behaviors that may require workarounds or are scheduled for future resolution.

---

## 📌 Overview

**Purpose:** This page documents:

- Known bugs that have not been fixed
- System limitations by design
- Platform-specific issues
- Workarounds for common problems
- Issues scheduled for future releases

**Status Definitions:**

- 🔴 **Open** - Issue exists, no fix planned yet
- 🟡 **In Progress** - Fix is being developed
- 🟢 **Workaround Available** - Temporary solution exists
- 🔵 **By Design** - Intentional behavior, not a bug
- ⚪ **Deferred** - Fix postponed to future release

---

## 🗄️ Database & Data Issues

### Issue #1: Load Profile Table Growing Rapidly

**Status:** 🔵 By Design  
**Severity:** Medium  
**Component:** Database / Load Profile Storage

**Description:**

The `meter_load_profile_table` grows rapidly with high-frequency meter readings (15-minute intervals). With many meters, this table can reach millions of rows within months.

**Example Growth:**
- 100 meters × 96 readings/day × 30 days = ~288,000 rows/month
- 500 meters × 96 readings/day × 365 days = ~17.5 million rows/year

**Impact:**
- Slower query performance over time
- Increased backup times
- Higher storage requirements

**Workaround:**

```sql
-- Archive old data periodically (older than 12 months)
CREATE TABLE meter_load_profile_archive_2023 AS
SELECT * FROM meter_load_profile_table
WHERE timestamp < '2023-01-01';

-- Delete archived data from main table
DELETE FROM meter_load_profile_table
WHERE timestamp < '2023-01-01';

-- Optimize table
OPTIMIZE TABLE meter_load_profile_table;
```

**Best Practice:**
- Implement monthly archival process
- Keep only 12-18 months in main table
- Use partitioning for very large deployments

---

### Issue #2: No Automatic Data Retention Policy

**Status:** 🔴 Open  
**Severity:** Low  
**Component:** Database Management

**Description:**

The system does not automatically archive or delete old meter readings. All data is retained indefinitely, requiring manual intervention for data lifecycle management.

**Impact:**
- Database grows without bounds
- Manual intervention required
- No built-in compliance with data retention policies

**Workaround:**

Create custom Artisan command for periodic cleanup:

```php
// app/Console/Commands/ArchiveOldReadings.php
namespace App\Console\Commands;

use Illuminate\Console\Command;
use Illuminate\Support\Facades\DB;
use Carbon\Carbon;

class ArchiveOldReadings extends Command
{
    protected $signature = 'readings:archive {--months=12}';
    protected $description = 'Archive meter readings older than specified months';

    public function handle()
    {
        $months = $this->option('months');
        $cutoffDate = Carbon::now()->subMonths($months);
        
        $this->info("Archiving readings older than {$cutoffDate->toDateString()}...");
        
        // Archive to separate table
        DB::statement("
            INSERT INTO meter_load_profile_archive
            SELECT * FROM meter_load_profile_table
            WHERE timestamp < ?
        ", [$cutoffDate]);
        
        // Delete from main table
        $deleted = DB::table('meter_load_profile_table')
            ->where('timestamp', '<', $cutoffDate)
            ->delete();
        
        $this->info("Archived and deleted {$deleted} old readings.");
    }
}
```

```php
// Schedule in app/Console/Kernel.php
protected function schedule(Schedule $schedule)
{
    $schedule->command('readings:archive --months=12')->monthly();
}
```

---

### Issue #3: Orphaned Meters After Gateway Deletion

**Status:** 🟢 Workaround Available  
**Severity:** Medium  
**Component:** Gateway Management / Data Integrity

**Description:**

When a gateway is deleted, its associated meters become orphaned (foreign key set to NULL or deletion cascades). The system does not prompt to reassign meters before gateway deletion.

**Impact:**
- Meters lose gateway association
- Data collection stops for affected meters
- Manual reassignment required

**Workaround:**

**Before deleting gateway:**

```bash
# Check for associated meters
php artisan tinker
>>> $gateway = Gateway::find($gateway_id);
>>> $meterCount = $gateway->meters()->count();
>>> echo "This gateway has {$meterCount} meters";

# Reassign meters to another gateway
>>> $newGateway = Gateway::find($new_gateway_id);
>>> $gateway->meters()->update(['meter_gateway_id' => $newGateway->gateway_id]);
```

**Preventive measure:**

Add validation in controller:

```php
// Before deletion in CAMRGatewayController.php
public function delete_gateway_confirmed(Request $request)
{
    $gateway = Gateway::find($request->gateway_id);
    
    if ($gateway->meters()->count() > 0) {
        return response()->json([
            'error' => 'Cannot delete gateway with assigned meters. '
                      .'Please reassign meters first.'
        ], 400);
    }
    
    // Proceed with deletion
}
```

---

## 🔌 Gateway & Device Issues

### Issue #4: Gateway Offline Detection Delay

**Status:** 🔵 By Design  
**Severity:** Low  
**Component:** Gateway Monitoring

**Description:**

Gateways are marked as "offline" only when explicitly detected, not based on last-seen timestamp. A gateway that stops communicating may still appear "online" until the next monitoring cycle.

**Expected Behavior:**
- Gateway should be marked offline if no communication for >1 hour

**Actual Behavior:**
- Offline status only updated when gateway actively reports status
- Can lag by hours or days

**Impact:**
- Delayed notification of gateway failures
- False "online" status in dashboard

**Workaround:**

Implement automated offline detection:

```php
// Create scheduled task
// app/Console/Commands/DetectOfflineGateways.php
public function handle()
{
    $thresholdTime = Carbon::now()->subHour();
    
    Gateway::where('gateway_last_seen', '<', $thresholdTime)
           ->where('gateway_status', 'online')
           ->update(['gateway_status' => 'offline']);
    
    $this->info('Updated offline gateway statuses');
}
```

```php
// Schedule in app/Console/Kernel.php
protected function schedule(Schedule $schedule)
{
    $schedule->command('gateways:detect-offline')->hourly();
}
```

---

### Issue #5: CSV Update Flag Not Auto-Resetting

**Status:** 🟢 Workaround Available  
**Severity:** Low  
**Component:** Gateway Configuration Update

**Description:**

In rare cases, the `gateway_csv_update` flag may remain set to `1` even after successful CSV download by gateway, preventing future updates.

**Symptoms:**
- Gateway reports "CSV already downloaded"
- New configuration changes not pushed
- Manual flag reset required

**Cause:**
- Network interruption during download
- Gateway not calling reset endpoint
- Race condition in concurrent requests

**Workaround:**

**Manual reset:**
```bash
php artisan tinker
>>> Gateway::where('gateway_csv_update', 1)
        ->where('updated_at', '<', now()->subHours(24))
        ->update(['gateway_csv_update' => 0]);
```

**Automated cleanup:**
```php
// Reset stale CSV update flags (>24 hours old)
$schedule->call(function() {
    Gateway::where('gateway_csv_update', 1)
           ->where('updated_at', '<', now()->subDay())
           ->update(['gateway_csv_update' => 0]);
})->daily();
```

---

### Issue #6: Load Profile Upload Size Limit

**Status:** 🔵 By Design  
**Severity:** Medium  
**Component:** Load Profile API

**Description:**

Load profile file uploads are limited by PHP's `upload_max_filesize` (default 2MB) and `post_max_size` settings. Large files from gateways with many meters may be rejected.

**Typical File Sizes:**
- 10 meters, 1 day: ~500KB
- 50 meters, 1 day: ~2.5MB (exceeds default limit)
- 100 meters, 1 day: ~5MB

**Impact:**
- Gateway uploads fail with 413 or 500 error
- Data loss until gateway retries
- May require manual data recovery

**Solution:**

Increase PHP limits:

```ini
; php.ini
upload_max_filesize = 100M
post_max_size = 100M
max_execution_time = 300
memory_limit = 256M
```

```bash
# Restart web server after changes
sudo systemctl restart apache2  # or nginx/php-fpm
```

**Verification:**
```bash
php -i | grep -E "upload_max_filesize|post_max_size|max_execution_time"
```

---

## 📊 Report Generation Issues

### Issue #7: SAP Report Missing Partial Hour Data

**Status:** 🔵 By Design  
**Severity:** Low  
**Component:** SAP Report Generation

**Description:**

SAP reports aggregate data by full hours. Partial hour data at the beginning/end of the selected date range may be excluded to maintain hourly consistency.

**Example:**
- Query range: 2024-01-15 10:30 to 2024-01-15 14:45
- Report shows: 2024-01-15 11:00 to 2024-01-15 14:00
- Missing: 10:30-11:00 and 14:00-14:45 data

**Impact:**
- Slight data discrepancy vs RAW report
- End-of-day reports may miss last partial hour

**Workaround:**

Query full hour ranges:
- Start: 00:00:00
- End: 23:59:59

**Or use RAW report** for complete data.

---

### Issue #8: Excel Export Memory Exhaustion with Large Datasets

**Status:** 🟢 Workaround Available  
**Severity:** High  
**Component:** Report Export / Excel Generation

**Description:**

Generating Excel exports for very large datasets (>50,000 rows) can exhaust PHP memory, causing 500 errors or incomplete files.

**Affected Reports:**
- RAW Report (all meters, 30+ days)
- Consumption Report (many meters)
- Demand Report (hourly data)

**Symptoms:**
- "Allowed memory size of X bytes exhausted"
- Browser download fails
- Blank/corrupted Excel file

**Workarounds:**

**1. Increase PHP memory:**
```ini
; php.ini
memory_limit = 1024M  ; or higher
```

**2. Use chunked export:**
```php
// In report controller
return Excel::download(new LargeReportExport($filters), 'report.xlsx', \Maatwebsite\Excel\Excel::XLSX, [
    'memory' => 1024 * 1024 * 1024, // 1GB
]);
```

**3. Use CSV instead of Excel:**
- CSV format uses less memory
- Faster generation
- Can handle millions of rows

**4. Limit date range:**
- Query smaller time periods
- Generate multiple reports
- Combine manually if needed

---

### Issue #9: Report Generation Blocks Web Server

**Status:** ⚪ Deferred  
**Severity:** Medium  
**Component:** Report Generation / Performance

**Description:**

Large report generation happens synchronously, blocking the PHP worker until complete. This can cause:
- Slow response times for other users
- Timeout errors
- Server resource exhaustion

**Impact:**
- Poor user experience during report generation
- Concurrent users affected
- Server appears unresponsive

**Workaround:**

**Use queue system:**

```php
// Create queued job
php artisan make:job GenerateReport

// In job class
public function handle()
{
    $report = // generate report
    
    // Email to user when complete
    Mail::to($this->user->email)->send(new ReportReady($report));
}

// Dispatch job
GenerateReport::dispatch($user, $filters);
```

**Configure queue worker:**
```bash
# Install supervisor to manage queue worker
sudo apt-get install supervisor

# Configure in /etc/supervisor/conf.d/camr-queue.conf
[program:camr-queue-worker]
command=php /var/www/camr/artisan queue:work --sleep=3 --tries=3
autorestart=true
user=www-data
```

---

## 🔐 Security & Authentication Issues

### Issue #10: No Multi-Factor Authentication (MFA)

**Status:** 🔴 Open  
**Severity:** Medium  
**Component:** Authentication

**Description:**

The system only supports username/password authentication. No support for:
- Two-factor authentication (2FA/MFA)
- Single Sign-On (SSO)
- OAuth/SAML integration

**Impact:**
- Reduced security for sensitive accounts
- Non-compliance with some security standards
- Higher risk of unauthorized access

**Workaround:**

**Implement at infrastructure level:**

1. **VPN requirement:** Require VPN connection before accessing CAMR
2. **IP whitelisting:** Restrict access to known IP ranges
3. **Fail2ban:** Automatic IP blocking after failed logins

```bash
# Example fail2ban configuration
# /etc/fail2ban/jail.local
[camr-auth]
enabled = true
filter = camr-auth
logpath = /var/www/camr/storage/logs/laravel.log
maxretry = 5
bantime = 3600
```

**Future Enhancement:**
Implement Laravel 2FA packages like `pragmarx/google2fa-laravel`

---

### Issue #11: Session Timeout Too Short

**Status:** 🟢 Workaround Available  
**Severity:** Low  
**Component:** Session Management

**Description:**

Default session timeout of 120 minutes may be too short for users working on long reports or monitoring dashboards.

**Impact:**
- Users logged out during active work
- Unsaved work lost
- Frequent re-authentication required

**Solution:**

Adjust session lifetime:

```ini
# .env
SESSION_LIFETIME=480  # 8 hours
```

**Or implement "Remember Me" functionality** (already available on login form).

---

### Issue #12: Password Reset Email Contains Plain-Text Password

**Status:** 🔵 By Design  
**Severity:** Medium  
**Component:** Password Reset / Email Security

**Description:**

Password reset emails contain the temporary password in plain text. While necessary for the reset process, this violates some security best practices.

**Example Email:**
```
Your temporary password is: TempPass123
```

**Security Concerns:**
- Email can be intercepted
- Password visible in email logs
- Email client may cache password

**Best Practices:**

1. **Use one-time links instead:**
   - Send password reset link with token
   - User sets new password via form
   - Token expires after use

2. **Current system mitigation:**
   - Password is randomly generated
   - Forces password change on first login
   - Email uses TLS/SSL encryption

**Workaround:**

Instruct users to:
- Delete password reset email after use
- Change password immediately upon login
- Use secure email providers (Gmail, Outlook)

---

## 🐛 UI/UX Issues

### Issue #13: DataTables Not Persisting Page State

**Status:** ⚪ Deferred  
**Severity:** Low  
**Component:** UI / DataTables

**Description:**

When navigating away from a DataTable and returning, page number, sorting, and filters reset to defaults.

**Impact:**
- User must re-apply filters
- Re-navigate to desired page
- Poor user experience for large datasets

**Workaround:**

Enable DataTables state saving:

```javascript
// In JavaScript initialization
$('#meterTable').DataTable({
    stateSave: true,
    stateDuration: 60 * 60 * 24, // 24 hours
    // ... other options
});
```

---

### Issue #14: No Bulk Operations for Meters

**Status:** 🔴 Open  
**Severity:** Medium  
**Component:** Meter Management / UI

**Description:**

No ability to perform bulk operations on meters:
- Bulk delete
- Bulk gateway reassignment
- Bulk configuration file update
- Bulk enable/disable

**Impact:**
- Time-consuming for large deployments
- Prone to manual errors
- Requires individual updates

**Workaround:**

Use database queries for bulk operations:

```bash
# Bulk update via tinker
php artisan tinker

# Reassign meters to new gateway
>>> Meter::where('meter_gateway_id', $old_gateway_id)
       ->update(['meter_gateway_id' => $new_gateway_id]);

# Update configuration file
>>> Meter::where('meter_site_id', $site_id)
       ->update(['meter_configuration_file_id' => $config_id]);

# Bulk delete (CAUTION)
>>> Meter::where('meter_status', 'inactive')
       ->where('updated_at', '<', now()->subYears(2))
       ->delete();
```

---

### Issue #15: Mobile Responsiveness Issues

**Status:** 🔴 Open  
**Severity:** Low  
**Component:** UI / Responsive Design

**Description:**

While the application uses Bootstrap and is partially responsive, some pages have issues on mobile devices:
- DataTables overflow on small screens
- Modal dialogs too large for mobile
- Forms require horizontal scrolling
- Navigation sidebar overlaps content

**Impact:**
- Poor mobile user experience
- Difficult to use on tablets
- Requires desktop for full functionality

**Workaround:**

Use desktop/laptop for:
- Report generation
- Bulk data entry
- Complex configurations

Mobile suitable for:
- Dashboard viewing
- Quick meter lookups
- Status monitoring

**Future Enhancement:**
Implement mobile-first redesign with responsive DataTables configuration.

---

## 🛠️ System Limitations

### Limitation #1: Single Database Server Only

**Status:** 🔵 By Design  
**Component:** Architecture

**Description:**

No built-in support for database replication, clustering, or read replicas. System designed for single database server.

**Implications:**
- Single point of failure
- Limited horizontal scaling
- No automatic failover

**Mitigation:**
- Implement MySQL replication manually
- Use database backups
- Consider database clustering at infrastructure level

---

### Limitation #2: No Real-Time Dashboard Updates

**Status:** 🔵 By Design  
**Component:** Dashboard / WebSocket

**Description:**

Dashboards require manual refresh to see latest data. No WebSocket or real-time push updates.

**Implications:**
- Users must refresh page
- Delayed awareness of issues
- No real-time alerts

**Workaround:**

Implement auto-refresh:

```javascript
// In dashboard view
setInterval(function() {
    location.reload();
}, 300000); // Refresh every 5 minutes
```

---

### Limitation #3: Maximum 15-Minute Reading Interval

**Status:** 🔵 By Design  
**Component:** Data Collection

**Description:**

System designed for 15-minute interval readings. Higher frequency (1-minute, 5-minute) not officially supported.

**Reason:**
- Gateway hardware limitations
- Database performance considerations
- Storage requirements

**Impact:**
- Cannot capture sub-15-minute events
- Limited granularity for power quality analysis
- May miss short-duration anomalies

---

## 📝 Reporting Issues

When reporting a new issue, please include:

1. **Issue Description:** What is the problem?
2. **Steps to Reproduce:** How to trigger the issue?
3. **Expected Behavior:** What should happen?
4. **Actual Behavior:** What actually happens?
5. **Environment:**
   - Laravel version
   - PHP version
   - MySQL version
   - Browser (for UI issues)
6. **Screenshots/Logs:** Visual evidence or error messages
7. **Workaround:** Any temporary solution found?

---

## 🔗 Related Documentation

- [Troubleshooting Guide](troubleshooting.md) - Common problems and solutions
- [Installation Guide](../installation.md) - Setup and configuration
- [Architecture](../architecture.md) - System design and components
- [Database Schema](../database-schema.md) - Database structure

---

## 📝 Summary

This document tracks known issues and limitations in CAMR Robinsons. Issues are categorized by severity and status, with workarounds provided where available.

**For urgent production issues**, refer to the [Troubleshooting Guide](troubleshooting.md) first.

**To contribute:** If you discover new issues or workarounds, please document them here to help other users and developers.
