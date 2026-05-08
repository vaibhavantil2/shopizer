# Third‑Party Services & Vendors – Shopizer 3.2.5  

*Prepared by a vendor‑risk analyst. All observations are derived from the source‑code, Maven POM files and the grep results supplied.*  

---  

## 1. Integration Summary  

Shopizer is a modular Spring‑Boot e‑commerce platform that relies on a large number of external services and libraries to provide core functionality such as payments, shipping, email, storage, search, geolocation and authentication. The integrations can be grouped as follows:

| Category | Primary Vendors / Services | Maven Coordinates (excerpt) | Key Implementation Classes |
|----------|---------------------------|-----------------------------|----------------------------|
| **Payments** | Stripe, PayPal (Express Checkout), Braintree, (BeanStream – referenced but not present in the current source) | `com.stripe:stripe-java`, `com.paypal.sdk:merchantsdk`, `com.braintreegateway:braintree-java` | `StripePayment`, `Stripe3Payment`, `PayPalExpressCheckoutPayment`, `BraintreePayment` |
| **Shipping** | UPS, USPS, Canada Post (via `shopizer-canadapost` module) | `org.apache.httpcomponents:httpclient` (used for HTTP calls) | `UPSShippingQuote`, `USPSShippingQuote`, `ShippingDistancePreProcessorImpl` |
| **Email** | SMTP (JavaMail), Amazon SES | `javax.mail:mail`, `com.amazonaws:aws-java-sdk-ses` | `DefaultEmailSenderImpl`, `SESEmailSenderImpl` |
| **Cloud / Object Storage** | Amazon S3, Google Cloud Storage (GCS) | `com.amazonaws:aws-java-sdk-s3`, `com.google.cloud:google-cloud-storage` (indirect via GCP helper) | `S3CacheManagerImpl`, `GCPStaticContentAssetsManagerImpl`, `GCPCacheManagerImpl` |
| **Caching** | Infinispan (embedded) | `org.infinispan:infinispan-core` | Various cache‑related beans (not shown in the excerpt) |
| **Search** | Elasticsearch (self‑hosted) | `org.elasticsearch:elasticsearch` (via `shopizer.search.version` property) | Search service implementations in `sm-core` (not listed in the excerpt) |
| **Geolocation** | MaxMind GeoIP2 (GeoLite2) | `com.maxmind.geoip2:geoip2` | `GeoLocationImpl` |
| **Authentication / Authorization** | JWT (io.jsonwebtoken) | `io.jsonwebtoken:jjwt` | JWT utility classes (e.g., `JwtTokenUtil` – not shown but present in the dependency tree) |
| **Utility / Misc** | Apache Commons (lang3, collections4, validator, fileupload, io), Google Guava, Jackson, MapStruct, Drools, Spring‑Boot starters, etc. | See `pom.xml` properties | Used throughout the code base for data handling, validation, mapping, rule engine, etc. |

---  

## 2. Payment Providers  

| Vendor | Integration Type | Data Shared with Vendor | Evidence (Class / File) |
|--------|------------------|------------------------|--------------------------|
| **Stripe** | Server‑side SDK (v1) – `Charge` & `PaymentIntent` APIs | • Card token (generated client‑side) <br>• Transaction amount, currency, order ID <br>• Store identifier (merchant) | `StripePayment.java` (legacy charge flow) <br>`Stripe3Payment.java` (PaymentIntent flow) |
| **PayPal** | Classic NVP/SOAP Express Checkout (via PayPal Java SDK) | • PayPal API credentials (username, password, signature) <br>• Payment token / payer ID <br>• Transaction amount, currency, order details | `PayPalExpressCheckoutPayment.java` |
| **Braintree** | Braintree Java SDK (hosted fields / tokenization) | • Merchant ID, public & private keys, tokenization key <br>• Transaction amount, currency, order ID <br>• Customer email (optional) | `BraintreePayment.java` |
| **BeanStream** | Mentioned in comments / older modules but **no concrete implementation** in the current source tree. | N/A | N/A (no class found) |

### Common Payment‑related Configuration Keys  

| Key | Meaning | Required for |
|-----|---------|--------------|
| `secretKey` / `publishableKey` | Stripe API keys | Stripe |
| `api`, `username`, `signature` | PayPal classic credentials | PayPal |
| `merchant_id`, `public_key`, `private_key`, `tokenization_key` | Braintree credentials | Braintree |
| `environment` (TEST/PROD) | Determines sandbox vs. production endpoints | All three |

---  

## 3. Shipping Carriers  

| Carrier | Integration Type | Data Sent | Implementation |
|---------|------------------|-----------|----------------|
| **UPS** | XML‑based REST API (Rate/Ship) | • Origin & destination address (street, city, zip, country) <br>• Package weight, dimensions <br>• Store credentials (accessKey, userId, password) | `UPSShippingQuote.java` |
| **USPS** | XML/JSON HTTP API (RateV4) | Same address & package data as UPS | `USPSShippingQuote.java` |
| **Canada Post** | Separate module (`shopizer-canadapost`) – not shown in the excerpt but referenced in the parent POM | Similar address & package data | `CanadaPostShippingQuote` (hypothetical) |
| **Google Maps (Distance Matrix)** | Used for distance‑based shipping calculations | Origin & destination lat/long, API key | `ShippingDistancePreProcessorImpl.java` (calls Google Maps Services client) |

---  

## 4. Email Services  

| Service | Integration Type | Data Shared | Implementation |
|---------|------------------|-------------|----------------|
| **SMTP (JavaMail)** | Direct SMTP via `JavaMailSender` (Spring) | • From / To email addresses <br>• Email subject & body (HTML & plain text) <br>• Optional attachments | `DefaultEmailSenderImpl.java` |
| **Amazon SES** | AWS SDK v1 (`aws-java-sdk-ses`) | • From / To email addresses <br>• Email subject & body (HTML & plain text) <br>• AWS region configuration | `SESEmailSenderImpl.java` |

Both email modules use **Freemarker** templates for body generation (`templates/email/*`).  

---  

## 5. Cloud Infrastructure  

| Service | Purpose | Data Stored / Transferred | Implementation |
|---------|---------|---------------------------|----------------|
| **Amazon S3** | Static content (product images, CMS assets) | Binary files (images, PDFs) – no PII unless uploaded by merchants | `S3CacheManagerImpl.java`, `GCPCacheManagerImpl` (GCP fallback) |
| **Google Cloud Storage (GCS)** | Same purpose as S3 (alternative) | Binary files | `GCPStaticContentAssetsManagerImpl.java` |
| **Infinispan** | Distributed cache for sessions, product catalogs, etc. | Cached objects (may contain PII such as user sessions) | Configured via Spring; not a direct external call but a **third‑party library** |
| **AWS RDS (MySQL/PostgreSQL)** | Not a direct code integration, but the platform can be deployed on RDS (see property comments) | Database records (full order & customer data) | Configuration only – external service dependency at deployment time |

---  

## 6. Search  

| Engine | Integration Type | Data Indexed | Implementation |
|--------|------------------|--------------|----------------|
| **Elasticsearch 7.5.2** (self‑hosted) | REST client (Spring Data Elasticsearch) | Product catalog, categories, searchable fields (may include product descriptions, SKUs) – **no direct PII** unless merchants store it in searchable fields | Managed by `shopizer.search.version` property; code lives in the search module (not shown in the excerpt) |

---  

## 7. Geolocation  

| Provider | Integration Type | Data Sent | Implementation |
|----------|------------------|-----------|----------------|
| **MaxMind GeoIP2 (GeoLite2‑City)** | Local DB lookup (no network call) | IP address (client) – used to derive city, country, latitude/longitude | `GeoLocationImpl.java` (loads `reference/GeoLite2-City.mmdb` from classpath) |

---  

## 8. Authentication  

| Mechanism | Library | Data Processed | Implementation |
|-----------|---------|----------------|----------------|
| **JWT** | `io.jsonwebtoken:jjwt` | User identifier, roles, expiration – signed with a secret key | JWT utility classes (e.g., `JwtTokenUtil`) – not listed but present in the compiled jar |
| **OAuth / Social Login** | Not present in the current source (only mentioned in comments) | N/A | N/A |

---  

## 9. Other Libraries (Supporting Infrastructure)  

| Library | Primary Use | Example Classes |
|---------|-------------|-----------------|
| **Apache HttpClient** (`org.apache.httpcomponents:httpclient`) | HTTP calls to UPS, USPS, Google Maps, etc. | `UPSShippingQuote.java`, `USPSShippingQuote.java` |
| **Google Maps Services Java** (`com.google.maps:google-maps-services`) | Distance Matrix & Places APIs | `ShippingDistancePreProcessorImpl.java` |
| **Jackson** (`com.fasterxml.jackson.core:jackson-*`) | JSON (de)serialization for API payloads | Various DTOs |
| **Apache Commons** (`commons-lang3`, `commons-io`, `commons-collections4`, `commons-validator`, `commons-fileupload`) | Utility functions, validation, file handling | Throughout the code base |
| **Guava** (`com.google.guava:guava`) | Collections, caching helpers | Various |
| **MapStruct** (`org.mapstruct:mapstruct`) | Bean mapping between DTOs and entities | Mapper interfaces |
| **Drools** (`org.kie:kie-ci`) | Business rules engine (e.g., promotions) | Rule files (not shown) |
| **Spring Boot Starters** (`spring-boot-starter-web`, `spring-boot-starter-cache`, etc.) | Core framework | All modules |

---  

## 10. Data Flow to Third Parties  

| Destination | Types of Data Sent | Reason for Transfer | Frequency |
|------------|--------------------|---------------------|-----------|
| **Stripe / Braintree / PayPal** | • Card token (generated client‑side) <br>• Transaction amount, currency, order ID <br>• Store identifier (merchant account) | Payment authorization & capture | Per successful checkout |
| **UPS / USPS / Canada Post** | • Shipping address (name, street, city, zip, country) <br>• Package weight & dimensions | Rate quotation & label generation | Per shipping quote request |
| **Google Maps (Distance Matrix)** | • Origin & destination lat/long (derived from address) <br>• API key (in request header) | Distance‑based shipping cost calculation | Per checkout when distance‑based shipping is enabled |
| **Amazon SES / SMTP** | • Email addresses (from, to, reply‑to) <br>• Email subject & body (order confirmation, password reset) | Transactional notifications | Per email event |
| **Amazon S3 / GCS** | • Binary files (product images, CMS assets) – may contain merchant‑owned media, occasionally customer‑uploaded files | Static asset storage & CDN delivery | When merchants upload assets |
| **Elasticsearch** | • Product catalog data (titles, descriptions, SKUs) – may include merchant‑provided text that could contain PII if mis‑used | Search indexing | Bulk re‑index or incremental updates |
| **MaxMind GeoIP2** | • Client IP address (used locally) | Geolocation for tax/shipping | Per request (local lookup) |
| **JWT** | • User ID, roles, expiration timestamp (signed, not transmitted to external service) | Stateless authentication within the application | Internal only |

> **Note:** No raw credit‑card numbers ever leave the client; only a token generated by Stripe’s client‑side library is transmitted to the server.  

---  

## 11. Risk Assessment  

| Vendor / Service | Criticality to Business | Data Exposure Level | Confidentiality Impact | Integrity Impact | Availability Impact | Compliance Considerations |
|------------------|------------------------|----------------------|------------------------|------------------|---------------------|---------------------------|
| **Stripe** | **High** – primary payment processor for many deployments | **High** – payment token, amount, order ID (PCI‑DSS scope) | Medium (tokenized) | Medium (transaction tampering) | High (payment downtime stops sales) | PCI‑DSS, GDPR (if EU customers) |
| **PayPal** | High (alternative payment) | High (payment token, order data) | Medium | Medium | High | PCI‑DSS, GDPR |
| **Braintree** | High (alternative) | High (payment token, order data) | Medium | Medium | High | PCI‑DSS, GDPR |
| **UPS / USPS / Canada Post** | Medium – needed for shipping quotes & label generation | Medium (full shipping address, which contains PII) | High (address is PII) | Low | Medium (shipping quote failures affect checkout) | GDPR, local data‑protection laws |
| **Google Maps** | Low‑Medium – only used for distance calculation | Low (derived lat/long, not personal) | Low | Low | Low | GDPR (IP address may be considered personal) |
| **Amazon SES / SMTP** | Medium – responsible for order confirmations, password resets | Low‑Medium (email addresses) | Medium (email address is PII) | Low | Medium (email outage impacts communication) | GDPR, CAN‑SPAM, AWS compliance |
| **Amazon S3 / GCS** | High – stores all merchant‑uploaded media | Medium (binary files may contain PII if merchants upload such data) | High (if PII stored) | Low | High (loss of assets breaks storefront) | GDPR, data residency requirements |
| **Elasticsearch** | Medium – search functionality | Low (product data only) | Low | Medium (tampered index could affect search results) | Medium (search outage degrades UX) | GDPR (if product data contains personal info) |
| **MaxMind GeoIP2** | Low – local lookup, no external call | Low (IP address) | Low‑Medium (IP can be personal) | Low | Low | GDPR (IP is personal data) |
| **JWT (jjwt)** | High – authentication backbone | Low (user ID, roles) – stays within the app | Medium (if token is leaked) | High (token tampering leads to privilege escalation) | High (auth outage blocks access) | GDPR, OWASP ASVS |
| **Apache HttpClient, Guava, Commons, etc.** | Low – utility libraries | N/A | N/A | N/A | N/A | No direct data exposure, but keep libraries up‑to‑date to avoid known CVEs |

### Overall Risk Rating  

| Overall Vendor Criticality | Aggregate Data Sensitivity | Recommended Risk Level |
|----------------------------|----------------------------|------------------------|
| **Payments (Stripe, PayPal, Braintree)** | High (payment token, amount) | **Critical** – must be monitored, encrypted in‑transit, and stored only as tokens. |
| **Cloud Storage (S3/GCS)** | Medium‑High (media files) | **High** – enforce bucket policies, encryption‑at‑rest, and IAM least‑privilege. |
| **Shipping APIs** | Medium (address) | **Medium‑High** – ensure TLS, limit data to what is required, and purge after use. |
| **Email (SES/SMTP)** | Low‑Medium (email address) | **Medium** – protect credentials, enforce SPF/DKIM, and monitor for abuse. |
| **Search & Caching** | Low (catalog data) | **Medium** – keep software patched, restrict network access. |
| **Geolocation** | Low (IP) | **Low** – local DB, keep updated. |
| **Authentication (JWT)** | Medium (user ID) | **High** – rotate signing keys, short token lifetimes, revocation list. |

---  

## 12. Recommendations & Mitigations  

1. **Payment Security**
   * Use **PCI‑DSS validated** SDKs only (already done).  
   * Ensure **TLS 1.2+** for all outbound calls to Stripe/PayPal/Braintree.  
   * Store only **payment tokens**; never log raw tokens or card data.  
   * Rotate API keys regularly and restrict them to required scopes.  

2. **Shipping Data Protection**
   * Transmit shipping address data over **HTTPS** (already the default for HttpClient).  
   * Consider **address masking** when persisting quotes (store only a hash or reference).  
   * Delete temporary quote objects after a configurable TTL.  

3. **Email Service Hardening**
   * Keep **SMTP credentials** encrypted in the database or vault.  
   * Enable **DKIM/SPF** for the domain used in `fromEmail`.  
   * Rate‑limit email sending to prevent abuse.  

4. **Cloud Storage Governance**
   * Enable **server‑side encryption** (S3 SSE‑AES256 or SSE‑KMS).  
   * Apply **bucket policies** that deny public read unless explicitly required.  
   * Use **IAM roles** with the least privilege for the application.  
   * Implement **object lifecycle rules** to purge unused files.  

5. **Search & Caching**
   * Run Elasticsearch behind a **private network** or VPN; expose only via API gateway.  
   * Keep Elasticsearch and Infinispan versions up‑to‑date (patch known CVEs).  

6. **Geolocation**
   * Update the MaxMind DB regularly (monthly) to maintain accuracy and address GDPR‑related licensing.  

7. **Authentication**
   * Use **strong secret keys** for JWT signing (minimum 256‑bit).  
   * Set **short expiration** (e.g., 15 min) and implement refresh‑token flow.  
   * Store JWT secret in a **secrets manager** (AWS Secrets Manager, HashiCorp Vault, etc.).  

8. **Dependency Management**
   * Adopt a **software‑bill‑of‑materials (SBOM)** and regularly scan dependencies with tools such as **OWASP Dependency‑Check** or **Snyk**.  
   * Pin versions (already done via Maven properties) and avoid transitive upgrades without testing.  

9. **Logging & Monitoring**
   * Mask or redact PII (card tokens, email addresses) in logs.  
   * Enable **audit logging** for all third‑party API calls (including request IDs, timestamps, and response codes).  
   * Set up **alerting** on failed payment or shipping API calls to detect service outages quickly.  

10. **Legal & Compliance**
    * Review **Data Processing Agreements (DPAs)** with each vendor (Stripe, PayPal, Braintree, AWS, Google).  
    * Ensure **cross‑border data transfer** clauses are satisfied (e.g., EU‑US Privacy Shield replacements).  
    * Provide **privacy notices** to end‑users describing which third parties receive their data.  

---  

### Closing Statement  

Shopizer’s architecture heavily depends on external payment, shipping, email and cloud services. While the code demonstrates proper use of SDKs and secure transport (HTTPS), the **risk profile is dominated by the payment processors and cloud storage** because they handle the most sensitive data. Implementing the mitigations above will reduce the likelihood of data leakage, service disruption, and regulatory non‑compliance.  

*Prepared for the vendor‑risk assessment of Shopizer 3.2.5.*