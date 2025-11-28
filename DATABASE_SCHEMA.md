# 🗄️ GlobalMarket: Complete Database Schema Documentation
## Multi-Database Architecture & Design

**Version:** 1.0  
**Date:** November 28, 2025  
**Status:** Production-Ready Design  
**Team:** Database & Backend Team

---

## 📋 Table of Contents

1. [Database Strategy Overview](#database-strategy)
2. [Azure Cosmos DB Schema](#cosmos-db-schema)
3. [PostgreSQL Schema](#postgresql-schema)
4. [Redis Data Structures](#redis-schema)
5. [Elasticsearch Indices](#elasticsearch-schema)
6. [Data Relationships](#data-relationships)
7. [Indexing Strategy](#indexing-strategy)
8. [Partitioning & Sharding](#partitioning-strategy)
9. [Migration Strategy](#migration-strategy)
10. [Performance Optimization](#performance-optimization)

---

<a name="database-strategy"></a>
## 🎯 1. Database Strategy Overview

### 1.1 Multi-Database Approach

```yaml
Database Distribution Strategy:

┌─────────────────────────────────────────────────────────┐
│                   Application Layer                      │
└────┬────────────────┬──────────────┬────────────────┬───┘
     │                │              │                │
     ▼                ▼              ▼                ▼
┌─────────┐    ┌──────────┐   ┌──────────┐   ┌──────────┐
│ Cosmos  │    │PostgreSQL│   │  Redis   │   │Elastic   │
│   DB    │    │          │   │ Cluster  │   │ search   │
│         │    │          │   │          │   │          │
│ 80%     │    │ 15%      │   │ 5%       │   │ Search   │
│ of data │    │ of data  │   │ of data  │   │ Layer    │
└─────────┘    └──────────┘   └──────────┘   └──────────┘

Data Distribution Rules:

1. Cosmos DB (Primary - Hot Data):
   ✓ High-traffic, real-time data
   ✓ User-facing data (products, orders, profiles)
   ✓ Session data
   ✓ Shopping carts
   ✓ Reviews & ratings
   ✓ Product catalog
   
   Why: <10ms latency, global distribution, auto-scaling

2. PostgreSQL (Transactional - Critical Data):
   ✓ Financial transactions
   ✓ Vendor payouts
   ✓ Audit logs
   ✓ Admin operations
   ✓ Reporting data
   
   Why: ACID compliance, complex joins, data integrity

3. Redis (Cache - Ultra-Hot Data):
   ✓ Session tokens
   ✓ Rate limiting
   ✓ Real-time inventory
   ✓ Product cache
   ✓ User preferences
   
   Why: Sub-millisecond latency, perfect for cache

4. Elasticsearch (Search - Indexed Data):
   ✓ Product search
   ✓ Vendor search
   ✓ Order search
   ✓ Analytics logs
   
   Why: Full-text search, faceting, aggregations
```

### 1.2 Data Flow

```yaml
Write Flow:
  Customer places order
    ↓
  1. Write to Cosmos DB (order document)
  2. Write to PostgreSQL (financial transaction)
  3. Invalidate Redis cache
  4. Update Elasticsearch index (async)
  5. Send to message queue (notifications)

Read Flow:
  Customer views product
    ↓
  1. Check Redis cache → HIT? Return
  2. Cache MISS → Query Cosmos DB
  3. Store in Redis (TTL: 5 minutes)
  4. Return to customer
```

---

<a name="cosmos-db-schema"></a>
## 🌍 2. Azure Cosmos DB Schema

### 2.1 Database Structure

```javascript
// Two databases (one per tenant)
Databases:
  - globalmarket_eu      (Europe tenant)
  - globalmarket_mena    (MENA tenant)

Collections per database:
  - products             (partition: /tenant_id/vendor_id)
  - orders               (partition: /tenant_id/user_id)
  - users                (partition: /tenant_id/user_id)
  - carts                (partition: /tenant_id/session_id)
  - reviews              (partition: /tenant_id/product_id)
  - sessions             (partition: /tenant_id/session_id)
  - notifications        (partition: /tenant_id/user_id)
  - wishlists            (partition: /tenant_id/user_id)
```

### 2.2 Products Collection

```javascript
// Collection: products
// Partition Key: /tenant_id/vendor_id
// RU/s: 4000 (auto-scale to 40,000)

{
  // Identifiers
  "id": "prod_01HGW5XM7N2K8P9Q4R5S6T7V8W",
  "tenant_id": "eu-west-1",
  "vendor_id": "vendor_abc123",
  
  // Basic Info (Multi-language)
  "name": {
    "en": "Premium Wireless Headphones",
    "nl": "Premium Draadloze Koptelefoon",
    "de": "Premium Kabellose Kopfhörer",
    "fr": "Écouteurs Sans Fil Premium",
    "ar": "سماعات لاسلكية متميزة"
  },
  
  "slug": {
    "en": "premium-wireless-headphones",
    "nl": "premium-draadloze-koptelefoon",
    "de": "premium-kabellose-kopfhoerer",
    "fr": "ecouteurs-sans-fil-premium",
    "ar": "سماعات-لاسلكية-متميزة"
  },
  
  "description": {
    "en": "Experience crystal-clear audio with our premium wireless headphones...",
    "nl": "Ervaar kristalhelder geluid met onze premium draadloze koptelefoon...",
    "de": "Erleben Sie kristallklaren Sound mit unseren Premium...",
    "fr": "Découvrez un son cristallin avec nos écouteurs...",
    "ar": "استمتع بصوت نقي مع سماعاتنا اللاسلكية المتميزة..."
  },
  
  // Product Details
  "sku": "HDPH-WL-PRO-001",
  "barcode": "1234567890123",
  "brand": "AudioTech",
  "manufacturer": "AudioTech Industries",
  
  // Category & Tags
  "category_id": "cat_electronics_audio_headphones",
  "category_path": ["electronics", "audio", "headphones"],
  "tags": ["wireless", "bluetooth", "noise-cancelling", "premium"],
  
  // Pricing (per tenant currency)
  "price": 299.99,
  "currency": "EUR",
  "compare_at_price": 399.99,
  "cost_price": 150.00,
  "profit_margin": 0.50,
  
  // Dynamic Pricing (Buy Box competition)
  "buy_box": {
    "current_winner": "vendor_abc123",
    "price": 299.99,
    "competitors": [
      {
        "vendor_id": "vendor_xyz789",
        "price": 305.99,
        "quality_score": 4.2
      }
    ],
    "updated_at": "2025-11-28T10:30:00Z"
  },
  
  // Inventory Management
  "inventory": {
    "stock_quantity": 150,
    "reserved_quantity": 5,  // In carts
    "available_quantity": 145,  // stock - reserved
    "low_stock_threshold": 10,
    "track_inventory": true,
    "allow_backorder": false,
    "warehouse_locations": [
      {
        "warehouse_id": "wh_amsterdam",
        "quantity": 100
      },
      {
        "warehouse_id": "wh_rotterdam",
        "quantity": 50
      }
    ]
  },
  
  // Media Assets
  "images": [
    {
      "id": "img_1",
      "url": "https://cdn.globalmarket.eu/products/hdph-wl-001-1.jpg",
      "url_thumbnail": "https://cdn.globalmarket.eu/products/thumbs/hdph-wl-001-1.jpg",
      "alt": {
        "en": "Premium Wireless Headphones - Front View",
        "nl": "Premium Draadloze Koptelefoon - Vooraanzicht",
        "ar": "سماعات لاسلكية متميزة - المنظر الأمامي"
      },
      "position": 1,
      "is_primary": true
    },
    {
      "id": "img_2",
      "url": "https://cdn.globalmarket.eu/products/hdph-wl-001-2.jpg",
      "url_thumbnail": "https://cdn.globalmarket.eu/products/thumbs/hdph-wl-001-2.jpg",
      "alt": {
        "en": "Side view with controls",
        "nl": "Zijaanzicht met bedieningselementen",
        "ar": "المنظر الجانبي مع عناصر التحكم"
      },
      "position": 2,
      "is_primary": false
    }
  ],
  
  "videos": [
    {
      "url": "https://cdn.globalmarket.eu/videos/hdph-wl-001-demo.mp4",
      "thumbnail": "https://cdn.globalmarket.eu/videos/thumbs/hdph-wl-001-demo.jpg",
      "duration": 120,
      "title": "Product Demo"
    }
  ],
  
  // Variants & Options
  "has_variants": true,
  "variants": [
    {
      "id": "var_color_black",
      "sku": "HDPH-WL-PRO-001-BLK",
      "attributes": {
        "color": "Black",
        "color_hex": "#000000"
      },
      "price_modifier": 0,
      "stock": 80,
      "image_id": "img_1"
    },
    {
      "id": "var_color_white",
      "sku": "HDPH-WL-PRO-001-WHT",
      "attributes": {
        "color": "White",
        "color_hex": "#FFFFFF"
      },
      "price_modifier": 10,
      "stock": 70,
      "image_id": "img_3"
    }
  ],
  
  // Specifications
  "specifications": {
    "technical": [
      {
        "name": "Bluetooth Version",
        "value": "5.3",
        "unit": null
      },
      {
        "name": "Battery Life",
        "value": "30",
        "unit": "hours"
      },
      {
        "name": "Charging Time",
        "value": "2",
        "unit": "hours"
      },
      {
        "name": "Frequency Response",
        "value": "20Hz - 20kHz",
        "unit": null
      }
    ],
    "physical": [
      {
        "name": "Weight",
        "value": "250",
        "unit": "g"
      },
      {
        "name": "Dimensions",
        "value": "18 x 15 x 8",
        "unit": "cm"
      }
    ]
  },
  
  // Geographic Availability
  "geographic_markets": {
    "available_in_countries": ["NL", "DE", "BE", "FR", "UK"],
    "restricted_countries": ["US", "CN"],  // Export restrictions
    "shipping_zones": [
      {
        "zone_id": "zone_benelux",
        "countries": ["NL", "BE", "LU"],
        "shipping_cost": 0,
        "free_shipping_threshold": 0,
        "delivery_time_min": 1,
        "delivery_time_max": 2,
        "delivery_time_unit": "days"
      },
      {
        "zone_id": "zone_germany",
        "countries": ["DE"],
        "shipping_cost": 5.99,
        "free_shipping_threshold": 50,
        "delivery_time_min": 2,
        "delivery_time_max": 4,
        "delivery_time_unit": "days"
      }
    ]
  },
  
  // SEO & Marketing
  "seo": {
    "meta_title": {
      "en": "Premium Wireless Headphones | AudioTech | GlobalMarket",
      "nl": "Premium Draadloze Koptelefoon | AudioTech | GlobalMarket",
      "ar": "سماعات لاسلكية متميزة | AudioTech | GlobalMarket"
    },
    "meta_description": {
      "en": "Shop premium wireless headphones with 30h battery life...",
      "nl": "Koop premium draadloze koptelefoon met 30u batterijduur...",
      "ar": "تسوق سماعات لاسلكية متميزة مع بطارية 30 ساعة..."
    },
    "keywords": ["wireless headphones", "bluetooth", "noise cancelling", "audiotech"],
    "canonical_url": "https://globalmarket.eu/products/premium-wireless-headphones"
  },
  
  // Analytics & Performance Metrics
  "metrics": {
    "views_total": 1250,
    "views_last_30_days": 450,
    "sales_total": 45,
    "sales_last_30_days": 12,
    "revenue_total": 13499.55,
    "revenue_last_30_days": 3599.88,
    "conversion_rate": 0.036,
    "cart_add_rate": 0.15,
    "wishlist_add_count": 78,
    
    // Reviews & Ratings
    "rating_average": 4.7,
    "rating_count": 32,
    "rating_distribution": {
      "5": 20,
      "4": 8,
      "3": 3,
      "2": 1,
      "1": 0
    },
    
    // Buy Box Performance
    "buy_box_win_rate": 0.85,  // 85% of time
    "buy_box_last_updated": "2025-11-28T10:30:00Z"
  },
  
  // Quality Score (for Buy Box)
  "quality_score": {
    "overall": 4.7,
    "components": {
      "product_rating": 4.7,
      "vendor_rating": 4.8,
      "delivery_time": 4.5,
      "customer_service": 4.9,
      "return_rate": 0.02
    },
    "calculated_at": "2025-11-28T00:00:00Z"
  },
  
  // Status & Visibility
  "status": "published",  // draft, published, archived, deleted
  "visibility": "public",  // public, private, hidden
  "is_featured": true,
  "is_on_sale": true,
  "is_new_arrival": false,
  "is_best_seller": true,
  
  // Compliance & Legal
  "compliance": {
    "ce_marked": true,
    "rohs_compliant": true,
    "weee_registered": true,
    "energy_label": null,
    "hazardous_materials": false,
    "age_restriction": null
  },
  
  // Timestamps
  "created_at": "2025-01-15T10:30:00Z",
  "updated_at": "2025-11-28T09:15:00Z",
  "published_at": "2025-01-15T12:00:00Z",
  "deleted_at": null,
  
  // Partition Key (Cosmos DB)
  "_partitionKey": "eu-west-1/vendor_abc123",
  
  // TTL (Time To Live - optional)
  "ttl": null,  // null = never expire
  
  // Version Control
  "version": 5,
  "last_modified_by": "admin_user_123"
}
```

### 2.3 Orders Collection

```javascript
// Collection: orders
// Partition Key: /tenant_id/user_id
// RU/s: 6000 (auto-scale to 60,000)

{
  // Identifiers
  "id": "order_01HGW6YN8O3L9Q0R5S6T7U8V9X",
  "tenant_id": "eu-west-1",
  "user_id": "user_customer_456",
  
  // Order Info
  "order_number": "EU-2025-001234",
  "order_type": "standard",  // standard, express, gift
  "status": "processing",
  
  // Status Flow
  "status_history": [
    {
      "status": "pending",
      "timestamp": "2025-11-28T10:15:00Z",
      "note": "Order received",
      "updated_by": "system"
    },
    {
      "status": "payment_confirmed",
      "timestamp": "2025-11-28T10:16:00Z",
      "note": "Payment confirmed via iDEAL",
      "updated_by": "payment_gateway"
    },
    {
      "status": "processing",
      "timestamp": "2025-11-28T10:20:00Z",
      "note": "Order being prepared",
      "updated_by": "vendor_abc123"
    }
  ],
  
  // Customer Information
  "customer": {
    "id": "user_customer_456",
    "email": "john.doe@example.com",
    "first_name": "John",
    "last_name": "Doe",
    "phone": "+31612345678",
    "customer_type": "registered",  // guest, registered
    "customer_since": "2024-05-15T00:00:00Z"
  },
  
  // Shipping Address
  "shipping_address": {
    "id": "addr_123",
    "recipient_name": "John Doe",
    "company": null,
    "street_line1": "Kalverstraat 123",
    "street_line2": "Apartment 4B",
    "city": "Amsterdam",
    "state": "Noord-Holland",
    "postal_code": "1012 AB",
    "country": "NL",
    "country_name": "Netherlands",
    "phone": "+31612345678",
    "delivery_instructions": "Please ring doorbell twice"
  },
  
  // Billing Address
  "billing_address": {
    "id": "addr_124",
    "name": "John Doe",
    "company": null,
    "street_line1": "Kalverstraat 123",
    "street_line2": "Apartment 4B",
    "city": "Amsterdam",
    "state": "Noord-Holland",
    "postal_code": "1012 AB",
    "country": "NL",
    "country_name": "Netherlands",
    "phone": "+31612345678"
  },
  
  // Order Items (can be from multiple vendors)
  "items": [
    {
      "id": "item_1",
      "vendor_id": "vendor_abc123",
      "vendor_name": "Tech Store Amsterdam",
      
      // Product Info
      "product_id": "prod_01HGW5XM7N2K8P9Q4R5S6T7V8W",
      "product_name": "Premium Wireless Headphones",
      "product_slug": "premium-wireless-headphones",
      "sku": "HDPH-WL-PRO-001-BLK",
      "variant": "Black",
      
      // Pricing
      "quantity": 1,
      "unit_price": 299.99,
      "subtotal": 299.99,
      "discount": 0,
      "tax_rate": 0.21,  // 21% VAT
      "tax_amount": 62.99,
      "total": 362.98,
      
      // Media
      "image_url": "https://cdn.globalmarket.eu/products/thumbs/hdph-wl-001-1.jpg",
      
      // Fulfillment
      "fulfillment_status": "pending",  // pending, processing, shipped, delivered
      "tracking_number": null,
      "shipped_at": null,
      "delivered_at": null
    }
  ],
  
  // Order Totals
  "totals": {
    "subtotal": 299.99,
    "shipping_cost": 0,
    "shipping_discount": 0,
    "tax": 62.99,
    "discount_code": null,
    "discount_amount": 0,
    "total": 362.98,
    "currency": "EUR"
  },
  
  // Payment Information
  "payment": {
    "method": "ideal",  // ideal, card, paypal, banktransfer, cod
    "method_title": "iDEAL",
    "status": "paid",  // pending, paid, failed, refunded, partially_refunded
    "
    "provider": "mollie",
    "transaction_id": "tr_WDqYK6vllg",
    "payment_details": {
      "bank": "ABN AMRO",
      "account_holder": "John Doe"
    },
    "paid_at": "2025-11-28T10:16:00Z",
    "refunded_amount": 0,
    "refund_history": []
  },
  
  // Shipping Information
  "shipping": {
    "method": "standard",
    "method_title": "Standard Shipping (1-2 days)",
    "carrier": "PostNL",
    "carrier_service": "Pakket",
    "tracking_number": "3SABCD1234567890",
    "tracking_url": "https://postnl.nl/tracktrace/?B=3SABCD1234567890",
    "estimated_delivery_date": "2025-11-30",
    "shipped_at": null,
    "delivered_at": null,
    "delivery_confirmation": null
  },
  
  // Notes & Communication
  "customer_note": "Please call before delivery",
  "internal_note": "VIP customer - priority handling",
  "gift_message": null,
  
  // Timeline Events
  "timeline": [
    {
      "event": "order_placed",
      "timestamp": "2025-11-28T10:15:00Z",
      "actor": "customer",
      "note": "Order placed by customer"
    },
    {
      "event": "payment_received",
      "timestamp": "2025-11-28T10:16:00Z",
      "actor": "system",
      "note": "Payment confirmed via iDEAL"
    },
    {
      "event": "order_confirmed",
      "timestamp": "2025-11-28T10:20:00Z",
      "actor": "vendor",
      "note": "Vendor confirmed order"
    }
  ],
  
  // Marketing & Attribution
  "marketing": {
    "utm_source": "google",
    "utm_medium": "cpc",
    "utm_campaign": "black_friday_2025",
    "referrer": "https://google.com",
    "landing_page": "https://globalmarket.eu/products/premium-wireless-headphones"
  },
  
  // Fraud Detection
  "fraud_analysis": {
    "risk_score": 0.15,  // 0-1 (lower is better)
    "risk_level": "low",  // low, medium, high
    "flags": [],
    "checked_at": "2025-11-28T10:15:30Z"
  },
  
  // Returns & Refunds
  "returns": {
    "eligible_until": "2025-12-28T23:59:59Z",  // 30 days
    "return_requested": false,
    "return_approved": false,
    "return_tracking": null
  },
  
  // Timestamps
  "created_at": "2025-11-28T10:15:00Z",
  "updated_at": "2025-11-28T10:20:00Z",
  "completed_at": null,
  "cancelled_at": null,
  
  // Partition Key
  "_partitionKey": "eu-west-1/user_customer_456",
  
  // TTL - Archive after 2 years
  "ttl": 63072000  // 2 years in seconds
}
```

### 2.4 Users Collection

```javascript
// Collection: users
// Partition Key: /tenant_id/user_id
// RU/s: 2000 (auto-scale to 20,000)

{
  // Identifiers
  "id": "user_customer_456",
  "tenant_id": "eu-west-1",
  "user_type": "customer",  // customer, vendor, admin
  
  // Authentication
  "email": "john.doe@example.com",
  "email_verified": true,
  "email_verified_at": "2024-05-15T12:30:00Z",
  "password_hash": "$2b$12$KIXXr5fQY...",  // bcrypt
  "password_changed_at": "2024-05-15T12:00:00Z",
  
  // Personal Information
  "first_name": "John",
  "last_name": "Doe",
  "phone": "+31612345678",
  "phone_verified": true,
  "date_of_birth": "1990-05-15",
  "gender": "male",
  
  // Addresses
  "addresses": [
    {
      "id": "addr_123",
      "type": "home",
      "is_default": true,
      "recipient_name": "John Doe",
      "street_line1": "Kalverstraat 123",
      "street_line2": "Apartment 4B",
      "city": "Amsterdam",
      "state": "Noord-Holland",
      "postal_code": "1012 AB",
      "country": "NL",
      "phone": "+31612345678"
    }
  ],
  
  // Preferences
  "preferences": {
    "language": "nl",
    "currency": "EUR",
    "timezone": "Europe/Amsterdam",
    "newsletter_subscribed": true,
    "sms_notifications": false,
    "push_notifications": true
  },
  
  // Profile
  "avatar_url": "https://cdn.globalmarket.eu/avatars/user_456.jpg",
  "bio": null,
  
  // Customer Metrics
  "metrics": {
    "total_orders": 15,
    "total_spent": 4567.89,
    "average_order_value": 304.52,
    "lifetime_value": 4567.89,
    "first_order_date": "2024-06-10T14:20:00Z",
    "last_order_date": "2025-11-28T10:15:00Z",
    "loyalty_points": 456
  },
  
  // Segmentation
  "segments": ["high_value", "repeat_customer", "newsletter_subscriber"],
  
  // Status
  "status": "active",  // active, suspended, deleted
  "is_verified": true,
  
  // Timestamps
  "created_at": "2024-05-15T12:00:00Z",
  "updated_at": "2025-11-28T10:15:00Z",
  "last_login_at": "2025-11-28T09:00:00Z",
  
  // Partition Key
  "_partitionKey": "eu-west-1/user_customer_456"
}
```

### 2.5 Reviews Collection

```javascript
// Collection: reviews
// Partition Key: /tenant_id/product_id
// RU/s: 1000 (auto-scale to 10,000)

{
  "id": "review_01HGX1YZ2P4Q5R6S7T8U9V0W1X",
  "tenant_id": "eu-west-1",
  "product_id": "prod_01HGW5XM7N2K8P9Q4R5S6T7V8W",
  "vendor_id": "vendor_abc123",
  
  // Reviewer
  "user_id": "user_customer_789",
  "user_name": "Sarah M.",
  "user_avatar": "https://cdn.globalmarket.eu/avatars/user_789.jpg",
  "verified_purchase": true,
  "order_id": "order_01HGW6YN8O3L9Q0R5S6T7U8V9X",
  
  // Review Content
  "rating": 5,
  "title": "Amazing sound quality!",
  "comment": "These headphones exceeded my expectations. The sound is crystal clear...",
  "language": "en",
  
  // Media
  "images": [
    "https://cdn.globalmarket.eu/reviews/review_123_1.jpg",
    "https://cdn.globalmarket.eu/reviews/review_123_2.jpg"
  ],
  "videos": [],
  
  // Helpful Votes
  "helpful_count": 12,
  "not_helpful_count": 1,
  
  // Vendor Response
  "vendor_response": {
    "comment": "Thank you for your positive feedback!",
    "responded_at": "2025-11-29T10:00:00Z",
    "responded_by": "vendor_abc123"
  },
  
  // Moderation
  "status": "approved",  // pending, approved, rejected, flagged
  "moderation_note": null,
  "moderated_by": "admin_user_001",
  "moderated_at": "2025-11-28T12:00:00Z",
  
  // Timestamps
  "created_at": "2025-11-28T11:30:00Z",
  "updated_at": "2025-11-28T12:00:00Z",
  
  "_partitionKey": "eu-west-1/prod_01HGW5XM7N2K8P9Q4R5S6T7V8W"
}
```

---

<a name="postgresql-schema"></a>
## 🐘 3. PostgreSQL Schema

### 3.1 Schema Overview

```sql
-- Database: globalmarket_financials
-- Purpose: ACID-compliant transactional data

-- Schemas:
CREATE SCHEMA IF NOT EXISTS eu_tenant;
CREATE SCHEMA IF NOT EXISTS mena_tenant;
CREATE SCHEMA IF NOT EXISTS shared;
```

### 3.2 Vendor Payouts Table

```sql
-- Table: vendor_payouts
-- Purpose: Track vendor payments and commissions

CREATE TABLE shared.vendor_payouts (
    -- Primary Key
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- Tenant & Vendor
    tenant_id VARCHAR(50) NOT NULL,
    vendor_id VARCHAR(50) NOT NULL,
    
    -- Payout Information
    payout_number VARCHAR(50) UNIQUE NOT NULL,
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    
    -- Financial Amounts
    total_orders INTEGER NOT NULL DEFAULT 0,
    gross_sales DECIMAL(12, 2) NOT NULL DEFAULT 0.00,
    returns_amount DECIMAL(12, 2) NOT NULL DEFAULT 0.00,
    net_sales DECIMAL(12, 2) NOT NULL DEFAULT 0.00,
    
    -- Commission Calculation
    commission_rate DECIMAL(5, 2) NOT NULL,  -- 0.15 = 15%
    commission_amount DECIMAL(12, 2) NOT NULL,
    
    -- Additional Fees
    transaction_fees DECIMAL(12, 2) NOT NULL DEFAULT 0.00,
    shipping_fees DECIMAL(12, 2) NOT NULL DEFAULT 0.00,
    adjustment_amount DECIMAL(12, 2) NOT NULL DEFAULT 0.00,
    
    -- Final Payout
    payout_amount DECIMAL(12, 2) NOT NULL,
    currency VARCHAR(3) NOT NULL DEFAULT 'EUR',
    
    -- Payment Method
    payment_method VARCHAR(20) NOT NULL,  -- bank_transfer, paypal, stripe
    bank_account_last4 VARCHAR(4),
    bank_name VARCHAR(100),
    paypal_email VARCHAR(255),
    
    -- Status
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    -- Statuses: pending, processing, paid, failed, cancelled
    
    -- Payment Reference
    payment_reference VARCHAR(255),  -- Bank transaction ID
    payment_proof_url TEXT,  -- Receipt upload
    
    -- Notes
    internal_note TEXT,
    vendor_note TEXT,
    
    -- Timestamps
    scheduled_at TIMESTAMP,
    processed_at TIMESTAMP,
    paid_at TIMESTAMP,
    failed_at TIMESTAMP,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    -- Foreign Keys
    CONSTRAINT fk_vendor_payouts_tenant 
        FOREIGN KEY (tenant_id) 
        REFERENCES shared.tenants(id) 
        ON DELETE RESTRICT,
    
    -- Indexes
    CONSTRAINT chk_payout_amount_positive 
        CHECK (payout_amount >= 0)
);

-- Indexes for Performance
CREATE INDEX idx_vendor_payouts_tenant 
    ON shared.vendor_payouts(tenant_id);

CREATE INDEX idx_vendor_payouts_vendor 
    ON shared.vendor_payouts(vendor_id);

CREATE INDEX idx_vendor_payouts_status 
    ON shared.vendor_payouts(status);

CREATE INDEX idx_vendor_payouts_period 
    ON shared.vendor_payouts(period_start, period_end);

CREATE INDEX idx_vendor_payouts_scheduled 
    ON shared.vendor_payouts(scheduled_at) 
    WHERE status = 'pending';

-- Composite Index for Common Queries
CREATE INDEX idx_vendor_payouts_vendor_period 
    ON shared.vendor_payouts(vendor_id, period_start DESC);

-- Trigger for updated_at
CREATE TRIGGER update_vendor_payouts_updated_at
    BEFORE UPDATE ON shared.vendor_payouts
    FOR EACH ROW
    EXECUTE FUNCTION shared.update_updated_at_column();
```

### 3.3 Financial Transactions Table

```sql
-- Table: financial_transactions
-- Purpose: Complete audit trail of all financial movements

CREATE TABLE shared.financial_transactions (
    -- Primary Key
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- Tenant
    tenant_id VARCHAR(50) NOT NULL,
    
    -- Transaction Type
    transaction_type VARCHAR(30) NOT NULL,
    -- Types: order_payment, order_refund, vendor_payout, 
    --        platform_commission, shipping_fee, adjustment
    
    -- Reference to Source
    reference_type VARCHAR(30) NOT NULL,  -- order, payout, adjustment
    reference_id VARCHAR(100) NOT NULL,
    
    -- Parties Involved
    from_party_type VARCHAR(20),  -- customer, vendor, platform, gateway
    from_party_id VARCHAR(50),
    to_party_type VARCHAR(20),
    to_party_id VARCHAR(50),
    
    -- Financial Details
    amount DECIMAL(12, 2) NOT NULL,
    currency VARCHAR(3) NOT NULL DEFAULT 'EUR',
    exchange_rate DECIMAL(10, 6),  -- If currency conversion
    amount_base_currency DECIMAL(12, 2),  -- Converted to base currency
    
    -- Payment Method
    payment_method VARCHAR(30),  -- ideal, card, paypal, bank_transfer
    payment_provider VARCHAR(30),  -- mollie, stripe, paypal
    payment_reference VARCHAR(255),  -- External transaction ID
    
    -- Status
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    -- Statuses: pending, completed, failed, cancelled, refunded
    
    -- Metadata
    description TEXT,
    metadata JSONB,  -- Additional flexible data
    
    -- Accounting
    debit_account VARCHAR(50),  -- Chart of accounts reference
    credit_account VARCHAR(50),
    
    -- Timestamps
    transaction_date TIMESTAMP NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMP,
    failed_at TIMESTAMP,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    -- Foreign Keys
    CONSTRAINT fk_financial_transactions_tenant 
        FOREIGN KEY (tenant_id) 
        REFERENCES shared.tenants(id) 
        ON DELETE RESTRICT,
    
    -- Constraints
    CONSTRAINT chk_transaction_amount_not_zero 
        CHECK (amount != 0)
);

-- Indexes
CREATE INDEX idx_financial_transactions_tenant 
    ON shared.financial_transactions(tenant_id);

CREATE INDEX idx_financial_transactions_reference 
    ON shared.financial_transactions(reference_type, reference_id);

CREATE INDEX idx_financial_transactions_date 
    ON shared.financial_transactions(transaction_date DESC);

CREATE INDEX idx_financial_transactions_type 
    ON shared.financial_transactions(transaction_type);

CREATE INDEX idx_financial_transactions_status 
    ON shared.financial_transactions(status);

CREATE INDEX idx_financial_transactions_parties 
    ON shared.financial_transactions(from_party_id, to_party_id);

-- JSONB Index for Metadata Queries
CREATE INDEX idx_financial_transactions_metadata 
    ON shared.financial_transactions USING GIN (metadata);
```

### 3.4 Audit Logs Table

```sql
-- Table: audit_logs
-- Purpose: Complete audit trail of all system actions

CREATE TABLE shared.audit_logs (
    -- Primary Key (BIGSERIAL for high volume)
    id BIGSERIAL PRIMARY KEY,
    
    -- Tenant
    tenant_id VARCHAR(50) NOT NULL,
    
    -- Actor Information
    user_id VARCHAR(50),
    user_email VARCHAR(255),
    user_type VARCHAR(20),  -- customer, vendor, admin, system
    user_role VARCHAR(50),
    
    -- Request Information
    ip_address INET,
    user_agent TEXT,
    request_id VARCHAR(100),  -- For tracing
    session_id VARCHAR(100),
    
    -- Action Details
    action VARCHAR(100) NOT NULL,
    -- Examples: create_product, update_order, delete_user, login, logout
    
    resource_type VARCHAR(50) NOT NULL,  -- product, order, user, etc.
    resource_id VARCHAR(50),
    
    -- Changes (for updates)
    old_values JSONB,
    new_values JSONB,
    changed_fields TEXT[],  -- Array of field names
    
    -- Result
    status VARCHAR(20) NOT NULL DEFAULT 'success',
    -- Statuses: success, failed, unauthorized, error
    
    error_message TEXT,
    
    -- Context
    description TEXT,
    metadata JSONB,
    
    -- Timestamp
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    -- Foreign Keys
    CONSTRAINT fk_audit_logs_tenant 
        FOREIGN KEY (tenant_id) 
        REFERENCES shared.tenants(id) 
        ON DELETE RESTRICT
);

-- Indexes for Fast Queries
CREATE INDEX idx_audit_logs_tenant 
    ON shared.audit_logs(tenant_id);

CREATE INDEX idx_audit_logs_user 
    ON shared.audit_logs(user_id);

CREATE INDEX idx_audit_logs_action 
    ON shared.audit_logs(action);

CREATE INDEX idx_audit_logs_resource 
    ON shared.audit_logs(resource_type, resource_id);

CREATE INDEX idx_audit_logs_date 
    ON shared.audit_logs(created_at DESC);

CREATE INDEX idx_audit_logs_status 
    ON shared.audit_logs(status) 
    WHERE status != 'success';

-- Composite Index for Common Queries
CREATE INDEX idx_audit_logs_user_date 
    ON shared.audit_logs(user_id, created_at DESC);

-- JSONB Indexes
CREATE INDEX idx_audit_logs_old_values 
    ON shared.audit_logs USING GIN (old_values);

CREATE INDEX idx_audit_logs_new_values 
    ON shared.audit_logs USING GIN (new_values);

CREATE INDEX idx_audit_logs_metadata 
    ON shared.audit_logs USING GIN (metadata);

-- Partitioning by Month (for scalability)
-- This is optional but recommended for high-volume logs

-- Create partitions
CREATE TABLE shared.audit_logs_2025_11 PARTITION OF shared.audit_logs
    FOR VALUES FROM ('2025-11-01') TO ('2025-12-01');

CREATE TABLE shared.audit_logs_2025_12 PARTITION OF shared.audit_logs
    FOR VALUES FROM ('2025-12-01') TO ('2026-01-01');

-- Auto-create partitions with a stored procedure
```

---

**🎯 الجزء الأول تم!**

رفعت:
- ✅ Database Strategy Overview
- ✅ Complete Cosmos DB Schema (Products, Orders, Users, Reviews)
- ✅ PostgreSQL Schema (Vendor Payouts, Financial Transactions, Audit Logs)

**باقي:** Redis, Elasticsearch, Relationships, Indexing Strategy, Partitioning, Migration Strategy

بدك أكمل الباقي بنفس المستوى التفصيلي؟ 🚀