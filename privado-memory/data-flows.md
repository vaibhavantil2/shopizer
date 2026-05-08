# Data Flow Mapping – Shopizer (Java e‑commerce platform)

> **Scope** – This mapping focuses on the movement of **personal data (PII)** through the Shopizer 3.2.5 code‑base, covering ingestion, internal processing, and egress. It draws on the repository‑level analysis of entities, services, third‑party integrations, and persistence mechanisms that were identified earlier.

---

## 1. Data Ingress Points  

| Entry Point | Personal Data Collected (PII) | Origin / Trigger |
|-------------|------------------------------|------------------|
| **Customer registration** | • Email address  <br>• Password (hashed) <br>• First / last name (via billing/delivery) <br>• Date of birth <br>• Gender <br>• Nickname / username <br>• Company name <br>• Phone number <br>• Full billing & delivery address (street, city, postcode, state, country, zone, latitude/longitude) | `/api/v1/customers` – registration form submitted by a visitor |
| **Checkout** | • Billing address (same fields as registration) <br>• Delivery address (same fields) <br>• Phone number <br>• Email (snapshot) <br>• IP address (captured at order creation) <br>• Credit‑card data (number, CVV, expiry, owner – **deprecated** but still present in the model) | `/api/v1/orders` – checkout flow (cart → order) |
| **Admin user creation** | • Admin login name <br>• Admin email <br>• Admin password (hashed) <br>• First / last name <br>• Security questions & answers (high‑risk) | `/api/v1/users` – admin UI / API |
| **Merchant store setup** | • Store contact email <br>• Store phone <br>• Physical address (street, city, postcode, state, country) <br>• Domain name | `/api/v1/merchantstores` – merchant onboarding UI |
| **Newsletter opt‑in** | • Email address (opt‑in flag) | Newsletter subscription widget |
| **Shopping cart** | • IP address (used for fraud detection, cart‑to‑order correlation) | Cart creation / update API (`/api/v1/cart`) |
| **Product reviews** | • Customer name (or nickname) <br>• Review text (may contain personal identifiers) <br>• Rating (numeric) | `/api/v1/reviews` – review submission endpoint |

*All fields are defined in the JPA entities under `com.salesmanager.core.model.*` (e.g., `Customer`, `Order`, `User`, `MerchantStore`, `ProductReview`).*  

---

## 2. Internal Data Movement  

### 2.1 Layered Flow  

```
+-------------------+      +-------------------+      +-------------------+      +-------------------+
|  sm-shop (REST    | ---> |  sm-core (service | ---> |  sm-core-model    | ---> |  Relational DB   |
|  API Controllers) |      |  layer)           |      |  (JPA entities)   |      |  (MySQL / PG)    |
+-------------------+      +-------------------+      +-------------------+      +-------------------+
```

* **sm-shop** – Spring‑Boot controllers expose the public REST API. They perform request validation, authentication (JWT), and map JSON payloads to DTOs.  
* **sm-core** – Service beans contain business rules (e.g., `CustomerService`, `OrderService`, `PaymentService`). They orchestrate calls to repositories, external APIs, and internal utilities.  
* **sm-core-model** – JPA entity classes map directly to database tables (`CUSTOMER`, `ORDERS`, `USERS`, `MERCHANT_STORE`, `PRODUCT_REVIEW`, etc.).  
* **Database** – All persisted PII resides in the relational DB (MySQL by default). Sensitive fields (passwords) are stored as BCrypt hashes; other fields are stored in clear text unless the operator adds column‑level encryption.

### 2.2 Session & Cart Lifecycle  

```
[Browser] --(session cookie)--> sm-shop (SessionController)
          |
          v
   In‑memory / Infinispan cache (cart object)
          |
          |  Checkout → Order conversion
          v
   sm-core.OrderService → sm-core-model.Order entity → DB
```

* The shopping cart lives in the **Infinispan** cache (keyed by session ID). When the user proceeds to checkout, the cart is transformed into an `Order` entity, enriched with the captured IP address and snapshot of the billing/delivery data.

### 2.3 Payment & Shipping Sub‑flows  

* **Payment** – `OrderService` calls `PaymentService` → selects a concrete implementation (`StripePayment`, `PayPalExpressCheckoutPayment`, `BraintreePayment`). The service builds a request object containing the **amount**, **currency**, **order ID**, **customer email**, and (if the deprecated flow is used) **raw card data**. The request is sent over HTTPS to the third‑party gateway.  
* **Shipping** – `ShippingService` builds a quote request that includes **origin/destination addresses**, **phone**, **package weight/dimensions**, and **store credentials**. The request is sent to UPS/USPS (or Canada Post) via their REST/XML APIs.

---

## 3. Data Egress Points  

| Destination (Third‑Party) | Personal Data Sent | Legal / Business Purpose |
|---------------------------|--------------------|--------------------------|
| **Stripe / PayPal / Braintree** | • Card token / (deprecated) raw card number, CVV, expiry, owner <br>• Transaction amount, currency, order ID <br>• Customer email (for receipt) | Payment authorization & settlement |
| **UPS / USPS / Canada Post** | • Origin & destination street address, city, postcode, country <br>• Phone number (contact) <br>• Package weight/dimensions | Shipping rate calculation & label generation |
| **Amazon SES / SMTP (JavaMail)** | • Customer email address (order confirmations, registration, newsletters) <br>• Optional name fields for personalization | Email communication with customers |
| **Amazon S3 / Google Cloud Storage** | • Binary files (product images, digital downloads) – **no PII** unless a merchant uploads personal content | Static asset storage & CDN delivery |
| **Elasticsearch** | • Product catalogue data (title, description, SKU, price) – typically **no PII**; however, if merchants store personalised product attributes they may be indexed | Search indexing for fast product lookup |
| **GeoLocation service (MaxMind GeoIP2)** | • IP address (from order/cart) | Geolocation for tax/shipping calculations and fraud detection |
| **Google Maps Distance Matrix** | • Origin & destination latitude/longitude (derived from addresses) | Distance‑based shipping cost calculation |

*All outbound calls are performed over TLS (HTTPS). The platform does **not** encrypt PII before transmission to third parties; it relies on the transport layer security and the third‑party’s own compliance.*

---

## 4. Textual Flow Diagrams (ASCII)

### 4.1 Customer Registration Flow
```
+-----------+          +----------------+          +----------------+          +-------------------+
|  Browser  |  POST /api/v1/customers  |  sm-shop (REST) |  sm-core (CustomerService) |  sm-core-model (Customer) |
+-----------+  (JSON payload)  +----------------+  (validation, hashing) +----------------+  (JPA persist)  |
        |                               |                               |                               |
        |                               v                               v                               v
        |                     +----------------+               +----------------+               +-------------------+
        |                     |  BCrypt hash   |               |  Save entity   |               |  INSERT into DB   |
        |                     +----------------+               +----------------+               +-------------------+
        |                               |                               |                               |
        +-------------------------------+-------------------------------+-------------------------------+
                                            |
                                            v
                                   +-------------------+
                                   |  Confirmation    |
                                   |  Email (SES/SMTP)|
                                   +-------------------+
```

### 4.2 Order / Checkout Flow
```
+-----------+   POST /api/v1/orders (cart → order)   +----------------+   invoke   +----------------+   persist   +-------------------+
|  Browser  |--------------------------------------->|  sm-shop (API) |----------->|  sm-core (OrderService) |----------->|  sm-core-model (Order) |
+-----------+   (JSON with billing, shipping, IP)   +----------------+   (validation, cart‑to‑order)   +----------------+   (JPA)   |
        |                                                                                                   |
        |                                                                                                   v
        |                                                                                         +-------------------+
        |                                                                                         |  INSERT into DB   |
        |                                                                                         +-------------------+
        |                                                                                                   |
        |                                                                                                   v
        |                                                                                         +-------------------+
        |                                                                                         |  Trigger Payment  |
        |                                                                                         +-------------------+
        |                                                                                                   |
        |                                                                                                   v
        |                                                                                         +-------------------+
        |                                                                                         |  Payment Service  |
        |                                                                                         +-------------------+
```

### 4.3 Payment Processing Flow
```
+-----------+   POST /api/v1/payments   +----------------+   build request   +----------------+   HTTPS POST   +-------------------+
|  Order    |------------------------->|  sm-shop (API) |----------------->|  PaymentService|--------------->|  Stripe / PayPal / |
| Service   |   (orderId, token)       |                |   (gateway SDK)  |                |   (TLS)       |  Braintree API    |
+-----------+                         +----------------+                  +----------------+                +-------------------+
        |                                                                                                   |
        |   Receive response (success/failure)                                                               |
        +<----------------------------------------------------------------------------------------------<---+
        |
        v
+----------------+   update order status   +-------------------+
|  OrderService  |----------------------->|  Order entity (DB)|
+----------------+   (status = PAID)      +-------------------+
```

---

## 5. Cross‑Border / Jurisdictional Considerations  

| Cloud / Third‑Party Service | Typical Data Residency | Potential Legal Impact |
|-----------------------------|------------------------|------------------------|
| **AWS S3 / SES** | Data stored in the AWS region configured by the operator (e.g., `eu‑west‑1` for EU, `us‑east‑1` for US) | Subject to the laws of that region (e.g., GDPR for EU, CLOUD Act for US). Data‑transfer agreements (Standard Contractual Clauses) may be required when EU data is sent to US‑based AWS. |
| **Google Cloud Storage** | Region selectable (e.g., `europe‑west1`, `us‑central1`) | Same considerations as AWS; ensure the chosen region aligns with the organization’s data‑localisation policy. |
| **Stripe** | EU‑hosted Stripe accounts store payment data in EU data centres; US‑hosted accounts store in US. | PCI‑DSS compliance is mandatory; GDPR applies to any personal identifiers (email, name) transmitted. |
| **PayPal / Braintree** | Global infrastructure; data may be replicated across multiple jurisdictions. | Must rely on PayPal/Braintree’s GDPR‑compliant data‑processing agreements. |
| **UPS / USPS** | Shipping carriers operate globally; address data may be processed in the carrier’s headquarters country. | Addresses are considered personal data; carriers provide contractual clauses for data protection. |
| **MaxMind GeoIP2** | IP‑to‑location database is downloaded and used locally; no data leaves the host. | No cross‑border transfer, but the IP address itself is personal data under GDPR. |
| **Elasticsearch (self‑hosted)** | Typically co‑located with the primary DB (same region). | If hosted in a different jurisdiction, ensure appropriate safeguards. |

**Recommendation:**  
*Document the exact regions used for each cloud service in the system architecture register.*  
*When EU resident data is sent to a non‑EU service, attach a Standard Contractual Clause (SCC) or rely on an adequacy decision.*  

---

## 6. Data Minimisation Assessment  

| Data Category | Is it strictly necessary? | Comments / Potential Reduction |
|---------------|---------------------------|--------------------------------|
| **Email address (customer, admin, merchant, newsletter)** | **Yes** – required for login, order receipts, notifications, and legal communications. | Ensure opt‑in status is stored separately and that marketing emails respect the flag. |
| **Password** | **Yes** – required for authentication. | Already stored as BCrypt hash; no further reduction possible. |
| **Full name (first/last)** | **Yes** – needed for invoicing, shipping labels, and legal compliance. | Could be split into “billing name” and “shipping name” to avoid unnecessary duplication. |
| **Date of birth** | **Partial** – useful for age‑restricted products or analytics. | If the store does not sell age‑restricted goods, consider removing or making optional. |
| **Gender** | **Low** – rarely required for commerce. | Make optional or remove unless needed for analytics with explicit consent. |
| **Phone number** | **Yes** – required for shipping carrier contact and fraud checks. | Keep but mask in logs and UI where not needed. |
| **Full address (billing & delivery)** | **Yes** – essential for invoicing, shipping, tax calculation. | No reduction possible. |
| **Latitude / Longitude** | **Medium** – used for geo‑targeting and distance‑based shipping. | Could be derived on‑the‑fly from address; if not required, drop the fields. |
| **IP address** | **Medium** – used for fraud detection and geo‑location. | Store only a hash or truncate (e.g., /24) after risk assessment. |
| **Credit‑card data (number, CVV, expiry, owner)** | **No** – deprecated in the model; PCI‑DSS forbids storing raw card data unless you are a PCI‑validated merchant. | Remove these columns entirely; rely on tokenisation via Stripe/PayPal/Braintree. |
| **Security questions / answers (admin)** | **High** – used for password recovery. | Consider replacing with email‑based password reset links; if retained, encrypt at rest. |
| **Merchant store contact details** | **Yes** – needed for order notifications and legal contact. | No reduction. |
| **Product review author name** | **Yes** – required for attribution. | Provide an option for pseudonymous reviews. |
| **Newsletter opt‑in flag** | **Yes** – required for consent management. | Store separately from the email address to simplify consent revocation. |

### Overall Findings  

* The platform **collects more data than strictly required** for a basic e‑commerce flow (e.g., gender, DOB, latitude/longitude, deprecated credit‑card fields).  
* **Data minimisation** can be achieved by:  
  1. Removing or making optional the low‑value fields (gender, DOB, latitude/longitude).  
  2. Deleting the raw credit‑card columns and enforcing token‑only storage via payment gateways.  
  3. Masking or hashing IP addresses after fraud checks.  
  4. Encrypting high‑risk fields (security answers, any future PII) at rest.  
* Implement **privacy‑by‑design** defaults: optional fields are omitted from the registration form unless a merchant explicitly enables them.

---

### Quick Reference Checklist for the DPO  

| ✅ | Action |
|----|--------|
| **Inventory** | Verify that the database schema matches the PII inventory (Customer, Order, User, MerchantStore, ProductReview). |
| **Encryption** | Ensure BCrypt for passwords; add column‑level encryption for security answers and any future high‑risk fields. |
| **Retention** | Define retention periods (e.g., delete order snapshots after 7 years, purge IP logs after 30 days). |
| **Access Controls** | Restrict DB access to service accounts; audit admin UI actions on PII. |
| **Third‑Party Agreements** | Review contracts with Stripe, PayPal, Braintree, UPS, USPS, AWS, GCP for GDPR SCCs / Data Processing Addendums. |
| **Data Subject Rights** | Implement API endpoints or admin tools to export, rectify, or delete a customer’s data on request. |
| **Incident Response** | Log all outbound PII transmissions; set up alerts for abnormal data flows (e.g., bulk export to S3). |
| **Documentation** | Keep a Data Flow Diagram (DFD) versioned with the codebase; update whenever a new integration is added. |

--- 

**Conclusion** – The mapping above provides a complete view of how personal data enters Shopizer, traverses internal modules, and leaves the system to external services. It also highlights areas where data minimisation, encryption, and cross‑border safeguards can be strengthened to meet GDPR, CCPA, and other privacy regulations.