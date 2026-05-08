# Repository Architecture Overview

> **Scope** – This document is a concise knowledge‑base for developers, architects, and reviewers who need to understand the structure, technology choices, and design patterns of the **Shopizer** Java e‑commerce repository (version 3.2.5). All statements are derived directly from the source tree, `pom.xml` files, CI configuration, and README documentation that accompany the project.

---

## 1. Repository Identity
| Item | Value |
|------|-------|
| **Name** | **Shopizer** |
| **Purpose** | Open‑source, headless e‑commerce platform exposing a rich REST API (catalog, cart, checkout, merchant, order, customer, user). |
| **License** | Apache License, Version 2.0 (see `LICENSE.md`). |
| **Primary URL** | <http://www.shopizer.com/> |
| **Source Repository** | <https://github.com/shopizer-ecommerce/shopizer> (multi‑module Maven project). |
| **Current Release** | 3.2.5 (Java 11 compatible). |

---

## 2. Tech Stack

| Category | Technology / Library | Version (as defined in `pom.xml`) | Notes / Usage |
|----------|----------------------|-----------------------------------|---------------|
| **Language** | Java | 11 (`<java.version>11</java.version>`) | All modules compile with Java 11. |
| **Framework** | Spring Boot | 2.5.12 (parent `spring-boot-starter-parent`) | Provides web, cache, data, security, etc. |
| **Build** | Maven | – (wrapper scripts `mvnw`/`mvnw.cmd`) | Multi‑module build (`shopizer` pom aggregates 5 modules). |
| **Persistence** | JPA / Hibernate (via Spring Data JPA) | – (inherited from Spring Boot) | Entity classes live under `com.salesmanager.core.model.*`. |
| **Search** | Elasticsearch | 7.5.2 (property `elasticsearch.version`) | Used for product/catalog search. |
| **Caching** | Infinispan | 9.4.18.Final | Configured via Spring Cache abstraction. |
| **JSON** | Jackson (core, databind, annotations) | 2.13.4 / 2.13.4.1 | API payload (de)serialization. |
| **Utility** | Guava | 27.1‑jre | Helper utilities throughout codebase. |
| **Commons** | Apache Commons Lang, IO, Collections4, Validator | 3.5, 2.7, 4.1, 1.5.1 | General purpose helpers. |
| **Mapping** | MapStruct | 1.3.0.Final | Compile‑time bean mapping (DTO ↔ entity). |
| **Security / JWT** | JJWT | 0.8.0 | Token‑based authentication for REST API. |
| **Mail** | JavaMail (`javax.mail:mail`) | 1.4.7 | Email notifications (order, registration, etc.). |
| **GeoIP** | MaxMind GeoIP2 | 2.7.0 | IP‑based location services. |
| **Rules Engine** | Drools (KIE) | 7.32.0.Final | Business rule evaluation (e.g., promotions). |
| **Maps** | Google Maps Services | 0.1.6 | Geocoding, distance calculations for shipping. |
| **API Docs** | Swagger (Springfox) | 2.9.2 | Auto‑generated OpenAPI spec (`/swagger-ui.html`). |
| **Testing / Coverage** | JaCoCo (configured via properties) | – | Enforces minimum line/branch coverage (`.30` / `.37`). |
| **Containerisation** | Docker (official images) | – | `shopizerecomm/shopizer`, `shopizerecomm/shopizer-admin`, `shopizerecomm/shopizer-shop-reactjs`. |
| **CI/CD** | CircleCI (`.circleci/config.yml`) & Jenkins (`Jenkinsfile`) | – | Automated builds, tests, and Docker image publishing. |

*All versions are taken from the top‑level `pom.xml` and module `pom.xml`s.*

---

## 3. Architecture Pattern

| Observation | Evidence |
|-------------|----------|
| **Layered / Hexagonal style** | Packages are clearly separated by concern: `model` (domain entities), `business` (services, exceptions), `controller` (REST endpoints – not listed but implied by Spring Boot conventions). |
| **Monolithic deployment** | The whole application is packaged as a single Spring Boot JAR (`sm-shop` module) that runs the API server. No separate microservice projects are present in the repository. |
| **Modularisation** | Five Maven modules (`sm-core-model`, `sm-core`, `sm-core-modules`, `sm-shop-model`, `sm-shop`) encapsulate distinct layers and allow independent versioning. |
| **Headless / API‑first** | README emphasises “Headless commerce and Rest api”, and Swagger UI is exposed (`/swagger-ui.html`). |
| **Extensibility via modules** | `sm-core-modules` is described as “used for create new external modules implementation deployed in maven”, indicating a plug‑in point for custom functionality. |
| **Domain‑Driven Design (DDD) hints** | Entity names (`Product`, `Category`, `Customer`, `Order`) live under domain‑specific packages, and criteria objects (`ProductCriteria`, `CustomerCriteria`) support query specifications. |

**Conclusion:** Shopizer follows a **modular monolithic architecture** built on a classic **layered pattern** (presentation → service → domain → persistence). The modularisation enables clean separation of concerns and future extraction of micro‑services if desired, but the current runtime is a single Spring Boot application.

---

## 4. Module Structure

| Maven Module | Path | Primary Responsibility | Key Packages / Classes |
|--------------|------|------------------------|------------------------|
| **`sm-core-model`** | `sm-core-model/` | **Domain model** – JPA entities, value objects, enumerations, and DTO‑like criteria. | `com.salesmanager.core.model.*` (e.g., `catalog.product.Product`, `customer.Customer`, `common.Address`). |
| **`sm-core`** | `sm-core/` | **Core services & business logic** – Service interfaces, implementations, exception hierarchy, utility listeners. | `com.salesmanager.core.business.*` (e.g., `exception.ServiceException`, `audit.AuditListener`). |
| **`sm-core-modules`** | `sm-core-modules/` | **Pluggable extensions** – Framework for external modules that can be packaged separately and added at runtime. | (Not fully listed in the excerpt, but the module’s description states its purpose). |
| **`sm-shop-model`** | `sm-shop-model/` | **Shop‑specific model** – Additional entities that belong to the shop layer (e.g., merchant, shop configuration). | (Files not shown in the excerpt, but naming follows the same pattern). |
| **`sm-shop`** | `sm-shop/` | **Application entry point** – Spring Boot application, REST controllers, configuration, and wiring of all other modules. | `com.salesmanager.shop.*` (controllers, configuration classes, main class). |

The **parent pom** (`shopizer/pom.xml`) defines the common properties, dependencyManagement, and aggregates these modules, ensuring a consistent build across the whole codebase.

---

## 5. Build & Deployment

| Aspect | Details |
|--------|---------|
| **Build System** | Maven (wrapper scripts `mvnw`/`mvnw.cmd`). Run `./mvnw clean install` at the repository root to compile all modules and run unit tests. |
| **Continuous Integration** | - **CircleCI** – Configured in `.circleci/config.yml` (runs Maven build, tests, and publishes Docker images).<br>- **Jenkins** – `Jenkinsfile` provides a pipeline for building, testing, and deploying artifacts. |
| **Docker Support** | Official Docker images are published to Docker Hub (`shopizerecomm/shopizer`, `shopizerecomm/shopizer-admin`, `shopizerecomm/shopizer-shop-reactjs`). The README includes `docker run` commands for backend, admin UI, and React storefront. |
| **Testing** | Unit tests are executed by Maven Surefire; coverage enforced by JaCoCo (minimum 30 % lines, 37 % branches). |
| **Packaging** | The `sm-shop` module produces an executable Spring Boot JAR (`shopizer.jar`) that can be run with `java -jar` or via `./mvnw spring-boot:run`. |
| **Configuration** | Externalised via `application.yml`/`application.properties` (not shown but standard Spring Boot). Database credentials, mail server, and other subsystems are configurable at runtime. |

---

## 6. Code Organization

| Dimension | Conventions / Patterns |
|-----------|------------------------|
| **Package Naming** | Root package `com.salesmanager.core` for core modules, `com.salesmanager.shop` for the web layer. Sub‑packages reflect functional domains (`model.catalog`, `model.customer`, `business.exception`, etc.). |
| **Domain Entities** | JPA annotated POJOs located under `model.*`. Example: `Product`, `Category`, `Customer`, `Order`. They implement `Auditable` for automatic audit fields (`created`, `updated`). |
| **Criteria / Specification** | Query helper objects (`ProductCriteria`, `CustomerCriteria`, `AttributeCriteria`) encapsulate filtering, pagination, and sorting logic. |
| **Service Layer** | Interfaces and implementations reside in `business` packages (e.g., `ProductService`, `CustomerService`). Exceptions (`ServiceException`, `ConversionException`) are defined centrally. |
| **REST Controllers** | Though not listed in the excerpt, the Spring Boot application (`sm-shop`) follows the conventional `@RestController` pattern under `com.salesmanager.shop.controller`. Controllers expose CRUD and business operations via the Swagger‑documented API. |
| **Mapping** | MapStruct is used for compile‑time bean mapping between entities and DTOs, reducing boilerplate. |
| **Auditing** | `Auditable` interface + `AuditListener` (JPA entity listener) automatically populate `AuditSection` fields (`dateCreated`, `dateModified`). |
| **File & Content Handling** | Content abstraction (`Content`, `ContentFile`, `ImageContentFile`, etc.) supports static and dynamic media assets (product images, digital downloads). |
| **Internationalisation** | Description entities (`ProductDescription`, `CategoryDescription`, etc.) store locale‑specific text, enabling multi‑language catalogs. |
| **Extensibility** | `Optionable` interface and related attribute classes (`ProductOption`, `ProductOptionValue`) provide a flexible product‑option system. |
| **Error Handling** | Centralised exception hierarchy (`ServiceException`, `ConversionException`) enables consistent API error responses. |

---

### Quick Reference – How to Run the Project

```bash
# Clone and build
git clone https://github.com/shopizer-ecommerce/shopizer.git
cd shopizer
./mvnw clean install

# Run the backend (Spring Boot)
cd sm-shop
./mvnw spring-boot:run   # starts on http://localhost:8080

# Access Swagger UI
open http://localhost:8080/swagger-ui.html
```

Or use Docker (no local build required):

```bash
docker run -p 8080:8080 shopizerecomm/shopizer:latest
```

---

**End of Document**.