# Eloquent Models

## 📋 Overview

The CAMR system uses Laravel's Eloquent ORM for database interactions. All models extend `Illuminate\Database\Eloquent\Model` and implement activity logging using the Spatie Laravel ActivityLog package.

## 🏛️ Model Architecture

All models (except `User`) share common characteristics:
- **Activity Logging:** Uses `LogsActivity` trait from Spatie
- **Audit Trail:** Tracks create, update, and delete operations
- **Session Integration:** Logs the user ID from session for audit purposes
- **Mass Assignment:** Protected via `$fillable` arrays
- **Timestamps:** Laravel's automatic `created_at` and `updated_at`

### Common Pattern

```php
use Spatie\Activitylog\Traits\LogsActivity;
use Spatie\Activitylog\Contracts\Activity;

class ExampleModel extends Model
{
    use LogsActivity;
    
    // Set causer_id from session for activity log
    public function tapActivity(Activity $activity, string $eventName)
    {
        $activity->causer_id = Session::get('loginID');
    }
    
    protected $table = 'table_name';
    protected $primaryKey = 'id_column';
    protected $fillable = [/* mass assignable fields */];
    
    // Activity logging configuration
    protected static $logName = 'Model Name';
    protected static $logOnlyDirty = true;
    protected static $logAttributes = [/* logged fields */];
}
```

## 🏢 Core Infrastructure Models

### SiteModel

Represents physical locations (malls, properties) where metering equipment is deployed.

**File:** `app/Models/SiteModel.php`  
**Table:** `meter_site`  
**Primary Key:** `site_id`

**Fillable Attributes:**
- `division_idx` - Foreign key to division
- `company_idx` - Foreign key to company
- `building_idx` - Foreign key to building (nullable)
- `site_code` - Building code/identifier
- `created_by_user_idx` - Creator user ID
- `modified_by_user_idx` - Last modifier user ID

**Activity Log Name:** "Site Details"

**Relationships:**
```php
// Suggested relationships (not explicitly defined in code)
// belongsTo(CompanyModel::class, 'company_idx', 'company_id')
// belongsTo(DivisionModel::class, 'division_idx', 'division_id')
// hasMany(GatewayModel::class, 'site_idx', 'site_id')
// hasMany(MeterModel::class, 'site_idx', 'site_id')
// hasMany(BuildingModel::class, 'site_idx', 'site_id')
```

### GatewayModel

Represents Remote Terminal Units (RTUs) that collect meter data.

**File:** `app/Models/GatewayModel.php`  
**Table:** `meter_rtu`  
**Primary Key:** `rtu_id`

**Fillable Attributes:**
- `gateway_sn` - Gateway serial number
- `gateway_mac` - MAC address (used in device API)
- `gateway_ip` - IP address
- `rtu_physical_location` - Physical location description
- `update_rtu` - CSV update flag (0/1)
- `update_rtu_location` - Site code update flag (0/1)
- `update_rtu_ssh` - SSH enabled flag (0/1)
- `update_rtu_force_lp` - Force load profile flag (0/1)
- `idf_number` - IDF identifier
- `switch_name` - Network switch name
- `idf_port` - Switch port number
- `last_log_update` - Last communication timestamp
- `soft_rev` - Software/firmware revision
- `created_by_user_idx` - Creator user ID
- `modified_by_user_idx` - Last modifier user ID

**Activity Log Name:** "Gateway Details"

**Key Flags:**
- `update_rtu` = 1: Gateway will download CSV configuration
- `update_rtu_location` = 1: Gateway will download site code
- `update_rtu_ssh` = 1: SSH access enabled
- `update_rtu_force_lp` = 1: Force load profile collection

**Relationships:**
```php
// belongsTo(SiteModel::class, 'site_idx', 'site_id')
// belongsTo(MeterLocationModel::class, 'location_idx', 'location_id')
// hasMany(MeterModel::class, 'rtu_idx', 'rtu_id')
```

### MeterModel

Represents electricity meters connected to gateways.

**File:** `app/Models/MeterModel.php`  
**Table:** `meter_details`  
**Primary Key:** `meter_id`

**Fillable Attributes:**
- `site_idx` - Foreign key to site
- `rtu_idx` - Foreign key to gateway
- `location_idx` - Foreign key to meter location
- `config_idx` - Foreign key to configuration file
- `site_code` - Site code
- `meter_name` - Meter identifier/name
- `meter_name_addressable` - Is addressable (1/0)
- `meter_default_name` - Default meter name
- `meter_type` - Meter type
- `meter_brand` - Manufacturer
- `meter_role` - Role (e.g., "Client Meter")
- `meter_remarks` - Additional notes
- `customer_name` - Customer/tenant name
- `meter_multiplier` - Reading multiplier (default: 1)
- `meter_status` - Current status
- `created_by_user_idx` - Creator user ID
- `modified_by_user_idx` - Last modifier user ID

**Activity Log Name:** "Meter Details"

**Relationships:**
```php
// belongsTo(SiteModel::class, 'site_idx', 'site_id')
// belongsTo(GatewayModel::class, 'rtu_idx', 'rtu_id')
// belongsTo(BuildingModel::class, 'building_idx', 'building_id')
// belongsTo(MeterLocationModel::class, 'location_idx', 'location_id')
// belongsTo(ConfigurationFileModel::class, 'config_idx', 'config_id')
```

### BuildingModel

Represents buildings within sites.

**File:** `app/Models/BuildingModel.php`  
**Table:** `meter_building_table`  
**Primary Key:** `building_id`

**Fillable Attributes:**
- `site_idx` - Foreign key to site
- `building_code` - Building code
- `building_description` - Building name/description
- `cut_off` - Billing cut-off day
- `device_ip_range` - IP address range
- `ip_network` - Network address
- `ip_netmask` - Network mask
- `ip_gateway` - Gateway IP
- `created_by_user_idx` - Creator user ID
- `modified_by_user_idx` - Last modifier user ID

**Activity Log Name:** "Building Details"

**Relationships:**
```php
// belongsTo(SiteModel::class, 'site_idx', 'site_id')
// hasMany(MeterLocationModel::class, 'building_id', 'building_id')
// hasMany(MeterModel::class, 'building_idx', 'building_id')
```

### MeterLocationModel

Represents meter locations (EE rooms) within buildings.

**File:** `app/Models/MeterLocationModel.php`  
**Table:** `meter_location_table`  
**Primary Key:** `location_id`

**Fillable Attributes:**
- `site_idx` - Foreign key to site
- `building_id` - Foreign key to building
- `location_code` - Location code (e.g., "EER-01")
- `location_description` - Location description
- `created_by_user_idx` - Creator user ID
- `modified_by_user_idx` - Last modifier user ID

**Activity Log Name:** "Location Details"

**Relationships:**
```php
// belongsTo(SiteModel::class, 'site_idx', 'site_id')
// belongsTo(BuildingModel::class, 'building_id', 'building_id')
// hasMany(GatewayModel::class, 'location_idx', 'location_id')
// hasMany(MeterModel::class, 'location_idx', 'location_id')
```

## 🏛️ Organization Models

### CompanyModel

Represents companies/organizations.

**File:** `app/Models/CompanyModel.php`  
**Table:** `meter_company_table`  
**Primary Key:** `company_id`

**Fillable Attributes:**
- `company_code` - Company code
- `company_name` - Company name
- `created_by_user_idx` - Creator user ID
- `modified_by_user_idx` - Last modifier user ID

**Activity Log Name:** "Company Details"

**Relationships:**
```php
// hasMany(SiteModel::class, 'company_idx', 'company_id')
```

### DivisionModel

Represents divisions within companies.

**File:** `app/Models/DivisionModel.php`  
**Table:** `meter_division_table`  
**Primary Key:** `division_id`

**Fillable Attributes:**
- `division_code` - Division code
- `division_name` - Division name
- `created_by_user_idx` - Creator user ID
- `modified_by_user_idx` - Last modifier user ID

**Activity Log Name:** "Division Details"

**Relationships:**
```php
// hasMany(SiteModel::class, 'division_idx', 'division_id')
```

## 👤 User Models

### User

Authentication model extending Laravel's `Authenticatable`.

**File:** `app/Models/User.php`  
**Table:** `user_tb`  
**Primary Key:** `user_id` (auto-increment)

**Fillable Attributes:**
- `user_real_name` - Full name
- `user_name` - Username (login)
- `user_password` - Hashed password

**Hidden Attributes:**
- `user_password` - Never exposed in JSON
- `remember_token` - Session token

**Key Features:**
- Extends Laravel's `Authenticatable` for auth integration
- Uses `Notifiable` trait for notifications
- Password automatically hashed using bcrypt

**Additional Fields (not in fillable):**
- `user_type` - User type ("Admin", "User")
- `user_access` - Access level ("All", "Selected")
- `user_email_address` - Email for password reset
- `user_site_list_ids` - Comma-separated site IDs (if "Selected")
- `user_expiration` - Account expiration date

### UserAccountModel

**File:** `app/Models/UserAccountModel.php`  
**Table:** `user_tb`  
**Purpose:** Additional user operations (separate from auth)

### UserSiteAccessModel

**File:** `app/Models/UserSiteAccessModel.php`  
**Table:** `user_access_group`  
**Purpose:** User-site access mapping (many-to-many)

## ⚙️ Configuration Models

### ConfigurationFileModel

Meter configuration files for different meter models.

**File:** `app/Models/ConfigurationFileModel.php`  
**Table:** `meter_configuration_file`  
**Primary Key:** `config_id`

**Purpose:** Stores meter model configurations used by gateways

### WebPageSettingsModel

Web interface customization settings.

**File:** `app/Models/WebPageSettingsModel.php`  
**Table:** `web_page_settings`  
**Primary Key:** `setting_id`

**Purpose:** Logo, header title, navigation customization

## 📊 Data Models

### MeterDataModel

Time-series meter reading data.

**File:** `app/Models/MeterDataModel.php`  
**Table:** `meter_data`  
**Primary Key:** `id` (BIGINT)

**Purpose:** Stores electrical measurements (voltage, current, power, energy)

**Note:** This table grows continuously. Consider archival strategy.

## 🔗 Relationships Summary

### Hierarchy
```
Company (1) → (N) Site
Division (1) → (N) Site
Site (1) → (N) Building
Site (1) → (N) MeterLocation
Site (1) → (N) Gateway
Site (1) → (N) Meter
Building (1) → (N) MeterLocation
Gateway (1) → (N) Meter
MeterLocation (1) → (N) Gateway
MeterLocation (1) → (N) Meter
ConfigurationFile (1) → (N) Meter
```

### Defining Relationships in Models

While relationships are not explicitly defined in the current models, they can be added:

```php path=null start=null
// Example: In SiteModel.php
public function company()
{
    return $this->belongsTo(CompanyModel::class, 'company_idx', 'company_id');
}

public function division()
{
    return $this->belongsTo(DivisionModel::class, 'division_idx', 'division_id');
}

public function gateways()
{
    return $this->hasMany(GatewayModel::class, 'site_idx', 'site_id');
}

public function meters()
{
    return $this->hasMany(MeterModel::class, 'site_idx', 'site_id');
}
```

## 📋 Activity Logging

All models use **Spatie Laravel ActivityLog** for audit trails.

### Configuration

```php path=null start=null
protected static $logName = 'Model Name';  // Log category
protected static $logOnlyDirty = true;     // Log only changed attributes
protected static $logAttributes = [...];    // Which fields to log
```

### Custom Causer

All models override `tapActivity()` to set the user from session:

```php path=null start=null
public function tapActivity(Activity $activity, string $eventName)
{
    $activity->causer_id = Session::get('loginID');
}
```

### Viewing Activity Logs

Activity logs are stored in the `activity_log` table:

```sql
SELECT * FROM activity_log 
WHERE subject_type = 'App\\Models\\SiteModel' 
  AND subject_id = 1 
ORDER BY created_at DESC;
```

## 🛠️ Usage Examples

### Creating a Site

```php path=null start=null
$site = SiteModel::create([
    'division_idx' => 1,
    'company_idx' => 1,
    'site_code' => 'MALL-01',
    'created_by_user_idx' => Session::get('loginID')
]);
// Automatically logged to activity_log
```

### Updating a Gateway

```php path=null start=null
$gateway = GatewayModel::find($id);
$gateway->update_rtu = 1;  // Enable CSV update
$gateway->save();
// Only 'update_rtu' change is logged (logOnlyDirty = true)
```

### Querying Meters

```php path=null start=null
// Get all meters for a site
$meters = MeterModel::where('site_idx', $siteId)->get();

// Get meters for a specific gateway
$meters = MeterModel::where('rtu_idx', $gatewayId)->get();

// With eager loading (if relationships defined)
$site = SiteModel::with(['meters', 'gateways'])->find($id);
```

## 🐛 Common Patterns

### Soft Deletes

Models do **not** use soft deletes. The `meter_site` table has a `deleted_at` column but it's a VARCHAR, not a proper soft delete implementation.

### Timestamps

Models use Laravel's automatic timestamps (`created_at`, `updated_at`), but also track:
- `created_by_user_idx` - User who created
- `modified_by_user_idx` - User who last modified

### Foreign Key Naming

Foreign keys use `_idx` suffix:
- `site_idx` → references `site_id`
- `rtu_idx` → references `rtu_id`
- `company_idx` → references `company_id`

## ⚠️ Important Notes

1. **No Explicit Relationships:** Models don't define Eloquent relationships. Joins are done manually in controllers.
2. **Session-Based User Tracking:** Uses `Session::get('loginID')` instead of `Auth::id()`
3. **Activity Logging:** All CRUD operations are automatically logged for audit purposes
4. **Mass Assignment Protection:** Only fields in `$fillable` can be mass-assigned
5. **Primary Keys:** All use auto-incrementing integer IDs with `_id` suffix

## 📚 Related Documentation

- [Database Schema](database-schema.md) - Complete table structures
- [Site Management](modules/site-management.md) - Using SiteModel
- [Gateway Management](modules/gateway-management.md) - Using GatewayModel
- [Meter Management](modules/meter-management.md) - Using MeterModel

---

**Model Count:** 13 models  
**Activity Logging:** Enabled on all except User  
**Eloquent Relationships:** Not explicitly defined (manual joins used)
