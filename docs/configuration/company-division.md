# 🏢 Company & Division

The CAMR system uses a hierarchical **organizational structure** to manage multiple companies, divisions, and sites. Companies represent parent organizations, Divisions represent business units within companies, and Sites represent physical properties/buildings.

---

## 📋 Overview

**Controllers:** `CompanyController.php`, `DivisionController.php`  
**Models:** `CompanyModel.php`, `DivisionModel.php`  
**Tables:** `meter_company_table`, `meter_division_table`

**Organizational Hierarchy:**
```
Company (Robinson's Land Corporation)
  └─ Division (Retail Division)
       └─ Site (Robinson's Galleria)
            └─ Building (Main Building)
                 └─ Meter Location (EE Room 1)
                      └─ Gateway (RTU-001)
                           └─ Meter (Main Meter)
```

---

## 🔑 Company Management

### Database Structure

**Table:** `meter_company_table`

| Column | Type | Description |
|--------|------|-------------|
| `company_id` | INT (PK) | Auto-increment primary key |
| `company_name` | VARCHAR(255) | Company name (unique) |
| `company_code` | VARCHAR(50) | Company code (optional) |
| `created_by_user_idx` | INT | User who created record |
| `modified_by_user_idx` | INT | User who last modified |
| `created_at` | TIMESTAMP | Creation timestamp |
| `updated_at` | TIMESTAMP | Last update timestamp |

### Company Controller

```php path=/Users/rli/Documents/DEC/camr_robinsons-main/camr_robinsons-main/app/Http/Controllers/CompanyController.php start=19
public function Company(){
    if(Session::has('loginID')){
        $title = 'Company List';
        $data = array();
        $WebPageSettingsdata = WebPageSettingsModel::first();
        $data = User::where('user_id', '=', Session::get('loginID'))->first();
    
        return view("amr.company", compact('data','title', 'WebPageSettingsdata'));
    }
}
```

### Company CRUD Operations

#### Create Company

```php path=/Users/rli/Documents/DEC/camr_robinsons-main/camr_robinsons-main/app/Http/Controllers/CompanyController.php start=99
public function create_company_post(Request $request){
    $request->validate([
        'company_name' => 'required|unique:meter_company_table,company_name',
    ], [
        'company_name.required' => 'Company Name is Required',
    ]);

    $CompanyList = new CompanyModel();
    $CompanyList->company_name = $request->company_name;
    $CompanyList->created_by_user_idx = Session::get('loginID');
    $result = $CompanyList->save();
    
    if($result){
        return response()->json(['success'=>'Company Information Successfully Created!']);
    }else{
        return response()->json(['success'=>'Error on Insert Company Information']);
    }
}
```

**Validation:**
- Company name is required
- Company name must be unique

#### Update Company

```php path=/Users/rli/Documents/DEC/camr_robinsons-main/camr_robinsons-main/app/Http/Controllers/CompanyController.php start=125
public function update_company_post(Request $request){
    $request->validate([
        'company_name' => 'required|unique:meter_company_table,company_name,'.$request->CompanyID.',company_id',
    ], [
        'company_name.required' => 'Company Name is Required',
    ]);
    
    $CompanyList = CompanyModel::find($request->CompanyID);
    $CompanyList->company_name = $request->company_name;
    $CompanyList->modified_by_user_idx = Session::get('loginID');
    $result = $CompanyList->update();
    
    if($result){
        return response()->json(['success'=>'Company Information Successfully Updated!']);
    }else{
        return response()->json(['success'=>'Error on Update Company Information']);
    }
}
```

**Validation:** Unique check excludes current company ID

#### Delete Company

```php path=/Users/rli/Documents/DEC/camr_robinsons-main/camr_robinsons-main/app/Http/Controllers/CompanyController.php start=91
public function delete_company_confirmed(Request $request){
    $CompanyID = $request->CompanyID;
    CompanyModel::find($CompanyID)->delete();
    return 'Deleted';
}
```

**⚠️ Warning:** Deleting a company will cascade to divisions and sites if foreign key constraints are configured.

---

## 🏭 Division Management

### Database Structure

**Table:** `meter_division_table`

| Column | Type | Description |
|--------|------|-------------|
| `division_id` | INT (PK) | Auto-increment primary key |
| `division_code` | VARCHAR(50) | Division code (unique) |
| `division_name` | VARCHAR(255) | Division name (unique) |
| `created_by_user_idx` | INT | User who created record |
| `modified_by_user_idx` | INT | User who last modified |
| `created_at` | TIMESTAMP | Creation timestamp |
| `updated_at` | TIMESTAMP | Last update timestamp |

### Division Controller

```php path=/Users/rli/Documents/DEC/camr_robinsons-main/camr_robinsons-main/app/Http/Controllers/DivisionController.php start=19
public function division(){
    if(Session::has('loginID')){
        $title = 'Division List';
        $data = array();
        $WebPageSettingsdata = WebPageSettingsModel::first();
        $data = User::where('user_id', '=', Session::get('loginID'))->first();
    
        return view("amr.division", compact('data','title', 'WebPageSettingsdata'));
    }
}
```

### Division CRUD Operations

#### Create Division

```php path=/Users/rli/Documents/DEC/camr_robinsons-main/camr_robinsons-main/app/Http/Controllers/DivisionController.php start=101
public function create_division_post(Request $request){
    $request->validate([
        'division_code' => 'required|unique:meter_division_table,division_code',
        'division_name' => 'required|unique:meter_division_table,division_name',
    ], [
        'division_code.required' => 'Division Code is Required',
        'division_name.required' => 'Division Name is Required',
    ]);

    $DivisionList = new DivisionModel();
    $DivisionList->division_code = $request->division_code;
    $DivisionList->division_name = $request->division_name;
    $DivisionList->created_by_user_idx = Session::get('loginID');
    $result = $DivisionList->save();
    
    if($result){
        return response()->json(['success'=>'Division Information Successfully Created!']);
    }else{
        return response()->json(['success'=>'Error on Insert Division Information']);
    }
}
```

**Validation:**
- Division code is required and unique
- Division name is required and unique

#### Update Division

```php path=/Users/rli/Documents/DEC/camr_robinsons-main/camr_robinsons-main/app/Http/Controllers/DivisionController.php start=130
public function update_division_post(Request $request){
    $request->validate([
        'division_code' => 'required|unique:meter_division_table,division_code,'.$request->DivisionID.',division_id',
        'division_name' => 'required|unique:meter_division_table,division_name,'.$request->DivisionID.',division_id',
    ], [
        'division_code.required' => 'Division Code is Required',
        'division_name.required' => 'Division Name is Required',
    ]);
    
    $DivisionList = DivisionModel::find($request->DivisionID);
    $DivisionList->division_code = $request->division_code;
    $DivisionList->division_name = $request->division_name;
    $DivisionList->modified_by_user_idx = Session::get('loginID');
    $result = $DivisionList->update();
    
    if($result){
        return response()->json(['success'=>'Division Information Successfully Updated!']);
    }else{
        return response()->json(['success'=>'Error on Update Division Information']);
    }
}
```

#### Delete Division

```php path=/Users/rli/Documents/DEC/camr_robinsons-main/camr_robinsons-main/app/Http/Controllers/DivisionController.php start=93
public function delete_division_confirmed(Request $request){
    $DivisionID = $request->DivisionID;
    DivisionModel::find($DivisionID)->delete();
    return 'Deleted';
}
```

---

## 📊 DataTables Integration

### Company List DataTable

```php path=/Users/rli/Documents/DEC/camr_robinsons-main/camr_robinsons-main/app/Http/Controllers/CompanyController.php start=35
public function getCompanyList(Request $request)
{
    $Company = CompanyModel::get();
    if ($request->ajax()) {
    
        $data = CompanyModel::select(
            'company_id',
            'company_name',
            'created_at',
            'updated_at'
        );		

        return DataTables::of($data)
            ->addIndexColumn()
            ->addColumn('action', function($row){	
                $actionBtn = '
                <div class="action_table_menu_switch">
                <a href="#" data-id="'.$row->company_id.'" class="bi bi-pencil-fill btn_icon_table btn_icon_table_edit" id="editCompany" title="Update Company Information"></a>
                <a href="#" data-id="'.$row->company_id.'" class="bi-trash3-fill btn_icon_table btn_icon_table_delete" id="deleteCompany" title="Delete Company Information"></a>
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

**Features:**
- Server-side processing
- Edit/delete action buttons
- Formatted timestamps
- Bootstrap icon buttons

### Division List DataTable

Similar implementation with both `division_code` and `division_name` columns.

---

## 🔗 Integration with Site Management

### Site Creation Workflow

When creating a site, user must select company and division:

```php
// In CAMRSiteController::create_site_post
$request->validate([
    'building_code' => 'required|unique:meter_building_table,building_code',
    'building_description' => 'required|unique:meter_building_table,building_description',
    'division_id' => 'required',
    'company_id' => 'required'
]);

$site = new SiteModel();
$site->division_idx = $request->division_id;
$site->company_idx = $request->company_id;
$site->site_code = $request->building_code;
$site->created_by_user_idx = Session::get('loginID');
$site->save();
```

**Foreign Keys:**
- `meter_site.company_idx` → `meter_company_table.company_id`
- `meter_site.division_idx` → `meter_division_table.division_id`

---

## 📖 Usage Examples

### Example 1: Setup New Organization

**Scenario:** Setting up Robinson's Land Corporation structure

1. **Create Company**
   - Navigate to: Configuration → Company
   - Click: Add Company
   - Enter: "Robinson's Land Corporation"
   - Save

2. **Create Divisions**
   - Navigate to: Configuration → Division
   - Click: Add Division
   - Enter Code: "RET", Name: "Retail Division"
   - Save
   - Repeat for "Commercial Division" (COM), "Residential Division" (RES)

3. **Create Sites**
   - Navigate to: Site Management
   - Click: Add Site
   - Select Company: "Robinson's Land Corporation"
   - Select Division: "Retail Division"
   - Enter Building Code: "RG"
   - Enter Building Description: "Robinson's Galleria"
   - Save

### Example 2: Restructuring Divisions

**Scenario:** Merging two divisions

1. **Update Sites**
   ```sql
   -- Move all sites from old division to new division
   UPDATE meter_site 
   SET division_idx = 2 
   WHERE division_idx = 5;
   ```

2. **Delete Old Division**
   - Verify no sites reference old division
   - Delete division via UI

### Example 3: Multi-Company Setup

**Scenario:** Managing multiple property management companies

**Companies:**
- Robinson's Land Corporation
- Gokongwei Properties
- Ayala Land Inc.

**Divisions per Company:**
- Robinson's Land: Retail, Commercial, Residential
- Gokongwei: Malls, Offices, Hotels
- Ayala Land: Shopping Centers, Business Parks

**Sites:**
- Each site assigned to one company + one division
- Users granted access per site (not per company/division)

---

## 🛠️ Troubleshooting

### Issue: Cannot Delete Company

**Error:** Foreign key constraint violation

**Cause:** Sites or divisions still reference the company

**Solution:**
```sql
-- Check sites referencing company
SELECT * FROM meter_site WHERE company_idx = 1;

-- Check divisions (if company links to division)
SELECT * FROM meter_division_table WHERE company_id = 1;

-- Reassign or delete dependent records first
UPDATE meter_site SET company_idx = 2 WHERE company_idx = 1;

-- Then delete company
DELETE FROM meter_company_table WHERE company_id = 1;
```

### Issue: Duplicate Company Name Error

**Error:** "Company Name already exists"

**Cause:** Attempting to create/update with existing name

**Solution:**
- Use unique company name
- Or update validation to allow duplicates with different codes

### Issue: Division Code vs Name Confusion

**Best Practice:**
- **Division Code:** Short identifier (e.g., "RET", "COM")
- **Division Name:** Full descriptive name (e.g., "Retail Division")
- Both must be unique

---

## 📊 Organizational Reports

### Query: Sites by Company

```sql
SELECT 
    c.company_name,
    COUNT(s.site_id) AS total_sites,
    GROUP_CONCAT(b.building_code) AS sites
FROM meter_company_table c
LEFT JOIN meter_site s ON c.company_id = s.company_idx
LEFT JOIN meter_building_table b ON s.site_id = b.site_idx
GROUP BY c.company_id, c.company_name
ORDER BY total_sites DESC;
```

### Query: Sites by Division

```sql
SELECT 
    d.division_code,
    d.division_name,
    COUNT(s.site_id) AS total_sites,
    GROUP_CONCAT(b.building_description SEPARATOR ', ') AS sites
FROM meter_division_table d
LEFT JOIN meter_site s ON d.division_id = s.division_idx
LEFT JOIN meter_building_table b ON s.site_id = b.site_idx
GROUP BY d.division_id, d.division_code, d.division_name
ORDER BY d.division_code;
```

### Query: Complete Hierarchy

```sql
SELECT 
    c.company_name,
    d.division_code,
    d.division_name,
    b.building_code,
    b.building_description,
    COUNT(DISTINCT g.rtu_id) AS gateway_count,
    COUNT(DISTINCT m.meter_id) AS meter_count
FROM meter_company_table c
INNER JOIN meter_site s ON c.company_id = s.company_idx
INNER JOIN meter_division_table d ON s.division_idx = d.division_id
INNER JOIN meter_building_table b ON s.site_id = b.site_idx
LEFT JOIN meter_rtu g ON s.site_id = g.site_idx
LEFT JOIN meter_details m ON s.site_id = m.site_idx
GROUP BY c.company_id, d.division_id, s.site_id
ORDER BY c.company_name, d.division_code, b.building_code;
```

---

## 🔗 Related Documentation

- **[Site Management](../modules/site-management.md)** - Site creation and hierarchy
- **[Database Schema](../database-schema.md)** - Company and division table structures
- **[User Management](../user-management.md)** - Site access control
- **[Configuration](../configuration.md)** - System configuration overview

---

## 📝 Best Practices

### For Administrators

1. **Establish Hierarchy Early**
   - Define companies and divisions before creating sites
   - Use consistent naming conventions
   - Document organizational structure

2. **Use Meaningful Codes**
   ```
   Division Codes:
   - RET (Retail)
   - COM (Commercial)
   - RES (Residential)
   - HOT (Hospitality)
   ```

3. **Regular Audits**
   - Review company/division structure quarterly
   - Remove unused entries
   - Update names as organization changes

### For Developers

1. **Enforce Foreign Key Constraints**
   ```sql
   ALTER TABLE meter_site
   ADD CONSTRAINT fk_company
   FOREIGN KEY (company_idx) 
   REFERENCES meter_company_table(company_id)
   ON DELETE RESTRICT;
   ```

2. **Add Cascade Options Carefully**
   ```sql
   -- Use CASCADE only if you want automatic deletion
   ON DELETE CASCADE  -- Dangerous: deletes all child records
   ON DELETE RESTRICT -- Safer: prevents deletion if children exist
   ```

3. **Validate Relationships**
   ```php
   // Ensure division belongs to company (if applicable)
   $division = Division::where('division_id', $divisionId)
       ->where('company_id', $companyId)
       ->first();
   ```

---

**Last Updated:** 2024-03-15  
**Document Version:** 1.0  
**Maintainer:** CAMR Development Team
