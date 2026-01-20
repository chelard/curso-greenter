# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Laravel 10 application that integrates with Greenter (electronic invoice library for Peru's SUNAT system) to generate and send electronic invoices (facturas electrónicas). The application provides a REST API for managing companies and creating invoices with XML/PDF generation and SUNAT submission capabilities.

## Tech Stack

- Laravel 10 (PHP 8.1+)
- JWT Authentication (tymon/jwt-auth)
- Greenter (greenter/lite, greenter/report, greenter/htmltopdf)
- MySQL database
- Vite for frontend assets
- PHPUnit for testing

## Development Commands

### Installation & Setup
```bash
# Install PHP dependencies
composer install

# Install Node dependencies
npm install

# Copy environment file and configure
cp .env.example .env

# Generate application key
php artisan key:generate

# Generate JWT secret
php artisan jwt:secret

# Run database migrations
php artisan migrate

# Create storage link
php artisan storage:link
```

### Development
```bash
# Start development server
php artisan serve

# Run Vite dev server
npm run dev

# Build assets for production
npm run build

# Clear various caches
php artisan cache:clear
php artisan config:clear
php artisan route:clear
php artisan view:clear

# Regenerate autoload files
composer dump-autoload
```

### Testing
```bash
# Run all tests
php artisan test

# Run PHPUnit directly
vendor/bin/phpunit

# Run specific test file
php artisan test --filter InvoiceTest

# Run tests with coverage
vendor/bin/phpunit --coverage-html coverage
```

### Code Quality
```bash
# Format code with Laravel Pint
vendor/bin/pint

# Fix specific files
vendor/bin/pint app/Services/SunatService.php
```

## Architecture

### Core Business Logic

The application is structured around electronic invoice generation and SUNAT integration:

**SunatService** (`app/Services/SunatService.php`)
- Central service for all Greenter operations
- `getSee()`: Configures SUNAT connection with certificates and endpoints
- `getInvoice()`: Builds Invoice object from request data
- `sunatResponse()`: Processes SUNAT API response with error handling
- `getHtmlReport()`: Generates HTML invoice report
- `generatePdfReport()`: Creates PDF using wkhtmltopdf (requires `WKHTML_PDF_PATH` env variable)

**Invoice Controller** (`app/Http/Controllers/Api/InvoiceController.php`)
- `send()`: Sends invoice to SUNAT and returns XML + CDR response
- `xml()`: Generates signed XML without SUNAT submission
- `pdf()`: Generates PDF report
- `setTotales()`: Calculates invoice totals based on tax affectation codes (tipAfeIgv)
- `setLegends()`: Converts total amount to words using luecano/numero-a-letras

### Tax Calculation Logic

The application uses SUNAT tax affectation codes (`tipAfeIgv`):
- 10: Gravado (taxable)
- 20: Exonerado (exonerated)
- 30: Inafecto (unaffected)
- 40: Exportación (export)
- Other: Gratuitas (free/promotional)

Calculations in `InvoiceController::setTotales()`:
1. Groups details by tax type
2. Calculates IGV (18% VAT) and ICBPER (plastic bag tax)
3. Applies rounding: `floor($subTotal * 10) / 10`

### Authentication

JWT-based authentication using tymon/jwt-auth:
- Guard configured in `config/auth.php` as 'api' guard with 'jwt' driver
- Protected routes use `auth:api` middleware
- Token endpoints: login, logout, refresh, me

### Data Models

**Company Model**: Stores company information for invoice issuance
- Belongs to User
- Required fields: razon_social, ruc, direccion, sol_user, sol_pass, cert_path
- File storage: logo_path (company logo), cert_path (SUNAT certificate .pem)
- Production flag toggles between SUNAT beta and production endpoints

**User Model**: Standard Laravel user with JWT authentication
- Has many Companies (one-to-many relationship)

### File Storage

Invoice-related files stored using Laravel Storage facade:
- Company certificates: stored at `cert_path` (configured in Company model)
- Company logos: stored at `logo_path`
- Generated PDFs: stored in `storage/app/invoices/`

### API Structure

All routes defined in `routes/api.php`:

**Public endpoints:**
- POST /api/register
- POST /api/login

**Protected endpoints (require JWT):**
- POST /api/logout
- POST /api/refresh
- POST /api/me
- Resource: /api/companies (full CRUD)
- POST /api/invoices/send
- POST /api/invoices/xml
- POST /api/invoices/pdf

### Request Validation

Invoice requests require:
- `company` (array with address nested)
- `client` (array)
- `details` (array of line items)

Company validation includes custom rule `UniqueCompanyRule` to prevent duplicate RUC per user.

## Environment Configuration

Critical environment variables:
- `DB_*`: Database connection
- `JWT_SECRET`: JWT token signing (generate with `php artisan jwt:secret`)
- `WKHTML_PDF_PATH`: Path to wkhtmltopdf binary for PDF generation
- Standard Laravel configuration (APP_KEY, APP_ENV, etc.)

## Greenter Integration Notes

- SUNAT certificates must be in PEM format
- Production mode determined by Company model's `production` boolean
- Invoice XML follows SUNAT UBL 2.1 specification
- CDR (Constancia de Recepción) returned as base64-encoded ZIP
- Hash calculation uses Greenter's XmlUtils

## Common Development Patterns

When adding new invoice types (credit notes, debit notes, etc.):
1. Add method in SunatService to build the Greenter model
2. Create controller method with validation
3. Implement totals calculation logic
4. Add route in routes/api.php
5. Ensure proper tax affectation handling

When modifying invoice calculations:
- All calculation logic lives in `InvoiceController::setTotales()`
- Tax codes follow SUNAT catalog specifications
- Rounding is critical for SUNAT validation
