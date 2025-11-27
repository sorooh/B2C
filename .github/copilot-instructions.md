# GlobalMarket Empire (B2C) - AI Agent Instructions

## Project Overview
This is **GlobalMarket Empire** (codename: Borvat), a B2C e-commerce platform competing with Bol.com. The strategic focus is rescuing vendors rejected/expelled from Bol.com by offering lower fees (0-4% vs 3-12%) and easier onboarding.

## Architecture & Tech Stack

### Backend
- **Framework:** Django 5.0 + Django REST Framework
- **Database:** PostgreSQL (production), SQLite (development)
- **Cache:** Redis Cluster for sessions, products, and search results
- **Storage:** AWS S3 / CloudFlare R2 for product images

### Frontend
- **Customer Site:** Next.js 14 + React (www.globalmarket.com)
- **Vendor Dashboard:** Next.js (vendor.globalmarket.com) - separate micro-frontend
- **Admin Panel:** Django Admin (customized)
- **Styling:** Tailwind CSS + Shadcn/ui
- **State:** Zustand (cart) + TanStack Query (API)

### Infrastructure
- Multi-region deployment (EU-West primary, EU-Central secondary)
- Kubernetes auto-scaling (3-50 pods based on CPU/memory)
- CDN: CloudFlare global with WebP optimization
- Monitoring: Sentry + Prometheus + Grafana

## Key Components

### 1. **Vendor System** (`marketplace_integration/`)
- **Purpose:** Multi-marketplace integration hub - allows selling on Bol.com, Amazon, Kaufland, eBay simultaneously
- **Pattern:** Factory pattern for marketplace integrations (`MarketplaceIntegrationFactory`)
- **Models:** `MarketplaceConfig`, `MarketplaceAccount`, `ProductMarketplaceListing`, `MarketplaceOrder`
- **Integration classes:** `BolComIntegration`, `AmazonIntegration`, `KauflandIntegration`, `EbayIntegration`

### 2. **Product Management**
- **EAN/GTIN Integration:** Barcode-based product search (like Bol.com)
- **Image Processing:** Automated WebP conversion, multi-image upload
- **Variants:** Color/size variants with dynamic pricing
- **Categories:** Hierarchical category system

### 3. **Performance Scoring**
- **Vendor Rating:** 0-100 score based on fulfillment speed, customer satisfaction
- **Strikes System:** Penalty system for policy violations
- **Buy Box Competition:** Win/loss indicators for competing sellers

### 4. **Orders & Fulfillment**
- **Multi-step checkout:** Address → Delivery → Payment → Review
- **Payment:** Stripe Elements + Payment Intents (primary), iDEAL, PayPal support
- **Guest checkout:** No forced registration
- **Real-time sync:** Order status synced across all marketplaces

## Development Workflows

### Setup & Build
```bash
# Backend setup
python -m venv venv
source venv/bin/activate  # or venv\Scripts\activate on Windows
pip install -r requirements.txt
python manage.py migrate
python manage.py runserver

# Frontend setup (customer site)
cd frontend-customer
npm install
npm run dev

# Frontend setup (vendor dashboard)
cd frontend-vendor
npm install
npm run dev
```

### Database Sharding Strategy
- Geographic sharding: `DB_SHARD_EU_WEST`, `DB_SHARD_EU_CENTRAL`, `DB_SHARD_GLOBAL`
- Time-based partitioning for orders: `ORDERS_2025_Q1`, `ORDERS_2025_Q2`, etc.
- Read replicas for product queries (`read_replica_1`, `read_replica_2`)
- Database router in `settings.py` handles read/write splitting

### Caching Layers
- **3-level strategy:** In-memory → Redis → Database
- **Product cache:** 1 hour TTL (`products` Redis namespace)
- **Sessions:** 24 hour TTL (`sessions` Redis namespace)
- **Search results:** Cached in Redis (`SEARCH_RESULTS_CACHE`)

### Background Tasks (Celery)
- `process_vendor_products_bulk`: Handle 10,000+ product uploads asynchronously
- `generate_analytics_reports`: Vendor performance reports
- `send_marketing_emails_batch`: Batch email campaigns (100,000+)
- Broker: Redis cluster

## Project-Specific Conventions

### Naming
- API controllers: `controllers/<resource>Controller.py`
- Services: Suffixed with `Service` (e.g., `UserService`, `OrderService`)
- Marketplace integrations: Inherit from `BaseMarketplaceIntegration`

### Database Optimization
- **Composite indexes:** `idx_products_category_price`, `idx_products_vendor_active`
- **Full-text search:** `idx_products_search_vector` (PostgreSQL GIN index)
- **Use prefetch:** Always use `select_related()` and `prefetch_related()` for related objects

### API Design
- RESTful endpoints under `/api/`
- Customer APIs: `/api/products`, `/api/cart`, `/api/orders`
- Vendor APIs: `/api/vendor/products`, `/api/vendor/orders`, `/api/vendor/analytics`
- Marketplace sync: `/api/marketplace/sync`, `/api/marketplace/orders`

## Integration Points

### External APIs
- **Stripe:** Primary payment processor (`stripe.PaymentIntent`)
- **Marketplace APIs:** Bol.com Partner Plaza, Amazon SP-API, Kaufland API, eBay Trading API
- **Shipping:** PostNL, DHL integrations (TBD)
- **CDN:** CloudFlare R2 for image storage

### Authentication
- OAuth2 for marketplace integrations (Bol.com, Amazon, eBay)
- JWT tokens for API authentication
- Google/GitHub OAuth for customer/vendor login

## Critical Paths

### Vendor Onboarding Flow
1. Registration → Verification (24h target) → Store setup → Product upload → Order fulfillment
2. **Seller Rescue Program:** Expedited approval for Bol.com-rejected vendors (proof required)

### Customer Purchase Flow
1. Browse → Add to cart (real-time price/stock validation) → Checkout (guest or logged-in) → Payment → Order confirmation → Tracking

### Multi-Marketplace Sync
1. Product created in Borvat → Auto-sync to selected marketplaces (Bol.com, Amazon, etc.)
2. Order received from marketplace → Import to Borvat → Notify vendor → Update tracking → Sync status back

## Performance Targets

### Phase 1 (Months 1-3): 
- 500 vendors, 10K products, 10K daily visitors
- Infrastructure: 2x app servers, 1x DB, 1x Redis

### Phase 2 (Months 4-12):
- 5,000 vendors, 500K products, 100K daily visitors
- Infrastructure: 6x app servers (auto-scale), 1x master + 2x slave DBs, Redis cluster

### Phase 3 (Year 2):
- 25,000 vendors, 2M products, 500K daily visitors
- Infrastructure: 15x servers (multi-region), sharded DB, Elasticsearch, RabbitMQ

## Testing Strategy
- **Unit tests:** Django TestCase for models/views
- **Integration tests:** Test marketplace API integrations with mocked responses
- **Load testing:** Target 10,000 concurrent users (use Locust or k6)
- **E2E tests:** Cypress for customer journey testing

## References
- **Architecture:** See `SCALABLE_ARCHITECTURE.md` for scaling strategy
- **Roadmap:** See `IMPLEMENTATION_ROADMAP.md` for 12-week development plan
- **Blueprint:** See `PROJECT_BLUEPRINT.md` for complete project overview
- **Marketplace Integration:** See `MULTI_MARKETPLACE_INTEGRATION.md` for API integration details
