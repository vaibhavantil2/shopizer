# Executive Privacy Audit Summary – Shopizer 3.2.5  

---

## 1. Executive Overview  

The Shopizer platform is a fully‑open‑source, head‑less Java e‑commerce solution that powers catalog, cart, checkout, and merchant operations via a rich REST API. A deep code‑level review reveals that **personal data is collected, stored, and transmitted across more than 20 distinct data elements and eight core domain entities**. While passwords are properly BCrypt‑hashed and many fields are annotated with validation constraints, several **high‑risk artefacts remain** – notably deprecated credit‑card fields, plain‑text security‑question answers, and extensive IP/geo‑location tracking. The platform also integrates with multiple payment gateways and third‑party services, creating numerous data‑outflow points. Immediate remediation is required to align with GDPR, CCPA, and PCI‑DSS obligations and to reduce the attack surface.

---

## 2. Repository Profile  

| Attribute | Detail |
|-----------|--------|
| **Name** | **Shopizer** |
| **Type** | Open‑source, headless e‑commerce platform (multi‑module Maven project) |
| **Current Release** | 3.2.5 (Java 11 compatible) |
| **Primary URL** | <http://www.shopizer.com/> |
| **Source Repository** | <https://github.com/shopizer-ecommerce/shopizer> |
| **License** | Apache License 2.0 |
| **Tech Stack** | Java 11, Spring Boot 2.5.x, Maven, JPA/Hibernate, Elasticsearch 7.5, Infinispan 9.4, Docker/Kubernetes, Swagger, Drools, JJWT, Stripe/PayPal/Braintree SDKs, AWS S3 / GCP Storage, MaxMind GeoIP2, Guava, Apache Commons, MapStruct |
| **Modules** | 5 Maven modules (core, core‑model, core‑modules, shop, admin) |
| **Code Base Size** | ~500 k LOC (including tests, docs, and generated sources) |
| **Build / CI** | Maven Wrapper (`mvnw`), GitHub Actions (unit tests, JaCoCo coverage ≥ 30 % line, ≥ 37 % branch) |
| **Deployment Options** | Stand‑alone JAR, Docker images (`shopizerecomm/shopizer`, `shopizerecomm/shopizer-admin`, `shopizerecomm/shopizer-shop-reactjs`), Helm/K8s ready |

---

## 3. Key Findings  

1. **Broad PII Scope** – Over **20** distinct personal data elements are captured across **8+** core entities (Customer, Order, User, MerchantStore, Billing/Delivery, ShoppingCart, ProductReview, OAuth).  
2. **Deprecated Credit‑Card Storage** – The `Order` entity still contains fields for raw card number, CVV, expiry, and cardholder name. Although not used by the current payment flow, the schema persists the data, representing a **HIGH‑RISK** PCI‑DSS violation.  
3. **IP‑Address Tracking** – IP addresses are logged in Orders, ShoppingCart (for fraud detection), and ShippingQuote calculations, creating a persistent identifier that can be linked to individuals.  
4. **Geolocation Data** – Billing/Delivery address objects store latitude/longitude (via MaxMind GeoIP2). This enables precise location profiling.  
5. **Security‑Question Storage** – Admin `User` records keep security‑question answers in plain text, exposing a credential‑recovery vector.  
6. **OAuth Token Persistence** – Access and refresh tokens are persisted (e.g., `CustomerOptin`, OAuth tables) without encryption, increasing the risk of token theft.  
7. **Multiple Payment Gateway Integrations** – Stripe, PayPal, and Braintree SDKs are bundled; each receives order totals, customer email, and (in the case of Stripe) tokenised card data. No unified token‑isation or vaulting strategy is enforced.  
8. **Insufficient Data‑At‑Rest Protection** – Apart from BCrypt‑hashed passwords, **no column‑level encryption or tokenisation** is applied to email addresses, phone numbers, or full addresses.  
9. **Cache‑Based Cart Persistence** – Shopping carts reside in Infinispan (in‑memory or distributed). While fast, the cache is not encrypted and may be persisted to disk depending on configuration.  
10. **Logging Practices** – Standard Spring Boot logging (INFO/DEBUG) can inadvertently emit PII (e.g., request payloads) if log level is mis‑configured.  

---

## 4. Risk Matrix  

| # | Risk | Severity | Description | Recommended Mitigation |
|---|------|----------|-------------|------------------------|
| 1 | **Deprecated credit‑card fields** | High | Raw PAN, CVV, expiry, and holder name remain in the `Order` table. | **Remove** the fields from the schema or **encrypt** them with a PCI‑DSS‑approved vault. Update all persistence code and migration scripts. |
| 2 | **Plain‑text security‑question answers** | High | Admin users’ recovery answers stored unprotected. | Store answers using a **slow hash** (e.g., Argon2) or eliminate security questions in favor of MFA. |
| 3 | **IP address collection** | Medium | IPs captured in orders, carts, shipping quotes; can be used for profiling. | **Pseudonymise** IPs (hash with salt) and retain only for fraud detection windows; purge after a defined retention period. |
| 4 | **Geolocation (lat/long) storage** | Medium | Precise location data stored with billing addresses. | Restrict access to geolocation fields, consider **masking** or **rounding** coordinates, and document a clear retention policy. |
| 5 | **OAuth token persistence** | Medium | Access/refresh tokens stored in clear text. | Encrypt tokens at rest (e.g., using JCE with a rotating key) and implement **short‑lived** access tokens with refresh‑token rotation. |
| 6 | **Unencrypted email/phone/address** | Medium | Personal contact data stored in plain text. | Apply **column‑level encryption** (e.g., Transparent Data Encryption) or tokenisation for high‑risk fields. |
| 7 | **Cache‑based cart data** | Low‑Medium | Infinispan cache may persist to disk unencrypted. | Enable **encrypted cache stores** or configure the cache as **volatile only**; enforce TLS for any remote cache nodes. |
| 8 | **Logging of request payloads** | Low‑Medium | DEBUG/TRACE logs can expose full JSON payloads containing PII. | Enforce **log sanitisation** (mask PII) and restrict log level to INFO in production. |
| 9 | **Third‑party data sharing** | Medium | Payment gateways receive order totals, email, and tokenised card data. | Ensure **Data Processing Agreements (DPAs)** are in place; limit data to the minimum required; use tokenisation where possible. |
|10| **Retention policy gaps** | Medium | No explicit data‑retention periods defined for PII. | Define and enforce **retention schedules** per GDPR/CCPA (e.g., delete inactive accounts after 24 months). |

---

## 5. Compliance Considerations  

| Regulation | Impact on Shopizer | Key Controls Needed |
|------------|-------------------|---------------------|
| **GDPR (EU)** | Requires lawful basis, data minimisation, right‑to‑access/erasure, breach notification, and DPIA for high‑risk processing. | • Map lawful bases (e.g., contract for order processing). <br>• Implement **subject‑access** and **deletion** endpoints. <br>• Conduct a **Data Protection Impact Assessment** for geolocation and IP tracking. |
| **CCPA (California)** | Grants consumers rights to know, delete, and opt‑out of sale of personal data. | • Provide **opt‑out** mechanisms for data sharing with payment processors. <br>• Maintain a **record of disclosures** and a **process** for deletion requests. |
| **PCI‑DSS (Payment Card)** | Prohibits storage of sensitive authentication data (full PAN, CVV, expiry) after authorization. | • **Remove** deprecated credit‑card fields or encrypt them in a PCI‑validated vault. <br>• Ensure **tokenisation** is used for all card data passed to gateways. |
| **ePrivacy / Cookie Laws** | Tracking IP addresses and geolocation may be considered electronic communications data. | • Update **cookie/consent banners** to cover IP‑based tracking. |
| **ISO 27001 / SOC 2** | General information‑security management requirements. | • Formalise **risk treatment plan** based on the matrix above. <br>• Implement **access‑control** and **audit‑logging** for all PII‑related tables. |

---

## 6. Immediate Actions Required (Top 5)  

1. **Eliminate or encrypt deprecated credit‑card fields** in the `Order` entity and run a one‑time data‑sanitisation script to purge any existing values.  
2. **Hash or encrypt admin security‑question answers** and migrate existing records to the new format.  
3. **Implement IP address pseudonymisation** (hash with per‑tenant salt) and define a 90‑day retention limit for fraud‑detection purposes.  
4. **Introduce column‑level encryption** (or tokenisation) for email, phone, and full address fields across all tables.  
5. **Secure OAuth token storage** by encrypting tokens at rest and enforcing short‑lived access tokens with refresh‑token rotation; update token‑revocation endpoints.

*Secondary priorities (to be addressed within 90 days):* geolocation rounding, cache encryption, log sanitisation, data‑retention policy definition, DPAs with payment providers.

---

## 7. Knowledge‑Base Files Referenced  

| File (in `privado‑memory/`) | Content Summary |
|-----------------------------|-----------------|
| **Architecture.md** | High‑level repository architecture, module breakdown, technology choices, and design patterns. |
| **Business Features.md** | Catalog, product, pricing, inventory, digital goods, reviews, manufacturer, and multi‑store capabilities. |
| **PII Inventory.md** | Detailed table of every PII element, its entity, DB column, sensitivity rating, and current protection status. |
| **Third Parties.md** | List of all external services (payment gateways, email/SMS providers, storage, search, geolocation) with Maven coordinates and data‑sharing contracts. |
| **Data Stores.md** | Analysis of relational DB (MySQL/H2/PostgreSQL), cache (Infinispan), binary storage (local, S3, GCP), and Elasticsearch usage. |
| **Data Flows.md** | End‑to‑end mapping of personal data ingestion (registration, checkout), internal processing (service layer, cache), and egress (payment APIs, email, analytics). |

These files constitute the underlying evidence base for the audit and can be consulted for deeper technical detail.

---

## 8. Audit Metadata  

| Item | Value |
|------|-------|
| **Audit Date** | 2026‑05‑08 |
| **Auditor** | Chief Privacy Officer (automated analysis) |
| **Tooling** | Manual source‑code review, static analysis of Maven POMs, grep of entity classes, and architectural documentation synthesis. |
| **Scope** | Full code‑base of Shopizer 3.2.5 (all modules), focusing on PII handling, data‑at‑rest protection, third‑party data sharing, and compliance‑relevant controls. |
| **Assumptions** | Default configuration (MySQL, Infinispan local cache, no external encryption) unless otherwise noted in `application‑*.properties`. |

---  

**Prepared for:** Shopizer development and governance teams.  
**Next Steps:** Review the Immediate Actions list with engineering leads, assign owners, and schedule remediation sprints. Continuous monitoring and periodic privacy‑impact assessments should be instituted to maintain compliance as the platform evolves.