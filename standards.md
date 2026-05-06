# Standards & API Reference

> Project: Mining Operations Management · Generated: 2026-05-06

## Industry Standards & Specifications

### ISO Standards

**ISO/TS 15143-3:2020 — Earth-moving machinery and mobile road construction machinery: Worksite data exchange — Part 3: Telematics data (AEMP 2.0)**
- Official URL: https://www.iso.org/standard/76394.html
- The foundational standard for mixed-fleet telematics interoperability. Defines a web-service protocol and common JSON schema for exchanging equipment operational parameters (operating hours, fuel consumption, GPS location, idle time, fault codes) from OEM systems into third-party fleet management platforms. Adopted by Caterpillar, Komatsu, Hitachi, and other major OEMs. Essential for any mining fleet management integration layer.

**ISO 15926 — Integration of life-cycle data for process plants including oil and gas production facilities**
- Official URL: https://15926.org/ · https://en.wikipedia.org/wiki/ISO_15926
- Provides a standard data model and ontology for representing plant, equipment, and process data across the engineering, construction, and maintenance lifecycle. Applicable to mine processing plants, mineral handling facilities, and process-heavy operations where equipment data must be exchanged between engineering, procurement, and operations systems.

**ISO 45001:2018 — Occupational health and safety management systems**
- Official URL: https://www.iso.org/standard/63787.html
- The primary international standard for occupational health and safety management systems. Defines requirements for incident reporting, hazard identification, risk assessment, and legal compliance that must be reflected in any mine safety module, including pre-shift inspections, permit-to-work, and MSHA/regulatory reporting workflows.

**ISO 14001:2015 — Environmental management systems**
- Official URL: https://www.iso.org/standard/60857.html
- Specifies requirements for an environmental management system enabling organisations to improve environmental performance through efficient use of resources and waste reduction. Directly relevant to tailings management, water usage tracking, dust and emissions monitoring, and ESG reporting features in mining operations platforms.

**ISO 55000/55001/55002 — Asset management**
- Official URL: https://www.iso.org/standard/55088.html
- The ISO 55000 family defines principles and requirements for asset management systems in asset-intensive industries. Provides the conceptual framework for enterprise asset management (EAM) modules covering asset lifecycle, maintenance strategy, and performance measurement — directly applicable to mining fleet and equipment management.

**ISO/IEC 62443 — Security for industrial automation and control systems**
- Official URL: https://www.iec.ch/cyber-security
- Defines security requirements for industrial control systems including SCADA, PLCs, and OPC-UA endpoints. Critical for securing OT/IT integration points in mine operations management platforms, particularly where sensor telemetry from process equipment connects to cloud-based management systems.

---

### W3C & IETF Standards

**OPC Unified Architecture (OPC-UA) — IEC 62541**
- Official URL: https://opcfoundation.org/about/opc-technologies/opc-ua/
- The dominant industrial communication standard for SCADA, PLC, and process historian integration. OPC-UA provides secure, platform-independent communication between field devices and management software using client-server and pub-sub (MQTT) transport. Any mining operations platform ingesting real-time sensor data from process equipment or mobile machinery should support OPC-UA as a primary data collection protocol.

**MQTT v5.0 — OASIS Standard**
- Official URL: https://mqtt.org/ · https://docs.oasis-open.org/mqtt/mqtt/v5.0/
- Lightweight publish-subscribe messaging protocol designed for constrained IoT environments and unreliable networks. Used extensively for sensor telemetry transmission from edge devices at remote mine sites to cloud or on-premises management platforms. Commonly paired with OPC-UA in IIoT architectures: OPC-UA for horizontal device communication, MQTT for vertical cloud transmission.

**RFC 9110 — HTTP Semantics**
- Official URL: https://www.rfc-editor.org/rfc/rfc9110
- The foundational HTTP/1.1 semantics specification underpinning all REST API design for mining software integrations. Relevant for defining API behaviour, status codes, caching, and content negotiation in the platform's public API surface.

**RFC 7519 — JSON Web Token (JWT)**
- Official URL: https://www.rfc-editor.org/rfc/rfc7519
- Defines the JWT format used for authentication token exchange in REST APIs and OAuth 2.0 flows. Applicable to all API authentication between the platform and OEM telematics feeds, ERP connectors, and mobile client authentication.

**RFC 8288 — Web Linking**
- Official URL: https://www.rfc-editor.org/rfc/rfc8288
- Defines the Link header and link relation types used in REST API hypermedia responses. Relevant for paginated API endpoints in work order, asset, and incident APIs.

---

### Data Model & API Specifications

**OpenAPI Specification 3.1 (OAS 3.1)**
- Official URL: https://spec.openapis.org/oas/v3.1.0
- The industry standard for describing REST API interfaces. The mining platform's public API should be described in OpenAPI 3.1 to enable SDK generation, documentation tooling, and integration with API gateways. Modular Mining's public API already uses OpenAPI as its specification format.

**AsyncAPI 3.0**
- Official URL: https://www.asyncapi.com/docs/reference/specification/v3.0.0
- Extends OpenAPI concepts to event-driven and message-based APIs (MQTT, WebSockets, AMQP). Relevant for describing the real-time telemetry and event streams from mine equipment — Modular Mining's public API uses AsyncAPI for WebSocket endpoints alongside OpenAPI for REST endpoints.

**JSON Schema (Draft 2020-12)**
- Official URL: https://json-schema.org/specification.html
- Used to define and validate data structures for API request/response bodies and configuration payloads. Critical for ensuring data quality in telemetry ingestion pipelines from heterogeneous OEM sources.

**GeoJSON (RFC 7946)**
- Official URL: https://www.rfc-editor.org/rfc/rfc7946
- Defines the standard JSON representation for geospatial features. Relevant for representing GPS equipment positions, geofenced zones, mine pit boundaries, haul road networks, and blast exclusion areas in fleet management and safety modules.

---

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749) and OpenID Connect 1.0**
- Official URLs: https://www.rfc-editor.org/rfc/rfc6749 · https://openid.net/connect/
- The standard authorization framework and identity layer for securing API access between the mining platform, mobile clients, OEM telematics feeds, and ERP integrations. Enables SSO with enterprise identity providers (Azure AD, Okta) commonly used in mining organizations.

**OWASP API Security Top 10**
- Official URL: https://owasp.org/www-project-api-security/
- The authoritative reference for API security risks including broken object-level authorisation, excessive data exposure, and injection vulnerabilities. Directly applicable to securing work order, incident, and telemetry APIs against external and insider threats.

**NIST Cybersecurity Framework (CSF) 2.0**
- Official URL: https://www.nist.gov/cyberframework
- Provides the risk management framework for securing OT/IT convergence in mining environments. Particularly relevant for securing OPC-UA endpoints, SCADA integrations, and edge computing nodes at remote mine sites where cybersecurity hardening of operational technology is required.

---

### Regulatory & Reporting Frameworks

**GRI 14: Mining Sector Standard (2024)**
- Official URL: https://www.globalreporting.org/standards/standards-development/sector-standard-for-mining/
- The Global Reporting Initiative's mining-sector sustainability standard covering 25 material topics including tailings management, water use, emissions, biodiversity, human rights, land rights, and community engagement. The primary ESG reporting framework that mining operations software must support for automated data capture and report generation.

**ICMM Mining Principles and Performance Expectations**
- Official URL: https://www.icmm.com/en-gb/icmm-principles
- The International Council on Mining & Metals framework for responsible mining. Aligned with GRI 14 and covers health and safety, environmental stewardship, social performance, and ethical business. An AI-native platform should map its environmental data fields to both GRI 14 and ICMM expectations.

**CRIRSCO International Reporting Template**
- Official URL: https://crirsco.com/
- The Committee for Mineral Reserves International Reporting Standards template for public reporting of Exploration Results, Mineral Resources, and Mineral Reserves. Relevant if the platform includes ore resource tracking or reconciliation between mine plan and actual production.

**MSHA — Mine Safety and Health Administration (USA)**
- Official URL: https://www.msha.gov/
- US federal regulator for mine safety. Requires immediate accident reporting (within 15 minutes), 7000-1 accident/injury forms within 10 working days, and quarterly 7000-2 employment and production reports. Any compliance module targeting US operations must support MSHA-specific forms, not just OSHA.

**Global Industry Standard on Tailings Management (GISTM)**
- Official URL: https://globaltailingsreview.org/global-industry-standard/
- The international standard for tailings facility governance, requiring annual independent reviews, consequence classification, and emergency preparedness documentation. Environmental reporting modules should support GISTM data capture and reporting workflows.

---

## Similar Products — Developer Documentation & APIs

### Cat Digital Marketplace — ISO 15143-3 (AEMP 2.0) API

- **Description:** Caterpillar's public telematics API exposing real-time equipment data (location, operating hours, fuel level, fault codes, payload) for Cat and non-Cat equipment in mixed fleets. Used by fleet management systems to ingest Cat machine data.
- **API Documentation:** https://digital.cat.com/knowledge-hub/faq/iso-15143-3-aemp-20-api-faqs
- **Standards:** ISO/TS 15143-3 (AEMP 2.0), REST/JSON
- **Authentication:** OAuth 2.0 / API Key

---

### Komatsu — ISO 15143-3 API (My Komatsu)

- **Description:** Komatsu's mixed-fleet telematics API via My Komatsu portal. Exposes KOMTRAX telemetry data for Komatsu and third-party machines. No additional API connection fee charged by Komatsu.
- **API Documentation:** https://www.komatsu.com/en-us/services-and-support/equipment-monitoring-and-analysis/my-komatsu
- **Standards:** ISO/TS 15143-3 (AEMP 2.0), REST/JSON
- **Authentication:** OAuth 2.0

---

### Modular Mining (Komatsu) DISPATCH Public API

- **Description:** REST + WebSocket API for Modular Mining's DISPATCH Fleet Management System. Provides real-time equipment positioning, status, cycle-state changes, and crusher telemetry. Supports bi-directional integration with ERP and SCADA systems. Specified using OpenAPI and AsyncAPI.
- **API Documentation:** https://www.mining-technology.com/contractors/data//pressreleases/public-api-interoperability/
- **Standards:** OpenAPI 3.x, AsyncAPI, REST/JSON, WebSockets
- **Authentication:** API Key / OAuth 2.0

---

### IBM Maximo Application Suite — REST APIs

- **Description:** Enterprise EAM REST APIs covering assets, work orders, inventory, and labour records. Supports integration with IoT platforms, ERP systems, and third-party maintenance tools.
- **API Documentation:** https://www.ibm.com/docs/en/mas-cd/continuous-delivery?topic=apis
- **SDKs/Libraries:** Java, Python (community)
- **Developer Guide:** https://www.ibm.com/docs/en/maximo-application-suite
- **Standards:** REST/JSON, OpenAPI, OAuth 2.0
- **Authentication:** OAuth 2.0 / API Key

---

### IFS Cloud — REST APIs

- **Description:** IFS Cloud exposes a comprehensive REST API for ERP, EAM, and EHS modules including work orders, assets, incidents, and environmental metrics. Supports integration with OT systems and third-party analytics.
- **API Documentation:** https://docs.ifs.com/techdocs/Foundation1/010_connect/030_odata_and_rest/
- **Standards:** REST/JSON, OData, OpenAPI
- **Authentication:** OAuth 2.0 / OpenID Connect

---

### RPMGlobal AMT — Integration APIs

- **Description:** RPMGlobal AMT ships with a suite of integration APIs for connecting to ERPs, fleet systems, and production/health monitoring systems. Enables bi-directional data exchange for maintenance cost and life-cycle data.
- **API Documentation:** https://rpmglobal.com/product/amt/
- **Standards:** REST/JSON
- **Authentication:** API Key / OAuth 2.0

---

### MaintainX — REST API

- **Description:** MaintainX REST API enables integration with ERP systems, IoT sensor platforms, and BI tools. Supports work order CRUD, asset management, and parts inventory operations. Zapier integration provides no-code connector options.
- **API Documentation:** https://api.getmaintainx.com/
- **SDKs/Libraries:** JavaScript, Python (via community wrappers)
- **Standards:** REST/JSON, OpenAPI
- **Authentication:** API Key / OAuth 2.0

---

### Tractian CMMS — REST API

- **Description:** Tractian's REST API integrates the CMMS with ERP and EAM systems, enabling synchronisation of work orders, asset records, and sensor-triggered maintenance events.
- **API Documentation:** https://tractian.com/en/industry/mining-sector
- **Standards:** REST/JSON
- **Authentication:** API Key / OAuth 2.0

---

### Hexagon Mining — HxGN MineOperate APIs

- **Description:** HxGN MineOperate provides integration interfaces for connecting fleet management, machine guidance, and drill-and-blast data to third-party ERP and asset management systems. Sensor data from blast movement technology can be accessed programmatically.
- **API Documentation:** https://hexagon.com/products/product-groups/hxgn-mineoperate (contact vendor for developer documentation)
- **Standards:** REST/JSON (specifics vendor-disclosed)
- **Authentication:** Enterprise SSO / API Key

---

### OPC Foundation — OPC-UA Reference Implementation

- **Description:** Open-source reference implementation of OPC-UA for integrating SCADA and PLC data. Used to build adapters between industrial control systems and cloud platforms. Supports both client-server and MQTT pub-sub transport modes.
- **API Documentation:** https://opcfoundation.org/developer-tools/specifications-unified-architecture
- **SDKs/Libraries:** C, C++, Java, .NET, Python (open62541 — MIT licence)
- **Developer Guide:** https://reference.opcfoundation.org/
- **Standards:** IEC 62541, OPC-UA, MQTT 5.0
- **Authentication:** X.509 certificates / username-password

---

## Notes

**Mixed-fleet telematics fragmentation:** Despite ISO 15143-3 (AEMP 2.0) adoption, OEM implementations vary in data field coverage and update frequency. Building a normalisation layer above the standard is necessary for reliable mixed-fleet analytics.

**OT/IT security gap:** Most OEM telematics APIs and SCADA integrations were not designed with modern OAuth 2.0 patterns. Legacy API key authentication is common; platforms should support both while enforcing mTLS at the network boundary.

**ESG reporting APIs:** No standardised machine-readable API exists for GRI 14 or ICMM reporting submission. Current practice involves generating structured data exports (PDF, XBRL, or CSV) rather than API submission. This is an emerging area where the platform can define its own schema.

**MSHA electronic submission:** MSHA is progressively moving to electronic form submission via its online portal; direct API integration for 7000-1 and 7000-2 forms is not yet available (as of 2026) but anticipated in future regulatory upgrades.
