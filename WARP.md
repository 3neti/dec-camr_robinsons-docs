# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Overview

This is a **MkDocs-based documentation repository** for the CAMR (Centralized Automated Meter Reading) Robinsons system. The documentation is created as part of a **turnover process** to help new development teams understand, maintain, and enhance the Laravel 8-based meter reading application.

## Source Code Location

The documented Laravel application is located at:
```
~/Documents/DEC/camr_robinsons-main/camr_robinsons-main/
```

This documentation repository should reference that codebase when providing code examples and technical details.

## Common Commands

### Development
```bash
# Install dependencies
pip install -r requirements.txt

# Run local development server (auto-reloads on changes)
mkdocs serve

# Access at http://localhost:8000
```

### Building
```bash
# Build static site to site/ directory
mkdocs build

# Build and check for broken links
mkdocs build --strict
```

### Testing
```bash
# Validate YAML configuration
python -c "import yaml; yaml.safe_load(open('mkdocs.yml'))"

# Check for broken internal links (requires mkdocs build first)
mkdocs build --strict
```

## Documentation Architecture

### Content Structure
- **index.md** - Landing page with system overview and feature highlights
- **overview.md** - Detailed system architecture and business objectives
- **modules/** - Core module documentation (Site, Gateway, Meter, Building, Location management)
- **reports/** - Report types documentation (SAP, RAW, Site, Consumption, Demand, Offline)
- **api/** - Gateway device API and load profile API documentation
- **configuration/** - Environment setup, email, company/division, config files
- **maintenance/** - Troubleshooting and known issues
- **appendices/** - Glossary and developer notes

### Mermaid Diagrams
This project uses Mermaid for flow diagrams. The Mermaid library is loaded via CDN and initialized in `docs/javascripts/mermaid-init.js`.

**Important conventions:**
- Use `flowchart TB` (top-to-bottom) or `flowchart LR` (left-to-right) for process flows
- Use subgraphs for logical grouping of related components
- Use dotted lines (`-. context .-`) for annotations
- Apply the `note` class to annotation nodes for consistent styling
- Keep diagrams focused on high-level flows rather than implementation details

### Custom Styling
- **extra.css** - Aggressive print/PDF styling to eliminate blank pages
- **print.css** - Print-specific optimizations
- Both are referenced in `mkdocs.yml` under `extra_css`

## Key Concepts & Domain Terms

### CAMR-Specific Terms
- **CAMR** - Centralized Automated Meter Reading (the system being documented)
- **RTU** - Remote Terminal Unit (synonymous with Gateway in this context)
- **Gateway** - Device that collects data from meters and sends to central system
- **Load Profile** - Time-series data showing electricity consumption patterns
- **Site** - Physical location (mall, property) with gateways and meters
- **Building** - Subdivision within a site
- **Meter Location** - Specific EE room within a building
- **EE Room** - Electrical Equipment room where meters are located
- **SAP Report** - Report formatted for SAP system integration
- **RAW Report** - Unprocessed meter data export
- **Site As-Built** - Infrastructure documentation showing all gateways and meters

### Laravel/Technical Terms
- **Blade** - Laravel's templating engine
- **Eloquent** - Laravel's ORM (Object-Relational Mapping)
- **Migration** - Database schema version control
- **Middleware** - Request filtering (authentication, authorization)
- **DataTables** - JavaScript library for interactive tables (server-side processing)
- **Artisan** - Laravel's command-line tool

## Technology Stack

### Documentation Framework
- **MkDocs** (>=1.5.0) with Material theme
- **mkdocs-material** (>=9.5.0) for theme and UI components
- **pymdown-extensions** (>=10.7.0) for enhanced markdown (admonitions, code highlighting, superfences)

### Features Enabled
- Navigation tabs, sections, expand, indexes, and top-level navigation
- Code annotations, copy buttons, and syntax highlighting
- Mermaid diagram support via custom fence formatting
- Admonitions and collapsible details
- Search with suggestions and highlighting

### CAMR Application Stack (Being Documented)

**Backend:**
- **Laravel 8** (PHP 7.3+ / 8.0+)
- **MySQL** database (`meter_reading_robinsons`)
- **Blade templates** for frontend rendering
- **Laravel Mix** for asset compilation

**Key Packages:**
- **maatwebsite/excel** - Excel import/export functionality
- **yajra/laravel-datatables-oracle** - Server-side DataTables processing
- **spatie/laravel-activitylog** - Activity logging for audit trail
- **phpoffice/phpspreadsheet** - Advanced spreadsheet operations

**Infrastructure:**
- Apache/Nginx web server
- MySQL 5.7+ / 8.0+
- PHP 7.3+ / 8.0+
- Gateway devices (RTUs) in the field

## ReadTheDocs Configuration

The project is configured for ReadTheDocs deployment via `.readthedocs.yaml`:
- Python 3.11 on Ubuntu 22.04
- Automatically builds from `mkdocs.yml`
- Installs dependencies from `requirements.txt`

## Writing Guidelines

### Content Principles
1. **Turnover focus** - Write for developers inheriting the project
2. **Code examples** - Reference actual files from the Laravel codebase
3. **Clear architecture** - Explain how components interact
4. **Practical guidance** - Include troubleshooting and maintenance info
5. **Assume Laravel knowledge** - Readers should have basic Laravel experience

### Markdown Conventions
- Use emoji prefixes for section headers (e.g., 📊, 🔑, ✅, 🔐, 📡)
- Use tables for technical comparisons and feature matrices
- Use admonitions for important notes, warnings, and tips
- Use Mermaid for architecture diagrams and data flows
- Keep paragraphs concise and scannable

### Code Blocks
When referencing code from the Laravel application:
```php
// Reference actual file paths
// File: app/Http/Controllers/CAMRSiteController.php
public function site() {
    // ...
}
```

### Document Organization
- Start with high-level overview and purpose
- Provide table of contents for long documents
- Use horizontal rules (`---`) to separate major sections
- Include "Next Steps" or "Related Documentation" sections where appropriate

## Development Workflow

### Adding New Documentation
1. Create markdown file in appropriate `docs/` subdirectory
2. Add entry to `nav:` section in `mkdocs.yml`
3. Test locally with `mkdocs serve`
4. Verify Mermaid diagrams render correctly
5. Check internal links work properly

### Modifying Existing Content
1. Always preserve emoji prefixes and heading structure
2. Maintain consistent table formatting
3. Test Mermaid diagram changes in browser before committing
4. Update cross-references if page structure changes

### Adding Custom Assets
- JavaScript files → `docs/javascripts/`
- CSS files → `docs/stylesheets/`
- Images → `docs/images/` or `docs/assets/`
- Reference in `mkdocs.yml` under `extra_javascript` or `extra_css`

## Important Notes

- **No backend code** - This is a static documentation site for the Laravel application
- **Reference actual code** - When documenting features, reference the actual source files
- **Turnover context** - This is documentation for knowledge transfer to new developers
- **Mermaid rendering** - Requires JavaScript enabled; diagrams won't render in plain markdown viewers
- **Laravel 8 context** - All examples should be Laravel 8 compatible
- **Database name** - The production database is `meter_reading_robinsons`

## Documentation TODO

Priority areas to document:
1. ✅ Index and Overview pages (completed)
2. Installation and system requirements
3. Database schema and model relationships
4. Core module documentation (Site, Gateway, Meter, Building, Location)
5. Report generation workflows
6. Gateway device API endpoints
7. User management and access control
8. Configuration and customization
9. Troubleshooting guide
10. Developer notes with code insights

## Reference Materials

When documenting the CAMR application, refer to:
- `routes/web.php` - All route definitions
- `app/Http/Controllers/` - Controller logic
- `app/Models/` - Eloquent models
- `database/migrations/` - Database schema
- `composer.json` - Dependency list
- `.env.example` - Configuration variables
- `README.md` - Original project README (if exists)
