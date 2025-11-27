# 🏗️ GlobalMarket: Technical Architecture Document
## Complete System Design & Implementation Guide

**Version:** 1.0  
**Date:** November 28, 2025  
**Status:** Ready for Implementation  
**Team:** Development Team

---

## 📋 Table of Contents

1. [System Overview](#system-overview)
2. [Architecture Principles](#architecture-principles)
3. [Multi-Tenant Architecture](#multi-tenant-architecture)
4. [Database Architecture](#database-architecture)
5. [Backend Architecture](#backend-architecture)
6. [Frontend Architecture](#frontend-architecture)
7. [API Design](#api-design)
8. [Security Architecture](#security-architecture)
9. [Infrastructure & DevOps](#infrastructure-devops)
10. [Monitoring & Observability](#monitoring-observability)
11. [Deployment Strategy](#deployment-strategy)

---

<a name="system-overview"></a>
## 🌍 1. System Overview

### 1.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    CloudFlare CDN + WAF                     │
│                   (Global Edge Network)                      │
└────────────────────────┬────────────────────────────────────┘
                         │
            ┌────────────┴────────────┐
            │                         │
    ┌───────▼────────┐       ┌───────▼────────┐
    │  Europe Region │       │   MENA Region  │
    │   (West EU)    │       │   (UAE North)  │
    └───────┬────────┘       └───────┬────────┘
            │                         │
    ┌───────▼─────────────────────────▼────────┐
    │         Azure Kubernetes Service          │
    │              (Multi-Tenant)               │
    │  ┌──────────┬──────────┬──────────────┐  │
    │  │ API GW   │ Services │ Background   │  │
    │  │ (Kong)   │          │ Workers      │  │
    │  └──────────┴──────────┴──────────────┘  │
    └────────┬─────────────────────┬────────────┘
             │                     │
    ┌────────▼────────┐   ┌────────▼─────────┐
    │  Cosmos DB      │   │  PostgreSQL      │
    │  (Products,     │   │  (Financials,    │
    │   Orders,       │   │   Audit)         │
    │   Users)        │   │                  │
    └─────────────────┘   └──────────────────┘
             │
    ┌────────▼────────────────────┐
    │  Redis Cluster + ES         │
    │  (Cache, Search, Sessions)  │
    └─────────────────────────────┘
```

### 1.2 Core Components

```yaml
Frontend Layer:
  - Customer Web App (Next.js 15)
  - Vendor Dashboard (Next.js 15)
  - Admin Panel (Next.js 15)
  - Mobile Apps (React Native - Phase 6)

API Layer:
  - API Gateway (Kong)
  - REST APIs (Django REST Framework)
  - GraphQL API (Future)
  - WebSocket Server (Real-time)

Service Layer:
  - Product Service
  - Order Service
  - Payment Service
  - Shipping Service
  - Notification Service
  - AI Service
  - Analytics Service

Data Layer:
  - Cosmos DB (Primary NoSQL)
  - PostgreSQL (Relational)
  - Redis (Cache & Sessions)
  - Elasticsearch (Search)
  - Azure Blob Storage (Files)

Infrastructure:
  - Azure Kubernetes (AKS)
  - CloudFlare (CDN + Security)
  - Azure Monitor (Observability)
  - GitHub Actions (CI/CD)
```

---

<a name="architecture-principles"></a>
## 🎯 2. Architecture Principles

### 2.1 Design Principles

```yaml
1. Multi-Tenancy First:
   - Every entity belongs to a tenant (region)
   - Data isolation at database level
   - Shared infrastructure, isolated data

2. Scalability:
   - Horizontal scaling (Kubernetes)
   - Stateless services
   - Database sharding ready
   - CDN for static content

3. Security:
   - Zero Trust Architecture
   - Encryption everywhere (at rest & in transit)
   - RBAC (Role-Based Access Control)
   - Regular security audits

4. Performance:
   - <100ms API response time (p95)
   - Multi-layer caching
   - Async processing for heavy tasks
   - Optimized database queries

5. Reliability:
   - 99.9% uptime SLA
   - Auto-scaling
   - Health checks
   - Graceful degradation

6. Maintainability:
   - Clean code principles
   - Comprehensive documentation
   - Automated testing
   - Continuous integration
```

### 2.2 Technology Decisions

```yaml
Backend Framework: Django 5.0
Why:
  ✓ Mature & battle-tested
  ✓ Excellent ORM
  ✓ Built-in admin
  ✓ Large ecosystem
  ✓ Good security defaults
  ✓ Fast development

Frontend Framework: Next.js 15
Why:
  ✓ React-based (popular)
  ✓ Server-side rendering
  ✓ App Router (modern)
  ✓ Great DX
  ✓ TypeScript support
  ✓ API routes

Primary Database: Azure Cosmos DB
Why:
  ✓ Global distribution
  ✓ Low latency (<10ms)
  ✓ Elastic scaling
  ✓ Multi-model
  ✓ 99.999% availability
  ✓ Perfect for multi-region

Cloud Provider: Azure
Why:
  ✓ Best Cosmos DB integration
  ✓ Excellent AKS
  ✓ Good pricing
  ✓ Enterprise support
  ✓ GDPR compliant data centers
```

---

<a name="multi-tenant-architecture"></a>
## 🏢 3. Multi-Tenant Architecture

### 3.1 Tenant Model

```python
# models/tenant.py

class Tenant(models.Model):
    """
    Represents a regional marketplace (Europe or MENA)
    """
    
    class TenantType(models.TextChoices):
        EUROPE = 'EU', 'Europe'
        MENA = 'MENA', 'Middle East & North Africa'
        ASIA = 'ASIA', 'Asia Pacific'  # Future
    
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)
    
    # Basic Info
    name = models.CharField(max_length=100)  # "GlobalMarket Europe"
    slug = models.SlugField(unique=True)  # "globalmarket-europe"
    type = models.CharField(max_length=10, choices=TenantType.choices)
    
    # Domain Configuration
    domain = models.CharField(max_length=255, unique=True)  # "globalmarket.eu"
    api_domain = models.CharField(max_length=255)  # "api.globalmarket.eu"
    
    # Regional Settings
    default_currency = models.CharField(max_length=3)  # "EUR" or "USD"
    default_language = models.CharField(max_length=5)  # "en", "nl", "ar"
    timezone = models.CharField(max_length=50)  # "Europe/Amsterdam"
    
    # Countries
    countries = models.JSONField(default=list)  # ["NL", "DE", "BE"]
    
    # Database Configuration
    cosmos_database_name = models.CharField(max_length=100)  # "globalmarket_eu"
    postgres_schema = models.CharField(max_length=100)  # "eu_tenant"
    
    # Features
    features = models.JSONField(default=dict)  # Feature flags per tenant
    
    # Status
    is_active = models.BooleanField(default=True)
    is_in_maintenance = models.BooleanField(default=False)
    
    # Timestamps
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        db_table = 'tenants'
        indexes = [
            models.Index(fields=['slug']),
            models.Index(fields=['domain']),
        ]
    
    def __str__(self):
        return f"{self.name} ({self.type})"
```

### 3.2 Tenant Identification Strategy

```python
# middleware/tenant.py

class TenantMiddleware:
    """
    Identifies tenant from request and sets it in thread-local
    """
    
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        # Method 1: Subdomain-based
        # eu.globalmarket.com -> Europe
        # mena.globalmarket.com -> MENA
        
        # Method 2: Domain-based (preferred)
        # globalmarket.eu -> Europe
        # globalmarket.ae -> MENA
        
        host = request.get_host().split(':')[0]
        
        try:
            tenant = Tenant.objects.get(domain=host, is_active=True)
            request.tenant = tenant
            
            # Set in thread-local for database routing
            set_current_tenant(tenant)
            
        except Tenant.DoesNotExist:
            # Default to Europe or return 404
            tenant = Tenant.objects.get(slug='globalmarket-europe')
            request.tenant = tenant
        
        response = self.get_response(request)
        
        # Clear thread-local
        clear_current_tenant()
        
        return response
```

### 3.3 Database Routing

```python
# database/router.py

class TenantDatabaseRouter:
    """
    Routes database queries based on current tenant
    """
    
    def db_for_read(self, model, **hints):
        # Tenant-specific models go to Cosmos DB
        if model._meta.app_label in ['products', 'orders', 'users']:
            tenant = get_current_tenant()
            if tenant:
                return f'cosmos_{tenant.slug}'
        
        # Financial models go to PostgreSQL
        if model._meta.app_label in ['financials', 'audit']:
            return 'postgres'
        
        return 'default'
    
    def db_for_write(self, model, **hints):
        return self.db_for_read(model, **hints)
    
    def allow_migrate(self, db, app_label, model_name=None, **hints):
        # Cosmos DB doesn't use migrations
        if db.startswith('cosmos_'):
            return False
        return True
```

### 3.4 Data Isolation

```python
# models/base.py

class TenantAwareModel(models.Model):
    """
    Base model for all tenant-specific data
    """
    
    tenant = models.ForeignKey(
        'core.Tenant',
        on_delete=models.CASCADE,
        related_name='%(app_label)s_%(class)s_set'
    )
    
    class Meta:
        abstract = True
        # Every query automatically filtered by tenant
        indexes = [
            models.Index(fields=['tenant']),
        ]
    
    def save(self, *args, **kwargs):
        # Automatically set tenant from thread-local
        if not self.tenant_id:
            self.tenant = get_current_tenant()
        super().save(*args, **kwargs)


# Usage Example:
class Product(TenantAwareModel):
    name = models.CharField(max_length=255)
    price = models.DecimalField(max_digits=10, decimal_places=2)
    # tenant is automatically added
    
    class Meta:
        db_table = 'products'
```

---

<a name="database-architecture"></a>
## 🗄️ 4. Database Architecture

### 4.1 Database Strategy

```yaml
Multi-Database Approach:

1. Azure Cosmos DB (Primary - 80% of data):
   Purpose: High-traffic, real-time data
   
   Collections:
     - products (partition: tenant_id + vendor_id)
     - orders (partition: tenant_id + user_id)
     - users (partition: tenant_id + user_id)
     - carts (partition: tenant_id + user_id)
     - reviews (partition: tenant_id + product_id)
     - sessions (partition: tenant_id + session_id)
   
   Advantages:
     ✓ <10ms latency
     ✓ Global replication
     ✓ Auto-scaling
     ✓ Multi-region writes
     ✓ 99.999% availability

2. PostgreSQL (Secondary - 15% of data):
   Purpose: Transactions, complex queries
   
   Tables:
     - vendor_payouts (ACID required)
     - financial_transactions
     - audit_logs
     - analytics_aggregates
     - admin_actions
   
   Advantages:
     ✓ ACID compliance
     ✓ Complex joins
     ✓ Mature tooling
     ✓ Reporting queries

3. Redis Cluster (Cache - 5% of data):
   Purpose: High-speed cache, sessions
   
   Keys:
     - session:{session_id}
     - product:{product_id}
     - user:{user_id}
     - cart:{cart_id}
     - rate_limit:{user_id}
   
   Advantages:
     ✓ Sub-millisecond latency
     ✓ Perfect for cache
     ✓ Pub/Sub for real-time
     ✓ Leaderboards

4. Elasticsearch:
   Purpose: Full-text search
   
   Indices:
     - products_eu
     - products_mena
     - orders
     - logs
   
   Advantages:
     ✓ Powerful search
     ✓ Faceted search
     ✓ Analytics
     ✓ Log aggregation
```

### 4.2 Cosmos DB Schema

```javascript
// Cosmos DB - Products Collection

{
  "id": "prod_01HGW5XM7N2K8P9Q4R5S6T7V8W",
  "tenant_id": "eu-west-1",
  "vendor_id": "vendor_123",
  
  // Product Info
  "name": {
    "en": "Premium Wireless Headphones",
    "nl": "Premium draadloze koptelefoon",
    "de": "Premium kabellose Kopfhörer"
  },
  "slug": "premium-wireless-headphones",
  "description": {
    "en": "High-quality wireless headphones...",
    "nl": "Hoogwaardige draadloze koptelefoon...",
    "de": "Hochwertige kabellose Kopfhörer..."
  },
  "sku": "HDPH-WL-001",
  "brand": "AudioTech",
  "category_id": "cat_electronics_audio",
  
  // Pricing
  "price": 299.99,
  "currency": "EUR",
  "compare_at_price": 399.99,
  "cost_price": 150.00,
  
  // Inventory
  "stock_quantity": 150,
  "low_stock_threshold": 10,
  "track_inventory": true,
  
  // Media
  "images": [
    {
      "url": "https://cdn.globalmarket.eu/products/hdph-wl-001-1.jpg",
      "alt": "Front view",
      "position": 1
    },
    {
      "url": "https://cdn.globalmarket.eu/products/hdph-wl-001-2.jpg",
      "alt": "Side view",
      "position": 2
    }
  ],
  
  // Variants
  "variants": [
    {
      "id": "var_color_black",
      "name": "Color",
      "value": "Black",
      "price_modifier": 0,
      "sku": "HDPH-WL-001-BLK",
      "stock": 80
    },
    {
      "id": "var_color_white",
      "name": "Color",
      "value": "White",
      "price_modifier": 10,
      "sku": "HDPH-WL-001-WHT",
      "stock": 70
    }
  ],
  
  // SEO
  "seo": {
    "meta_title": "Premium Wireless Headphones | GlobalMarket",
    "meta_description": "Shop premium wireless headphones...",
    "keywords": ["wireless", "headphones", "audio", "premium"]
  },
  
  // Geographic Availability
  "available_in_countries": ["NL", "DE", "BE", "FR"],
  "shipping_info": {
    "NL": {
      "shipping_cost": 0,
      "delivery_days_min": 1,
      "delivery_days_max": 2
    },
    "DE": {
      "shipping_cost": 5.99,
      "delivery_days_min": 2,
      "delivery_days_max": 4
    }
  },
  
  // Performance Metrics
  "metrics": {
    "views": 1250,
    "sales": 45,
    "revenue": 13499.55,
    "rating_avg": 4.7,
    "rating_count": 32,
    "conversion_rate": 0.036
  },
  
  // Status
  "status": "published",  // draft, published, archived
  "is_featured": true,
  "is_on_sale": true,
  
  // Timestamps
  "created_at": "2025-01-15T10:30:00Z",
  "updated_at": "2025-01-20T14:25:00Z",
  "published_at": "2025-01-15T12:00:00Z",
  
  // Partition Key (for Cosmos DB)
  "_partitionKey": "eu-west-1:vendor_123",
  
  // TTL (optional - for temporary items)
  "ttl": null
}
```

```javascript
// Cosmos DB - Orders Collection

{
  "id": "order_01HGW6YN8O3L9Q0R5S6T7U8V9X",
  "tenant_id": "eu-west-1",
  "user_id": "user_456",
  
  // Order Info
  "order_number": "EU-2025-001234",
  "status": "processing",  // pending, processing, shipped, delivered, cancelled
  
  // Customer
  "customer": {
    "id": "user_456",
    "email": "john@example.com",
    "name": "John Doe",
    "phone": "+31612345678"
  },
  
  // Shipping Address
  "shipping_address": {
    "name": "John Doe",
    "street": "Kalverstraat 123",
    "city": "Amsterdam",
    "state": "Noord-Holland",
    "postal_code": "1012 AB",
    "country": "NL",
    "phone": "+31612345678"
  },
  
  // Billing Address
  "billing_address": {
    // Same structure as shipping_address
  },
  
  // Items
  "items": [
    {
      "id": "item_1",
      "product_id": "prod_01HGW5XM7N2K8P9Q4R5S6T7V8W",
      "vendor_id": "vendor_123",
      "name": "Premium Wireless Headphones",
      "sku": "HDPH-WL-001-BLK",
      "variant": "Black",
      "quantity": 1,
      "unit_price": 299.99,
      "total_price": 299.99,
      "image_url": "https://cdn.globalmarket.eu/products/hdph-wl-001-1.jpg"
    }
  ],
  
  // Pricing
  "subtotal": 299.99,
  "shipping_cost": 0,
  "tax": 62.99,  // 21% VAT
  "discount": 0,
  "total": 362.98,
  "currency": "EUR",
  
  // Payment
  "payment": {
    "method": "ideal",  // ideal, card, paypal, cod
    "status": "paid",  // pending, paid, failed, refunded
    "transaction_id": "txn_abc123",
    "paid_at": "2025-01-20T10:15:00Z"
  },
  
  // Shipping
  "shipping": {
    "method": "standard",
    "carrier": "PostNL",
    "tracking_number": "3SABCD1234567890",
    "tracking_url": "https://postnl.nl/track/3SABCD1234567890",
    "estimated_delivery": "2025-01-22",
    "shipped_at": "2025-01-20T16:00:00Z",
    "delivered_at": null
  },
  
  // Notes
  "customer_note": "Please ring doorbell",
  "internal_note": "",
  
  // Timeline
  "timeline": [
    {
      "status": "pending",
      "timestamp": "2025-01-20T10:15:00Z",
      "note": "Order received"
    },
    {
      "status": "processing",
      "timestamp": "2025-01-20T10:20:00Z",
      "note": "Payment confirmed"
    },
    {
      "status": "shipped",
      "timestamp": "2025-01-20T16:00:00Z",
      "note": "Package shipped via PostNL"
    }
  ],
  
  // Timestamps
  "created_at": "2025-01-20T10:15:00Z",
  "updated_at": "2025-01-20T16:00:00Z",
  
  // Partition Key
  "_partitionKey": "eu-west-1:user_456"
}
```

### 4.3 PostgreSQL Schema

```sql
-- PostgreSQL - Financials Schema

-- Vendor Payouts
CREATE TABLE vendor_payouts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id VARCHAR(50) NOT NULL,
    vendor_id VARCHAR(50) NOT NULL,
    
    -- Payout Info
    payout_number VARCHAR(50) UNIQUE NOT NULL,
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    
    -- Amounts
    total_sales DECIMAL(12, 2) NOT NULL,
    commission_rate DECIMAL(5, 2) NOT NULL,
    commission_amount DECIMAL(12, 2) NOT NULL,
    payout_amount DECIMAL(12, 2) NOT NULL,
    currency VARCHAR(3) NOT NULL,
    
    -- Payment Details
    payment_method VARCHAR(20) NOT NULL,  -- bank_transfer, paypal
    bank_account VARCHAR(255),
    paypal_email VARCHAR(255),
    
    -- Status
    status VARCHAR(20) NOT NULL DEFAULT 'pending',  -- pending, processing, paid, failed
    
    -- Timestamps
    processed_at TIMESTAMP,
    paid_at TIMESTAMP,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    -- Indexes
    CONSTRAINT fk_tenant FOREIGN KEY (tenant_id) REFERENCES tenants(id),
    INDEX idx_vendor_payouts_tenant (tenant_id),
    INDEX idx_vendor_payouts_vendor (vendor_id),
    INDEX idx_vendor_payouts_status (status),
    INDEX idx_vendor_payouts_period (period_start, period_end)
);

-- Financial Transactions (Audit Trail)
CREATE TABLE financial_transactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id VARCHAR(50) NOT NULL,
    
    -- Transaction Info
    transaction_type VARCHAR(30) NOT NULL,  -- order, refund, payout, commission
    reference_id VARCHAR(100) NOT NULL,  -- order_id, payout_id, etc.
    
    -- Parties
    from_party_type VARCHAR(20),  -- customer, vendor, platform
    from_party_id VARCHAR(50),
    to_party_type VARCHAR(20),
    to_party_id VARCHAR(50),
    
    -- Amount
    amount DECIMAL(12, 2) NOT NULL,
    currency VARCHAR(3) NOT NULL,
    
    -- Metadata
    description TEXT,
    metadata JSONB,
    
    -- Timestamps
    transaction_date TIMESTAMP NOT NULL DEFAULT NOW(),
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    -- Indexes
    INDEX idx_transactions_tenant (tenant_id),
    INDEX idx_transactions_reference (reference_id),
    INDEX idx_transactions_date (transaction_date),
    INDEX idx_transactions_type (transaction_type)
);

-- Audit Logs
CREATE TABLE audit_logs (
    id BIGSERIAL PRIMARY KEY,
    tenant_id VARCHAR(50) NOT NULL,
    
    -- Actor
    user_id VARCHAR(50),
    user_email VARCHAR(255),
    user_role VARCHAR(50),
    ip_address INET,
    user_agent TEXT,
    
    -- Action
    action VARCHAR(100) NOT NULL,  -- create_product, update_order, etc.
    resource_type VARCHAR(50) NOT NULL,  -- product, order, user
    resource_id VARCHAR(50),
    
    -- Changes
    old_values JSONB,
    new_values JSONB,
    
    -- Timestamps
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    -- Indexes
    INDEX idx_audit_logs_tenant (tenant_id),
    INDEX idx_audit_logs_user (user_id),
    INDEX idx_audit_logs_action (action),
    INDEX idx_audit_logs_resource (resource_type, resource_id),
    INDEX idx_audit_logs_date (created_at)
);
```

---

<a name="backend-architecture"></a>
## 🔧 5. Backend Architecture

### 5.1 Project Structure

```
backend/
├── config/
│   ├── __init__.py
│   ├── settings/
│   │   ├── __init__.py
│   │   ├── base.py          # Base settings
│   │   ├── development.py   # Dev settings
│   │   ├── production.py    # Prod settings
│   │   └── test.py          # Test settings
│   ├── urls.py              # Root URL config
│   ├── wsgi.py              # WSGI application
│   └── asgi.py              # ASGI application (WebSocket)
│
├── apps/
│   ├── core/                # Core functionality
│   │   ├── models/
│   │   │   ├── tenant.py
│   │   │   └── user.py
│   │   ├── middleware/
│   │   │   └── tenant.py
│   │   ├── utils/
│   │   └── management/
│   │
│   ├── products/            # Product management
│   │   ├── models/
│   │   │   ├── product.py
│   │   │   ├── category.py
│   │   │   └── variant.py
│   │   ├── serializers/
│   │   ├── views/
│   │   ├── urls.py
│   │   └── services/
│   │
│   ├── orders/              # Order management
│   │   ├── models/
│   │   │   ├── order.py
│   │   │   ├── cart.py
│   │   │   └── checkout.py
│   │   ├── serializers/
│   │   ├── views/
│   │   ├── services/
│   │   └── tasks.py         # Celery tasks
│   │
│   ├── vendors/             # Vendor management
│   │   ├── models/
│   │   ├── serializers/
│   │   ├── views/
│   │   └── services/
│   │
│   ├── payments/            # Payment processing
│   │   ├── providers/
│   │   │   ├── stripe.py
│   │   │   ├── ideal.py
│   │   │   └── cod.py
│   │   ├── models/
│   │   ├── services/
│   │   └── webhooks.py
│   │
│   ├── shipping/            # Shipping integration
│   │   ├── providers/
│   │   │   ├── postnl.py
│   │   │   ├── dhl.py
│   │   │   └── aramex.py
│   │   ├── models/
│   │   └── services/
│   │
│   ├── ai/                  # AI features
│   │   ├── services/
│   │   │   ├── gpt4.py
│   │   │   ├── recommendations.py
│   │   │   └── visual_search.py
│   │   └── tasks.py
│   │
│   └── analytics/           # Analytics
│       ├── models/
│       ├── services/
│       └── reports/
│
├── common/                  # Shared utilities
│   ├── exceptions.py
│   ├── pagination.py
│   ├── permissions.py
│   └── validators.py
│
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
│
├── docker/
│   ├── Dockerfile
│   ├── docker-compose.yml
│   └── entrypoint.sh
│
├── requirements/
│   ├── base.txt
│   ├── development.txt
│   ├── production.txt
│   └── test.txt
│
├── manage.py
└── pytest.ini
```

### 5.2 Service Layer Pattern

```python
# apps/products/services/product_service.py

from typing import List, Optional
from django.db import transaction
from apps.products.models import Product
from apps.core.utils import get_current_tenant

class ProductService:
    """
    Business logic for product management
    """
    
    @staticmethod
    def create_product(vendor_id: str, data: dict) -> Product:
        """
        Create a new product
        """
        tenant = get_current_tenant()
        
        with transaction.atomic():
            product = Product.objects.create(
                tenant=tenant,
                vendor_id=vendor_id,
                **data
            )
            
            # Generate AI description if not provided
            if not product.description:
                from apps.ai.services import AIService
                product.description = AIService.generate_description(product)
                product.save()
            
            # Index in Elasticsearch
            product.index_in_search_engine()
            
            # Clear cache
            cache.delete(f'vendor_products:{vendor_id}')
            
            return product
    
    @staticmethod
    def get_products(
        vendor_id: Optional[str] = None,
        category_id: Optional[str] = None,
        status: str = 'published',
        page: int = 1,
        per_page: int = 20
    ) -> dict:
        """
        Get products with filters and pagination
        """
        tenant = get_current_tenant()
        
        # Build query
        query = Product.objects.filter(
            tenant=tenant,
            status=status
        )
        
        if vendor_id:
            query = query.filter(vendor_id=vendor_id)
        
        if category_id:
            query = query.filter(category_id=category_id)
        
        # Paginate
        start = (page - 1) * per_page
        end = start + per_page
        
        products = list(query[start:end])
        total = query.count()
        
        return {
            'items': products,
            'total': total,
            'page': page,
            'per_page': per_page,
            'total_pages': (total + per_page - 1) // per_page
        }
    
    @staticmethod
    def update_product(product_id: str, data: dict) -> Product:
        """
        Update existing product
        """
        tenant = get_current_tenant()
        
        with transaction.atomic():
            product = Product.objects.get(
                id=product_id,
                tenant=tenant
            )
            
            # Update fields
            for key, value in data.items():
                setattr(product, key, value)
            
            product.save()
            
            # Re-index
            product.index_in_search_engine()
            
            # Clear cache
            cache.delete(f'product:{product_id}')
            
            return product
```

---

**الوثيقة طويلة جداً! 🚀**

رفعت لك الجزء الأول من **TECHNICAL_ARCHITECTURE.md** (15KB).

بدك أكمل باقي الأجزاء؟
- API Design
- Security Architecture  
- Infrastructure & DevOps
- Monitoring
- Deployment Strategy

أو بدك أبدأ بوثيقة ثانية (Database Schema أو API Documentation)؟ 🤔