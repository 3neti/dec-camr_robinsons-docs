# CAMR Robinsons - Centralized Automated Meter Reading System

## 📊 Overview

The **Centralized Automated Meter Reading (CAMR)** system for Robinsons is a Laravel 8-based web application designed to monitor, manage, and report on electricity consumption across multiple sites. The system collects data from remote gateways and meters, providing comprehensive reporting and analysis capabilities.

## ✨ Key Features

### 🏢 Site & Building Management
- Multi-site support with hierarchical organization
- Building and meter location tracking
- Company and division management
- Customizable site access controls

### 📡 Gateway & Meter Operations
- Real-time gateway monitoring and configuration
- Remote gateway management (SSH, CSV updates, site code updates)
- Meter data collection and storage
- Load profile receiving and processing
- Offline detection and alerting

### 📈 Comprehensive Reporting
- **SAP Report** - Integration with SAP systems
- **RAW Report** - Raw meter data exports
- **Site Report** - Site-level consumption summaries
- **Consumption Report** - Hourly and daily consumption analysis
- **Demand Report** - Peak demand tracking (15-min and hourly intervals)
- **Offline Report** - Gateway and meter availability reports
- **Site As-Built** - Infrastructure documentation exports

### 👥 User Management
- Role-based access control
- Site-specific access permissions
- User account management
- Password reset functionality via email

### ⚙️ Configuration & Customization
- Web page branding (logo, navigation, header customization)
- Configuration file management
- Email notifications
- Company and division hierarchies

## 🛠️ Technology Stack

- **Framework:** Laravel 8.x
- **PHP Version:** 7.3+ / 8.0+
- **Database:** MySQL
- **Frontend:** Blade templates with Bootstrap
- **Build Tools:** Laravel Mix, Webpack
- **Key Packages:**
  - `maatwebsite/excel` - Excel import/export
  - `yajra/laravel-datatables-oracle` - DataTables server-side processing
  - `spatie/laravel-activitylog` - Activity logging
  - `phpoffice/phpspreadsheet` - Advanced spreadsheet operations

## 📋 System Components

### Core Modules
1. **Site Management** - Site creation, configuration, and dashboard
2. **Gateway Management** - Gateway registration, monitoring, and remote control
3. **Meter Management** - Meter registration, data collection, and CSV import
4. **Building Management** - Building structure and organization
5. **Meter Location** - EE room and meter placement tracking

### Reporting Engine
- Web-based report generation with preview
- Direct Excel export functionality
- Customizable date ranges and filters
- Building and meter-level granularity

### Device Communication
- Gateway polling endpoints (`/rtu/index.php/rtu/...`)
- CSV configuration updates
- Load profile data reception
- Remote SSH access control

## 🚀 Quick Start

1. **System Requirements** - PHP 7.3+, MySQL, Composer, Node.js
2. **Installation** - Clone repository, install dependencies, configure `.env`
3. **Database Setup** - Run migrations to create database schema
4. **Web Server** - Configure Apache/Nginx to serve the application
5. **Gateway Configuration** - Register gateways and configure meter endpoints

## 📚 Documentation Structure

This documentation is organized into the following sections:

- **Getting Started** - Installation and initial setup
- **System Architecture** - Technical design and data flow
- **Core Modules** - Detailed feature documentation
- **Reporting** - Report types and usage
- **User Management** - Access control and authentication
- **Database** - Schema and models
- **API Reference** - Gateway and load profile APIs
- **Configuration** - Environment and customization
- **Maintenance** - Troubleshooting and known issues

## 🎯 Project Context

This documentation was created as part of a **turnover process** to enable new development teams to understand, maintain, and enhance the CAMR Robinsons system. The system is currently in production use for monitoring electricity consumption across Robinsons properties.

## 📞 Support

For technical questions or issues:
- Review the **Troubleshooting** section
- Check **Known Issues** in the Maintenance section
- Consult **Developer Notes** in the Appendices

---

**Last Updated:** November 2024  
**Laravel Version:** 8.x  
**Database:** meter_reading_robinsons
