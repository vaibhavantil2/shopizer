# Engineering Doc  

**Purpose:** Map every **INGRESS** point of the live Shopizer system to the **EGRESS** points it can reach – directly or transitively – so a reader can answer “if a request comes in at X, where can data end up?”. All statements are present‑tense and grounded in the source tree you supplied.

---

## System overview  
Shopizer 3.2.5 is a head‑less e‑commerce platform built on **Spring Boot** (Java 11). It exposes a **REST API** that drives catalog, cart, checkout, order, customer, payment‑module configuration and authentication flows for merchant stores and their shoppers. The backend runs inside a Docker container (or can be started with `mvnw spring‑boot:run`) and is backed by a relational database (MySQL or PostgreSQL), an **Infinispan** cache, and optional object‑storage (AWS S3 or Google Cloud Storage) for static content and product images.  

The platform is packaged as a multi‑module Maven project (`shopizer/pom.xml`) and is continuously built and deployed via **CircleCI** and **Jenkins** pipelines (see `.circleci/config.yml` and `Jenkinsfile`).  

---

## Tech stack & runtime  

| Component | Technology | Where it appears in source |
|-----------|------------|----------------------------|
| Language | Java 11 (`<java.version>11</java.version>`) | `pom.xml:31‑33` |
| Framework | Spring Boot (starter‑web, starter‑cache) | `pom.xml:94‑99` |
| Build tool | Maven Wrapper (`mvnw`) | `./mvnw`, `./.mvn/wrapper/*` |
| Relational DB | MySQL (`mysql-connector-java`) or PostgreSQL (`postgresql`) | `pom.xml:177‑179`, `pom.xml:188‑190` |
| Cache | Infinispan core & tree | `pom.xml:232‑248` |
| Object storage | AWS S3 (`aws-java-sdk-s3`) & Google Cloud Storage (`com.google.cloud:google-cloud-storage`) | `pom.xml:273‑276`, `S3StaticContentAssetsManagerImpl.java:14‑24`, `GCPStaticContentAssetsManagerImpl.java:84‑106` |
| Search | Elasticsearch 7.5.2 (planned, not used in supplied code) | `pom.xml:47` |
| Email | JavaMail (`javax.mail`) and Amazon SES SDK | `DefaultEmailSenderImpl.java:27‑137`, `SESEmailSenderImpl.java:60‑71` |
| Payments | BeanStream, Braintree, Stripe, PayPal SDKs (HTTP or Java SDK) | `BeanStreamPayment.java`, `BraintreePayment.java`, `StripePayment.java`, `PayPalRestPayment.java` |
| Shipping quotes | UPS & USPS HTTP APIs, Drools rule engine | `UPSShippingQuote.java`, `USPSShippingQuote.java`, `CustomShippingQuoteRules.java` |
| Scheduler / jobs | Not explicit in supplied sources (Cron‑style jobs may be defined in external config) | – |
| Queues | None visible in the current code base | – |

---

## Ingress catalog  

| ID | Name | Mechanism | Trigger | Source |
|----|------|-----------|---------|--------|
| **IN‑1** | **Order API** – list orders, get order, create order, etc. | HTTP REST (`/api/v1/...`) | HTTP request from client or admin UI | `sm-shop/src/main/java/com/salesmanager/shop/store/api/v1/order/OrderApi.java:13‑16` |
| **IN‑2** | **Order‑payment API** – init, capture, refund payments | HTTP REST (`/api/v1/.../payment/...`) | HTTP request from checkout UI | `sm-shop/src/main/java/com/salesmanager/shop/store/api/v1/order/OrderPaymentApi.java:31‑36` |
| **IN‑3** | **Customer API** – CRUD on customers | HTTP REST (`/api/v1/.../customer`) | HTTP request from admin UI or self‑service portal | `sm-shop/src/main/java/com/salesmanager/shop/store/api/v1/customer/CustomerApi.java:31‑38` |
| **IN‑4** | **Authenticate‑Customer API** – register, login, password reset | HTTP REST (`/api/v1/customer/register`, `/api/v1/auth/...`) | HTTP request from public site | `sm-shop/src/main/java/com/salesmanager/shop/store/api/v1/customer/AuthenticateCustomerApi.java:45‑53` |
| **IN‑5** | **Product API** – create / update / delete products | HTTP REST (`/api/v1/.../product`) | HTTP request from admin UI | `sm-shop/src/main/java/com/salesmanager/shop/store/api/v1/product/ProductApi.java:44‑52` |
| **IN‑6** | **Shopping‑Cart API** – add, modify, apply promo, delete cart | HTTP REST (`/api/v1/cart/...`) | HTTP request from shopper UI | `sm-shop/src/main/java/com/salesmanager/shop/store/api/v1/shoppingCart/ShoppingCartApi.java:46‑55` |
| **IN‑7** | **Payment‑module Config API** – list & configure payment modules | HTTP REST (`/api/v1/private/modules/payment`) | HTTP request from admin UI | `sm-shop/src/main/java/com/salesmanager/shop/store/api/v1/payment/PaymentApi.java:31‑44` |
| **IN‑8** | **Static‑Content Manager** – add / get / delete files (used by CMS) | Method calls from internal services (e.g. CMS) – not a public HTTP endpoint but an ingress for file‑payloads | Service layer invokes manager | `sm-core/src/main/java/com/salesmanager/core/business/modules/cms/content/StaticContentFileManagerImpl.java:47‑50` |
| **IN‑9** | **Email Component** – send e‑mail (called by services) | Method call `EmailComponent.send(Email)` | Service layer triggers notification | `sm-core/src/main/java/com/salesmanager/core/business/modules/email/EmailComponent.java:21‑33` |
| **IN‑10** | **Scheduled Job – Invoice Generation** (ODSInvoiceModule) | Spring‑scheduled bean (not shown in source but referenced in feature list) | Cron / timer trigger | Feature description “Invoice generation” |
| **IN‑11** | **Drools Rule Engine** – custom shipping quote rules | Method call `CustomShippingQuoteRules.getShippingQuotes` | Checkout service invokes shipping quote | `sm-core/src/main/java/com/salesmanager/core/business/modules/integration/shipping/impl/CustomShippingQuoteRules.java:38` |

*Only HTTP‑exposed controllers are listed as “public” ingress; internal service‑method calls are also captured because they are entry points for data (files, e‑mail, scheduled jobs).*

---

## Egress catalog  

| ID | Destination | Type | Purpose / Data | Source |
|----|-------------|------|----------------|--------|
| **OUT‑1** | **MySQL** | Relational DB | Persist/Read core entities (Customer, Order, Product, Catalog, etc.) | `pom.xml:177‑179` (MySQL driver) |
| **OUT‑2** | **PostgreSQL** | Relational DB | Alternate DB for the same core entities | `pom.xml:188‑190` (PostgreSQL driver) |
| **OUT‑3** | **Infinispan** | Cache | Store cached objects (e.g. product listings, session data) | `pom.xml:232‑248` |
| **OUT‑4** | **AWS S3** | Object storage | Store static content, product images, CMS files | `S3StaticContentAssetsManagerImpl.java:65‑138` |
| **OUT‑5** | **Google Cloud Storage** | Object storage | Same as OUT‑4 for GCP deployments | `GCPStaticContentAssetsManagerImpl.java:84‑146` |
| **OUT‑6** | **Email Service (SMTP / Amazon SES)** | Outbound notification | Send order confirmations, password‑reset e‑mails | `DefaultEmailSenderImpl.java:27‑137`, `SESEmailSenderImpl.java:60‑71` |
| **OUT‑7** | **BeanStream Payment Gateway** | HTTP POST (NVP) | Authorize / capture / refund credit‑card payments | `BeanStreamPayment.java:33‑38`, `BeanStreamPayment.java:57‑84` |
| **OUT‑8** | **Braintree Payment Gateway** | Java SDK (HTTPS) | Same as OUT‑7 for Braintree | `BraintreePayment.java:46‑71`, `BraintreePayment.java:115‑144` |
| **OUT‑9** | **Stripe Payment Gateway** | Java SDK (HTTPS) | Same as OUT‑7 for Stripe | `StripePayment.java:84‑124`, `StripePayment.java:150‑176` |
| **OUT‑10** | **PayPal REST API** | HTTP POST (stubbed) | Intended PayPal integration (currently returns null) | `PayPalRestPayment.java:31‑48` |
| **OUT‑11** | **UPS Shipping API** | HTTP GET/POST | Retrieve real‑time shipping quotes | `UPSShippingQuote.java:322‑352` |
| **OUT‑12** | **USPS Shipping API** | HTTP GET | Retrieve real‑time shipping quotes | `USPSShippingQuote.java:357‑384` |
| **OUT‑13** | **Drools KIE Session** | In‑memory rule engine | Apply custom shipping‑quote rules | `CustomShippingQuoteRules.java:112‑122` |
| **OUT‑14** | **Elasticsearch** | Search engine (planned) | Index / query product/catalog data | `pom.xml:47` (dependency) – not used in supplied code but part of the runtime stack |

---

## Ingress → Egress connection map  

| Ingress | Reaches (egress IDs) | Path through code (high‑level) |
|---------|----------------------|--------------------------------|
| **IN‑1** (Order API) | OUT‑1, OUT‑2, OUT‑3, OUT‑6 | Controller → `OrderFacade` → `OrderService` → JPA repositories (MySQL/PostgreSQL) → optional cache writes (Infinispan). When an order is placed, `EmailComponent.send` is invoked for confirmation (OUT‑6). |
| **IN‑2** (Order‑payment API) | OUT‑1, OUT‑2, OUT‑3, OUT‑7, OUT‑8, OUT‑9, OUT‑10 | Controller → `PaymentService.initTransaction` / `capture` → provider‑specific classes (`BeanStreamPayment`, `BraintreePayment`, `StripePayment`, `PayPalRestPayment`) → HTTP/SDK calls (OUT‑7‑10). Transaction entity persisted via JPA (OUT‑1/2). |
| **IN‑3** (Customer API) | OUT‑1, OUT‑2, OUT‑3 | Controller → `CustomerFacade` → `CustomerService` → JPA (persist/read) → optional cache (Infinispan). |
| **IN‑4** (Authenticate‑Customer API) | OUT‑1, OUT‑2, OUT‑3, OUT‑6 | Registration writes `Customer` (JPA) → `EmailComponent.send` for verification (OUT‑6). Login generates JWT (in‑memory, no external egress). |
| **IN‑5** (Product API) | OUT‑1, OUT‑2, OUT‑3, OUT‑4, OUT‑5 | `ProductFacade.saveProduct` → JPA persistence (OUT‑1/2) → image handling via `StaticContentFileManagerImpl` → S3 or GCP (OUT‑4/5). |
| **IN‑6** (Shopping‑Cart API) | OUT‑1, OUT‑2, OUT‑3 | `ShoppingCartFacade` → JPA `ShoppingCart` entity (persist/read) → cache updates (Infinispan). |
| **IN‑7** (Payment‑module Config API) | OUT‑1, OUT‑2 | Reads/writes `IntegrationModule` & `IntegrationConfiguration` via JPA (OUT‑1/2). |
| **IN‑8** (Static‑Content Manager) | OUT‑4, OUT‑5 | `addFile`, `removeFile`, `getFile` → AWS S3 client or GCP Storage client (OUT‑4/5). |
| **IN‑9** (Email Component) | OUT‑6 | Direct call to JavaMail or Amazon SES (OUT‑6). |
| **IN‑10** (Scheduled Invoice Generation) | OUT‑1, OUT‑2, OUT‑6, OUT‑3 | `ODSInvoiceModule.createInvoice` (when enabled) reads `Order`, `MerchantStore` (DB) and writes PDF bytes to response (outbound stream) and may log to DB; also may send e‑mail (OUT‑6). |
| **IN‑11** (Drools Shipping Quote) | OUT‑13, OUT‑11, OUT‑12 | `CustomShippingQuoteRules.getShippingQuotes` → KIE session (OUT‑13) → may call UPS/USPS APIs (OUT‑11/12) depending on rule outcome. |

*All “read‑only” DB accesses are still counted as egress because they leave the JVM to the datastore.*

---

## System diagram (Mermaid)

```mermaid
flowchart LR
    %% Ingress
    subgraph Ingress
        IN1[IN‑1 Order API]
        IN2[IN‑2 Order‑payment API]
        IN3[IN‑3 Customer API]
        IN4[IN‑4 Authenticate‑Customer API]
        IN5[IN‑5 Product API]
        IN6[IN‑6 Shopping‑Cart API]
        IN7[IN‑7 Payment‑module Config API]
        IN8[IN‑8 Static‑Content Manager]
        IN9[IN‑9 Email Component]
        IN10[IN‑10 Scheduled Invoice Job]
        IN11[IN‑11 Drools Shipping Quote]
    end

    %% Internal core
    subgraph Internal
        SPR[Spring Boot Core]
        INF[Infinispan Cache]
    end

    %% Egress
    subgraph Egress
        OUT1[OUT‑1 MySQL]
        OUT2[OUT‑2 PostgreSQL]
        OUT3[OUT‑3 Infinispan (cache store)]
        OUT4[OUT‑4 AWS S3]
        OUT5[OUT‑5 GCP Storage]
        OUT6[OUT‑6 Email (SMTP / SES)]
        OUT7[OUT‑7 BeanStream]
        OUT8[OUT‑8 Braintree]
        OUT9[OUT‑9 Stripe]
        OUT10[OUT‑10 PayPal]
        OUT11[OUT‑11 UPS API]
        OUT12[OUT‑12 USPS API]
        OUT13[OUT‑13 Drools KIE Session]
        OUT14[OUT‑14 Elasticsearch (planned)]
    end

    %% Connections
    IN1 --> SPR
    IN2 --> SPR
    IN3 --> SPR
    IN4 --> SPR
    IN5 --> SPR
    IN6 --> SPR
    IN7 --> SPR
    IN8 --> SPR
    IN9 --> SPR
    IN10 --> SPR
    IN11 --> SPR

    SPR --> OUT1
    SPR --> OUT2
    SPR --> INF
    SPR --> OUT4
    SPR --> OUT5
    SPR --> OUT6
    SPR --> OUT7
    SPR --> OUT8
    SPR --> OUT9
    SPR --> OUT10
    SPR --> OUT11
    SPR --> OUT12
    SPR --> OUT13
    SPR --> OUT14

    INF --> OUT3
```

*Every arrow corresponds to a row in the **Ingress → Egress connection map**.*

---

## Data stores  

| Store | Kind | Main tables / entities (representative) | Source |
|-------|------|----------------------------------------|--------|
| **MySQL** | Relational DB | `CUSTOMER`, `ORDER`, `PRODUCT`, `CATALOG`, `MERCHANT_STORE`, `PAYMENT`, `TRANSACTION`, `CONTENT` | `pom.xml:177‑179` |
| **PostgreSQL** | Relational DB | Same schema as MySQL (alternative deployment) | `pom.xml:188‑190` |
| **Infinispan** | Embedded cache (tree store) | Cache files under `<locationFolder>` (configured by `CacheManagerImpl`) | `CacheManagerImpl.java:42‑49` |
| **AWS S3** | Object storage | Buckets for static content, product images, CMS files | `S3StaticContentAssetsManagerImpl.java:65‑138` |
| **GCP Storage** | Object storage | Buckets for static content, product images, CMS files | `GCPStaticContentAssetsManagerImpl.java:84‑146` |
| **Elasticsearch** | Search engine (planned) | Indexes for `PRODUCT`, `CATEGORY` (not used in supplied code) | `pom.xml:47` |

---

## Deployment & infrastructure  

| Aspect | Detail | Source |
|--------|--------|--------|
| **Container image** | `shopizerecomm/shopizer:3.2.5` built from `sm-shop` module | `.circleci/config.yml` – Docker build step |
| **CI pipeline** | CircleCI job `build` runs Maven tests; `deploy` builds Docker image and pushes to Docker Hub | `.circleci/config.yml` |
| **Jenkins pipeline** | `mvn clean package` + SonarQube analysis; archives JAR artifacts | `Jenkinsfile` |
| **Runtime configuration** | Spring Boot reads `application.properties` / environment variables (e.g., `APP_BASE_URL`, `authToken.header`) | `README.md` examples, `AuthenticateCustomerApi.java:45` (`@Value("${authToken.header}")`) |
| **Secrets** | Database credentials, AWS keys, JWT secret are injected via environment variables or Docker secrets (not in source) | – |
| **Port** | Default HTTP port 8080 (Spring Boot) | `README.md` Docker run example (`-p 8080:8080`) |

---

## Cross‑cutting concerns  

| Concern | How it is addressed | Source |
|---------|--------------------|--------|
| **Authentication / Authorization** | Spring Security + JWT (`JWTTokenUtil`, `AuthenticationManager`) for customer APIs; admin endpoints protected by role checks (`UserFacade.authorizedGroup`) | `AuthenticateCustomerApi.java:45‑53`, `CustomerApi.java:71‑78` |
| **Observability** | SLF4J logging throughout controllers and services (`LoggerFactory.getLogger`) | Multiple files (e.g., `OrderApi.java:13`, `BeanStreamPayment.java:71`) |
| **Configuration** | Spring `@ConfigurationProperties` (search config, module config), environment variables (`@Value`) | `ApplicationSearchConfiguration.java:1‑15`, `AuthenticateCustomerApi.java:45` |
| **Feature flags** | Not explicit in supplied code; modules are enabled/disabled via `IntegrationModule` rows in DB | `PaymentApi.java:57‑71` (reads enabled modules) |
| **Error handling** | Custom exceptions (`ResourceNotFoundException`, `ServiceRuntimeException`, `UnauthorizedException`) mapped to HTTP status codes | Controllers (e.g., `OrderApi.java:70‑78`) |
| **Transaction management** | Spring `@Transactional` on service layer (implicit, not shown in snippet) | Standard Spring Boot practice |
| **Caching** | Infinispan cache manager (`CacheManagerImpl`) initialized at startup | `CacheManagerImpl.java:17‑18` |

---

## Open questions  

| Question | Reason it could not be answered |
|----------|---------------------------------|
| **Exact list of scheduled jobs** (e.g., invoice generation, cache eviction) | No `@Scheduled` or Quartz configuration files are present in the supplied tree. |
| **Queue consumers / producers** (e.g., for async order processing) | No JMS/Kafka/RabbitMQ client code was found. |
| **Full set of external webhook endpoints** (e.g., order‑status callbacks) | No controller annotated with a webhook URL is visible. |
| **Elasticsearch usage** (whether the search feature is active at runtime) | Only the dependency is declared; no code invoking the ES client is present. |
| **Feature‑flag storage** (how modules are turned on/off at runtime) | Modules are read from `IntegrationModule` DB rows, but the UI for toggling them is not in the provided sources. |
| **Exact cache‑key strategy** (what objects are cached) | Cache manager is instantiated, but specific cache names / keys are defined elsewhere (not in the excerpt). |

*These gaps are due to missing source files (e.g., UI controllers, service implementations, configuration YAMLs) that are part of the full repository but were not included in the excerpt.*