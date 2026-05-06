# Standards & API Reference

> Project: Vacation Rental Management · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 27001:2022 — Information Security Management Systems**
- URL: https://www.iso.org/standard/27001
- The globally recognised standard for establishing, implementing, and maintaining an Information Security Management System (ISMS). Directly relevant for any vacation rental SaaS storing sensitive guest PII, payment data, and property access credentials. Enterprise property managers and channel partners increasingly require ISO 27001 certification as a procurement prerequisite.

**ISO/IEC 27701:2019 — Privacy Information Management**
- URL: https://www.iso.org/standard/71670.html
- Extension to ISO 27001 that addresses privacy information management. Relevant when handling EU guest data and aligning with GDPR obligations, particularly for processing guest identity, booking history, and payment records across jurisdictions.

### W3C & IETF Standards

**RFC 5545 — Internet Calendaring and Scheduling Core Object Specification (iCalendar)**
- URL: https://datatracker.ietf.org/doc/html/rfc5545
- The foundational standard for the `.ics` calendar format used universally in vacation rental availability sync. Airbnb, Vrbo, and Booking.com all support iCalendar export/import as a baseline sync mechanism for platforms lacking direct API access. Every vacation rental PMS must implement RFC 5545-compliant iCal export and import.

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The industry-standard protocol for delegated authorisation. Airbnb, Guesty, Hostaway, Beds24, and Booking.com all use OAuth 2.0 as the authentication mechanism for their APIs. Any vacation rental management platform integrating with OTAs or third-party apps must implement OAuth 2.0 flows (Authorization Code + PKCE for user-delegated access; Client Credentials for machine-to-machine).

**RFC 7519 — JSON Web Token (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- Standard for compact, URL-safe claims representation used extensively as the bearer token format in REST API authentication across the vacation rental ecosystem (Guesty, Hostaway, Apaleo).

**RFC 8288 — Web Linking**
- URL: https://datatracker.ietf.org/doc/html/rfc8288
- Defines the `Link` header and relation types used in paginated REST API responses — relevant when building channel manager APIs that return large collections of listings, reservations, or pricing records.

### Data Model & API Specifications

**OpenAPI Specification (OAS) 3.1 / 3.2**
- URL: https://spec.openapis.org/oas/v3.1.0.html | https://spec.openapis.org/oas/v3.2.0.html
- The de-facto standard for documenting REST APIs. OAS 3.1 (2021) introduced full JSON Schema Draft 2020-12 compatibility. OAS 3.2 (September 2025) adds structured tag nesting, streaming media type support (Server-Sent Events, JSON Lines), and native QUERY HTTP method support — relevant for streaming AI pricing recommendations and real-time channel event feeds. All major vacation rental SaaS APIs (Guesty, Hostaway, Beds24) publish OpenAPI specs.

**JSON Schema Draft 2020-12**
- URL: https://json-schema.org/draft/2020-12
- The underlying schema language for validating OTA request/response payloads, listing content, reservation objects, and pricing rules. Used natively in OpenAPI 3.1+ and by the OpenTravel Alliance 2.0 message format.

**OpenTravel Alliance (OTA) 2.0 — Hospitality XML/JSON Messaging Standard**
- URL: https://opentravel.org/category/specification/
- Industry consortium standard (since 1999) for interoperable data exchange across hotels, vacation rentals, and OTAs. OTA 1.0 is XML-based; OTA 2.0 uses model-driven development producing both XML XSD and JSON/OAS 3.0 formats. The Hospitality Workgroup explicitly covers vacation rentals. The 2024A release modernised XML messages; a parallel JSON/REST track continues under OTA 2.0.

**iCal.NET / ical-org specification**
- URL: https://github.com/ical-org/ical.net/wiki/iCalendar-Specification-(RFC5545)
- Community reference implementation and schema documentation for RFC 5545, widely used in .NET vacation rental integrations.

### Security & Authentication Standards

**PCI DSS v4.0 — Payment Card Industry Data Security Standard**
- URL: https://www.pcisecuritystandards.org/standards/
- Mandatory for any platform that stores, processes, or transmits cardholder data. Applies to all vacation rental software accepting direct card payments. Compliance is typically achieved by routing payments through a PCI-compliant gateway (Stripe, Braintree) to reduce scope. OwnerRez, Guesty, and Hostaway all document their PCI compliance posture explicitly. Non-compliant platforms face fines and loss of card processing rights.

**PSD2 / Strong Customer Authentication (SCA) — EU Directive 2015/2366**
- URL: https://www.eba.europa.eu/regulation-and-policy/payment-services-and-electronic-money/regulatory-technical-standards-on-strong-customer-authentication-and-secure-communication
- EU regulation requiring multi-factor authentication for online payment transactions above €30. Directly affects vacation rental booking flows for European guests. Platforms must support SCA-compliant payment flows via 3D Secure 2 (3DS2) through their payment gateway.

**GDPR — General Data Protection Regulation (EU) 2016/679**
- URL: https://gdpr-info.eu/
- Applies to any vacation rental platform collecting data from EU guests, regardless of where the platform is hosted. Key obligations: lawful basis for processing (consent or legitimate interest), data subject rights (access, erasure, portability), data minimisation, and breach notification within 72 hours. Lodgify, Guesty, and Beds24 publish explicit GDPR compliance documentation.

**OAuth 2.1 (Draft) — Consolidated OAuth 2.0**
- URL: https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/
- Draft consolidation of OAuth 2.0 + security BCP that mandates PKCE for all grant types and removes implicit and password credential flows. Emerging as the recommended baseline for new API integrations; vacation rental platforms building new OAuth integrations should target OAuth 2.1 compliance.

### MCP Server Specifications

**Model Context Protocol (MCP)**
- URL: https://modelcontextprotocol.io/
- Anthropic's open protocol for connecting AI agents to external systems via a standardised tool-calling interface. Directly relevant to AI-native vacation rental management: Apaleo launched the first hospitality MCP server in 2025, enabling AI agents to perform reservation management, pricing updates, and guest communication operations without custom per-integration code. An open-source vacation rental PMS should consider publishing an MCP server to allow LLM agents (Claude, GPT-4o, etc.) to operate the platform autonomously.

---

## Similar Products — Developer Documentation & APIs

### Airbnb Connectivity API
- **Description:** Official partner API for two-way calendar sync, pricing, listing management, messaging, and reviews. Restricted to approved connectivity partners and premier software integrations; not publicly accessible to individual hosts.
- **API Documentation:** https://developer.withairbnb.com/
- **SDKs/Libraries:** No official SDK; partners build against REST/JSON endpoints using OAuth 2.0.
- **Developer Guide:** https://developer.withairbnb.com/ (requires approved partner status)
- **Standards:** REST/JSON, OAuth 2.0 (Authorization Code), iCal (RFC 5545) as fallback
- **Authentication:** OAuth 2.0; Instant Booking mandatory for all API-connected listings

### Vrbo / Expedia Rapid API (Vacation Rentals)
- **Description:** The Rapid API integrates 650k+ Vrbo properties alongside 265k+ traditional vacation rental and hotel inventory through a unified REST API. Partners access availability, rates, content, and reservations via a single endpoint. Rollout to new connectivity partners is staged.
- **API Documentation:** https://developers.expediagroup.com/docs/products/rapid/lodging/vacation-rentals
- **Developer Guide:** https://developers.expediagroup.com/docs/products/rapid/lodging/vacation-rentals/vrbo-integration-guide
- **Standards:** REST/JSON, OpenAPI documented, supply_source=vrbo parameter for Vrbo-specific inventory
- **Authentication:** API key + HMAC signature

### Booking.com Connectivity API
- **Description:** Comprehensive REST/XML connectivity API enabling property managers to manage room availability, pricing, reservations, property content, contacts, facilities, and charges for properties listed on Booking.com. Two base URLs: non-PCI (content/availability) and PCI-scoped (reservation retrieval).
- **API Documentation:** https://developers.booking.com/connectivity/docs
- **User Guide:** https://connect.booking.com/user_guide/site/en-US
- **Standards:** REST + XML, Connections API for partner permissioning, modular Property Management APIs
- **Authentication:** OAuth 2.0 / API credentials; Connections API governs permission grants between PMS and properties

### Guesty Open API
- **Description:** REST API exposing Guesty's full PMS functionality: listings, reservations, guests, tasks, accounting, owner statements, and inbox. Targeted at property managers integrating bespoke tools or building on top of Guesty.
- **API Documentation:** https://open-api-docs.guesty.com/
- **Booking Engine API:** https://booking-api-docs.guesty.com/
- **Postman Collection:** https://www.postman.com/guesty-cs-tier2/guesty-openapi-collection/
- **Standards:** REST/JSON, OpenAPI documented
- **Authentication:** OAuth 2.0 (access tokens, 24-hour TTL, up to 5 tokens per API key per day)

### Hostaway Public API
- **Description:** RESTful API covering listings, reservations, calendar management, financials, and webhooks. Among the most mature and thoroughly documented APIs in the vacation rental PMS segment; includes sandbox environment and 200+ pre-built integrations.
- **API Documentation:** https://api.hostaway.com/documentation
- **GitHub:** https://github.com/Hostaway/api
- **Standards:** REST/JSON; rate limits enforced per IP and account
- **Authentication:** API Key (Account ID + Secret Key, activated via dashboard)

### Beds24 API v2
- **Description:** Highly configurable REST API for channel integration, booking management, property configuration, and guest services. Interactive Swagger/OpenAPI documentation available. Supports both channel (OTA) and guest-service integration patterns.
- **API Documentation:** https://beds24.com/developer-api.html
- **Wiki:** https://wiki.beds24.com/index.php/OTAs:_How_to_connect_to_Beds24_using_API_V2
- **Standards:** REST/JSON, Swagger/OpenAPI interactive docs; V1 (legacy) and V2 available
- **Authentication:** API token (dashboard-issued)

### PriceLabs Dynamic Pricing API
- **Description:** Programmatic access to PriceLabs' ML-driven dynamic pricing engine and market data. Two distinct APIs: the Dynamic Pricing API for PMSs pushing/pulling price recommendations, and the Revenue Estimator API for embedding occupancy and revenue forecasts in external tools.
- **API Documentation:** https://hello.pricelabs.co/dynamic-pricing-api/
- **Revenue Estimator API:** https://hello.pricelabs.co/revenue-estimator-api-widget/
- **Standards:** REST/JSON
- **Authentication:** API key; contact support@pricelabs.co for access

### Beyond Pricing API
- **Description:** Revenue management API enabling PMS partners to pull AI-driven price recommendations and push listing data into the Beyond pricing engine. The Dynamic Integration API is specifically designed for in-house PMS platforms seeking to offer Beyond as a pricing layer.
- **API Documentation:** https://api-docs.beyondpricing.com/
- **Dynamic Integration API:** https://dynamic-api-docs.beyondpricing.com/
- **Standards:** REST/JSON
- **Authentication:** API key

### Avalara MyLodgeTax API
- **Description:** Automated occupancy and lodging tax compliance API for short-term rentals. Handles tax rate calculation, registration/permit management, return filing, and remittance across US jurisdictions. Integrated with OwnerRez, Lodgify, and other major PMSs.
- **API Documentation:** https://developer.avalara.com/mylodge/
- **API Reference:** https://developer.avalara.com/api-reference/myLodgeAPI/overview-onboarding/
- **Standards:** REST/JSON; partnerleads endpoint for onboarding, ownertaxsummaries/ownertaxdetails for rate data
- **Authentication:** Partner API credentials

### Apaleo Open PMS API
- **Description:** API-first, cloud-native PMS exposing Booking, Finance, Payment, and Distribution APIs. Primarily targets hotels and serviced apartments but used by vacation rental operators. Notable for launching the first hospitality MCP server (2025), enabling AI agents to operate the PMS directly.
- **API Documentation:** https://apaleo.com/open-apis
- **MCP Server:** https://www.hospitalitynet.org/news/4129031.html
- **Standards:** REST/JSON, OpenAPI 3.x, MCP (Model Context Protocol)
- **Authentication:** OAuth 2.0

### Stripe Payments API
- **Description:** The most widely adopted payment processing API in the vacation rental sector. Supports card payments, ACH, SEPA, wallets, subscriptions, and payout automation. SDKs available for JavaScript/Node.js, Python, Ruby, PHP, Java, Go, and .NET. Handles PCI DSS scope reduction by tokenising card data client-side.
- **API Documentation:** https://docs.stripe.com/api
- **SDKs:** https://docs.stripe.com/sdks (Node, Python, Ruby, PHP, Java, Go, .NET)
- **Standards:** REST/JSON, OpenAPI documented; PCI DSS SAQ A-EP compliant; 3DS2/SCA support built-in
- **Authentication:** Secret API key (server-side); Publishable key (client-side)

### Twilio Programmable Messaging API
- **Description:** SMS, MMS, RCS, and WhatsApp messaging API widely used by vacation rental platforms (Hostfully, iGMS, OwnerRez) for automated guest communication: booking confirmations, check-in instructions, pre-arrival messages, and post-stay follow-ups. Twilio explicitly documents a vacation rental notification workflow in its sample apps.
- **API Documentation:** https://www.twilio.com/docs/messaging/api
- **SDKs:** https://www.twilio.com/docs (Node, Python, Ruby, PHP, Java, C#)
- **Standards:** REST/JSON; supports webhook delivery receipts
- **Authentication:** HTTP Basic Auth using Account SID + Auth Token (or API Key + Secret for production)

---

## Notes

**Channel API access is gated:** Airbnb, Vrbo, and Booking.com do not offer open developer access — connectivity requires approved partner status. Startups typically use an existing channel manager (Rentals United, Lodgify, Beds24) as a middleware layer until they qualify for direct connectivity partnerships.

**iCalendar remains the universal fallback:** Despite rich REST APIs from major OTAs, RFC 5545 iCal is still the most universally supported calendar sync mechanism, particularly for smaller OTAs (Hipcamp, Vacasa affiliates, direct booking sites) that lack REST APIs.

**MCP is an emerging standard for AI-native PMS:** Apaleo's 2025 MCP server launch signals that the Model Context Protocol is becoming the preferred integration surface for AI agents operating property management systems. An open-source AI-native vacation rental PMS should treat MCP server publication as a first-class requirement.

**Tax compliance APIs are jurisdiction-fragmented:** No single standard covers short-term rental tax remittance globally. Avalara MyLodgeTax covers US lodging taxes; Taxamo (now Vertex) and Fonoa address EU VAT and global indirect tax; jurisdiction-specific APIs exist in some markets (e.g., Airbnb's collection-and-remittance in 30+ jurisdictions bypasses PMS involvement entirely).

**Smart home / IoT integration lacks a unified standard:** Smart lock vendors (Schlage Encode, Yale Assure, August Wi-Fi) each provide proprietary APIs or rely on platform bridges (SmartThings, Apple Home, Google Home). The Matter standard (CSA) is emerging as a potential cross-vendor protocol but adoption in the vacation rental segment remains early.
