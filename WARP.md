# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Overview

This is a **MkDocs-based documentation repository** for the Electronic Notarization Facility (ENF) project in the Philippines. The documentation covers architecture, compliance requirements, and development planning for an ENF platform that meets Philippine Supreme Court standards (A.M. No. 24-10-14-SC).

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
- **index.md** - Landing page with overview and feature highlights
- **STRATEGY_OUTLINE_v3.md** - Accreditation requirements, technical stack, implementation plan
- **CUSTOMER_JOURNEY_MAP.md** - User experience flows (on-demand vs account-based)
- **DUAL-PATH_CANONICALIZATION_AND_NOTARIZATION_FLOW.md** - Technical workflow for document processing

### Mermaid Diagrams
This project uses Mermaid for flow diagrams. The Mermaid library is loaded via CDN and initialized in `docs/javascripts/mermaid-init.js`.

**Important conventions:**
- Use `flowchart TB` (top-to-bottom) for process flows
- Use subgraphs for swimlanes and logical grouping
- Use dotted lines (`-. context .-`) for annotations
- Apply the `note` class to annotation nodes for consistent styling
- Keep diagrams focused on high-level flows rather than implementation details

### Custom Styling
- **extra.css** - Aggressive print/PDF styling to eliminate blank pages
- **print.css** - Print-specific optimizations
- Both are referenced in `mkdocs.yml` under `extra_css`

## Key Concepts & Domain Terms

- **ENF** - Electronic Notarization Facility (accredited platform for e-notarization)
- **ENP** - Electronic Notary Public (certified notary using the platform)
- **ENA** - Electronic Notary Administrator (Supreme Court oversight role)
- **REN** - Remote Electronic Notarization (video conferencing-based)
- **IEN** - In-Person Electronic Notarization (physical presence with e-signatures)
- **TRUTH Engine** - Document canonicalization and verification system (produces tamper-evident envelopes)
- **HyperVerge eKYC** - Identity verification system with liveness detection
- **PKI** - Public Key Infrastructure (asymmetric cryptography for digital signatures)

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

### ENF Platform Backend Stack

**Backend:**
- **Laravel 12** (PHP 8.2+)
- **Inertia.js** + Vue.js for frontend
- **Laravel Sanctum** for API authentication
- **bavix/laravel-wallet** for wallet system (from Commerce package)
- **lorisleiva/laravel-actions** for action-based architecture
- **spatie/laravel-medialibrary** for document handling
- **pestphp/pest** for testing

**Existing Packages (from sss-acop):**
- **app/Commerce** - Payment, wallet, and transaction handling
- **app/KYC** - HyperVerge eKYC integration with liveness detection

**Third-Party Services:**
- **HyperVerge** - eKYC + Liveness Detection
- **TRUTH Engine** - Document canonicalization
- **Twilio Video** - Video call functionality
- **PayMongo/Xendit** - Payment processing
- **AWS S3** - Document storage (encrypted)
- **PostgreSQL** - Primary database
- **Redis** - Caching and queues

**Infrastructure:**
- Blockchain anchoring (optional) for immutability
- 10-year retention policy
- Supreme Court reporting pipeline

## ReadTheDocs Configuration

The project is configured for ReadTheDocs deployment via `.readthedocs.yaml`:
- Python 3.11 on Ubuntu 22.04
- Automatically builds from `mkdocs.yml`
- Installs dependencies from `requirements.txt`

## Writing Guidelines

### Content Principles
1. **Regulatory accuracy** - Cite specific rule sections (e.g., A.M. No. 24-10-14-SC)
2. **Philippine context** - Focus on Philippine regulations (Data Privacy Act RA 10173, Supreme Court rules)
3. **Technical precision** - Use correct terminology for cryptographic and eKYC concepts
4. **Compliance focus** - Emphasize 10-year retention, ENA access, and audit requirements

### Markdown Conventions
- Use emoji prefixes for section headers (e.g., 🇵🇭, 📌, ✅, 🔐)
- Use tables for technical comparisons and checklists
- Use admonitions for regulatory notes and warnings
- Use Mermaid for process flows and journey maps
- Keep paragraphs concise and scannable

### Document Organization
- Start with high-level overview and key features
- Provide table of contents for long documents
- Use horizontal rules (`---`) to separate major sections
- Include "Next Steps" or "Conclusion" sections where appropriate

## Development Workflow

### Adding New Documentation
1. Create markdown file in `docs/` directory
2. Add entry to `nav:` section in `mkdocs.yml`
3. Test locally with `mkdocs serve`
4. Verify Mermaid diagrams render correctly
5. Check print/PDF output if document will be exported

### Modifying Existing Content
1. Always preserve emoji prefixes and heading structure
2. Maintain consistent table formatting (especially checklists)
3. Test Mermaid diagram changes in browser before committing
4. Preserve regulatory citations and legal language

### Adding Custom Assets
- JavaScript files → `docs/javascripts/`
- CSS files → `docs/stylesheets/`
- Images → `docs/images/`
- Other assets → `docs/assets/`
- Reference in `mkdocs.yml` under `extra_javascript` or `extra_css`

## Important Notes

- **No backend code** - This is a static documentation site, no API or database
- **Philippine jurisdiction** - All compliance and regulatory content is PH-specific
- **Mermaid rendering** - Requires JavaScript enabled; diagrams won't render in plain markdown viewers
- **PDF generation** - The `mkdocs-with-pdf` plugin is installed but not configured in `mkdocs.yml`; add configuration if PDF export is needed
- **10-year retention** - Referenced frequently in compliance context; core requirement for ENF accreditation
