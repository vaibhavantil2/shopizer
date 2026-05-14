# Shipping and delivery management  

## Overview  
The shipping‑and‑delivery management feature calculates the available shipping options for an order at checkout. When the checkout flow asks for a quote (the **shipping‑quote request**), the system invokes one of the three concrete `ShippingQuoteModule` implementations – `CustomShippingQuoteRules`, `UPSShippingQuote` or `USPSShippingQuote`. Each implementation validates the request, builds the data required by its pricing engine (Drools rules, UPS XML API, or USPS XML API), calls the external service (or rule engine), parses the response and populates a `ShippingQuote` with a list of `ShippingOption` objects that contain price, service code, name and optional delivery‑date information. The populated `ShippingQuote` is then returned to the caller (typically the checkout controller) for display to the shopper.  

## Behavior  

| Step | What the code does (with source citation) |
|------|-------------------------------------------|
| **Trigger** | The checkout service calls `ShippingQuoteModule.getShippingQuotes(...)` on the configured module (e.g. `CustomShippingQuoteRules`, `UPSShippingQuote`, `USPSShippingQuote`). |
| **Input validation** | <ul><li>`Validate.notNull(delivery, …)` and `Validate.notNull(delivery.getCountry(), …)` ensure a delivery address and country are present (`CustomShippingQuoteRules.java:45‑46`).</li><li>`Validate.notNull(packages, …)` and `Validate.notEmpty(packages, …)` guarantee at least one package (`CustomShippingQuoteRules.java:47‑48`).</li><li>Both UPS and USPS implementations return `null` early if `delivery.getPostalCode()` is blank (`UPSShippingQuote.java:71`, `USPSShippingQuote.java:78`).</li></ul> |
| **Read configuration** | <ul><li>All modules receive `IntegrationConfiguration`, `IntegrationModule`, `ShippingConfiguration`, `MerchantStore`, and `Locale` arguments.</li><li>UPS validates required keys (`accessKey`, `userId`, `password`) and the presence of a `packages` option list (`UPSShippingQuote.java:84‑106`).</li><li>USPS validates the `account` key and a `packages` option list (`USPSShippingQuote.java:84‑106`).</li></ul> |
| **Calculate auxiliary data** | <ul><li>`CustomShippingQuoteRules` computes total weight, the largest package volume, and the largest single dimension across all `PackageDetails` (`CustomShippingQuoteRules.java:58‑78`).</li><li>UPS converts store weight/size units to the codes required by UPS (`UPSShippingQuote.java:158‑176`).</li><li>USPS converts dimensions to inches and weight to pounds using `DataUtils.getMeasure` / `DataUtils.getWeight` (`USPSShippingQuote.java:165‑190`).</li></ul> |
| **Build request object** | <ul><li>`CustomShippingQuoteRules` creates a `ShippingInputParameters` instance and populates weight, country, province/zone, distance (if already calculated) and volume (`CustomShippingQuoteRules.java:84‑106`).</li><li>UPS builds an XML request string that contains shipper address, destination address, package weight, dimensions and the selected packaging code (`UPSShippingQuote.java:210‑306`).</li><li>USPS builds either a domestic `<RateV3Request>` or an international `<IntlRateRequest>` XML payload with service, zip codes, weight, dimensions, container, size, and ship‑date (`USPSShippingQuote.java:226‑332`).</li></ul> |
| **External call / rule execution** | <ul><li>`CustomShippingQuoteRules` obtains a Drools `KieSession` from `DroolsBeanFactory` (`CustomShippingQuoteRules.java:112‑114`) and fires all rules (`CustomShippingQuoteRules.java:119‑122`).</li><li>UPS opens an `HttpPost` to the configured host/port/uri, sends the XML, and reads the response via a `ResponseHandler` (`UPSShippingQuote.java:322‑352`).</li><li>USPS opens an `HttpGet` with the XML encoded as a query parameter, executes it, and reads the response (`USPSShippingQuote.java:357‑384`).</li></ul> |
| **Parse response** | <ul><li>`CustomShippingQuoteRules` reads the global `DecisionResponse` populated by the Drools rules; if `customPrice` is set, it creates a `ShippingOption` with that price and the module code (`CustomShippingQuoteRules.java:124‑138`).</li><li>UPS uses an Apache `Digester` to map `<RatedShipment>` elements to `ShippingOption` objects, extracting service code, price text, and estimated delivery days (`UPSShippingQuote.java:380‑416`).</li><li>USPS uses a `Digester` to map `<Postage>` (domestic) or `<Service>` (international) elements to `ShippingOption` objects, extracting name, code, and price text (`USPSShippingQuote.java:426‑466`).</li></ul> |
| **Populate `ShippingQuote`** | <ul><li>If the quote’s `shippingOptions` list is `null`, `CustomShippingQuoteRules` creates a new `ArrayList` and stores it back into the `ShippingQuote` (`CustomShippingQuoteRules.java:108‑112`).</li><li>UPS and USPS return the list of `ShippingOption` objects directly; the caller assigns them to the `ShippingQuote`.</li></ul> |
| **Success path** | The method returns a non‑null `List<ShippingOption>` containing one or more populated options. |
| **Validation‑failure path** | If required fields are missing (e.g., null delivery, missing postal code, missing integration keys) the method either throws `IntegrationException` (UPS/USPS) or returns `null` (Custom). |
| **Downstream‑failure path** | <ul><li>UPS: non‑2xx HTTP status → `ClientProtocolException` → wrapped in `IntegrationException` (`UPSShippingQuote.java:340‑345`).</li><li>USPS: non‑2xx HTTP status → `ClientProtocolException` → wrapped (`USPSShippingQuote.java:371‑376`).</li><li>Both parsers check for error nodes (`<Error>` or `<ErrorDescription>`) and throw `IntegrationException` with the parsed message (`UPSShippingQuote.java:424‑432`, `USPSShippingQuote.java:447‑456`).</li></ul> |

## Triggers / Entry points  

| Module | Entry point (file:line) |
|--------|------------------------|
| Custom rule‑based quotes | `CustomShippingQuoteRules.getShippingQuotes(…)` – `./sm-core/src/main/java/com/salesmanager/core/business/modules/integration/shipping/impl/CustomShippingQuoteRules.java:38` |
| UPS real‑time quotes | `UPSShippingQuote.getShippingQuotes(…)` – `./sm-core/src/main/java/com/salesmanager/core/business/modules/integration/shipping/impl/UPSShippingQuote.java:58` |
| USPS real‑time quotes | `USPSShippingQuote.getShippingQuotes(…)` – `./sm-core/src/main/java/com/salesmanager/core/business/modules/integration/shipping/impl/USPSShippingQuote.java:58` |

These methods are invoked by the checkout service (not shown in the provided sources) when the platform needs a shipping quote.

## End‑to‑end flow (Mermaid)

```mermaid
sequenceDiagram
    participant UI as "Checkout UI"
    participant CheckoutService as "Checkout Service"
    participant QuoteModule as "ShippingQuoteModule"
    participant Custom as "CustomShippingQuoteRules"
    participant Drools as "Drools KieSession"
    participant UPS as "UPSShippingQuote"
    participant UPS_HTTP as "UPS HTTP API"
    participant USPS as "USPSShippingQuote"
    participant USPS_HTTP as "USPS HTTP API"
    participant Quote as "ShippingQuote"
    participant Option as "ShippingOption"

    UI->>CheckoutService: request shipping quote (order, packages, address)
    CheckoutService->>QuoteModule: getShippingQuotes(...)
    alt Custom module selected
        QuoteModule->>Custom: getShippingQuotes(...)
        Custom->>Custom: validate inputs (delivery, packages)   %% CustomShippingQuoteRules.java:45‑48
        Custom->>Custom: compute weight/volume/size                %% CustomShippingQuoteRules.java:58‑78
        Custom->>Drools: getKieSession & fire rules                %% CustomShippingQuoteRules.java:112‑122
        Drools-->>Custom: DecisionResponse (customPrice)
        alt customPrice present
            Custom->>Option: new ShippingOption (price, codes)      %% CustomShippingQuoteRules.java:124‑138
            Custom->>Quote: add option to quote.options
        end
        Custom-->>QuoteModule: return List<ShippingOption>
    else UPS module selected
        QuoteModule->>UPS: getShippingQuotes(...)
        UPS->>UPS: validate keys & options                      %% UPSShippingQuote.java:84‑106
        UPS->>UPS: build XML request                             %% UPSShippingQuote.java:210‑306
        UPS->>UPS_HTTP: POST XML
        UPS_HTTP-->>UPS: XML response
        UPS->>UPS: parse response with Digester                  %% UPSShippingQuote.java:380‑416
        UPS->>Option: populate ShippingOption objects
        UPS->>Quote: set options list
        UPS-->>QuoteModule: return List<ShippingOption>
    else USPS module selected
        QuoteModule->>USPS: getShippingQuotes(...)
        USPS->>USPS: validate keys & options                     %% USPSShippingQuote.java:84‑106
        USPS->>USPS: build XML request (RateV3 or IntlRate)     %% USPSShippingQuote.java:226‑332
        USPS->>USPS_HTTP: GET with encoded XML
        USPS_HTTP-->>USPS: XML response
        USPS->>USPS: parse response with Digester                %% USPSShippingQuote.java:426‑466
        USPS->>Option: populate ShippingOption objects
        USPS->>Quote: set options list
        USPS-->>QuoteModule: return List<ShippingOption>
    end
    QuoteModule-->>CheckoutService: List<ShippingOption>
    CheckoutService-->>UI: display shipping options
```

## State / data touched  

| Entity | File (line) | How it is used |
|--------|--------------|----------------|
| `ShippingQuote` | `ShippingQuote.java:31‑44` | Holds the list of `ShippingOption` objects; created by the checkout service and passed into the module. |
| `ShippingOption` | `ShippingOption.java:31‑84` | Populated with price, code, name, etc.; returned to the caller. |
| `PackageDetails` | (model, not shown) | Input list describing each package’s weight, dimensions; iterated in all three modules (`CustomShippingQuoteRules.java:58‑78`, `UPSShippingQuote.java:226‑306`, `USPSShippingQuote.java:165‑190`). |
| `IntegrationConfiguration` | (model, not shown) | Provides API keys and option lists; read in UPS/USPS validation blocks (`UPSShippingQuote.java:84‑106`, `USPSShippingQuote.java:84‑106`). |
| `IntegrationModule` | (model, not shown) | Supplies module‑specific configuration (host, port, uri, region set) used by UPS/USPS to locate the external endpoint (`UPSShippingQuote.java:124‑146`, `USPSShippingQuote.java:124‑146`). |
| `MerchantStore` | (model, not shown) | Supplies store country, zone, weight/size unit codes, address fields used when building XML (`UPSShippingQuote.java:158‑176`, `USPSShippingQuote.java:226‑306`). |
| `DroolsBeanFactory` | `CustomShippingQuoteRules.java:27‑30` | Provides the Drools `KieSession` used for rule‑based pricing (`CustomShippingQuoteRules.java:112‑122`). |

No persistent database writes occur in these classes; they only read configuration objects and write to the in‑memory `ShippingQuote` instance.

## External dependencies  

| Dependency | Where it is invoked (file:line) |
|------------|---------------------------------|
| Drools KIE session | `CustomShippingQuoteRules.java:112‑122` |
| Apache HttpClient (POST) | `UPSShippingQuote.java:322‑352` |
| Apache HttpClient (GET) | `USPSShippingQuote.java:357‑384` |
| Apache Digester (XML parsing) | `UPSShippingQuote.java:380‑416` and `USPSShippingQuote.java:426‑466` |
| `DataUtils` (unit conversion) | `UPSShippingQuote.java:158‑176`, `USPSShippingQuote.java:165‑190` |
| `ProductPriceUtils` (formatting) | `USPSShippingQuote.java:254‑259` |
| `org.kie.api.runtime.KieSession` | `CustomShippingQuoteRules.java:112` |
| `org.apache.commons.lang3.Validate` | validation in all three modules (`CustomShippingQuoteRules.java:45‑48`, `UPSShippingQuote.java:71`, `USPSShippingQuote.java:78`) |

## Configuration / parameters  

| Parameter | Source location | Effect |
|-----------|----------------|--------|
| Drools rule file `PriceByDistance.drl` | Loaded via `ResourceFactory.newClassPathResource("com/salesmanager/drools/rules/PriceByDistance.drl")` (`CustomShippingQuoteRules.java:112`). |
| UPS integration keys: `accessKey`, `userId`, `password` | Expected in `IntegrationConfiguration.integrationKeys` (`UPSShippingQuote.java:84‑106`). |
| UPS `packages` option list | `IntegrationConfiguration.integrationOptions.get("packages")` (`UPSShippingQuote.java:84‑106`). |
| USPS integration key: `account` | `IntegrationConfiguration.integrationKeys.get("account")` (`USPSShippingQuote.java:84‑106`). |
| USPS `packages` option list | Same as UPS (`USPSShippingQuote.java:84‑106`). |
| Store weight unit code (`KG` → `KGS`, else `LBS`) | `UPSShippingQuote.java:158‑166`. |
| Store size unit code (`IN` for inches) | `UPSShippingQuote.java:168‑176`. |
| Locale language fallback to English | Both UPS and USPS (`UPSShippingQuote.java:124‑130`, `USPSShippingQuote.java:124‑130`). |
| Region whitelist (`module.getRegionsSet()`) | UPS validates store country against allowed regions (`UPSShippingQuote.java:124‑130`). |
| Module environment (`moduleConfig.getEnv()`) | Used to pick host/port/uri for UPS/USPS (`UPSShippingQuote.java:138‑146`, `USPSShippingQuote.java:138‑146`). |
| Distance key (`Constants.DISTANCE_KEY`) | If present in `quote.getQuoteInformations()` it is passed to Drools (`CustomShippingQuoteRules.java:53‑61`). |

## Edge cases & failure modes (observed in code)  

| Situation | Handling (source) |
|-----------|-------------------|
| Missing delivery or country | `Validate.notNull` throws `IllegalArgumentException` (`CustomShippingQuoteRules.java:45‑46`). |
| Empty or null package list | `Validate.notEmpty` throws `IllegalArgumentException` (`CustomShippingQuoteRules.java:47‑48`). |
| Blank postal code | Method returns `null` early (`UPSShippingQuote.java:71`, `USPSShippingQuote.java:78`). |
| UPS/USPS integration keys missing | Validation builds `errorFields` list and throws `IntegrationException` (`UPSShippingQuote.java:84‑106`, `USPSShippingQuote.java:84‑106`). |
| Store country not allowed for UPS | Returns `null` (`UPSShippingQuote.java:124‑130`). |
| HTTP non‑2xx response from UPS/USPS | Logs error and throws `ClientProtocolException` wrapped in `IntegrationException` (`UPSShippingQuote.java:340‑345`, `USPSShippingQuote.java:371‑376`). |
| XML response contains `<Error>` nodes | Parsed into `UPSParsedElements` / `USPSParsedElements`; if present, an `IntegrationException` is thrown (`UPSShippingQuote.java:424‑432`, `USPSShippingQuote.java:447‑456`). |
| No shipping options returned by external API | Throws `IntegrationException` with the parsed error message (`UPSShippingQuote.java:438‑442`, `USPSShippingQuote.java:459‑463`). |
| `DecisionResponse.customPrice` missing | No option is added; method returns the (possibly empty) list (`CustomShippingQuoteRules.java:124‑138`). |
| Missing `optionPriceText` when building `ShippingOption` | `ShippingOption.getOptionPrice()` lazily parses the text; if parsing fails, an error is logged (`ShippingOption.java:31‑44`). |

## Open questions  

* **Drools rule details** – The rule file `PriceByDistance.drl` is loaded but its contents are not part of the provided sources, so the exact pricing logic is unknown. (`CustomShippingQuoteRules.java:112`).  
* **How the selected module is chosen** – The code that decides whether to use the custom, UPS, or USPS implementation (e.g., based on store configuration or shipping address) is not included in the supplied files.  
* **Currency conversion** – Both UPS and USPS parsers contain commented‑out code for currency conversion, but the active path never performs conversion; it is unclear whether conversion happens elsewhere.  
* **Retry / timeout policy** – The HTTP clients are created with default settings; there is no explicit retry or timeout configuration visible.  
* **Persistence of the quote** – The `ShippingQuote` object is populated in memory; any later persistence (e.g., saving to DB or caching) is not shown in the provided code.  
* **Handling of multiple packages** – The code loops over `packages` to build XML, but the response parsing assumes a single `ShippingOption` per request; how multiple‑package quotes are aggregated is not evident.  

These points would require additional source files (controller layer, configuration loader, Drools rule definitions, or persistence services) to answer definitively.