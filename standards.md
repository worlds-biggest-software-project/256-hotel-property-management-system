# Standards & API Reference

> Project: Hotel Property Management System · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 27001:2022 — Information Security Management Systems**
- URL: https://www.iso.org/standard/27001
- The primary information security standard required by hotel groups and OTAs as a condition of PMS integration partnerships. PMS vendors are increasingly required to hold ISO 27001 certification for procurement approval by major chains and distribution partners.

**ISO/IEC 27701:2019 — Privacy Information Management**
- URL: https://www.iso.org/standard/71670.html
- Extension of ISO 27001 addressing privacy information management (PIMS). Directly relevant to handling of guest personal data under GDPR and CCPA, mapping privacy controls onto the ISMS framework.

**ISO/IEC 29101:2018 — Privacy Architecture Framework**
- URL: https://www.iso.org/standard/45269.html
- Provides a privacy architecture framework for information and communication technology systems that process personally identifiable information (PII). Relevant to the design of guest profile storage, consent management, and data lifecycle.

**ISO 22300:2021 — Security and Resilience — Vocabulary**
- URL: https://www.iso.org/standard/77008.html
- Baseline terminology standard for business continuity and security planning. Relevant to disaster-recovery architecture for a cloud-hosted PMS.

---

### W3C & IETF Standards

**RFC 7231 — HTTP/1.1 Semantics and Content**
- URL: https://datatracker.ietf.org/doc/html/rfc7231
- Defines the semantics of HTTP methods, status codes, and headers. Foundational to all REST API design within a PMS integration platform.

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- OAuth 2.0 is the de-facto authentication and authorisation standard for hotel PMS APIs. Oracle OPERA Cloud (OHIP), Mews, Apaleo, and Cloudbeds all implement OAuth 2.0. Mandatory for any PMS exposing public APIs.

**RFC 7519 — JSON Web Token (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- JWT is the token format used for short-lived access tokens in OAuth 2.0 flows across the PMS ecosystem (OPERA OHIP, Apaleo). Token expiry checking and renewal patterns are documented in vendor guides.

**RFC 7517 — JSON Web Key (JWK) / RFC 7518 — JSON Web Algorithms (JWA)**
- URL: https://datatracker.ietf.org/doc/html/rfc7517
- Supporting standards for cryptographic key management in JWT-based authentication schemes.

**RFC 8288 — Web Linking**
- URL: https://datatracker.ietf.org/doc/html/rfc8288
- Defines the `Link` header for pagination and resource relationships in REST APIs. Relevant to large dataset APIs (reservations, guest profiles) in a PMS.

**W3C WebAuthn (FIDO2)**
- URL: https://www.w3.org/TR/webauthn-2/
- Passwordless authentication standard increasingly adopted for staff portal and kiosk login within cloud PMS platforms.

---

### Data Model & API Specifications

**OpenTravel Alliance (OTA) Specifications — OTA 1.0 (XML) and OTA 2.0 (JSON / OAS 3.0)**
- URL: https://opentravel.org/
- The foundational data-exchange standard for hospitality since 1999. OTA 1.0 uses XML schemas (.xsd); OTA 2.0 adds JSON (OpenAPI Specification 3.0) via the DEx transformation tool. Covers hotel availability, reservations, pricing, check-in/checkout, key management, upsell, and profiles. All major GDS and CRS connections reference OTA schemas. The 2024A release updated XML messages; OTA 2.0 JSON release expected in 2025/2026.
- Download: https://opentravel.org/download-the-opentravel-specification/

**HTNG (Hospitality Technology Next Generation) Specifications**
- URL: https://www.ahla.com/htng-completed-workgroups
- HTNG develops hospitality-specific integration specifications built on OTA schemas. Key specifications include: PMS-to-POS connectivity, Customer Profile standards, and the HTNG Express specification for lightweight post-booking and operational data exchange (not full PMS integration). HTNG collaborated with OpenTravel and HEDNA (Hotel Electronic Distribution Network Association) via the Open Payments Alliance.
- HTNG Express GitHub: https://github.com/HTNG/htng-express

**OpenAPI Specification 3.x (OAS)**
- URL: https://spec.openapis.org/oas/v3.1.0
- The REST API description format used by Mews (Connector API), Cloudbeds (Python SDK generated via OpenAPI Generator), Oracle OPERA (OHIP API specs published on GitHub), and Apaleo. The universal standard for machine-readable API contracts in the PMS space.

**JSON Schema (Draft 2020-12)**
- URL: https://json-schema.org/
- Used for validating request and response payloads in PMS REST APIs. Referenced by OpenAPI 3.1 for component schema definitions.

**iCalendar / RFC 5545**
- URL: https://datatracker.ietf.org/doc/html/rfc5545
- Standard format for calendar data exchange. Relevant to exporting reservation and housekeeping schedules to external calendar applications and staff planning tools.

---

### Security & Authentication Standards

**PCI DSS (Payment Card Industry Data Security Standard) v4.0**
- URL: https://www.pcisecuritystandards.org/
- Mandatory compliance framework for any PMS that processes, stores, or transmits credit card data. Non-compliance fines range from $5,000 to $100,000/month imposed by payment processors. PMS requirements include: encrypted cardholder data at rest and in transit, strong password policies, MFA, role-based access controls, regular security assessments, and network segmentation. All major commercial PMS vendors (OPERA, Mews, Cloudbeds, RoomKeyPMS) are PCI DSS certified.

**GDPR — General Data Protection Regulation (EU) 2016/679**
- URL: https://gdpr-info.eu/
- Applies to any PMS processing personal data of EU guests regardless of the hotel's location. Key requirements: lawful basis for processing, explicit consent collection, data subject rights (access, erasure, portability), data retention policies, breach notification within 72 hours. Fines up to €20M or 4% of global annual turnover. PMS must implement consent management, data minimisation, and retention automation.

**CCPA — California Consumer Privacy Act**
- URL: https://oag.ca.gov/privacy/ccpa
- US state privacy regulation with similar principles to GDPR. Applies to PMS deployed in or processing data from California residents. Relevant for North American market.

**NIST Cybersecurity Framework (CSF) 2.0**
- URL: https://www.nist.gov/cyberframework
- Widely referenced US framework for managing cybersecurity risk. Relevant to PMS security architecture design, risk assessment, and vendor due-diligence documentation.

**OWASP API Security Top 10 (2023)**
- URL: https://owasp.org/API-Security/editions/2023/en/0x00-header/
- The reference checklist for REST API security in the hospitality integration platform. Key risks applicable to PMS APIs: Broken Object Level Authorisation (BOLA), Broken Authentication, Excessive Data Exposure, and Lack of Rate Limiting.

**OAuth 2.0 Security Best Current Practice (RFC 9700)**
- URL: https://datatracker.ietf.org/doc/html/rfc9700
- Updated security guidance for OAuth 2.0 deployments, addressing token injection, PKCE requirements, and redirect URI validation. Recommended baseline for any PMS implementing OAuth.

**Open Payments Alliance Standards Specification v2.0 (2021)**
- URL: https://www.ahla.com/file/15516/download?token=nVDjMbXq
- Industry specification developed by HTNG, OpenTravel, and HEDNA for standardised payment data exchange in hospitality. Covers payment processing flows between PMS, payment gateways, and point-of-sale systems.

---

### MCP Server Specifications

**Model Context Protocol (MCP)**
- URL: https://modelcontextprotocol.io/
- Anthropic's open protocol for connecting AI agents and LLM applications to data sources and tools. Directly relevant to a next-generation AI-native PMS: exposing reservation data, guest profiles, rate plans, and operational actions as MCP resources/tools would allow hotel AI agents (revenue optimisation, guest communication, housekeeping scheduling) to operate natively against PMS data. Apaleo explicitly referenced MCP in a 2025 blog post as part of their AI agent architecture roadmap.

---

## Similar Products — Developer Documentation & APIs

### Oracle OPERA Cloud (OHIP)

- **Description:** Enterprise-grade cloud PMS by Oracle Hospitality. OHIP (Oracle Hospitality Integration Platform) exposes 3,000+ REST APIs covering virtually all PMS operations, used by major chains including Accor and IHG.
- **API Documentation:** https://docs.oracle.com/en/industries/hospitality/integration-platform/
- **API Specs (GitHub):** https://github.com/oracle/hospitality-api-docs
- **Postman Collection:** https://www.postman.com/hospitalityapis/workspace/oracle-hospitality-apis/overview
- **Developer Guide:** https://docs.oracle.com/en/industries/hospitality/integration-platform/msrig/t_using_the_opera_cloud_apis.htm
- **Standards:** REST/JSON; OpenAPI 3.x specs published on GitHub
- **Authentication:** OAuth 2.0 (fine-grained scopes); application-key-based access per API consumer

---

### Mews Connector API

- **Description:** The primary integration API for the Mews PMS, providing full access to reservations, guests, rates, billing, housekeeping, and services. Used by 1,000+ Marketplace integrations. 10 M+ messages/day in production.
- **API Documentation:** https://docs.mews.com/connector-api
- **GitBook Documentation:** https://mews-systems.gitbook.io/connector-api
- **GitHub (Open API spec):** https://github.com/MewsSystems/gitbook-open-api
- **Developer Guide:** https://docs.mews.com/
- **Standards:** REST/JSON; OpenAPI/Swagger specification maintained
- **Authentication:** Access tokens (API key); OAuth 2.0 for partner integrations

---

### Mews Channel Manager API

- **Description:** Dedicated API for OTA channel manager connectivity, enabling real-time inventory and rate synchronisation between Mews PMS and distribution channels.
- **API Documentation:** https://docs.mews.com/channel-manager-api
- **Standards:** REST/JSON; HTNG-aligned data models
- **Authentication:** API key / token-based

---

### Cloudbeds API

- **Description:** REST API enabling full management of hotel operations: reservations, guests, rooms, billing, payments, and house accounts. 50+ endpoint categories; Python SDK available.
- **API Documentation:** https://developers.cloudbeds.com/
- **API Reference:** https://developers.cloudbeds.com/reference
- **Python SDK:** https://github.com/cloudbeds/cloudbeds-api-python
- **Developer Guide:** https://developers.cloudbeds.com/docs/integration-guide
- **Standards:** REST/JSON; OpenAPI Generator-compatible (Python SDK auto-generated)
- **Authentication:** OAuth 2.0 and API Key (`x-api-key` header or Bearer token)

---

### Apaleo Open API

- **Description:** 100% open API covering all PMS operations: inventory, rate plans, reservations, folios, housekeeping, and settings. Designed for developer-first hotel operators and ISV app builders. Hosts the first AI agent marketplace (Agent Hub) for hospitality.
- **API Documentation:** https://apaleo.dev/
- **Developer Portal:** https://apaleo.com/open-apis
- **Standards:** REST/JSON; OpenAPI 3.x specification; OAuth 2.0 via Apaleo Identity API
- **Authentication:** OAuth 2.0 (Authorization Code and Client Credentials flows)
- **SDKs:** Community-contributed; official documentation at apaleo.dev

---

### SiteMinder API (Channel Manager)

- **Description:** The world's most widely connected hotel channel manager, linking to 450+ distribution channels and 350+ PMS systems. APIs expose rate and inventory management, booking retrieval, and property configuration.
- **API Documentation:** https://www.siteminder.com/integrations/
- **Developer Information:** Integration via certified partner programme
- **Standards:** REST/JSON; OTA XML for legacy GDS/CRS integrations
- **Authentication:** API key and OAuth 2.0 (partner programme access)

---

### OpenTravel Alliance (OTA) Specification Downloads

- **Description:** The foundational XML and JSON message library for hospitality data exchange. Covers hotel availability, reservations, profiles, key management, and payments. Used by all major GDS, CRS, and distribution platforms.
- **Specification Downloads:** https://opentravel.org/download-the-opentravel-specification/
- **OTA 2.0 (JSON / OAS 3.0):** Available to OTA Alliance members; XML version publicly downloadable
- **Standards:** XML (.xsd) and JSON (OAS 3.0); WSDL and Swagger artefacts included
- **Authentication:** N/A (message format standard, not a live API)

---

### QloApps REST API (Open Source Reference Implementation)

- **Description:** Open-source PMS (OSL 3.0) with a REST API available for custom integrations. Useful as a reference implementation for data models and open-source PMS architecture.
- **GitHub:** https://github.com/Qloapps/QloApps
- **Developer Documentation:** https://qloapps.com/
- **Standards:** REST/JSON; PHP/MySQL backend (PrestaShop-based)
- **Authentication:** API key

---

### Sabre Hospitality SynXis (CRS / GDS Connectivity)

- **Description:** Sabre's central reservation system for hotels, connecting properties to GDS (Amadeus, Sabre, Travelport, Galileo), OTAs, and direct channels. Exposes PMS reservation sync via HTNG-compliant APIs.
- **API Documentation:** https://developer.synxis.com/pms/htng/reservation_sync
- **Standards:** HTNG Express; OTA XML; REST/JSON for newer endpoints
- **Authentication:** Partner/certification programme required

---

## Notes

1. **OpenTravel OTA 2.0 JSON / OAS 3.0** is the emerging standard for greenfield PMS integrations; OTA 1.0 XML is maintained for backward compatibility with legacy GDS systems but should be treated as legacy.

2. **HTNG Express** is optimised for lightweight post-booking operational data exchange (not full PMS integration) and is the recommended starting point for connecting third-party operational tools (e.g. keycard systems, in-room IoT) to a PMS.

3. **MCP (Model Context Protocol)** is an emerging but rapidly adopted standard for AI agent connectivity. An AI-native PMS should expose an MCP server interface as a first-class feature to enable LLM-based automation across revenue management, housekeeping, and guest communication workflows.

4. **PCI DSS v4.0** became fully mandatory on 31 March 2025 (superseding v3.2.1). Any PMS built after this date must comply with v4.0 requirements from day one, including script integrity management, targeted risk analyses, and enhanced MFA requirements.

5. **GDS Connectivity** (Amadeus, Sabre, Travelport) requires certification through each GDS operator's partner programme. Direct GDS connectivity is not practical for a new PMS; using a certified channel manager (SiteMinder, D-EDGE, RateGain) as an intermediary is the standard approach.
