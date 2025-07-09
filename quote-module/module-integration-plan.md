# Loome Module Integration Plan for New Quotes App

## Overview
This document outlines how the new `quotes` app will integrate with existing Loome modules to provide a seamless experience while maintaining system integrity.

## Integration Points

### 1. Projects Module Integration

#### Data Flow
```
Projects App ←→ Quotes App
    ↓              ↓
Project Model   ModernQuote Model
    ↓              ↓
Address Info    Quote Details
Customer Info   Line Items
```

#### Implementation
```python
# src/apps/quotes/models/quote.py
class ModernQuote(BaseModel):
    # Direct relationship to project
    project = models.ForeignKey(
        'projects.Project', 
        related_name='modern_quotes',
        on_delete=models.PROTECT
    )
    
    # Denormalized for performance
    organization = models.ForeignKey(
        'organization.Organization',
        on_delete=models.PROTECT
    )
    
    # Inherit project metadata
    def save(self, *args, **kwargs):
        if self.project_id and not self.organization_id:
            self.organization = self.project.organization
        super().save(*args, **kwargs)
```

#### Project View Integration
```python
# src/apps/projects/views/project.py (modification)
def project_detail_view(request, project_id):
    project = get_object_or_404(Project, id=project_id)
    
    # Get both quote types
    legacy_quotes = project.quotes.all()
    modern_quotes = project.modern_quotes.all()
    
    context = {
        'project': project,
        'legacy_quotes': legacy_quotes,
        'modern_quotes': modern_quotes,
        'show_new_quote_system': settings.ENABLE_MODERN_QUOTES,
    }
    return render(request, 'projects/view.html', context)
```

### 2. Organization Module Integration

#### Customer Data Access
```python
# src/apps/quotes/services/customer.py
from apps.organization.models import Organization
from apps.finance.models import Customer

class QuoteCustomerService:
    
    def get_customer_defaults(self, organization):
        """Get customer-specific defaults for quotes"""
        try:
            customer = organization.customer
            return {
                'payment_terms': customer.payment_terms,
                'tax_exempt': customer.tax_exempt,
                'show_line_pricing': customer.show_line_num,
                'credit_limit': customer.credit_limit,
                'ap_email': customer.ap_inbox_email,
            }
        except Customer.DoesNotExist:
            return {}
    
    def check_credit_approval(self, organization, quote_total):
        """Verify if quote amount is within credit limits"""
        customer = organization.customer
        if customer.credit_hold:
            return False, "Customer is on credit hold"
        
        if customer.credit_limit:
            # Check outstanding balance + new quote
            outstanding = self._get_outstanding_balance(customer)
            if outstanding + quote_total > customer.credit_limit:
                return False, f"Exceeds credit limit by ${outstanding + quote_total - customer.credit_limit:,.2f}"
        
        return True, None
```

### 3. Finance Module Integration

#### Tax Calculation
```python
# src/apps/quotes/services/tax.py
from apps.finance.selectors.tax import get_tax_info_for_project
from apps.finance.models import TaxExemption

class QuoteTaxService:
    
    def calculate_tax(self, quote):
        """Calculate tax using finance module's tax engine"""
        # Check for tax exemption
        exemptions = TaxExemption.objects.filter(
            customer=quote.organization.customer,
            state=quote.project.address.get('state'),
            is_active=True
        )
        
        if exemptions.exists():
            return Decimal('0.00')
        
        # Use existing tax calculation
        tax_info = get_tax_info_for_project(
            quote.project, 
            quote.project.address.get('zipcode')
        )
        
        tax_rate = Decimal(str(tax_info['tax_rate']))
        tax_shipping = tax_info.get('tax_shipping', False)
        
        if tax_shipping:
            taxable_amount = quote.subtotal + quote.get_shipping_amount()
        else:
            taxable_amount = quote.subtotal
        
        return (taxable_amount * tax_rate).quantize(Decimal('0.01'))
```

#### Invoice Generation
```python
# src/apps/quotes/services/invoice.py
from apps.finance.models import Invoice
from apps.finance.services.invoice import InvoiceService

class QuoteInvoiceService:
    
    def create_invoice_from_quote(self, modern_quote, order_id):
        """Create invoice when quote becomes an order"""
        invoice_service = InvoiceService()
        
        # Build invoice data
        invoice_data = {
            'customer': modern_quote.organization.customer,
            'project': modern_quote.project,
            'order_number': order_id,
            'deposit': False,
            'data': {
                'quote_id': modern_quote.version_id,
                'line_items': [
                    {
                        'description': item.description,
                        'quantity': item.quantity,
                        'rate': float(item.sale_price),
                        'amount': float(item.extended_price),
                    }
                    for item in modern_quote.line_items.all()
                ],
                'subtotal': float(modern_quote.subtotal),
                'tax': float(modern_quote.tax_amount),
                'shipping': float(modern_quote.get_shipping_amount()),
                'total': float(modern_quote.total),
            }
        }
        
        return invoice_service.create_invoice(invoice_data)
```

### 4. Supply Chain Module Integration

#### Product Catalog
```python
# src/apps/quotes/services/product.py
from apps.supplychain.models import Product, ProductSupplier
from apps.supplychain.selectors import get_products_for_org

class QuoteProductService:
    
    def search_products(self, organization, search_term):
        """Search products available to organization"""
        # Get org-specific products
        products = get_products_for_org(organization)
        
        # Filter by search term
        products = products.filter(
            Q(sku__icontains=search_term) |
            Q(name__icontains=search_term) |
            Q(manufacturer__name__icontains=search_term)
        ).select_related('manufacturer')[:20]
        
        return products
    
    def get_product_pricing(self, product, organization):
        """Get customer-specific pricing"""
        # Check for customer-specific pricing
        try:
            supplier = ProductSupplier.objects.get(
                product=product,
                organization=organization
            )
            return {
                'cost': supplier.cost,
                'multiplier': supplier.multiplier,
                'sale_price': supplier.price,
            }
        except ProductSupplier.DoesNotExist:
            # Use default pricing
            return {
                'cost': product.cost,
                'multiplier': product.multiplier,
                'sale_price': product.price,
            }
```

#### Inventory Checks
```python
# src/apps/quotes/services/inventory.py
from apps.supplychain.models import Inventory
from apps.supplychain.selectors import get_available_quantity

class QuoteInventoryService:
    
    def check_availability(self, quote):
        """Check inventory availability for quote items"""
        availability = []
        
        for item in quote.line_items.all():
            available_qty = get_available_quantity(
                sku=item.part_number,
                warehouse=None  # Check all warehouses
            )
            
            availability.append({
                'line_item': item,
                'requested': item.quantity,
                'available': available_qty,
                'sufficient': available_qty >= item.quantity,
            })
        
        return availability
```

### 5. Notification Module Integration

#### Email Notifications
```python
# src/apps/quotes/services/notifications.py
from apps.notification.services import NotificationService

class QuoteNotificationService:
    
    def send_quote_created(self, quote):
        """Notify relevant parties of new quote"""
        notification_service = NotificationService()
        
        notification_service.send_notification(
            notification_type='QUOTE_CREATED',
            recipients=[quote.project.project_manager.email],
            context={
                'quote': quote,
                'project': quote.project,
                'view_url': quote.get_absolute_url(),
            }
        )
    
    def send_quote_approved(self, quote):
        """Notify when quote is approved"""
        notification_service = NotificationService()
        
        # Notify PM and sales team
        recipients = [
            quote.project.project_manager.email,
            quote.created_by.email,
        ]
        
        notification_service.send_notification(
            notification_type='QUOTE_APPROVED',
            recipients=recipients,
            context={
                'quote': quote,
                'approved_by': quote.approved_by,
                'order_url': reverse('projects:order-place', 
                                   args=[quote.project.id, quote.legacy_quote.id])
            }
        )
```

### 6. User Module Integration

#### Permissions
```python
# src/apps/quotes/permissions.py
from django.contrib.auth.models import Permission
from django.contrib.contenttypes.models import ContentType

# Define quote permissions
QUOTE_PERMISSIONS = [
    ('view_all_quotes', 'Can view all quotes'),
    ('create_quote', 'Can create quotes'),
    ('edit_quote', 'Can edit quotes'),
    ('approve_quote', 'Can approve quotes'),
    ('delete_quote', 'Can delete quotes'),
    ('view_internal_pricing', 'Can view cost and margin data'),
]

# Permission checks in views
class QuoteCreateView(PermissionRequiredMixin, CreateView):
    permission_required = 'quotes.create_quote'
    
class QuoteInternalView(UserPassesTestMixin, DetailView):
    def test_func(self):
        return self.request.user.has_perm('quotes.view_internal_pricing')
```

### 7. Webhook Module Integration

#### External System Updates
```python
# src/apps/quotes/receivers.py
from apps.webhook.services import WebhookService

@receiver(post_save, sender=ModernQuote)
def notify_external_systems(sender, instance, created, **kwargs):
    """Send webhooks for quote events"""
    webhook_service = WebhookService()
    
    if created:
        event_type = 'quote.created'
    elif instance.status == 'approved':
        event_type = 'quote.approved'
    else:
        event_type = 'quote.updated'
    
    webhook_service.send_webhook(
        event_type=event_type,
        payload={
            'quote_id': instance.version_id,
            'project_id': instance.project.id,
            'organization_id': instance.organization.id,
            'status': instance.status,
            'total': float(instance.total),
            'line_items': [
                {
                    'sku': item.part_number,
                    'quantity': item.quantity,
                    'price': float(item.sale_price),
                }
                for item in instance.line_items.all()
            ]
        }
    )
```

## HTMX Integration Patterns

### Dynamic UI Components
{% raw %}
```html
<!-- templates/quotes/partials/line_item_row.html -->
<tr id="line-item-{{ item.id }}">
    <td>
        <input type="text" 
               name="part_number"
               value="{{ item.part_number }}"
               hx-get="{% url 'quotes:product-search' %}"
               hx-trigger="keyup changed delay:300ms"
               hx-target="#suggestions-{{ item.id }}"
               hx-indicator="#loading-{{ item.id }}">
        <div id="suggestions-{{ item.id }}"></div>
    </td>
    <td>
        <input type="number"
               name="quantity"
               value="{{ item.quantity }}"
               hx-post="{% url 'quotes:update-line-item' item.id %}"
               hx-trigger="change"
               hx-target="#summary-panel"
               hx-swap="outerHTML">
    </td>
</tr>
```
{% endraw %}

### Real-time Calculations
```python
# src/apps/quotes/views/htmx.py
class UpdateLineItemView(View):
    def post(self, request, item_id):
        item = get_object_or_404(LineItem, id=item_id)
        
        # Update item
        item.quantity = int(request.POST.get('quantity', 1))
        item.save()
        
        # Recalculate quote totals
        quote = item.quote
        quote.calculate_totals()
        
        # Return updated summary
        return render(request, 'quotes/partials/summary.html', {
            'quote': quote
        })
```

## Migration Strategy

### Phase 1: Parallel Operation
1. Deploy new quotes app alongside existing system
2. Enable for select pilot projects
3. Both systems visible in project views

### Phase 2: Gradual Adoption
1. Default new projects to modern quotes
2. Provide migration tools for active quotes
3. Training and documentation

### Phase 3: Full Migration
1. Disable Google Sheets sync
2. Archive legacy quote system
3. Remove dual-system UI

## Performance Considerations

### Database Optimization
```python
# Indexes for common queries
class ModernQuote(BaseModel):
    class Meta:
        indexes = [
            models.Index(fields=['project', '-created_at']),
            models.Index(fields=['organization', 'status']),
            models.Index(fields=['version_id']),
        ]

# Prefetch related data
quotes = ModernQuote.objects.select_related(
    'project',
    'organization',
    'created_by',
    'approved_by'
).prefetch_related(
    'line_items',
    'line_items__product'
)
```

### Caching Strategy
```python
# Cache expensive calculations
from django.core.cache import cache

def get_quote_totals(quote_id):
    cache_key = f'quote_totals_{quote_id}'
    totals = cache.get(cache_key)
    
    if not totals:
        quote = ModernQuote.objects.get(id=quote_id)
        totals = {
            'subtotal': quote.calculate_subtotal(),
            'tax': quote.calculate_tax(),
            'freight': quote.get_shipping_amount(),
            'total': quote.calculate_total(),
        }
        cache.set(cache_key, totals, 300)  # 5 minutes
    
    return totals
```

## Security Considerations

### Data Access Control
```python
# Row-level security
class QuoteQuerySet(models.QuerySet):
    def for_user(self, user):
        if user.has_perm('quotes.view_all_quotes'):
            return self
        
        # Limit to user's organization
        return self.filter(
            Q(created_by=user) |
            Q(project__project_manager=user) |
            Q(organization__users=user)
        ).distinct()

class ModernQuote(BaseModel):
    objects = QuoteQuerySet.as_manager()
```

### Audit Trail
```python
# Track all quote changes
from simple_history.models import HistoricalRecords

class ModernQuote(BaseModel):
    history = HistoricalRecords()
    
    def get_audit_trail(self):
        return self.history.all().select_related('history_user')
```