# Quote Management Module - Loome Implementation Plan

## Overview
This document outlines the implementation plan for integrating the Quote Management module into the Loome ERP system, leveraging Django, HTMX, and the existing Loome architecture.

## 1. Database Schema Design

### New Models Required

#### Quote Model
```python
# src/apps/projects/models/quote.py
class Quote(BaseModel):
    """Main quote model with version tracking"""
    # Relationships
    organization = ForeignKey('organization.Organization')  # Client
    project = ForeignKey('projects.Project', null=True, blank=True)
    opportunity = ForeignKey('projects.Opportunity', null=True, blank=True)
    
    # Quote Header Fields
    version_id = CharField(max_length=15, unique=True)  # Q-yymm-xxxxxx
    quote_number = CharField(max_length=50)  # User-defined version label
    project_name = CharField(max_length=255)
    project_location = CharField(max_length=255, blank=True)
    
    # Dates
    quote_date = DateField(default=date.today)
    valid_days = IntegerField(choices=[(7,7), (14,14), (30,30), (60,60), (90,90)])
    expiry_date = DateField()  # Auto-calculated
    
    # Personnel
    quoted_by = ForeignKey('user.User', null=True, blank=True)
    
    # Settings
    show_line_pricing = BooleanField(default=True)
    apply_tax_to_freight = BooleanField(default=False)
    
    # Financial
    tax_rate = DecimalField(max_digits=6, decimal_places=3, default=0)
    freight_amount = DecimalField(max_digits=10, decimal_places=2, default=0)
    freight_type = CharField(choices=[('percent', 'Percent'), ('dollar', 'Dollar')])
    
    # Notes
    quotation_notes = TextField(blank=True)
    drawing_version = CharField(max_length=255, blank=True)
    
    # Status
    status = CharField(choices=[('draft', 'Draft'), ('sent', 'Sent'), ('accepted', 'Accepted'), ('rejected', 'Rejected')])
    
    # Version tracking
    parent_quote = ForeignKey('self', null=True, blank=True)  # For version history
    version_number = IntegerField(default=1)
    
    class Meta:
        unique_together = [['organization', 'project_name', 'version_number']]
```

#### QuoteLineItem Model
```python
# src/apps/projects/models/quote_line_item.py
class QuoteLineItem(BaseModel):
    """Individual line items for quotes"""
    quote = ForeignKey('Quote', related_name='line_items')
    product = ForeignKey('supplychain.Product', null=True, blank=True)
    
    # Required fields
    line_type = CharField(max_length=50)
    quantity = DecimalField(max_digits=10, decimal_places=2)
    manufacturer = CharField(max_length=255)
    part_number = CharField(max_length=255)
    
    # Pricing (internal)
    cost = DecimalField(max_digits=10, decimal_places=2)
    multiplier = DecimalField(max_digits=6, decimal_places=3)
    
    # Pricing (customer-facing)
    sale_price = DecimalField(max_digits=10, decimal_places=2)
    extended_price = DecimalField(max_digits=10, decimal_places=2)  # Calculated
    
    # Optional fields
    location = CharField(max_length=255, blank=True)
    description = TextField(blank=True)
    customer_notes = TextField(blank=True)
    vendor_quote_number = CharField(max_length=100, blank=True)
    lead_time = CharField(max_length=100, blank=True)
    internal_notes = TextField(blank=True)
    
    # Ordering
    sort_order = IntegerField(default=0)
```

#### QuoteTemplate Model
```python
# src/apps/projects/models/quote_template.py
class QuoteTemplate(BaseModel):
    """Reusable quote templates"""
    name = CharField(max_length=255)
    description = TextField(blank=True)
    
    # Template data (JSON field for flexibility)
    template_data = JSONField()  # Stores line items and default values
    
    # Access control
    created_by = ForeignKey('user.User')
    is_public = BooleanField(default=False)
```

### Database Migrations
1. Create initial migrations for new models
2. Add indexes on frequently queried fields (organization, project_name, version_id)
3. Create database views for quote summary calculations

## 2. URL Structure

```python
# src/apps/projects/urls.py additions
urlpatterns += [
    # Quote Management
    path('quotes/', QuoteListView.as_view(), name='quote-list'),
    path('quotes/create/', QuoteCreateView.as_view(), name='quote-create'),
    path('quotes/<int:pk>/', QuoteDetailView.as_view(), name='quote-detail'),
    path('quotes/<int:pk>/edit/', QuoteUpdateView.as_view(), name='quote-edit'),
    path('quotes/<int:pk>/duplicate/', QuoteDuplicateView.as_view(), name='quote-duplicate'),
    path('quotes/<int:pk>/pdf/', QuotePDFView.as_view(), name='quote-pdf'),
    
    # HTMX endpoints
    path('quotes/htmx/line-items/', LineItemsPartialView.as_view(), name='quote-line-items'),
    path('quotes/htmx/summary/', QuoteSummaryPartialView.as_view(), name='quote-summary'),
    path('quotes/htmx/product-search/', ProductSearchView.as_view(), name='product-search'),
    path('quotes/htmx/tax-lookup/', TaxLookupView.as_view(), name='tax-lookup'),
    
    # Templates
    path('quotes/templates/', QuoteTemplateListView.as_view(), name='quote-template-list'),
    path('quotes/templates/create/', QuoteTemplateCreateView.as_view(), name='quote-template-create'),
]
```

## 3. Views Implementation

### Main Views
```python
# src/apps/projects/views/quotes.py

class QuoteCreateView(LoginRequiredMixin, CreateView):
    """Create new quote with HTMX support"""
    model = Quote
    template_name = 'projects/quotes/create.html'
    
    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['organizations'] = Organization.objects.filter(is_active=True)
        context['standard_notes'] = get_standard_notes()
        return context
    
    def form_valid(self, form):
        # Generate version ID
        form.instance.version_id = generate_quote_version_id()
        # Set expiry date
        form.instance.expiry_date = calculate_expiry_date(
            form.instance.quote_date, 
            form.instance.valid_days
        )
        return super().form_valid(form)

class LineItemsPartialView(View):
    """HTMX view for line item management"""
    template_name = 'projects/quotes/partials/line_items.html'
    
    def get(self, request, *args, **kwargs):
        quote = get_object_or_404(Quote, pk=request.GET.get('quote_id'))
        return render(request, self.template_name, {
            'quote': quote,
            'line_items': quote.line_items.all()
        })
    
    def post(self, request, *args, **kwargs):
        # Handle line item CRUD operations
        action = request.POST.get('action')
        if action == 'add':
            # Add new line item
        elif action == 'update':
            # Update existing line item
        elif action == 'delete':
            # Delete line item
        
        # Return updated partial
        return self.get(request, *args, **kwargs)
```

### HTMX Integration Points
1. **Line Item Grid**: Dynamic add/remove with `hx-post` and `hx-swap`
2. **Product Search**: Autocomplete with `hx-trigger="keyup changed delay:300ms"`
3. **Summary Calculations**: Real-time updates with `hx-trigger="change"`
4. **Tax Lookup**: API integration with loading states

## 4. Templates Structure

```
templates/projects/quotes/
├── create.html              # Main quote creation page
├── edit.html               # Quote editing page
├── detail.html             # Quote view/preview
├── list.html               # Quote listing with filters
├── partials/
│   ├── line_items.html     # HTMX line item grid
│   ├── summary.html        # Project summary panel
│   ├── header_form.html   # Quote header fields
│   └── product_search.html # Autocomplete dropdown
└── pdf/
    └── quote.html          # PDF template
```

### Key Template Features
- Use Tabler.io components for consistent UI
- Implement collapsible sections with Bootstrap collapse
- DataTables for quote listing
- Select2 for enhanced dropdowns
- HTMX attributes for dynamic behavior

## 5. Business Logic Implementation

### Services Layer
```python
# src/apps/projects/services/quote_service.py

class QuoteService:
    @staticmethod
    def generate_version_id():
        """Generate Q-yymm-xxxxxx format ID"""
        now = datetime.now()
        prefix = f"Q-{now.strftime('%y%m')}-"
        suffix = generate_random_string(6)  # Alternating consonants/digits
        return prefix + suffix
    
    @staticmethod
    def calculate_totals(quote):
        """Calculate all quote totals"""
        subtotal = sum(item.extended_price for item in quote.line_items.all())
        
        # Calculate freight
        if quote.freight_type == 'percent':
            freight = subtotal * (quote.freight_amount / 100)
        else:
            freight = quote.freight_amount
        
        # Calculate tax
        tax_base = subtotal + freight if quote.apply_tax_to_freight else subtotal
        tax_amount = tax_base * (quote.tax_rate / 100)
        
        return {
            'subtotal': subtotal,
            'freight': freight,
            'tax': tax_amount,
            'total': subtotal + freight + tax_amount
        }
    
    @staticmethod
    def duplicate_quote(quote):
        """Create new version of existing quote"""
        # Logic to duplicate quote with new version number
```

### Celery Tasks
```python
# src/apps/projects/tasks.py

@shared_task
def sync_product_catalog():
    """Sync product catalog for autocomplete"""
    # Update product cache for quick lookups

@shared_task
def generate_quote_pdf(quote_id):
    """Generate PDF in background"""
    # Use existing PDF generation infrastructure
```

## 6. Integration Points

### With Existing Loome Modules

1. **Projects Module**
   - Link quotes to opportunities
   - Convert accepted quotes to projects
   - Quote-to-project workflow

2. **Organization Module**
   - Customer selection and management
   - Multi-tenant support

3. **Supply Chain Module**
   - Product catalog integration
   - Inventory availability checks
   - Cost data synchronization

4. **Finance Module**
   - Convert quotes to sales orders
   - Invoice generation from accepted quotes
   - Tax rate management

### External Integrations

1. **Tax API Integration**
   - Implement API client for tax lookups
   - Cache tax rates by ZIP code
   - Handle API failures gracefully

2. **QuickBooks Sync**
   - Map quote data to QB estimates
   - Sync accepted quotes

## 7. Implementation Phases

### Phase 1: Core Infrastructure (Week 1-2)
- [ ] Create database models and migrations
- [ ] Set up URL structure
- [ ] Create base views and templates
- [ ] Implement quote CRUD operations

### Phase 2: Line Item Management (Week 3)
- [ ] Build HTMX-powered line item grid
- [ ] Implement product search/autocomplete
- [ ] Add calculation logic
- [ ] Create summary panel

### Phase 3: Advanced Features (Week 4)
- [ ] Version management system
- [ ] Template functionality
- [ ] Tax API integration
- [ ] PDF generation

### Phase 4: Integration & Polish (Week 5)
- [ ] Connect to existing Loome modules
- [ ] Add permissions and access control
- [ ] Implement quote workflow (draft → sent → accepted)
- [ ] Testing and bug fixes

## 8. Technical Considerations

### Performance
- Use database views for complex calculations
- Implement caching for product catalog
- Optimize HTMX requests with partial updates
- Use Select2 with pagination for large datasets

### Security
- Implement row-level permissions
- Validate all calculations server-side
- Sanitize PDF output
- Secure API endpoints

### Data Migration
- Import existing quotes from prototype
- Map localStorage data to database
- Preserve version history

## 9. Testing Strategy

### Unit Tests
- Model validation and business logic
- Service layer calculations
- API endpoint responses

### Integration Tests
- HTMX interactions
- PDF generation
- External API calls

### UI Tests
- Form validation
- Dynamic calculations
- Autocomplete functionality

## 10. Deployment Checklist

- [ ] Database migrations applied
- [ ] Static files collected
- [ ] Celery workers configured
- [ ] PDF generation dependencies installed
- [ ] Environment variables set
- [ ] Permissions configured
- [ ] Initial data loaded (standard notes, tax rates)
- [ ] Documentation updated