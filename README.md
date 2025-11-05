# CAMR Robinsons Documentation

MkDocs-based documentation for the Centralized Automated Meter Reading (CAMR) system used by Robinsons properties.

## Purpose

This documentation was created as part of a **turnover process** to help new development teams understand, maintain, and enhance the Laravel 8-based CAMR application located at:

```
~/Documents/DEC/camr_robinsons-main/camr_robinsons-main/
```

## Quick Start

1. **Install dependencies:**
   ```bash
   pip install -r requirements.txt
   ```

2. **Run development server:**
   ```bash
   mkdocs serve
   ```

3. **View documentation:**
   Open http://localhost:8000 in your browser

## What's Included

### Completed Documentation
- ✅ **index.md** - Landing page with system overview
- ✅ **overview.md** - Detailed architecture and business objectives
- ✅ **WARP.md** - Guidance for AI-assisted documentation development

### Documentation Structure
The documentation is organized into the following sections:

1. **Getting Started** - Installation, requirements, and setup
2. **System Architecture** - Technical design and data flow
3. **Core Modules** - Site, Gateway, Meter, Building, Location management
4. **Reporting** - SAP, RAW, Site, Consumption, Demand, and Offline reports
5. **User Management** - Authentication and access control
6. **Database** - Schema and models
7. **API Reference** - Gateway device and load profile APIs
8. **Configuration** - Environment setup and customization
9. **Maintenance** - Troubleshooting and known issues
10. **Appendices** - Glossary and developer notes

### Template Files
Most documentation files are still placeholder stubs from the template and need to be populated with CAMR-specific content by analyzing the Laravel codebase.

## Technology Stack

- **MkDocs** with Material theme
- **Mermaid** for diagrams
- **Python 3.11+**

## Building the Docs

```bash
# Build static site
mkdocs build

# Build with strict link checking
mkdocs build --strict
```

Output will be in the `site/` directory.

## Next Steps for Documentation

Priority areas to complete:

1. **Installation Guide** - Document setup process for new developers
2. **Database Schema** - Document all tables and relationships from migrations
3. **Core Modules** - Document each major feature (Site, Gateway, Meter management)
4. **Report Workflows** - Explain how each report type works
5. **API Endpoints** - Document gateway device communication
6. **Troubleshooting** - Common issues and solutions
7. **Developer Notes** - Code insights and architectural decisions

## Source Code Analysis

The CAMR Laravel application has the following structure:

### Key Controllers
- `CAMRSiteController` - Site management
- `CAMRGatewayController` - Gateway operations
- `CAMRMeterController` - Meter management
- `CAMRBuildingController` - Building management
- `CAMRMeterLocationController` - Location management
- `CAMRGatewayDeviceController` - Device API endpoints
- Various report controllers (SAP, RAW, Site, Consumption, Demand, Offline)

### Key Models
- `SiteModel` - Sites
- `GatewayModel` - Gateways/RTUs
- `MeterModel` - Meters
- `BuildingModel` - Buildings
- `MeterLocationModel` - Meter locations
- `MeterDataModel` - Time-series data
- `CompanyModel` / `DivisionModel` - Organizational hierarchy
- `User` / `UserAccountModel` - User management

### Database
- Database name: `meter_reading_robinsons`
- 14 migration files defining schema
- Relationships: Sites → Gateways → Meters, with Building and Location hierarchies

## Contributing

When adding documentation:

1. Create markdown files in appropriate `docs/` subdirectories
2. Update `mkdocs.yml` navigation
3. Test locally with `mkdocs serve`
4. Follow the conventions in `WARP.md`

## Support

For questions about the CAMR system or this documentation, refer to:
- The documentation itself (once complete)
- Laravel codebase at `~/Documents/DEC/camr_robinsons-main/camr_robinsons-main/`
- `WARP.md` for AI-assisted documentation guidance

# ENF Platform Documentation

Comprehensive documentation library for the **Electronic Notarization Facility (ENF)** platform - a two-sided marketplace for electronic notarization services in the Philippines.

## 🚀 Quick Start

```bash
# Install dependencies
pip install -r requirements.txt

# Run documentation server
mkdocs serve

# Visit http://localhost:8000
```

## 📚 Documentation Structure

### Business Documentation
- **Development Plan** - Comprehensive business strategy and roadmap
- **Business Model** - Revenue streams and monetization
- **Go-to-Market** - Marketing and customer acquisition
- **Financial Projections** - Budget, funding, and projections

### Technical Documentation
- **Architecture Overview** - System design and components
- **Laravel Backend** - Backend architecture using Laravel 12
- **API Documentation** - RESTful API endpoints
- **Database Schema** - Data models and relationships

### Backend Implementation
- **Laravel Stack** - Framework and package overview
- **Commerce Package** - Existing wallet and payment system (from sss-acop)
- **KYC Package** - Existing HyperVerge integration (from sss-acop)
- **Payment Integration** - PayMongo/Xendit setup
- **Wallet System** - Transaction and balance management

### Compliance & Regulations
- **Supreme Court Rules** - A.M. No. 24-10-14-SC full text
- **Compliance Requirements** - Accreditation checklist
- **Data Privacy Act** - RA 10173 compliance
- **Supreme Court Reporting** - ENA reporting requirements

### System Design
- **Data Flow Diagrams** - Level 1 and Level 2 DFDs
- **Customer Journey** - User experience flows
- **Notarization Flow** - End-to-end process

### TRUTH Engine
- **Overview** - Document canonicalization system
- **Canonicalization** - PDF/A conversion and hashing
- **Document Strategy** - Input methods and processing

### Implementation Guides
- **Setup Guide** - Development environment
- **ENP Onboarding** - How ENPs join the platform
- **Customer Flow** - Customer registration and KYC
- **Testing Guide** - Test strategy and execution

### API Reference
- **Authentication** - Sanctum-based auth
- **KYC Endpoints** - Identity verification APIs
- **Notarization** - Core notarization endpoints
- **Payments** - Wallet and transaction APIs

## 🏗️ Technology Stack

### Documentation
- **MkDocs** 1.5.0+ with Material theme
- **Mermaid** for diagrams
- **Python** 3.11

### Backend (Actual Platform)
- **Laravel 12** (PHP 8.2+)
- **Inertia.js** + Vue.js
- **PostgreSQL** + Redis
- **Laravel Sanctum** (Auth)
- **Pest** (Testing)

### Existing Packages
- **app/Commerce** (from `/Users/rli/PhpstormProjects/sss-acop`)
  - Wallet system (bavix/laravel-wallet)
  - Payment processing
  - Transaction management
  
- **app/KYC** (from `/Users/rli/PhpstormProjects/sss-acop`)
  - HyperVerge eKYC integration
  - Liveness detection
  - Face matching

### Third-Party Services
- **HyperVerge** - eKYC + liveness
- **Twilio Video** - Video calls
- **TRUTH Engine** - Document canonicalization
- **PayMongo/Xendit** - Payments
- **AWS S3** - Document storage

## 📖 Key Concepts

- **ENF** - Electronic Notarization Facility (the platform)
- **ENP** - Electronic Notary Public (lawyer who notarizes)
- **ENA** - Electronic Notary Administrator (Supreme Court oversight)
- **REN** - Remote Electronic Notarization (video-based)
- **IEN** - In-Person Electronic Notarization (physical presence)
- **TRUTH Engine** - Document canonicalization system
- **KYC** - Know Your Customer (identity verification)

## 🔧 Development

### Build Documentation

```bash
# Build static site
mkdocs build

# Build with link checking
mkdocs build --strict

# Serve locally
mkdocs serve

# Deploy to GitHub Pages
mkdocs gh-deploy
```

### Validate Configuration

```bash
# Check YAML syntax
python -c "import yaml; yaml.safe_load(open('mkdocs.yml'))"
```

## 📝 Writing Documentation

### Markdown Conventions
- Use emoji prefixes for headers (🇵🇭, 📌, ✅, 🔐)
- Create Mermaid diagrams with `flowchart TB`
- Use admonitions for important notes
- Reference Supreme Court rules with specific sections (e.g., A.M. No. 24-10-14-SC, Section 4)

### File Organization
```
docs/
├── index.md                    # Landing page
├── business/                   # Business documentation
├── technical/                  # Technical architecture
├── backend/                    # Laravel implementation
├── compliance/                 # Regulations & requirements
├── regulations/                # Copied from e-notary project
├── guides/                     # Implementation guides
└── backend/api/                # API reference
```

## 🎯 Platform Overview

The ENF Platform is an "Uber for Notarization" - a two-sided marketplace:

**Customer Side:**
1. Download app
2. Upload PDF document
3. Complete KYC (ID + selfie + liveness)
4. Pay via wallet or GCash/Maya
5. Match with available ENP
6. Video call verification
7. Receive notarized document

**ENP Side:**
1. Apply for account
2. Verify credentials
3. Set pricing/tariff
4. Toggle online/offline
5. Accept requests
6. Video call with customer
7. E-sign document
8. Get paid instantly

**Platform Revenue:**
- 20-30% transaction fee
- ENP sets price (minimum ₱200)
- Example: Customer pays ₱400 → ENF gets ₱100, ENP gets ₱300

## 📊 Business Model

- **Two-Sided Marketplace** (like Uber)
- **Transaction-Based Revenue** (20-30% commission)
- **Wallet System** (prepaid credits or pay-per-use)
- **Target Market:** OFWs, remote workers, law firms, SMEs
- **TAM:** 10M+ potential customers in Philippines

## ⚖️ Compliance

- **Supreme Court Rules:** A.M. No. 24-10-14-SC
- **Data Privacy Act:** RA 10173
- **10-Year Retention:** All documents and recordings
- **ENA Reporting:** Quarterly reports to Supreme Court
- **KYC Required:** HyperVerge with liveness detection
- **Video Recording:** All REN sessions must be recorded

## 🔗 Related Projects

- **sss-acop** - Source of Commerce and KYC packages (`/Users/rli/PhpstormProjects/sss-acop`)
- **e-notary** - Source of regulations and DFDs (`/Users/rli/PhpstormProjects/e-notary`)
- **truth** - TRUTH Engine implementation (`/Users/rli/PhpstormProjects/truth`)

## 📞 Support

**Developed by:** 3neti R&D OPC

**Documentation Version:** 1.0  
**Last Updated:** November 2025

---

## 📂 Directory Overview

```
enf-docs/
├── docs/                       # Documentation source
│   ├── index.md               # Home page
│   ├── business/              # Business plans
│   ├── technical/             # Technical docs
│   ├── backend/               # Laravel implementation
│   ├── compliance/            # Regulatory compliance
│   ├── regulations/           # Supreme Court rules & DFDs
│   ├── guides/                # How-to guides
│   └── assets/                # Images and media
├── archive/                   # Archived old docs
├── mkdocs.yml                 # MkDocs configuration
├── requirements.txt           # Python dependencies
├── WARP.md                    # Warp AI guidance
└── README.md                  # This file
```

## 🎉 Getting Started

1. **Install dependencies:** `pip install -r requirements.txt`
2. **Run docs server:** `mkdocs serve`
3. **Visit:** http://localhost:8000
4. **Explore the navigation** - organized by topic
5. **Read Laravel Backend docs** for implementation details
6. **Check Supreme Court Rules** for compliance

Happy documenting! 🚀
