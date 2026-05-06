# Standards & API Reference

> Project: Car Dealership DMS · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO 3779:2009 — Road Vehicles: Vehicle Identification Number (VIN) — Content and Structure**
- URL: https://www.iso.org/standard/52200.html
- Defines the 17-character VIN structure used worldwide, including World Manufacturer Identifier (WMI, positions 1–3), Vehicle Descriptor Section (VDS, positions 4–9), and Vehicle Identifier Section (VIS, positions 10–17). Every DMS must store, validate, and decode VINs conforming to this standard when recording vehicle inventory, sales, and service records.

**ISO 4030 — Road Vehicles: VIN Location and Attachment**
- URL: https://www.iso.org/standard/
- Specifies where the VIN must physically appear on the vehicle. Relevant to DMS service and compliance workflows that require verification of the VIN against physical inspection records.

**ISO 3780 — Road Vehicles: World Manufacturer Identifier (WMI)**
- URL: https://www.iso.org/standard/
- Defines how the first three characters of a VIN are assigned to vehicle manufacturers by country. Required by any DMS performing manufacturer lookups or OEM-level reporting.

**ISO 11898 — Road Vehicles: Controller Area Network (CAN)**
- URL: https://www.iso.org/standard/
- Specifies the physical and data-link layers for the CAN bus used in vehicle telematics. Relevant to DMS modules that receive telematics data (mileage, fault codes) directly from vehicles or via third-party telematics providers.

**IATF 16949 — Quality Management Systems: Automotive Production**
- URL: https://www.iatf.com/
- Quality management standard for automotive production. Relevant to DMS deployments for OEM-affiliated dealers who must demonstrate process compliance as part of their franchise agreements.

---

### W3C & IETF Standards

**RFC 9110 / HTTP Semantics**
- URL: https://www.rfc-editor.org/rfc/rfc9110
- The foundational HTTP/1.1 specification governing request/response semantics. All REST-based DMS APIs (CDK, Tekion, Dealertrack Opentrack) operate over HTTP and must conform to correct use of methods (GET, POST, PUT, DELETE), status codes, and headers.

**RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://www.rfc-editor.org/rfc/rfc6749
- The standard authorization framework used by modern DMS API platforms (Fortellis, Tekion APC, Cox developer portal) for delegated third-party access. DMS integrations use OAuth 2.0 client credentials and authorization code flows to authenticate partner applications without sharing dealer credentials.

**RFC 7519 — JSON Web Tokens (JWT)**
- URL: https://www.rfc-editor.org/rfc/rfc7519
- Defines compact, self-contained tokens used for transmitting claims between DMS APIs and integration partners. Widely used alongside OAuth 2.0 in automotive API platforms such as Fortellis and the Tekion Automotive Partner Cloud.

**RFC 8288 — Web Linking**
- URL: https://www.rfc-editor.org/rfc/rfc8288
- Defines link relations used in hypermedia APIs. Relevant to DMS APIs that implement HATEOAS-style navigation for deal, inventory, and service record resources.

**RFC 5322 — Internet Message Format (Email)**
- URL: https://www.rfc-editor.org/rfc/rfc5322
- Standard email format underpinning DMS customer communication features, lead notifications, and appointment confirmations.

---

### Data Model & API Specifications

**STAR Automotive Retail Domain Model (2026)**
- URL: https://www.starstandard.org/
- The authoritative interoperability standard for the automotive retail industry, published by Standards for Technology in Automotive Retail (STAR). The 2026 Domain Model defines canonical JSON schemas and OpenAPI-aligned specifications across core DMS domains: Sales, Service, Parts, Accounts Payable, Accounting, Payroll, and Human Resources. Schemas are available for free download and are already in production use at OEMs including Nissan North America and Toyota Motor North America.

**STAR Sales Process Standards API (v6.1.4, 2023)**
- URL: https://www.starstandard.org/index.php/2023/04/25/star-releases-industry-first-sales-process-standards-api-alongside-updated-star6-xml/
- Industry-first standardised API for the vehicle sales process, covering deal creation, consumer information, vehicle specifications, pricing, and financing. Includes both STAR6 XML (legacy) and JSON representations. The Deal API supports create, retrieve, and update operations for automotive deals.

**STAR Service Scheduling API**
- URL: https://www.starstandard.org/index.php/service-scheduling-api/
- Open standard API for scheduling dealership service appointments, covering dealer identifier, vehicle information, service advisor availability, recommended service list, transportation options, and appointment history. Available free to businesses worldwide.

**STAR Fleet Vehicle Rentals & Loyalty Customer Lead APIs (2026)**
- URL: https://www.starstandard.org/index.php/2026/01/20/star-announces-new-apis-for-automotive-fleet-vehicle-rentals-and-loyalty-customer-leads/
- New APIs for fleet rental transactions and synchronising customer lead data between dealer CRM systems and OEM systems.

**STAR Retail Delivery Report (RDR) API (2026)**
- URL: https://www.starstandard.org/index.php/2026/04/15/advancing-industry-standards-retail-delivery-reporting-now-part-of-the-star-domain-model/
- Standard for reporting vehicle retail deliveries to OEMs, now part of the STAR Domain Model. Required by many franchise dealer agreements for warranty registration and OEM incentive reconciliation.

**OpenAPI Specification 3.1.1**
- URL: https://spec.openapis.org/oas/v3.1.1.html
- The de-facto standard for describing REST APIs. STAR's new domain model is aligned with JSON/OpenAPI practices. All major DMS API platforms (Tekion, CDK Fortellis, Dealertrack) expose or accept OpenAPI-described endpoints.

**JSON Schema (Draft 2020-12)**
- URL: https://json-schema.org/
- Standard for validating JSON data structures. Used extensively in STAR domain models and by DMS integration middleware to validate request/response payloads before routing to dealership systems.

**SAE J2540 — Terms and Definitions for Automotive Retail Software**
- URL: https://www.sae.org/
- SAE International standard defining terminology for the automotive retail software domain, used as a reference for consistent naming of DMS entities such as repair orders, deal jacket, and parts transactions.

---

### Security & Authentication Standards

**FTC Safeguards Rule (16 CFR Part 314, amended 2023/2025)**
- URL: https://www.ftc.gov/legal-library/browse/rules/safeguards-rule
- Mandates that franchised and independent auto dealers implement a comprehensive written information security programme (WISP) protecting customer financial data held in the DMS. Specific requirements include multi-factor authentication (MFA) for all DMS access, encryption of customer data at rest and in transit, continuous monitoring, annual penetration testing, and contractual data-security obligations for all third-party DMS vendors and integration partners. Violations carry fines up to USD 100,000 per violation.

**Gramm-Leach-Bliley Act (GLBA) — Financial Privacy Rule**
- URL: https://www.ftc.gov/business-guidance/privacy-security/gramm-leach-bliley-act
- Federal law governing the sharing of non-public personal financial information (NPI) by entities engaged in financial activities, including auto dealers who arrange financing. DMS must implement GLBA-compliant notice and opt-out workflows, and restrict data sharing with non-affiliated third parties.

**OFAC SDN List Compliance (31 CFR Part 500+)**
- URL: https://ofac.treasury.gov/
- Requires dealers to screen customers and transactions against the Office of Foreign Assets Control Specially Designated Nationals list before completing financed sales. As of March 2025, the statute of limitations was extended to 10 years, requiring DMS audit-log retention policies to be updated accordingly.

**IRS Form 8300 Reporting**
- URL: https://www.irs.gov/businesses/small-businesses-self-employed/form-8300-and-reporting-cash-payments-of-over-10000
- Cash transactions exceeding USD 10,000 must be reported to the IRS within 15 days. DMS must detect qualifying cash transactions, generate Form 8300 data, and maintain records for 5 years.

**NIST Cybersecurity Framework (CSF) 2.0**
- URL: https://www.nist.gov/cyberframework
- Widely referenced security framework for DMS risk management and FTC Safeguards Rule compliance programmes. Provides Identify, Protect, Detect, Respond, and Recover function categories applied to dealer data environments.

**SOC 2 Type II**
- URL: https://www.aicpa-cima.com/resources/landing/system-and-organization-controls-soc-suite-of-services
- Industry certification required by many franchise groups and OEMs as a condition for approving DMS vendors. Demonstrates audited security controls around availability, confidentiality, and processing integrity of dealer data.

**ISO/IEC 27001 — Information Security Management**
- URL: https://www.iso.org/standard/27001
- International standard for information security management systems (ISMS). DMS vendors increasingly hold ISO 27001 certification to meet enterprise dealer group and OEM data security requirements.

**AAMVA Data Element Dictionary (D20)**
- URL: https://www.aamva.org/technology/technology-standards
- Published by the American Association of Motor Vehicle Administrators; defines standard data elements for driver licences, vehicle titles, and registration records to ensure uniform exchange between DMV systems and systems integrating with them (including DMS titling modules).

---

### MCP Server Specifications

The Model Context Protocol (MCP) is directly relevant to AI-native DMS implementations that expose dealership data to AI agents (inventory advisors, service scheduling bots, pricing assistants). Key considerations:

- **MCP Tool Definitions** — DMS entities (repair orders, deals, inventory) should be exposed as MCP tools with structured input/output schemas so that LLM agents can query and update dealership records through natural language workflows.
- **MCP Resources** — Read-only DMS data (vehicle inventory, customer history, parts catalogue) can be exposed as MCP resources for grounding AI responses without write-access risks.
- **MCP Prompt Templates** — Standardised prompts for common dealership workflows (write-up, trade-in appraisal, service advisor upsell) can be defined as MCP prompt resources.
- MCP specification: https://modelcontextprotocol.io/

---

## Similar Products — Developer Documentation & APIs

### CDK Global (Fortellis)

- **Description:** The largest DMS provider by install base, serving over 30,000 dealerships worldwide. CDK exposes its DMS via the Fortellis Automotive Commerce Exchange platform, an open, agnostic API marketplace connecting DMS data to third-party applications.
- **API Documentation:** https://apidocs.fortellis.io/
- **Developer Network:** https://developer.fortellis.io/
- **API Platform:** https://fortellis.io/
- **Standards:** REST/JSON, OpenAPI; async event-driven APIs also available
- **Authentication:** OAuth 2.0 (client credentials and authorization code flows)
- **Key APIs:** Repair Order API, CRM API, Payment Settling API, Async Event APIs for real-time DMS data streaming
- **Notes:** CDK APIs are distributed via the Fortellis marketplace; third-party developers register as solution providers to access dealer-authorised data.

---

### Tekion Automotive Partner Cloud (APC)

- **Description:** Cloud-native, AI-first DMS launched by a former Apple engineering leader. Tekion's Automotive Partner Cloud (APC) provides tiered API access for integration partners.
- **API Documentation / Partner Portal:** https://preprod-apc.tekioncloud.com/user/home
- **Contact for Access:** APCintegrations@tekion.com
- **Standards:** REST/JSON, OpenAPI-aligned
- **Authentication:** OAuth 2.0
- **API Tiers:**
  - Standard Open APIs — near-real-time read access to sales, service, inventory, and parts data
  - Elevated APIs — read/write with enhanced workflow support and reference data
  - Premium APIs — full data mapping for established technology partners
- **Notes:** Data dictionary provided for all API fields; most suitable for modern cloud-native integrations.

---

### Dealertrack (Cox Automotive) — Opentrack

- **Description:** Cox Automotive's DMS with over 375 integration points via its Opentrack open API programme. Offers real-time, bidirectional integration with 275+ vendors across CRM, inventory, desking, service scheduling, and F&I.
- **API Documentation:** https://us.dealertrack.com/resources/dealertrack-dms-opentrack-integration/
- **Developer Portal:** https://developer.coxautoinc.com/
- **Standards:** REST/JSON, OpenAPI
- **Authentication:** OAuth 2.0
- **Key Integrations:** VinSolutions CRM, Kelley Blue Book pricing, Manheim vehicle auctions, RouteOne/DealerTrack finance desking
- **Notes:** Opentrack programme certifies third-party integrators; partners receive structured data access with lower data-access fees than legacy DMS providers.

---

### Cox Automotive Developer Platform

- **Description:** Umbrella developer platform exposing APIs from Cox brands: Autotrader, Kelley Blue Book (KBB), Manheim, vAuto, VinSolutions, Dealertrack, HomeNet, and Xtime.
- **Developer Portal:** https://developer.coxautoinc.com/
- **Interactive API Docs:** https://coxautoinc.mashery.com/interactive-documentation
- **Standards:** REST/JSON; GraphQL used internally for some Cox product aggregation (Cox uses Apollo GraphQL)
- **Authentication:** OAuth 2.0, API Keys
- **Key API Products:** Vehicle pricing and incentives (KBB), used-vehicle market data (vAuto), inventory listings (HomeNet), vehicle history/title (AutoCheck), wholesale auction (Manheim), service scheduling (Xtime)

---

### Reynolds & Reynolds (ERA-IGNITE / POWER)

- **Description:** One of the "big three" DMS providers with deep F&I, compliance, and document-management capabilities. Access to Reynolds data has historically been more restricted than competitors, with the official Reynolds Certified Interface (RCI) programme carrying significant entry costs.
- **API / Integration Info:** https://supergood.ai/docs/reynolds-and-reynolds-api
- **Partner Integration Guides:** https://us.dealertrack.com/wp-content/uploads/sites/2/2020/06/digital-contracting-dms-integration-guide-reynolds-reynolds.pdf
- **Standards:** Proprietary XML/SOAP (legacy); limited REST endpoints via certified partners
- **Authentication:** Proprietary; partner certification required via RCI programme
- **Notes:** Integration costs historically high (RCI entry fees reportedly up to USD 80,000). Open API access significantly more limited compared to CDK/Tekion/Dealertrack. Third-party middleware (e.g., Supergood) provides abstraction layers.

---

### DealerSocket (Solera)

- **Description:** CRM and DMS provider offering both inbound and outbound APIs for connecting dealership data with third-party tools. Strong in CRM, inventory management, and F&I.
- **API Documentation:** https://dealersocket.com/apis/
- **Standards:** REST/JSON; XML lead forwarding also supported
- **Authentication:** OAuth 2.0
- **Key APIs:** Lead Forwarding Service (XML or email), Deal Push (desking/financial data), CRM customer sync, Inventory data feeds

---

### PBS Systems (Partner Hub)

- **Description:** Cloud DMS focused on Canadian and US dealerships, with an open Partner Hub for third-party integrations covering customers, vehicles, appointments, parts invoices, and repair orders.
- **API Documentation:** https://partnerhub.pbsdealers.com/
- **Standards:** REST/JSON; SOAP also supported; direct XML/JSON HTTP posts accepted
- **Authentication:** API key / partner certification
- **Notes:** One of the more openly documented DMS APIs available without enterprise procurement barriers.

---

### DealerBuilt (ceDMS)

- **Description:** Independent DMS offering real-time read/write API access to sales, service, customer, and accounting data via its Open Integration Platform.
- **API Documentation:** https://dealerbuilt.com/dms-platform/open-integration-platform/
- **Standards:** REST/JSON
- **Authentication:** API key / OAuth 2.0
- **Notes:** Joined STAR standards organisation in early 2026, signalling commitment to interoperability.

---

### NHTSA APIs (US Government)

- **Description:** Free public APIs from the National Highway Traffic Safety Administration providing vehicle safety and specification data.
- **API Portal:** https://www.nhtsa.gov/nhtsa-datasets-and-apis
- **vPIC (VIN Decoder) API:** https://vpic.nhtsa.dot.gov/api/
- **Recalls API:** https://api.nhtsa.gov/recalls
- **Standards:** REST/JSON, no authentication required (public)
- **Key Capabilities:**
  - Decode any VIN to make, model, year, body style, engine, and 100+ vehicle attributes
  - Retrieve all open safety recall campaigns for a vehicle by VIN, make/model/year
  - Safety ratings (NCAP star ratings) by VIN
- **Notes:** Free, no rate-limit registration required for standard use. Essential baseline data layer for any DMS vehicle inventory module.

---

### AAMVA NMVTIS (National Motor Vehicle Title Information System)

- **Description:** Government-operated system connecting state DMV title and registration databases. Dealers and DMS providers integrate with NMVTIS to verify vehicle title history, detect odometer fraud, and confirm salvage/junk status before completing purchases or sales.
- **Programme Info:** https://www.aamva.org/technology/systems/vehicle-systems/nmvtis
- **Standards:** XML/SOAP (government batch and real-time); third-party API wrappers available (e.g., MarketCheck VINData)
- **Authentication:** State/agency credentials; commercial access via approved data providers
- **Notes:** Mandatory reporting for junk/salvage yards and insurance carriers; optional but highly recommended for dealer used-vehicle acquisition workflows.

---

### RouteOne

- **Description:** Finance and insurance (F&I) platform connecting dealers with 1,400+ lenders. RouteOne's API enables deal data exchange, credit application submission, and automated decisioning between DMS/desking tools and lenders.
- **Integration Partner Docs:** https://routeone.com/integration-partners/features
- **Standards:** REST/JSON and legacy XML
- **Authentication:** Partner certification required
- **Key APIs:** Credit Application Exchange, Deal Refresh, Auto Submit (automated lender routing), Loyalty Customer Lead sync

---

## Notes

**Fragmentation remains the primary technical challenge.** Despite STAR's progress in publishing open JSON/OpenAPI schemas, the three dominant legacy DMS providers (CDK, Reynolds & Reynolds, Dealertrack) each maintain proprietary integration programmes with varying levels of openness, cost, and latency. An AI-native open-source DMS has the opportunity to adopt STAR standards natively from day one, avoiding the retrofit complexity that incumbents face.

**The FTC Safeguards Rule (2023/2025 amendments) has materially raised the security baseline** required of any DMS vendor or integration partner. MFA, encryption at rest, penetration testing, and formal vendor management programmes are now compliance requirements, not optional security practices.

**STAR's 2026 Domain Model represents a significant milestone.** For the first time, core back-office domains (Accounting, Payroll, HR, Accounts Payable) have published interoperability schemas alongside the existing sales and service APIs. An open-source DMS that implements these schemas will have a structural interoperability advantage over proprietary incumbents.

**NHTSA's free public APIs** (VIN decoder, recalls, safety ratings) provide a zero-cost, no-authentication data layer for vehicle inventory enrichment that every DMS should integrate as a baseline.
