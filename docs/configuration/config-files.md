# 📄 Configuration Files

The CAMR system uses **Modbus configuration files** to define communication parameters for different meter models. These configuration files specify register addresses, data types, and parsing rules for extracting meter readings from Modbus-compliant energy meters.

---

## 📋 Overview

**Controller:** `ConfigurationFileController.php`  
**Model:** `ConfigurationFileModel.php`  
**Table:** `meter_configuration_file`  
**Purpose:** Manage Modbus register mappings for different meter manufacturers/models

**File Format:** CSV/Text files with Modbus register definitions

---

## 🔑 Database Structure

### meter_configuration_file Table

| Column | Type | Description |
|--------|------|-------------|
| `config_id` | INT (PK) | Auto-increment primary key |
| `meter_model` | VARCHAR(255) | Meter model identifier |
| `config_file` | VARCHAR(255) | Configuration file name (unique) |
| `created_by_user_idx` | INT | User who created record |
| `modified_by_user_idx` | INT | User who last modified |
| `created_at` | TIMESTAMP | Creation timestamp |
| `updated_at` | TIMESTAMP | Last update timestamp |

**Relationship:** `meter_details.config_idx` → `meter_configuration_file.config_id`

---

## ⚙️ Configuration File Management

### Controller Implementation

```php path=/Users/rli/Documents/DEC/camr_robinsons-main/camr_robinsons-main/app/Http/Controllers/ConfigurationFileController.php start=19
public function ConfigurationFile(){
    $title = 'Configuration File List';
    $data = array();
    if(Session::has('loginID')){
        $WebPageSettingsdata = WebPageSettingsModel::first();
        $data = User::where('user_id', '=', Session::get('loginID'))->first();
        return view("amr.configuration_file", compact('data','title', 'WebPageSettingsdata'));
    }
}
```

### CRUD Operations

#### Create Configuration File

```php path=/Users/rli/Documents/DEC/camr_robinsons-main/camr_robinsons-main/app/Http/Controllers/ConfigurationFileController.php start=100
public function create_configuration_file_post(Request $request){
    $request->validate([
        'configuration_file_name' => 'required|unique:meter_configuration_file,config_file',
    ], [
        'configuration_file_name.required' => 'File Name is Required',
    ]);

    $ConfigurationFileList = new ConfigurationFileModel();
    $ConfigurationFileList->meter_model = 'N/A';
    $ConfigurationFileList->config_file = $request->configuration_file_name;
    $ConfigurationFileList->created_by_user_idx = Session::get('loginID');
    $result = $ConfigurationFileList->save();
    
    if($result){
        return response()->json(['success'=>'Configuration File Information Successfully Created!']);
    }else{
        return response()->json(['success'=>'Error on Insert Configuration File Information']);
    }
}
```

**Validation:**
- Configuration file name is required
- Configuration file name must be unique

#### Update Configuration File

```php path=/Users/rli/Documents/DEC/camr_robinsons-main/camr_robinsons-main/app/Http/Controllers/ConfigurationFileController.php start=128
public function update_configuration_file_post(Request $request){
    $request->validate([
        'configuration_file_name' => 'required|unique:meter_configuration_file,config_file,'.$request->ConfigFileID.',config_id',
    ], [
        'configuration_file_name.required' => 'File Name is Required',
    ]);
    
    $ConfigurationFileList = ConfigurationFileModel::find($request->ConfigFileID);
    $ConfigurationFileList->config_file = $request->configuration_file_name;
    $ConfigurationFileList->modified_by_user_idx = Session::get('loginID');
    $result = $ConfigurationFileList->update();
    
    if($result){
        return response()->json(['success'=>'Configuration File Successfully Updated!']);
    }else{
        return response()->json(['success'=>'Error on Update Company Information']);
    }
}
```

#### Delete Configuration File

```php path=/Users/rli/Documents/DEC/camr_robinsons-main/camr_robinsons-main/app/Http/Controllers/ConfigurationFileController.php start=92
public function delete_configuration_file_confirmed(Request $request){
    $ConfigFileID = $request->ConfigFileID;
    ConfigurationFileModel::find($ConfigFileID)->delete();
    return 'Deleted';
}
```

**⚠️ Warning:** Deleting a configuration file may break meters that reference it.

---

## 📊 DataTables Integration

```php path=/Users/rli/Documents/DEC/camr_robinsons-main/camr_robinsons-main/app/Http/Controllers/ConfigurationFileController.php start=36
public function getConfigurationFileList(Request $request)
{
    $Company = ConfigurationFileModel::get();
    if ($request->ajax()) {
    
        $data = ConfigurationFileModel::select(
            'config_id',
            'config_file',
            'created_at',
            'updated_at'
        );		

        return DataTables::of($data)
            ->addIndexColumn()
            ->addColumn('action', function($row){	
                $actionBtn = '
                <div class="action_table_menu_switch">
                <a href="#" data-id="'.$row->config_id.'" class="bi bi-pencil-fill btn_icon_table btn_icon_table_edit" id="editconfiguration_file" title="Update Configuration File"></a>
                <a href="#" data-id="'.$row->config_id.'" class="bi-trash3-fill btn_icon_table btn_icon_table_delete" id="deleteconfiguration_file" title="Delete Configuration File"></a>
                </div>';
                return $actionBtn;
            })
            ->addColumn('created_at_dt_format', function($row){						
                return $row->created_at;
            })
            ->addColumn('updated_at_dt_format', function($row){		
                if($row->updated_at=="0000-00-00 00:00:00"){
                    return "$row->updated_at";
                }else{
                    return $row->created_at;
                }
            })
            ->rawColumns(['action','created_at_dt_format','updated_at_dt_format'])
            ->make(true);
    }	
}
```

---

## 📝 Modbus Configuration File Format

### Example Configuration File

**File Name:** `Schneider_PM5560.csv`

```csv
Register,Data Type,Description,Unit,Multiplier
3000,FLOAT32,Voltage L1-N,V,1
3002,FLOAT32,Voltage L2-N,V,1
3004,FLOAT32,Voltage L3-N,V,1
3020,FLOAT32,Current L1,A,1
3022,FLOAT32,Current L2,A,1
3024,FLOAT32,Current L3,A,1
3060,FLOAT32,Active Power Total,kW,1
3062,FLOAT32,Reactive Power Total,kVAr,1
3064,FLOAT32,Apparent Power Total,kVA,1
3110,FLOAT32,Power Factor Total,,0.01
3204,INT64,Active Energy Import,kWh,0.001
3208,INT64,Active Energy Export,kWh,0.001
```

### Configuration File Structure

| Column | Description | Example |
|--------|-------------|---------||
| Register | Modbus register address | 3000, 3020, 3204 |
| Data Type | Data format | FLOAT32, INT64, UINT16 |
| Description | Human-readable parameter name | Voltage L1-N, Current L1 |
| Unit | Measurement unit | V, A, kW, kWh |
| Multiplier | Scale factor | 1, 0.001, 0.01 |

### Supported Data Types

| Data Type | Size | Description |
|-----------|------|-------------|
| `UINT16` | 2 bytes | Unsigned 16-bit integer (0-65535) |
| `INT16` | 2 bytes | Signed 16-bit integer (-32768 to 32767) |
| `UINT32` | 4 bytes | Unsigned 32-bit integer |
| `INT32` | 4 bytes | Signed 32-bit integer |
| `INT64` | 8 bytes | Signed 64-bit integer |
| `FLOAT32` | 4 bytes | 32-bit floating point |
| `FLOAT64` | 8 bytes | 64-bit floating point |

---

## 🔌 Integration with Meters

### Assigning Configuration to Meter

When creating/editing a meter, select the appropriate configuration file:

```php
// In meter creation form
<select name="config_idx">
    <option value="">Select Configuration File</option>
    <?php foreach($configuration_file_data as $config): ?>
        <option value="<?= $config->config_id ?>">
            <?= $config->config_file ?>
        </option>
    <?php endforeach; ?>
</select>
```

### Configuration File Usage in Gateway

Gateways use the configuration file to:
1. **Determine register addresses** to read from meter
2. **Parse raw Modbus data** according to data type
3. **Apply multipliers** to convert to correct units
4. **Format data** for storage in database

**Example: Reading Active Energy**
```python
# Gateway Python script (simplified)
register = 3204  # From config file
data_type = "INT64"  # From config file
multiplier = 0.001  # From config file

# Read 4 registers (INT64 = 8 bytes = 4 registers)
raw_value = modbus_client.read_holding_registers(register, 4)

# Parse as INT64
energy_raw = struct.unpack('>q', raw_value)[0]

# Apply multiplier
energy_kwh = energy_raw * multiplier

# Store to database
store_reading(meter_id, 'active_energy_import', energy_kwh)
```

---

## 📖 Usage Examples

### Example 1: Adding New Meter Model

**Scenario:** New Schneider PM5560 meters installed

1. **Obtain Modbus Register Map**
   - From meter manufacturer documentation
   - Example: Schneider PM5560 Modbus register list

2. **Create CSV Configuration File**
   ```csv
   Register,Data Type,Description,Unit,Multiplier
   3000,FLOAT32,Voltage L1-N,V,1
   3020,FLOAT32,Current L1,A,1
   3060,FLOAT32,Active Power Total,kW,1
   3204,INT64,Active Energy Import,kWh,0.001
   ```

3. **Upload to CAMR System**
   - Navigate to: Configuration → Configuration Files
   - Click: Add Configuration File
   - Enter Name: "Schneider_PM5560"
   - Save

4. **Assign to Meters**
   - Navigate to: Site Details → Meters
   - Edit meter
   - Select Configuration File: "Schneider_PM5560"
   - Save

### Example 2: Updating Register Addresses

**Scenario:** Meter firmware update changed register addresses

1. **Create New Configuration File**
   - Name: "Schneider_PM5560_v2"
   - Update register addresses

2. **Test with One Meter**
   - Assign new config to test meter
   - Verify data is reading correctly

3. **Bulk Update**
   ```sql
   UPDATE meter_details 
   SET config_idx = 5  -- New config file ID
   WHERE config_idx = 2  -- Old config file ID
   AND meter_model = 'Schneider PM5560';
   ```

### Example 3: Multi-Manufacturer Setup

**Configuration Files:**
- `Schneider_PM5560.csv` - Schneider Electric meters
- `ABB_M2M.csv` - ABB M2M series
- `Siemens_7KM.csv` - Siemens 7KM series
- `Generic_Modbus.csv` - Generic Modbus meters

**Meter Assignment:**
- 50 meters → Schneider_PM5560
- 30 meters → ABB_M2M
- 20 meters → Siemens_7KM
- 10 meters → Generic_Modbus

---

## 🛠️ Troubleshooting

### Issue: Meter Not Reading Data

**Possible Causes:**
1. Incorrect configuration file assigned
2. Wrong register addresses in config
3. Wrong data type specified
4. Multiplier incorrect

**Solution:**
```bash
# Check meter configuration
SELECT 
    m.meter_name,
    m.meter_model,
    c.config_file
FROM meter_details m
LEFT JOIN meter_configuration_file c ON m.config_idx = c.config_id
WHERE m.meter_id = 123;

# Verify register addresses match meter documentation
# Test manual Modbus read from gateway
```

### Issue: Data Values Incorrect

**Cause:** Wrong multiplier or data type

**Example:**
```
Expected: 1234.5 kWh
Actual: 1234500 kWh

Problem: Multiplier should be 0.001, not 1
```

**Solution:**
- Review meter documentation
- Update configuration file
- Verify with manual Modbus read

### Issue: Cannot Delete Configuration File

**Error:** Foreign key constraint violation

**Cause:** Meters still reference the configuration file

**Solution:**
```sql
-- Check meters using this config
SELECT meter_name, meter_model 
FROM meter_details 
WHERE config_idx = 3;

-- Reassign meters to different config
UPDATE meter_details 
SET config_idx = 5 
WHERE config_idx = 3;

-- Then delete config file
DELETE FROM meter_configuration_file WHERE config_id = 3;
```

---

## 📊 Configuration File Reports

### Query: Meters by Configuration File

```sql
SELECT 
    c.config_file,
    COUNT(m.meter_id) AS meter_count,
    GROUP_CONCAT(m.meter_name SEPARATOR ', ') AS meters
FROM meter_configuration_file c
LEFT JOIN meter_details m ON c.config_id = m.config_idx
GROUP BY c.config_id, c.config_file
ORDER BY meter_count DESC;
```

### Query: Meters Without Configuration

```sql
SELECT 
    meter_id,
    meter_name,
    meter_model,
    meter_status
FROM meter_details
WHERE config_idx IS NULL
   OR config_idx = 0;
```

### Query: Configuration File Usage Statistics

```sql
SELECT 
    c.config_file,
    c.meter_model,
    COUNT(m.meter_id) AS total_meters,
    SUM(CASE WHEN m.meter_status = 'Active' THEN 1 ELSE 0 END) AS active_meters,
    SUM(CASE WHEN DATEDIFF(NOW(), m.last_log_update) < 1 THEN 1 ELSE 0 END) AS online_meters
FROM meter_configuration_file c
LEFT JOIN meter_details m ON c.config_id = m.config_idx
GROUP BY c.config_id, c.config_file, c.meter_model;
```

---

## 🔗 Related Documentation

- **[Meter Management](../modules/meter-management.md)** - Assigning configuration files to meters
- **[Gateway Device API](../api/gateway-device-api.md)** - How gateways use configuration files
- **[Database Schema](../database-schema.md)** - Configuration file table structure
- **[Models](../models.md)** - ConfigurationFileModel implementation

---

## 📝 Best Practices

### For Administrators

1. **Document Configuration Files**
   - Maintain README for each config file
   - Include meter model, firmware version, date created
   - Document any deviations from standard register map

2. **Version Control**
   ```
   Naming Convention:
   - Schneider_PM5560_v1.csv (Original)
   - Schneider_PM5560_v2.csv (Updated 2024-01)
   - Schneider_PM5560_v3.csv (Firmware 1.5.0)
   ```

3. **Test Before Deployment**
   - Create test config file
   - Assign to one meter
   - Verify readings for 24 hours
   - Deploy to production meters

### For Developers

1. **Validate Configuration File Format**
   ```php
   // Pseudo-code for config file validation
   function validateConfigFile($file) {
       // Check CSV format
       // Verify required columns exist
       // Validate data types
       // Check register addresses are numeric
       // Ensure multipliers are valid numbers
   }
   ```

2. **Handle Missing Configuration Gracefully**
   ```php
   $config = ConfigurationFileModel::find($meter->config_idx);
   if (!$config) {
       Log::warning("Meter {$meter->meter_id} has no configuration file");
       // Use default/fallback configuration
       $config = ConfigurationFileModel::where('config_file', 'Generic_Modbus')->first();
   }
   ```

3. **Cache Configuration Files**
   ```php
   // Cache config files to reduce database queries
   $config = Cache::remember("config_{$configId}", 3600, function() use ($configId) {
       return ConfigurationFileModel::find($configId);
   });
   ```

### For Field Technicians

1. **Verify Meter Model Match**
   - Physical meter model must match configuration file
   - Check meter nameplate
   - Verify firmware version if applicable

2. **Test Modbus Communication**
   - Use Modbus testing tool (e.g., QModMaster)
   - Read registers manually
   - Compare with configuration file

3. **Document Issues**
   - Note any discrepancies
   - Report to IT team
   - Update configuration if needed

---

**Last Updated:** 2024-03-15  
**Document Version:** 1.0  
**Maintainer:** CAMR Development Team
