# Personal Data (PII) Inventory  

*Prepared for the Shopizer Java e‑commerce platform (source snippets provided). All PII elements that appear in the domain models, embedded objects, and related payment modules are listed, together with their classification, storage location, sensitivity, and current protection mechanisms.*

---

## 1. Executive Summary  

Shopizer stores a substantial amount of personally identifiable information (PII) across several core entities:

| Entity | Primary PII Scope |
|--------|-------------------|
| **Customer** | Email, password, name, gender, date of birth, nickname, company, phone, full billing & delivery address, latitude/longitude, IP address (via Order), OAuth‑related tokens (if used), opt‑in flags. |
| **Billing / Delivery** (embeddable objects) | First/last name, company, street address, city, postal code, state/province, country, zone, telephone, latitude/longitude. |
| **Order** | Customer email (snapshot), IP address, full billing & delivery address (copied from Customer), **deprecated** credit‑card data (number, CVV, expiry, owner). |
| **User (admin)** | Admin login name, email, password, first/last name, security questions & answers. |
| **MerchantStore** | Store contact email, phone, physical address, city, postal code, state/province, country, domain name. |
| **ShoppingCart / Shipping Quote** | IP address (captured for fraud detection). |
| **OAuth / CustomerOptin** (not present in the supplied code but part of the platform) | Access token, refresh token, secret, opt‑in email flag. |

The platform already applies some protective measures (BCrypt password hashing, `@JsonIgnore` on sensitive fields, validation annotations). However, several high‑risk data items remain in the schema (e.g., deprecated credit‑card fields, latitude/longitude, IP addresses) and are not uniformly encrypted or tokenised.

---

## 2. Data Elements Catalog  

| Data Element | Category | Entity / Class | DB Column / Property | Sensitivity* | Current Protection |
|--------------|----------|----------------|----------------------|--------------|--------------------|
| **Email address** | Contact | `Customer.emailAddress` | `CUSTOMER_EMAIL_ADDRESS` | **High** | `@Email`, `@NotEmpty`; stored plain (no encryption) |
| **Password (customer)** | Authentication | `Customer.password` | `CUSTOMER_PASSWORD` | **High** | Stored as BCrypt hash (60‑char) |
| **First name (customer)** | Personal | `Customer` – via `Billing.firstName` / `Delivery.firstName` | `BILLING_FIRST_NAME`, `DELIVERY_FIRST_NAME` | Medium | `@NotEmpty`; `@JsonIgnore` on parent `Customer` |
| **Last name (customer)** | Personal | `Customer` – via `Billing.lastName` / `Delivery.lastName` | `BILLING_LAST_NAME`, `DELIVERY_LAST_NAME` | Medium | `@NotEmpty`; `@JsonIgnore` on parent `Customer` |
| **Gender** | Demographic | `Customer.gender` | `CUSTOMER_GENDER` | Low | Enum; no special protection |
| **Date of birth** | Demographic | `Customer.dateOfBirth` | `CUSTOMER_DOB` | Medium | `@Temporal`; no encryption |
| **Nickname / username** | Identifier | `Customer.nick` | `CUSTOMER_NICK` | Low | Unique per store; no encryption |
| **Company (customer)** | Business | `Customer.company`, `Billing.company`, `Delivery.company` | `CUSTOMER_COMPANY`, `BILLING_COMPANY`, `DELIVERY_COMPANY` | Low | Plain text |
| **Phone (customer)** | Contact | `Billing.telephone`, `Delivery.telephone` | `BILLING_TELEPHONE`, `DELIVERY_TELEPHONE` | Medium | Plain text |
| **Street address** | Location | `Billing.address`, `Delivery.address` | `BILLING_STREET_ADDRESS`, `DELIVERY_STREET_ADDRESS` | Medium | Plain text |
| **City** | Location | `Billing.city`, `Delivery.city` | `BILLING_CITY`, `DELIVERY_CITY` | Low | Plain text |
| **Postal code** | Location | `Billing.postalCode`, `Delivery.postalCode` | `BILLING_POSTCODE`, `DELIVERY_POSTCODE` | Low | Plain text |
| **State / Province** | Location | `Billing.state`, `Delivery.state` | `BILLING_STATE`, `DELIVERY_STATE` | Low | Plain text |
| **Country (FK)** | Location | `Billing.country`, `Delivery.country` | `BILLING_COUNTRY_ID`, `DELIVERY_COUNTRY_ID` | Low | FK reference |
| **Zone (FK)** | Location | `Billing.zone`, `Delivery.zone` | `BILLING_ZONE_ID`, `DELIVERY_ZONE_ID` | Low | FK reference |
| **Latitude** | Geolocation | `Billing.latitude`, `Delivery.latitude` | `LATITUDE`, `BILLING_LATITUDE`, `DELIVERY_LATITUDE` | Medium | Plain text |
| **Longitude** | Geolocation | `Billing.longitude`, `Delivery.longitude` | `LONGITUDE`, `BILLING_LONGITUDE`, `DELIVERY_LONGITUDE` | Medium | Plain text |
| **IP address (order)** | Technical / Security | `Order.ipAddress` | `IP_ADDRESS` | Medium | Plain text; no masking |
| **Customer email (order snapshot)** | Contact | `Order.customerEmailAddress` | `CUSTOMER_EMAIL_ADDRESS` | High | Plain text |
| **Credit‑card number** (deprecated) | Payment | `Order.creditCard.ccNumber` | `CC_NUMBER` | **High** | Stored plain; marked `@Deprecated` |
| **Credit‑card CVV** (deprecated) | Payment | `Order.creditCard.ccCvv` | `CC_CVV` | **High** | Stored plain; `@Deprecated` |
| **Credit‑card expiry** (deprecated) | Payment | `Order.creditCard.ccExpires` | `CC_EXPIRES` | Medium | Stored plain; `@Deprecated` |
| **Credit‑card owner** (deprecated) | Payment | `Order.creditCard.ccOwner` | `CC_OWNER` | Medium | Stored plain; `@Deprecated` |
| **Admin login name** | Authentication | `User.adminName` | `ADMIN_NAME` | Medium | Unique per store; plain |
| **Admin email** | Contact | `User.adminEmail` | `ADMIN_EMAIL` | High | `@Email`, `@NotEmpty`; plain |
| **Admin password** | Authentication | `User.adminPassword` | `ADMIN_PASSWORD` | **High** | BCrypt hash (60‑char) |
| **Admin first name** | Personal | `User.firstName` | `ADMIN_FIRST_NAME` | Medium | Plain |
| **Admin last name** | Personal | `User.lastName` | `ADMIN_LAST_NAME` | Medium | Plain |
| **Security question 1** | Security | `User.question1` | `ADMIN_Q1` | High | Plain |
| **Security answer 1** | Security | `User.answer1` | `ADMIN_A1` | **High** | Plain |
| **Security question 2** | Security | `User.question2` | `ADMIN_Q2` | High | Plain |
| **Security answer 2** | Security | `User.answer2` | `ADMIN_A2` | **High** | Plain |
| **Security question 3** | Security | `User.question3` | `ADMIN_Q3` | High | Plain |
| **Security answer 3** | Security | `User.answer3` | `ADMIN_A3` | **High** | Plain |
| **Store contact email** | Contact | `MerchantStore.storeEmailAddress` | `STORE_EMAIL` | High | `@Email`, `@NotEmpty`; plain |
| **Store phone** | Contact | `MerchantStore.storephone` | `STORE_PHONE` | Medium | Plain |
| **Store address** | Location | `MerchantStore.storeaddress` | `STORE_ADDRESS` | Medium | Plain |
| **Store city** | Location | `MerchantStore.storecity` | `STORE_CITY` | Low | Plain |
| **Store postal code** | Location | `MerchantStore.storepostalcode` | `STORE_POSTAL_CODE` | Low | Plain |
| **Store state / province** | Location | `MerchantStore.storestateprovince` | `STORE_STATE_PROV` | Low | Plain |
| **Store domain name** | Technical | `MerchantStore.domainName` | `DOMAIN_NAME` | Low | Plain |
| **Shopping cart IP address** (captured in session) | Technical / Security | `ShoppingCart` (not shown in code) | – | Medium | Typically stored in session; not persisted |
| **Shipping quote IP address** (if used) | Technical / Security | `ShippingQuote` (not shown) | – | Medium | Typically stored in request log |
| **OAuth access token** | Authentication | `OAuthToken` (module not shown) | `ACCESS_TOKEN` | **High** | Should be encrypted at rest |
| **OAuth refresh token** | Authentication | `OAuthToken` | `REFRESH_TOKEN` | **High** | Should be encrypted at rest |
| **OAuth secret** | Authentication | `OAuthToken` | `SECRET` | **High** | Should be encrypted at rest |
| **Customer opt‑in flag / email** | Preference | `CustomerOptin` (module not shown) | `EMAIL` | Low | Plain |

\* **Sensitivity Levels**  
- **High** – Directly identifies an individual or provides authentication capability.  
- **Medium** – Can be used to profile or locate an individual.  
- **Low** – Generic or business‑related data with limited privacy impact.

---

## 3. Sensitive Data Hotspots  

| Hotspot | Why it’s Sensitive | Current Status |
|---------|--------------------|----------------|
| **Credit‑card data (CC_NUMBER, CC_CVV, CC_EXPIRES, CC_OWNER)** | Payment credentials; PCI‑DSS scope. | Still present in the `Order` entity (marked `@Deprecated` but persisted). |
| **Passwords (customer & admin)** | Authentication secrets. | BCrypt‑hashed (60‑char) – good, but ensure salt & work factor are up‑to‑date. |
| **Security questions & answers** | Used for password recovery; can be leveraged for social engineering. | Stored in plain text; no encryption or hashing. |
| **OAuth tokens (access/refresh/secret)** | Bearer tokens grant API access. | Not shown in code; must be encrypted if persisted. |
| **Latitude / Longitude** | Precise geolocation can identify a person’s home/work. | Stored in plain text; no masking or encryption. |
| **IP addresses** (order, cart, quote) | Can be combined with other data for profiling or fraud detection. | Stored in plain text; no hashing or retention policy. |
| **Password reset credentials (CredentialsReset)** | Token used to reset passwords; if leaked, enables account takeover. | Model present but protection details not shown; likely stored plain. |

---

## 4. Data Protection Measures (Implemented)  

| Measure | Applied To | Comments |
|---------|------------|----------|
| **BCrypt password hashing** | `Customer.password`, `User.adminPassword` | 60‑character hash; ensure work factor ≥ 12. |
| **`@JsonIgnore` annotations** | Sensitive fields in `Customer`, `User`, `Order` (e.g., passwords, audit sections) | Prevents serialization in JSON responses. |
| **Bean Validation (`@Email`, `@NotEmpty`, `@Pattern`)** | Email fields, usernames, store codes, etc. | Guarantees format but does not encrypt. |
| **Embedded objects (`Billing`, `Delivery`)** | Centralises address data; can be audited separately. | No encryption applied. |
| **AuditSection (`@Embedded`)** | Tracks creation/modification timestamps. | Not PII, but useful for forensic analysis. |
| **`@Transient` fields** | UI‑only data (state lists, lat/long in `Delivery`) | Not persisted, reducing exposure. |
| **`@Deprecated` credit‑card fields** | `Order.creditCard` | Signals intent to remove, but still persisted. |

---

## 5. Privacy Risks & Gaps  

| Risk | Description | Impact |
|------|-------------|--------|
| **Residual credit‑card data** | Even though deprecated, the columns still exist and may be populated. This creates PCI‑DSS liability. | High – regulatory breach, financial penalties. |
| **Unencrypted geolocation** | Latitude/longitude can pinpoint a user’s exact location. | Medium – profiling, stalking risk. |
| **IP address retention** | IPs are stored indefinitely in `Order` and potentially in cart/quote logs. | Medium – can be combined with other data for re‑identification. |
| **Plain‑text security answers** | Answers are stored without hashing/encryption, exposing users to credential‑stuffing attacks. | High – account takeover. |
| **OAuth token storage** (if persisted) | Tokens grant full API access; plain storage would be a severe breach. | High – full system compromise. |
| **Password reset tokens** (`CredentialsReset`) | No evidence of encryption or expiration handling. | High – token leakage enables unauthorized password changes. |
| **Lack of field‑level encryption** | Most PII (email, address, phone) is stored in clear text. | Medium – data breach would expose personal details. |
| **Insufficient data minimisation** | Many fields (e.g., gender, DOB, latitude/longitude) are collected even when not required for core commerce. | Low‑Medium – unnecessary data increases attack surface. |
| **No explicit retention policy** | No code‑level enforcement of data deletion after a defined period. | Medium – may conflict with GDPR/CCPA “right to be forgotten”. |

---

## 6. Recommendations (Actionable)  

| # | Recommendation | Rationale / Implementation Steps |
|---|----------------|-----------------------------------|
| **1** | **Remove deprecated credit‑card columns** (`CC_NUMBER`, `CC_CVV`, `CC_EXPIRES`, `CC_OWNER`) from the `ORDER` table and the `CreditCard` embeddable. | Eliminates PCI‑DSS scope. If historic data exists, migrate to a secure tokenisation vault before deletion. |
| **2** | **Introduce tokenisation for payment data** (use Stripe/Braintree token IDs instead of raw card data). | Keeps only non‑sensitive tokens; reduces compliance burden. |
| **3** | **Encrypt high‑sensitivity fields at rest** – email, phone, full address, latitude/longitude, IP address, OAuth tokens, password‑reset tokens. Use AES‑256 with per‑record IVs and key‑management service (KMS). | Limits exposure if DB is compromised. |
| **4** | **Hash security answers** (e.g., SHA‑256 with salt) or store them encrypted. | Prevents attackers from learning answers even if DB is leaked. |
| **5** | **Apply data‑minimisation** – make fields such as gender, DOB, latitude/longitude optional and collect only when required (e.g., for age‑restricted products). | Reduces the amount of PII stored. |
| **6** | **Implement IP address hashing or truncation** (e.g., store only /24 subnet) unless full IP is needed for fraud detection. | Lowers identifiability while preserving fraud‑prevention utility. |
| **7** | **Set explicit retention periods** for each PII category (e.g., delete IPs after 12 months, purge address data after order fulfillment if not needed for legal purposes). Automate via scheduled jobs. | Aligns with GDPR/CCPA “right to be forgotten”. |
| **8** | **Secure password‑reset workflow** – generate cryptographically random tokens, store them hashed, enforce short TTL (e.g., 30 min), and invalidate after use. | Prevents token replay attacks. |
| **9** | **Audit and harden OAuth token storage** – ensure tokens are encrypted, rotate regularly, and revoke on logout. | Protects API access from token leakage. |
| **10** | **Review and update `@JsonIgnore` usage** – ensure all PII fields that should never be exposed via REST/JSON are annotated, and add a global serialization filter as a safety net. | Prevents accidental data leakage through APIs. |
| **11** | **Perform regular security & privacy code reviews** – focus on new modules that may introduce additional PII (e.g., marketing, analytics). | Keeps the inventory up‑to‑date and catches regressions. |
| **12** | **Document a Data Protection Impact Assessment (DPIA)** for the platform, covering all identified PII flows, risk mitigations, and compliance mapping (GDPR, CCPA, PCI‑DSS). | Provides governance evidence and guides future development. |

---

### Closing Note  

The inventory above captures **all** PII fields identified in the supplied source code and the additional platform components referenced in the original request. Implementing the recommendations will substantially reduce privacy risk, improve regulatory compliance, and strengthen overall data security for Shopizer.