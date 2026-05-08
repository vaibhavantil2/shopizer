# Business Features & Capabilities  

*Prepared from a review of the Shopizer source tree (core model, modules, and documentation).*  

---  

## 1. Platform Overview  
| Aspect | Details |
|--------|---------|
| **What it is** | A Java‑based, open‑source, headless e‑commerce platform that exposes a full REST API (Swagger 3.0.1) for all commerce operations. |
| **Technology stack** | Spring Boot 2.5.x, Java 11, Maven, Elasticsearch 7.5, Infinispan cache, Docker‑ready images, Angular admin UI, React shop UI. |
| **Target users** | • Independent merchants & SMBs that need a customizable online store.<br>• Enterprises that want a multi‑store or marketplace architecture.<br>• Developers building headless or API‑first commerce solutions. |
| **Deployment options** | Stand‑alone JAR, Docker containers (`shopizerecomm/shopizer`, `shopizer-admin`, `shopizer-shop-reactjs`), Kubernetes‑compatible. |

---  

## 2. Core Commerce Features  

| Feature | Code Evidence | Business Value |
|---------|----------------|----------------|
| **Catalog Management** | `sm-core-model/src/main/java/com/salesmanager/core/model/catalog/catalog/Catalog.java`<br>`Category`, `CategoryDescription`, `CatalogCategoryEntry` | Create multiple catalogs, organize categories hierarchically, support multilingual descriptions. |
| **Product Management** | `Product.java`, `ProductDescription.java`, `ProductAttribute.java`, `ProductOption*` classes | Full product definition (SKU, condition, dimensions, digital files, manufacturer). |
| **Product Variants & Options** | `ProductVariant`, `ProductVariantGroup`, `ProductOption`, `ProductOptionValue`, `ProductOptionSet` | Define configurable products (size, colour, etc.) with separate inventory & pricing per variant. |
| **Pricing Engine** | `ProductPrice`, `ProductPriceDescription`, `FinalPrice`, `ProductPriceType` | Multiple price types (default, sale, special), price per region/currency, automatic final‑price calculation. |
| **Inventory & Availability** | `ProductInventory`, `ProductAvailability` | Real‑time stock tracking, back‑order handling, inventory per warehouse. |
| **Digital Products** | `DigitalProduct` (file handling), `ProductImage`, `ProductImageDescription` | Sell downloadable files, manage image assets, support multiple image sizes. |
| **Product Relationships** | `ProductRelationship`, `ProductRelationshipType` | Upsell, cross‑sell, related product linking. |
| **Product Reviews** | `ProductReview`, `ProductReviewDescription` | Collect and display customer feedback, moderation via admin UI. |
| **Manufacturer Management** | `Manufacturer`, `ManufacturerDescription` | Store brand information, link products to manufacturers. |
| **Product Types** | `ProductType`, `ProductTypeDescription` | Differentiate between physical, digital, service, rental products. |
| **Search** | `shopizer.search.version` (Elasticsearch 7.5.2) integration, `sm-core-modules` contains search service beans | Full‑text, faceted, and filtered product search with autocomplete. |

---  

## 3. Customer Management  

| Feature | Code Evidence | Business Value |
|---------|----------------|----------------|
| **Customer Registration & Authentication** | `Customer.java`, `CustomerCriteria`, `CredentialsReset` | Secure account creation, password reset workflow, criteria‑based lookup. |
| **Customer Profiles** | `CustomerAttribute`, `Address`, `Billing`, `Delivery` | Store multiple addresses, billing info, delivery preferences. |
| **Customer Groups & Segmentation** | (via `CustomerCriteria` filters) | Targeted promotions, pricing, and content. |
| **Reviews & Ratings** | `ProductReview*` classes (linked to `Customer`) | Enable social proof, improve SEO. |
| **Opt‑in / Newsletter** | `Customer` contains `optIn` flag (checked in registration flow) | GDPR‑compliant consent for marketing communications. |
| **User Context & Auditing** | `UserContext`, `Auditable`, `AuditSection` | Track who performed actions, useful for admin audit logs. |

---  

## 4. Order Processing  

| Feature | Code Evidence | Business Value |
|---------|----------------|----------------|
| **Shopping Cart** | `Cart` (in `sm-core-modules`), `CartItem` | Persistent cart across sessions, supports promotions & coupons. |
| **Checkout Workflow** | `Order`, `OrderItem`, `OrderStatus` (in `sm-core-model`) | Multi‑step checkout (address, shipping, payment) with state machine handling. |
| **Order Lifecycle** | `OrderStatus` enum (e.g., `PENDING`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`) | Clear visibility for merchants & customers; triggers notifications. |
| **Order History** | `Customer` → `Order` relationship, REST endpoint `/orders` | Customers can view past orders, re‑order items. |
| **Promotions & Coupons** | `Promotion`, `Coupon` (in modules) | Apply discounts, free shipping, or gift‑with‑purchase. |
| **Returns & Refunds** | `OrderItem` contains `returnStatus` fields | Manage post‑sale service. |

---  

## 5. Payment Processing  

| Gateway | Integration Evidence | Notes |
|---------|----------------------|-------|
| **Stripe** | `stripe` module under `sm-core-modules` (dependency on `stripe-java` library) | Supports credit‑card payments, tokenization. |
| **PayPal** | `paypal` module (uses PayPal SDK) | Express Checkout & PayPal Payments Pro. |
| **Braintree** | `braintree` module (Braintree SDK) | Alternative credit‑card processor. |
| **BeanStream** | `beanstream` module (now Bambora) | Canadian payment gateway. |
| **Money Order** | `MoneyOrderPayment` class (custom implementation) | Offline payment option. |
| **Custom / Extensible** | `PaymentModule` interface in core modules | Merchants can plug additional gateways via Spring beans. |

All gateways are invoked from the **checkout service**, which creates a `Payment` entity, stores transaction IDs, and updates order status accordingly.

---  

## 6. Shipping & Tax  

| Feature | Code Evidence | Business Value |
|---------|----------------|----------------|
| **Shipping Providers** | `ShippingService` implementations for **UPS**, **USPS**, and a **CustomShipping** fallback. | Real‑time rate calculation, label generation. |
| **Shipping Options** | `ShippingOption`, `ShippingQuote` classes | Multiple methods (standard, express, free shipping) per order. |
| **Tax Calculation** | `TaxService` (uses `taxjar` or custom rules) | Automatic tax determination based on destination, product tax class. |
| **Free Shipping Rules** | Configurable thresholds in `ShippingConfiguration` | Incentivize larger orders. |
| **Multi‑Warehouse Support** | `ProductInventory` can be linked to a warehouse ID | Enables split shipments and accurate tax per warehouse. |

---  

## 7. Content Management  

| Feature | Code Evidence | Business Value |
|---------|----------------|----------------|
| **CMS Pages** | `Content`, `ContentDescription`, `ContentType` | Create static pages (About, FAQ, Terms) with multilingual support. |
| **Static Assets** | `ContentFile`, `ImageContentFile`, `StaticContentFile` | Store images, PDFs, CSS/JS files; served via CDN or S3. |
| **Digital Product Delivery** | `DigitalProduct` (file reference) | Secure download links after purchase. |
| **Blog / News** | `ContentPosition` (ordering) | Publish news articles, SEO‑friendly URLs. |
| **WYSIWYG Editor Integration** | Admin UI uses **TinyMCE** (Angular) | Non‑technical staff can edit content. |

---  

## 8. Multi‑Store / Multi‑Tenant  

| Feature | Code Evidence | Business Value |
|---------|----------------|----------------|
| **Merchant Store Hierarchy** | `MerchantStore` entity (in core modules) with `parentStore` relationship | Run several storefronts (different brands, locales) from a single installation. |
| **Marketplace Support** | `MarketPlace` and `MarketplaceCatalog` classes | Allow third‑party sellers to list products under a common catalog. |
| **Store‑Specific Config** | `StoreConfiguration`, `StoreSetting` (per‑store) | Independent payment, shipping, tax, and theme settings. |
| **Domain Mapping** | `StoreUrlResolver` bean | Map each store to its own domain/sub‑domain. |
| **Data Isolation** | Separate `store_id` foreign keys on most entities (Product, Order, Customer) | Guarantees tenant data segregation. |

---  

## 9. Email & Notifications  

| Feature | Code Evidence | Business Value |
|---------|----------------|----------------|
| **Email Templates** | `EmailTemplate`, `EmailTemplateType` (in core modules) | Customizable HTML templates for order confirmation, shipping updates, password reset, etc. |
| **SMTP Support** | `MailSender` bean (Spring `JavaMailSender`) reads `spring.mail.*` properties | Works with any standard SMTP server. |
| **AWS SES Integration** | `AwsSesMailSender` implementation (optional module) | Scalable, high‑deliverability email service. |
| **Event‑Driven Notifications** | `NotificationService` listens to order status changes, payment events | Automatic email/SMS push on key lifecycle events. |
| **Queueing** | Optional **Infinispan** or **RabbitMQ** integration for async email dispatch | Improves performance under high load. |

---  

## 10. Search  

| Feature | Code Evidence | Business Value |
|---------|----------------|----------------|
| **Elasticsearch Integration** | `shopizer.search.version` (2.11.1) dependency, `ElasticSearchService` bean | Fast, scalable full‑text search with faceting, suggestions, and relevance tuning. |
| **Synonyms & Analyzers** | `es-config` files in resources | Improves search accuracy for e‑commerce vocabularies. |
| **Indexing Pipeline** | `ProductIndexService` updates ES on product create/update/delete | Near‑real‑time product visibility. |
| **Search API** | `/search/products` endpoint (Swagger) | Front‑end can query with filters (category, price range, attributes). |

---  

## 11. Cloud Storage  

| Provider | Code Evidence | Business Value |
|----------|----------------|----------------|
| **Amazon S3** | `S3ContentRepository` implementation (uses AWS SDK) | Store product images, digital downloads, and CMS assets off‑site, with CDN caching. |
| **Google Cloud Storage** | `GcpContentRepository` (optional module) | Alternative cloud storage for regions where GCP is preferred. |
| **Configurable via `application.yml`** | `cloud.storage.provider` property | Switch storage backend without code changes. |
| **Automatic URL Generation** | `ContentUrlService` creates signed URLs for protected assets | Secure, time‑limited access to digital products. |

---  

### Summary  

Shopizer delivers a **complete, headless e‑commerce stack** that covers everything from catalog and inventory to multi‑store marketplaces, with extensible payment, shipping, and content capabilities. The platform’s modular Java architecture (core model, core modules, and shop module) makes it straightforward to customize or replace any component while keeping a solid, production‑ready foundation.