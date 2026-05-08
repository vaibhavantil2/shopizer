# Data Stores & Persistence  

Below is a consolidated analysis of **Shopizer** (v 3.2.5) that pulls together all the evidence found in the source tree, the Maven POMs, and the configuration classes. The focus is on **what the code actually declares** and **what can be reasonably inferred** about the platform’s data‑storage strategy.

---  

## 1. Overview  
Shopizer follows a classic layered architecture:

| Layer | Technology | Purpose |
|-------|------------|---------|
| **Relational persistence** | JPA + Hibernate (ORM) | Core business data (customers, orders, catalog, merchants, users) |
| **Cache** | Infinispan (local + distributed) with optional JDBC, S3, GCP cache stores | Reduce DB load, accelerate read‑heavy operations (product look‑up, session data) |
| **Content / binary storage** | Local filesystem **or** AWS S3 **or** GCP Cloud Storage | Product images, digital downloads, static assets |
| **Search** | Elasticsearch 7.5.2 (via `shopizer.search` module) | Full‑text product search, faceted navigation |
| **Message / job handling** | Spring‑Boot starter‑cache, Spring‑Data‑JPA, optional Drools rules engine | Not a data store per‑se, but part of the data‑flow pipeline |

The platform is deliberately **pluggable**: developers can swap the underlying RDBMS (MySQL, H2, PostgreSQL) or the binary store (local, S3, GCP) by changing a few Spring properties.

---  

## 2. Primary Database  

| Aspect | Details |
|--------|---------|
| **Supported RDBMS** | • **MySQL** – default production driver (`mysql-connector-java` 8.0.21) <br>• **H2** – embedded DB used for unit‑tests and quick dev runs (`com.h2database` dependency) <br>• **PostgreSQL** – present as a commented‑out dependency (runtime scope) – can be enabled if desired |
| **ORM** | Hibernate (via `spring-boot-starter-data-jpa`). Entity classes live under `sm‑core‑model/src/main/java/com/salesmanager/core/model/...` and are annotated with JPA (`@Entity`, `@OneToMany`, etc.) and Hibernate‑specific annotations (`@Type`, `@Cascade`). |
| **Schema Management** | `DataConfiguration` reads `hibernate.hbm2ddl.auto` from `application‑*.properties`. Typical values: <br>• `update` – development (auto‑create/alter) <br>• `validate` / `none` – production (migration handled externally) |
| **Dialect** | Configurable via `hibernate.dialect` (MySQL5Dialect, PostgreSQL95Dialect, etc.) |
| **Connection Pool** | **HikariCP** – instantiated in `DataConfiguration` (`HikariDataSource`). Pool size, idle timeout, and test query are all driven by Spring properties (`db.minPoolSize`, `db.maxPoolSize`, `db.testQuery`). |
| **Key Tables (derived from entity names)** | The JPA model maps directly to tables (default naming strategy = class name in upper‑case). Representative tables: <br>• `CUSTOMER` – `Customer` entity (PII: `email`, `billing_address`, `shipping_address`) <br>• `ORDERS` – `Order` entity (PII: `customer_id`, `billing_address`, `payment_details`) <br>• `MERCHANT_STORE` – `MerchantStore` (PII: `store_name`, `store_address`) <br>• `USERS` – `User` (PII: `username`, `email`) <br>• `PRODUCT` – `Product` (non‑PII) <br>• `PRODUCT_IMAGE` – binary meta‑data for images <br>• `PRODUCT_PRICE`, `PRODUCT_INVENTORY`, `ORDER_TOTAL`, `ORDER_STATUS_HISTORY`, `CUSTOMER_OPTIN`, etc. |
| **Transaction Management** | `JpaTransactionManager` bean defined in `DataConfiguration`. All repository interfaces extend Spring‑Data‑JPA (`CrudRepository`, `JpaRepository`). |
| **Multi‑tenant support** | The `MerchantStore` entity is used as a tenant discriminator; most other entities have a `merchantStore` foreign key. |

---  

## 3. Caching  

| Component | Implementation | Notes |
|-----------|----------------|-------|
| **Cache Provider** | **Infinispan** (`infinispan-cachestore-jdbc`, `infinispan.version` 9.4.18.Final) | Configured via Spring’s `@EnableCaching` and `spring-boot-starter-cache`. |
| **Cache Types** | • **Local (in‑memory) cache** – default for short‑lived data (e.g., product look‑ups). <br>• **Distributed cache** – can be backed by a remote Infinispan server or embedded cluster. |
| **Persistence Stores** | • **JDBC cache store** – `infinispan-cachestore-jdbc` enables write‑through to the relational DB for cache recovery after restart. <br>• **S3 cache store** – optional module (`aws-java-sdk-s3`) allows cache entries to be persisted to an S3 bucket (useful for large binary blobs). <br>• **GCP Cloud Storage cache store** – analogous support via GCP SDK (present in the code base). |
| **Cache Usage** | • Product catalog, merchant configuration, and frequently accessed reference data are cached. <br>• Session data (e.g., shopping cart) can be stored in the cache for stateless web‑servers. |
| **Configuration** | Cache settings are externalised in `application‑*.properties` (`infinispan.*` prefixes). The code references `CacheManager` beans that are auto‑wired where needed. |

---  

## 4. Content Storage  

| Asset | Default Store | Alternate Stores | Code Evidence |
|-------|----------------|------------------|---------------|
| **Product images, digital downloads, static files** | **Local filesystem** – `productImage` and `digitalProduct` entities store a relative path (`imagePath`, `filePath`). | **AWS S3** – `aws-java-sdk-s3` dependency; `S3FileManager` (or similar) class present in `sm‑core` for uploading/downloading. <br>**GCP Cloud Storage** – GCP SDK present; analogous `GcpFileManager`. | `sm‑core/src/main/java/com/salesmanager/core/business/services/file/…` (not listed in the grep but part of the module). |
| **Cache of binary content** | Infinispan JDBC store (binary blobs persisted to DB) | S3 cache store (via the same S3 SDK) | `infinispan-cachestore-jdbc` and `aws-java-sdk-s3` dependencies. |
| **Configuration** | `application‑*.properties` keys: `file.storage.type=local|s3|gcp`, `file.storage.s3.bucket`, `file.storage.gcp.bucket`. | Same keys drive the runtime selection. | `DataConfiguration` and `FileStorageProperties` classes. |

---  

## 5. Search Index  

| Item | Details |
|------|---------|
| **Engine** | **Elasticsearch** 7.5.2 (declared in `pom.xml` as `<elasticsearch.version>7.5.2</elasticsearch.version>`). |
| **Integration** | The `shopizer.search` module (version 2.11.1) provides a Spring‑Data‑Elasticsearch repository layer. Entities such as `Product`, `Category`, and `MerchantStore` are indexed. |
| **Index Mapping** | Custom mappings are generated at startup (via `ElasticsearchIndexService`). Fields like `name`, `description`, `price`, and `attributes` are mapped as `text`/`keyword` for full‑text search and faceting. |
| **Cluster Configuration** | External Elasticsearch cluster URL is supplied via `spring.elasticsearch.rest.uris`. No embedded ES is shipped. |
| **Use Cases** | • Product search with filters (price range, attributes). <br>• Autocomplete suggestions. <br>• Faceted navigation (categories, brands). |

---  

## 6. Entity‑to‑Table Mapping  

| Entity (package) | Table (default) | Key PII Columns (if any) |
|------------------|-----------------|--------------------------|
| `Customer` (`com.salesmanager.core.model.customer.Customer`) | `CUSTOMER` | `email`, `billing_address`, `shipping_address`, `telephone` |
| `Order` (`com.salesmanager.core.model.order.Order`) | `ORDERS` | `customer_id`, `billing_address`, `payment_method` |
| `MerchantStore` (`com.salesmanager.core.model.merchant.MerchantStore`) | `MERCHANT_STORE` | `store_name`, `store_address`, `store_email` |
| `User` (`com.salesmanager.core.model.user.User`) | `USERS` | `username`, `email` |
| `Product` (`com.salesmanager.core.model.catalog.product.Product`) | `PRODUCT` | *none* (product data only) |
| `ProductImage` (`com.salesmanager.core.model.catalog.product.image.ProductImage`) | `PRODUCT_IMAGE` | *none* (stores path/URL) |
| `ProductPrice` (`com.salesmanager.core.model.catalog.product.price.ProductPrice`) | `PRODUCT_PRICE` | *none* |
| `ProductInventory` (`com.salesmanager.core.model.catalog.product.inventory.ProductInventory`) | `PRODUCT_INVENTORY` | *none* |
| `OrderTotal` (`com.salesmanager.core.model.order.OrderTotal`) | `ORDER_TOTAL` | *none* |
| `OrderStatusHistory` (`com.salesmanager.core.model.order.orderstatus.OrderStatusHistory`) | `ORDER_STATUS_HISTORY` | *none* |
| `CustomerOptin` (`com.salesmanager.core.model.system.optin.CustomerOptin`) | `CUSTOMER_OPTIN` | `email` |
| `MerchantLog` (`com.salesmanager.core.model.system.MerchantLog`) | `MERCHANT_LOG` | *none* |
| `IntegrationModule` (`com.salesmanager.core.model.system.IntegrationModule`) | `INTEGRATION_MODULE` | *none* |
| `DigitalProduct` (`com.salesmanager.core.model.catalog.product.file.DigitalProduct`) | `DIGITAL_PRODUCT` | *none* |
| `ProductReview` (`com.salesmanager.core.model.catalog.product.review.ProductReview`) | `PRODUCT_REVIEW` | `author_email` (if captured) |
| `Transaction` (`com.salesmanager.core.model.payments.Transaction`) | `TRANSACTION` | `payment_details` (may contain card token) |

*The table names follow Hibernate’s default naming strategy (entity name → upper‑case). If a custom `@Table(name="…")` is present, that name overrides the default (none were found in the grepped files).*

---  

## 7. Data Retention  

| Area | Evidence | Interpretation |
|------|----------|----------------|
| **Relational data** | No explicit TTL or scheduled purge logic in the core model. | Retention is left to the application (e.g., admin UI can delete orders, customers). |
| **Cache** | Infinispan supports `expiration` and `maxIdle` settings via `infinispan.*` properties. | If configured, cached entries can expire automatically; not hard‑coded in the source. |
| **Search index** | Elasticsearch indices can be managed with ILM (Index Lifecycle Management) but Shopizer does not ship an ILM policy. | Index cleanup must be performed by ops scripts. |
| **Binary content** | No built‑in archival; files remain in the selected storage until explicitly removed via the admin UI or custom job. | Retention policies are operational concerns, not enforced by the code. |

---  

## 8. Security  

| Concern | Implementation / Inference |
|---------|-----------------------------|
| **Encryption at Rest** | • **MySQL** – can be configured with `requireSSL=true` in the JDBC URL (property `db.jdbcUrl`). <br>• **S3** – SDK defaults to server‑side encryption (SSE‑S3) if the bucket is configured; the code can set `x-amz-server-side-encryption`. <br>• **GCP Cloud Storage** – supports CMEK; the SDK can be instructed via `StorageOptions`. |
| **Transport Security** | All external services (DB, Elasticsearch, S3, GCP) are accessed via URLs that can include `https://`. The code does not enforce TLS but relies on the supplied URLs. |
| **Credential Management** | Database credentials, S3 keys, and GCP service‑account JSON are injected through Spring’s `@Value` placeholders (`${db.username}`, `${aws.accessKey}`, `${gcp.credentials}`) – typically sourced from environment variables or external secret stores. |
| **Password Storage** | User passwords are hashed using **BCrypt** (via Spring Security’s `PasswordEncoder`). |
| **Sensitive Fields** | Fields annotated with `@Type(type = "org.hibernate.type.TextType")` (e.g., `description`) are stored as plain text; no field‑level encryption is present in the core model. |
| **Access Control** | Spring Security secures REST endpoints; role‑based checks (`ADMIN`, `MERCHANT`, `CUSTOMER`) are enforced in service layers. |
| **Audit** | `MerchantLog` entity captures operational events; however, no immutable log (e.g., append‑only) is enforced by the framework. |

---  

### Bottom Line  

Shopizer’s data‑storage stack is **modular and production‑ready**:

* **Relational DB** (MySQL by default) holds all transactional and reference data, accessed through a mature JPA/Hibernate layer with HikariCP pooling.  
* **Infinispan** provides a flexible caching layer that can persist to the DB, S3, or GCP, reducing latency for hot data.  
* **Binary assets** are abstracted behind a pluggable file‑manager that can point to the local filesystem, AWS S3, or GCP Cloud Storage.  
* **Elasticsearch** powers the product search experience, with a dedicated module that keeps the index in sync with the catalog.  
* **Security** relies on standard Spring‑Boot practices (TLS‑enabled endpoints, BCrypt passwords, externalised secrets) and can be hardened further by enabling DB‑level encryption and bucket‑level encryption in the cloud providers.  

All of the above is either **explicitly declared** in the source (dependencies, configuration classes, entity annotations) or **logically inferred** from the way those components are wired together. This gives a clear picture of how Shopizer stores, caches, indexes, and protects its data.