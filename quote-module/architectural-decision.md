# Architectural Decision: New Quotes App vs. Modifying Existing System

## Decision Summary
Create a **new Django app called `quotes`** that implements the modern quote management functionality, while keeping the existing `projects.Quote` model for legacy Google Sheets integration.

## Rationale

### Why a Separate App Makes Sense

1. **Clean Separation of Concerns**
   - Existing system is tightly coupled to Google Sheets sync workflow
   - New system requires fundamentally different data model and workflow
   - Avoids breaking existing functionality during transition

2. **Different Data Models**
   - Existing: Single JSON field stores all quote data from sheets
   - New: Normalized relational models with proper line items, versioning, and calculations

3. **Migration Path**
   - Teams can gradually transition from Google Sheets to new system
   - Both systems can coexist during transition period
   - No risk to existing quotes or workflows

4. **Integration Simplicity**
   - Clear handoff point when quote is "approved"
   - Can transform data to match existing order placement logic
   - Maintains compatibility with Magento integration

## Proposed Architecture

### New App Structure
```
src/apps/quotes/
├── __init__.py
├── apps.py
├── models/
│   ├── __init__.py
│   ├── quote.py          # Main quote model with versioning
│   ├── line_item.py      # Quote line items
│   ├── template.py       # Quote templates
│   └── catalog.py        # Product catalog cache
├── views/
│   ├── __init__.py
│   ├── quote.py          # Quote CRUD views
│   ├── htmx.py           # HTMX partials
│   └── api.py            # API endpoints
├── services/
│   ├── __init__.py
│   ├── calculations.py   # Price/tax calculations
│   ├── version.py        # Version management
│   ├── pdf.py            # PDF generation
│   └── order.py          # Convert to order
├── selectors/
│   ├── __init__.py
│   └── quote.py          # Query helpers
├── forms/
│   ├── __init__.py
│   └── quote.py          # Quote forms
├── templates/quotes/     # Templates
├── static/quotes/        # Static files
├── urls.py
├── admin.py
└── migrations/
```

### Key Models

```python
# src/apps/quotes/models/quote.py
class ModernQuote(BaseModel):
    """New quote model with full feature set"""
    # Link to existing system
    project = models.ForeignKey('projects.Project', related_name='modern_quotes')
    organization = models.ForeignKey('organization.Organization')
    
    # Version management
    version_id = models.CharField(max_length=15, unique=True)  # Q-yymm-xxxxxx
    parent_quote = models.ForeignKey('self', null=True, blank=True)
    version_number = models.IntegerField(default=1)
    
    # Quote details
    quote_number = models.CharField(max_length=50)
    project_name = models.CharField(max_length=255)
    project_location = models.CharField(max_length=255, blank=True)
    
    # Status tracking
    status = models.CharField(choices=QUOTE_STATUS_CHOICES)
    approved_at = models.DateTimeField(null=True, blank=True)
    approved_by = models.ForeignKey('user.User', null=True, blank=True)
    
    # Financial settings
    show_line_pricing = models.BooleanField(default=True)
    tax_rate = models.DecimalField(max_digits=6, decimal_places=3)
    freight_amount = models.DecimalField(max_digits=10, decimal_places=2)
    freight_type = models.CharField(choices=[('percent', 'Percent'), ('dollar', 'Dollar')])
    apply_tax_to_freight = models.BooleanField(default=False)
    
    # Cached calculations
    subtotal = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    tax_amount = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    total = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    
    class Meta:
        db_table = 'quotes_modernquote'
        unique_together = [['organization', 'project_name', 'version_number']]
```

## Integration Strategy

### 1. Project Integration
- Link quotes to existing `Project` model
- Use project's organization and address data
- Inherit project metadata (PM, dates, etc.)

### 2. Order Placement Flow
When a quote is approved in the new system:

```python
# src/apps/quotes/services/order.py
def convert_quote_to_order_data(modern_quote):
    """
    Transform ModernQuote to format expected by existing order placement
    """
    # Build data structure matching existing Quote.data format
    order_data = {
        'parsed_data': {
            'header': {
                'subtotal': float(modern_quote.subtotal),
                'tax_amount': float(modern_quote.tax_amount),
                'shipping_amount': float(modern_quote.get_shipping_amount()),
                'grand_total': float(modern_quote.total),
            },
            'products': [
                {
                    'sku': item.part_number,
                    'qty': item.quantity,
                    'price': float(item.sale_price),
                    'cost': float(item.cost),
                    'vendor': item.manufacturer,
                    'description': item.description,
                    # Map other fields as needed
                }
                for item in modern_quote.line_items.all()
            ]
        }
    }
    
    # Create legacy Quote object for order placement
    legacy_quote = Quote.objects.create(
        project=modern_quote.project,
        quote_number=modern_quote.quote_number,
        data=order_data,
        status=Quote.Status.APPROVED,
        approved_at=modern_quote.approved_at,
        # Map other required fields
    )
    
    return legacy_quote
```

### 3. Coexistence Strategy

**Phase 1: Parallel Systems**
- New quotes created in new system
- Existing quotes remain in old system
- Both visible in project view

**Phase 2: Migration**
- Provide tools to import historical quotes
- Convert active Google Sheets quotes to new format
- Maintain read-only access to legacy quotes

**Phase 3: Deprecation**
- Disable Google Sheets sync for new projects
- Archive legacy quote system
- Full cutover to new system

## API Endpoints

### Tax Integration
Reuse existing tax lookup from finance app:
```python
from apps.finance.selectors.tax import get_tax_info_for_project

def lookup_tax_rate(request):
    """HTMX endpoint for tax lookup"""
    zip_code = request.GET.get('zip_code')
    project_id = request.GET.get('project_id')
    
    project = Project.objects.get(id=project_id)
    tax_info = get_tax_info_for_project(project, zip_code)
    
    return JsonResponse({
        'tax_rate': tax_info['tax_rate'],
        'tax_shipping': tax_info['tax_shipping']
    })
```

### Product Catalog
Create new endpoint for product search:
```python
def product_search(request):
    """HTMX autocomplete for products"""
    query = request.GET.get('q', '')
    
    # Search existing products in supplychain app
    products = Product.objects.filter(
        Q(sku__icontains=query) | 
        Q(name__icontains=query) |
        Q(manufacturer__icontains=query)
    )[:10]
    
    return render(request, 'quotes/partials/product_suggestions.html', {
        'products': products
    })
```

## Benefits of This Approach

1. **Zero Risk to Existing System**
   - No changes to current quote/order workflow
   - Existing integrations continue working
   - Gradual migration possible

2. **Clean Implementation**
   - Modern Django patterns
   - Proper model relationships
   - HTMX for dynamic UI

3. **Future Flexibility**
   - Easy to enhance without legacy constraints
   - Can evolve independently
   - Clear upgrade path

4. **Maintains Integration Points**
   - Works with existing Project model
   - Compatible with order placement flow
   - Preserves QuickBooks/Magento sync

## Implementation Timeline

**Week 1-2: Core Infrastructure**
- Create new `quotes` app
- Implement base models
- Set up URL routing

**Week 3-4: Quote Management**
- Build quote creation/editing UI
- Implement line item management
- Add calculation engine

**Week 5: Integration**
- Connect to Project model
- Implement order conversion
- Add to project views

**Week 6: Testing & Polish**
- End-to-end testing
- Performance optimization
- Documentation

## Risks and Mitigation

**Risk**: Confusion between two quote systems
- **Mitigation**: Clear UI differentiation, training materials

**Risk**: Data inconsistency during transition
- **Mitigation**: Read-only legacy quotes, clear status indicators

**Risk**: Integration complexity
- **Mitigation**: Well-defined transformation layer, extensive testing