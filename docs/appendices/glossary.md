# Glossary

This glossary provides definitions for technical terms, acronyms, and domain-specific terminology used throughout the CAMR (Centralized Automated Meter Reading) Robinsons documentation.

---

## A

### Activity Log
Audit trail functionality powered by Spatie's `laravel-activitylog` package that records all create, update, and delete operations performed by users. Includes user ID, timestamp, model changes, and action type.

### Admin User
User account with elevated privileges allowing access to all system functions including user management, system configuration, and global reports. Distinguished from regular users who have restricted site access.

### API Endpoint
Specific URL path that gateways use to communicate with the CAMR server. Examples: `/check_time.php`, `/lp/receive_file.php`, `/rtu/index.php/rtu/rtu_check_update/{mac}/get_update_csv`.

### Artisan
Laravel's command-line interface providing helpful commands for application management, including migrations, cache clearing, and custom command execution.

### As-Built Report
Comprehensive documentation showing the physical layout of all gateways and meters at a specific site. Also called "Site As-Built Report."

---

## B

### Base64 Encoding
Method used to encode binary logo images as text strings for storage in the `web_page_settings` table. Enables logo storage in database without file system dependencies.

### Blade Template
Laravel's templating engine used for rendering HTML views. Files use `.blade.php` extension and support directives like `@if`, `@foreach`, and `{{ }}` for variable output.

### Building
Physical structure within a site containing one or more meter locations (EE rooms). Example: "North Tower," "Main Building," "Parking Structure."

### Bulk Import
Feature allowing mass upload of meter configurations via CSV file, typically used during initial site setup or large-scale gateway updates.

---

## C

### CAMR
**Centralized Automated Meter Reading** - The full name of this system. A web-based application for collecting, storing, and reporting electrical meter data from remote locations.

### Chunk/Chunking
Laravel database method that processes large result sets in smaller batches to prevent memory exhaustion. Example: `Model::chunk(100, function($items) { })`.

### Company
Top-level organizational entity in the system hierarchy. A company can have multiple divisions and sites. Example: "Robinsons Land Corporation."

### Configuration File
Template defining meter-specific parameters such as multiplier, channel count, and data format. Stored in `meter_configuration_files_table` and assigned to individual meters.

### Consumption Report
Report showing total energy consumption (kWh) for selected meters over a specified period. Can display hourly or daily aggregations.

### Controller
Laravel component handling HTTP requests and returning responses. Located in `app/Http/Controllers/`. Examples: `CAMRSiteController.php`, `CAMRGatewayController.php`.

### CSV (Comma-Separated Values)
Text file format used for bulk meter imports and gateway configuration updates. Contains meter parameters like serial numbers, multipliers, and channel assignments.

### CSV Update Flag
Database field (`gateway_csv_update`) indicating whether a gateway has pending configuration changes. Set to `1` when update needed, reset to `0` after successful download.

---

## D

### Dashboard
Main landing page after login showing system overview, recent activity, online/offline gateway status, and quick access to key functions.

### DataTables
JavaScript library providing interactive tables with server-side processing, sorting, filtering, and pagination. Used extensively throughout CAMR for displaying large datasets.

### DEC
Original developer/implementer of the CAMR system. Default branding includes DEC logo and "Centralized Automated Meter Reading" title.

### Demand Report
Report showing peak power demand (kW) for selected meters. Useful for identifying maximum load periods and capacity planning.

### Division
Sub-unit within a company. Sites are assigned to specific divisions. Example: "Retail Division," "Commercial Properties Division."

---

## E

### EE Room
See **Meter Location**.

### Eloquent
Laravel's ORM (Object-Relational Mapping) system. Provides elegant ActiveRecord implementation for database operations. Models like `Site`, `Gateway`, and `Meter` extend `Illuminate\Database\Eloquent\Model`.

### Excel Export
Feature to download reports in `.xlsx` format using the `maatwebsite/excel` package. Supports large datasets with memory-efficient chunked exports.

---

## F

### Foreign Key
Database constraint ensuring referential integrity between tables. Example: `meter_gateway_id` in `meter_table` references `gateway_id` in `meter_gateway_table`.

### Force Load Profile
Manual trigger sent to gateway requesting immediate load profile data upload, bypassing normal schedule. Useful for testing or recovering missing data.

---

## G

### Gateway
Physical hardware device (RTU) installed at a site that collects meter data and transmits to CAMR server. Identified by serial number and MAC address. Also called **RTU**.

### Gateway Device API
HTTP endpoints that gateways call to check server time, download configuration updates, and report status. Implements legacy URL structure for backward compatibility.

---

## H

### Hash (Password)
One-way cryptographic function used to securely store passwords. CAMR uses Laravel's `bcrypt` hashing via `Hash::make()` and `Hash::check()`.

### Hourly Aggregation
Data summarization method grouping 15-minute readings into hourly totals. Used in SAP Report and some views of Consumption/Demand reports.

---

## I

### isLoggedIn Middleware
Custom authentication middleware protecting routes from unauthorized access. Checks for valid session and redirects to login if not authenticated.

---

## K

### kW (Kilowatt)
Unit of electrical power representing instantaneous demand. Demand reports show kW values.

### kWh (Kilowatt-hour)
Unit of electrical energy representing total consumption over time. Consumption reports show kWh values.

---

## L

### Laravel
PHP web application framework (version 8.x) that CAMR is built upon. Provides MVC architecture, routing, ORM, and extensive ecosystem.

### Load Profile
Time-series data file containing meter readings at 15-minute intervals. Uploaded by gateways via POST to `/lp/receive_file.php` endpoint.

### Load Profile API
Server endpoint receiving load profile data from gateways. Parses uploaded files and stores readings in `meter_load_profile_table`.

### LONGBLOB
MySQL data type storing up to ~4GB of binary data. Used for `image_logo` field in `web_page_settings` table to store base64-encoded logos.

---

## M

### MAC Address
Unique hardware identifier for gateway network interface. Format: `XX:XX:XX:XX:XX:XX`. Used as gateway identifier in API routes.

### Meter
Electrical metering device measuring energy consumption. Connected to gateway via RS-485 or similar protocol. Identified by serial number.

### Meter Location
Physical location within a building where meters are installed. Also called **EE Room** (Electrical Equipment Room). Example: "Ground Floor EE Room," "Basement Electrical."

### Migration
Laravel's version control system for database schema. Files in `database/migrations/` define table structures and can be rolled back if needed.

### Model
Laravel class representing database table. Located in `app/Models/`. Examples: `Site.php`, `Gateway.php`, `Meter.php`. Extends `Illuminate\Database\Eloquent\Model`.

### Multiplier
Configuration value applied to raw meter readings to calculate actual consumption. Example: CT ratio of 100:1 requires multiplier of 100.

---

## N

### N+1 Query Problem
Performance issue where related models trigger individual queries in a loop. Solved with eager loading: `Model::with('relation')->get()`.

---

## O

### Offline Gateway
Gateway that hasn't communicated with server within expected timeframe (typically 1+ hours). May indicate connectivity issues or hardware failure.

### Offline Meter
Meter assigned to an offline gateway, or meter that hasn't reported data recently. Reports available for offline gateway/meter lists.

### Offline Report
Excel export listing all offline gateways or offline meters with last-seen timestamps and site assignments.

---

## P

### Primary Key
Unique identifier for database records. Most CAMR tables use auto-incrementing integer IDs. Exception: `web_page_settings` has no primary key (singleton pattern).

### PUT/POST Method
HTTP request methods. CAMR API uses GET for queries and POST for data submissions (form data, file uploads, updates).

---

## R

### RAW Report
Unprocessed meter reading data export showing all 15-minute interval readings without aggregation. Provides complete dataset for external analysis.

### Redis
In-memory data store optionally used for caching and session management. Can improve performance versus file-based caching.

### Remote Terminal Unit
See **RTU**.

### Report Settings
Filtering options for reports including date range, site selection, building, meter location, and specific meters.

### Route
URL path mapping defined in `routes/web.php`. Maps URLs to controller methods. Example: `Route::get('/site', [CAMRSiteController::class,'site'])`.

### RTU
**Remote Terminal Unit** - Another term for **Gateway**. Physical device collecting meter data in the field.

---

## S

### SAP Report
Formatted report designed for import into SAP ERP systems. Aggregates 15-minute readings into hourly totals with specific column formatting.

### Serial Number
Unique identifier for meters and gateways. Used for physical device tracking and configuration assignment.

### Session
Server-side storage of user authentication state and preferences. CAMR uses Laravel's session management with file or database driver.

### Site
Physical location (mall, property, campus) containing gateways and meters. Top level of location hierarchy in CAMR. Example: "Robinsons Galleria," "Robinsons Place Manila."

### Site Access Control
Permission system restricting regular users to specific sites. Admins have access to all sites; regular users see only assigned sites.

### Site As-Built Report
See **As-Built Report**.

### Site Code
Unique identifier for a site. Example: "RG-001" for Robinsons Galleria. Used in reports and gateway configuration.

### SMTP
Simple Mail Transfer Protocol - Used for sending password reset emails. Requires configuration in `.env` file with mail server credentials.

### Spatie ActivityLog
Laravel package (`spatie/laravel-activitylog`) providing audit trail functionality. Automatically logs model changes with user attribution.

### SSH (Remote)
Secure Shell access feature allowing remote command execution on gateways for troubleshooting. Enabled via `gateway_ssh` flag.

---

## T

### Tinker
Laravel's interactive REPL (Read-Eval-Print Loop) accessed via `php artisan tinker`. Allows direct database queries and model manipulation.

### Timestamp
Date/time field recording when readings were taken or records were modified. Format: `YYYY-MM-DD HH:MM:SS`.

---

## U

### User Account
Authentication credentials and profile for system access. Stored in `user_table` with bcrypt-hashed passwords.

### User Role
Access level designation. CAMR has two roles: **Admin** (full access) and **User** (site-restricted access).

---

## V

### Validation
Laravel's request validation system ensuring data integrity before processing. Example: `$request->validate(['field' => 'required|email'])`.

### View
Blade template file rendering HTML. Located in `resources/views/`. Uses `.blade.php` extension.

---

## W

### Web Page Settings
System configuration interface for customizing navigation header title, logo, and branding. Stored in `web_page_settings` table.

### WebPageSettingsModel
Eloquent model for web page customization. Unique in having no primary key; uses `first()` to retrieve single configuration row.

---

## Y

### Yajra DataTables
Laravel package (`yajra/laravel-datatables-oracle`) enabling server-side DataTables processing. Used for efficient handling of large datasets in tables.

---

## Common Abbreviations

| Abbreviation | Full Term | Meaning |
|--------------|-----------|----------|
| API | Application Programming Interface | Set of endpoints for software communication |
| CAMR | Centralized Automated Meter Reading | Full system name |
| CSV | Comma-Separated Values | Text file format for data exchange |
| CT | Current Transformer | Device for measuring high currents |
| DB | Database | Data storage system (MySQL) |
| DEC | (Developer/Company Name) | Original system implementer |
| EE Room | Electrical Equipment Room | Physical meter location |
| ERP | Enterprise Resource Planning | Business management software (e.g., SAP) |
| HTTP | Hypertext Transfer Protocol | Web communication protocol |
| kW | Kilowatt | Unit of power (demand) |
| kWh | Kilowatt-hour | Unit of energy (consumption) |
| MAC | Media Access Control | Hardware network address |
| MFA | Multi-Factor Authentication | Additional security layer (not implemented) |
| MVC | Model-View-Controller | Application design pattern |
| ORM | Object-Relational Mapping | Database abstraction layer |
| PHP | PHP: Hypertext Preprocessor | Server-side programming language |
| POST | (HTTP Method) | Request method for sending data |
| RAW | (Unprocessed) | Original, unaggregated data |
| REPL | Read-Eval-Print Loop | Interactive programming shell |
| RTU | Remote Terminal Unit | Field data collection device (gateway) |
| SAP | Systems, Applications, and Products | ERP software system |
| SMTP | Simple Mail Transfer Protocol | Email sending protocol |
| SQL | Structured Query Language | Database query language |
| SSH | Secure Shell | Encrypted remote access protocol |
| SSL/TLS | Secure Sockets Layer / Transport Layer Security | Encryption protocols |
| URL | Uniform Resource Locator | Web address |
| UUID | Universally Unique Identifier | 128-bit unique identifier |
| VPN | Virtual Private Network | Secure network connection |

---

## Database Tables Quick Reference

| Table Name | Purpose | Key Fields |
|------------|---------|------------|
| `meter_site` | Site information | `site_id`, `site_code`, `site_name` |
| `meter_gateway_table` | Gateway devices | `gateway_id`, `gateway_serial_number`, `gateway_mac_address` |
| `meter_table` | Meter information | `meter_id`, `meter_serial_number`, `meter_gateway_id` |
| `meter_building_table` | Buildings within sites | `building_id`, `building_name`, `building_site_id` |
| `meter_location_table` | Meter locations (EE rooms) | `location_id`, `location_name`, `location_building_id` |
| `meter_load_profile_table` | Meter readings | `meter_id`, `timestamp`, `kwh`, `kw` |
| `meter_configuration_files_table` | Meter config templates | `configuration_file_id`, `configuration_file_name` |
| `meter_company_table` | Company entities | `company_id`, `company_name` |
| `meter_division_table` | Divisions within companies | `division_id`, `division_name`, `division_company_id` |
| `user_table` | User accounts | `user_id`, `user_name`, `user_password` |
| `user_site_access_table` | Site permissions | `user_id`, `site_id` |
| `web_page_settings` | System branding | `navigation_header_title`, `image_logo` |
| `activity_log` | Audit trail | `log_name`, `causer_id`, `subject_type`, `properties` |

---

## Related Documentation

- [Database Schema](../database-schema.md) - Complete table structures
- [Architecture](../architecture.md) - System design overview
- [Technology Stack](../technology-stack.md) - Frameworks and libraries
- [Installation](../installation.md) - Setup instructions

---

## Contributing to Glossary

If you encounter undefined terms or acronyms in the documentation:

1. Add them to this glossary with clear definitions
2. Include context-specific examples where applicable
3. Link to relevant documentation sections
4. Maintain alphabetical ordering

**Format:**
```markdown
### Term Name
Clear, concise definition. Include usage examples or additional context as needed.
```
