# Quote Management Module Requirements

## Overview
This document outlines the key requirements extracted from the prototype quote management application that need to be implemented in the Loome ERP system.

## 1. Quote Version Management

### Version ID Generation
- **Format**: `Q-yymm-xxxxxx` (e.g., Q-2506-A1C3B7)
  - yy: Two-digit year
  - mm: Two-digit month
  - xxxxxx: 6-character random string (alternating consonants and digits)

### Version Control
- Track changes to quotes with automatic versioning
- Detect changes in:
  - Quote header information
  - Line items (additions, deletions, modifications)
  - Notes and terms
  - Tax/freight settings
- Skip saving duplicate versions if no changes detected
- Store version history organized by Client → Project → Version

### Version Storage Structure
- Key format: `${customer} - ${project} - Quote`
- Support multiple versions per client/project combination
- Version retrieval via three-level dropdown navigation

## 2. Quote Header Information

### Required Fields
- Client Name
- Project Name
- Quote Number (used as version label)
- Date (defaults to current date)
- Days Valid (dropdown options: 7, 14, 30, 60, 90 days)

### Optional Fields
- Project Location
- Quoted By
- Drawing Version Comment
- ZIP Code (for tax calculation)

### Settings
- Show Line Item Pricing on PDF (toggle)
- Apply Tax to Freight (checkbox)

## 3. Line Item Management

### Required Line Item Fields
- Type
- Quantity
- Manufacturer
- Part Number
- Cost (internal use only)
- Multiplier (internal use only)
- Sale Price
- Extended Sale Price (auto-calculated: qty × sale price)

### Optional Line Item Fields
- Location
- Description
- Customer Notes
- Vendor Quote Number (internal only)
- Lead Time
- Internal Notes (internal only)

### Line Item Features
- **Add/Remove**: Dynamic addition and deletion of line items
- **Duplicate**: Clone existing line items
- **Show/Hide Optional Fields**: Toggle visibility of optional fields
- **Auto-calculations**:
  - Sale Price = Cost × Multiplier
  - Multiplier = Sale Price ÷ Cost (when sale price is edited)
  - Extended Sale Price = Quantity × Sale Price

### Product Catalog Integration
- **Autocomplete**: Part number search with dropdown suggestions
- **Auto-populate**: Fill manufacturer, cost, multiplier, sale price, and description from catalog
- **Auto-save**: New part numbers automatically added to catalog on blur
- **Search**: Filter by part number or manufacturer
- **Keyboard Navigation**: Arrow keys and Enter to select suggestions

## 4. Calculations and Totals

### Financial Calculations
- **Subtotal**: Sum of all extended sale prices
- **Freight**: 
  - Percentage of subtotal OR
  - Fixed dollar amount
  - Toggle between % and $ modes
- **Sales Tax**:
  - Applied to subtotal only OR
  - Applied to subtotal + freight (configurable)
  - Tax rate lookup by ZIP code
- **Grand Total**: Subtotal + Tax + Freight

### Internal Metrics (Not shown to customer)
- **Total Cost**: Sum of (quantity × cost) for all line items
- **Gross Profit**: Subtotal - Total Cost
- **Margin**: (Gross Profit ÷ Subtotal) × 100

## 5. Tax Integration

### API Integration
- **Endpoint**: `https://beta.myilluminate.com/p/tax-lookup/?zipcode={zipCode}`
- **Response**: Returns tax rate based on ZIP code
- **Tax on Shipping Check**: Separate API call with `?tax_shipping=True` parameter
- **Auto-populate**: Tax rate and "Apply Tax to Freight" setting based on ZIP

### Tax Display
- Show as percentage with 3 decimal places (e.g., 7.250%)
- Allow manual override
- Format display with % symbol when not in focus

## 6. Notes and Terms

### Quotation Notes
- Custom notes field for quote-specific information
- Preserved across versions

### Standard Notes/Terms
- Pre-populated terms and conditions
- Standard language that appears on all quotes
- Editable per quote if needed

## 7. Data Persistence

### Storage Requirements
- Quote versions by client/project
- Product catalog for autocomplete
- Quote templates for reuse
- All data currently stored in localStorage (needs database in Loome)

### Import/Export
- Export quotes, catalog, and templates as JSON
- Import with merge capability for quotes
- Direct replacement for catalog and templates

## 8. User Interface Requirements

### Collapsible Sections
- Quote Manager (new, templates, version retrieval)
- Quote Info (header fields)
- Line Items (product grid)
- Quotation Notes

### Project Summary Panel
- Fixed position on right side
- Shows real-time calculations
- Internal-only section (red background)
- PDF output options

### Form Behavior
- Real-time validation
- Auto-formatting for currency fields
- Percentage/dollar toggle for freight
- Expand/collapse all sections

## 9. PDF Output Requirements

### Customer-Facing PDF Should Include
- **Header**: Client Name, Project Name, Location, Quote #, Date, Quoted By, Valid Until
- **Line Items**: 
  - Always: Type, Quantity, Manufacturer, Part Number
  - Optional: Sale Price, Extended Price (based on toggle)
  - Conditional: Location, Description, Customer Notes, Lead Time (only if filled)
- **Totals**: Subtotal, Sales Tax, Freight, Grand Total
- **Footer**: "Subject to Illuminate terms & conditions"

### Internal Information (Never on Customer PDF)
- Cost, Multiplier, Margin
- Vendor Quote Numbers
- Internal Notes
- Total Cost, Gross Profit calculations

## 10. Business Rules

### Validation Rules
- Client Name and Project Name required for saving
- Line items require: Type, Quantity, Manufacturer, Part Number, pricing info
- Automatic quote expiration calculation based on Days Valid

### Calculation Rules
- All currency values formatted to 2 decimal places
- Multipliers formatted to 3 decimal places
- Tax rates formatted to 3 decimal places
- Percentages stored as decimals (e.g., 7.25% stored as 7.25)

### Templates
- Save current quote as reusable template
- Load template to start new quote
- Templates include all fields except client-specific information