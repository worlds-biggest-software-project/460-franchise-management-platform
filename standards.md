# Standards & API Reference

> Project: Franchise Management Platform (460) · Generated: 2026-05-03

## Industry Standards & Specifications

### Regulatory & Compliance Standards

#### Federal Trade Commission (FTC) Franchise Rule
- **Standard:** 16 CFR Part 436 - Franchise Rule
- **URL:** https://www.ftc.gov/business-guidance/resources/franchises
- **Relevance:** Mandates disclosure requirements for franchisors, including the Franchise Disclosure Document (FDD) with 23 required items. Franchisees must receive FDD at least 14 days before signing any agreement or payment.

#### Franchise Disclosure Document (FDD) Standard
- **Standard:** FTC-regulated 23-Item Disclosure Format
- **URL:** https://www.ftc.gov/business-guidance/blog/2023/05/franchise-fundamentals-taking-deep-dive-franchise-disclosure-document
- **Relevance:** Standardized disclosure framework with 23 required sections covering franchisor background, fees, litigation, financial performance, outlet data, and operational obligations.

#### State Franchise Disclosure Laws
- **Standard:** Multi-state registration and disclosure requirements
- **URL:** https://www.nasaa.org (North American Securities Administrators Association)
- **Relevance:** 14 states require franchise registration; state-specific requirements beyond federal FTC rules. Quarterly or annual updates of material changes required.

#### NASAA Electronic Filing System
- **Standard:** NASAA Electronic Filing Depository
- **URL:** https://nasaa.org/investor-protection/franchising/
- **Relevance:** Centralized electronic filing system for franchise disclosure documents across participating states.

### Financial Accounting Standards

#### GAAP Franchise Accounting (ASC 606, ASC 952, ASC 350)
- **Standard:** U.S. GAAP - Accounting Standards Codification
- **URL:** https://www.fasb.org/
- **Relevance:** ASC 606 (Revenue from Contracts with Customers), ASC 952 (Franchisors), and ASC 350 (Intangibles) govern franchise revenue recognition, initial fees, and ongoing royalty accounting.

#### Statement of Financial Accounting Standards No. 45 (FAS 45)
- **Standard:** FAS 45 - Accounting for Franchise Fee Revenue
- **URL:** https://www.xavierpaper.com/documents/usgaap/FAS45.pdf
- **Relevance:** Legacy standard addressing timing of franchise fee recognition and initial service obligations.

#### IFRS Franchise Accounting Standards
- **Standard:** IFRS 15 (Revenue from Contracts with Customers) and IAS 38 (Intangible Assets)
- **URL:** https://www.ifrs.org/
- **Relevance:** International alternatives to U.S. GAAP for multinational franchisors.

### Data & Security Standards

#### OAuth 2.0 Authorization Framework
- **Standard:** RFC 6749 and RFC 6750
- **URL:** https://tools.ietf.org/html/rfc6749
- **Relevance:** Industry-standard for secure API authorization and third-party application access without exposing credentials.

#### HTTPS/TLS 1.2 and 1.3
- **Standard:** RFC 5246, RFC 8446
- **URL:** https://tools.ietf.org/html/rfc5246
- **Relevance:** Secure transport layer for all data in transit; mandatory for handling confidential franchisee and financial data.

#### JSON (JavaScript Object Notation) Standard
- **Standard:** RFC 7158 (JSON Text Sequences), ECMA-404
- **URL:** https://www.json.org/
- **Relevance:** Lightweight data interchange format for REST APIs and franchise data exchange.

#### OpenAPI 3.1 Specification
- **Standard:** OpenAPI Initiative
- **URL:** https://spec.openapis.org/oas/v3.1.0
- **Relevance:** Standard for documenting REST APIs used in franchise software integrations and third-party development.

#### JSON Schema Standard
- **Standard:** JSON Schema (Draft 2020-12)
- **URL:** https://json-schema.org/
- **Relevance:** Schema validation for franchise data structures, agreement templates, and FDD sections.

### Privacy & Compliance Standards

#### GDPR (General Data Protection Regulation)
- **Standard:** EU Regulation 2016/679
- **URL:** https://gdpr-info.eu/
- **Relevance:** Data protection for franchisees in EU; right to access, erasure, portability; mandatory for any franchisor with EU franchisees.

#### CCPA (California Consumer Privacy Act)
- **Standard:** California Civil Code § 1798.100 et seq.
- **URL:** https://oag.ca.gov/privacy/ccpa
- **Relevance:** Data privacy rights for California franchisees; requires disclosure and consent for data collection.

#### SOC 2 Compliance
- **Standard:** System and Organization Controls (Type II)
- **URL:** https://www.aicpa.org/interestareas/informationtechnology/sodframework.html
- **Relevance:** Independent audit standard for cloud-based franchise software providers covering security, availability, and confidentiality.

#### ISO 27001 Information Security Management
- **Standard:** ISO/IEC 27001:2022
- **URL:** https://www.iso.org/standard/81805.html
- **Relevance:** Information security management system standard; increasingly required by franchisors managing multi-unit sensitive data.

#### FERPA (Family Educational Rights and Privacy Act)
- **Standard:** 20 U.S.C. § 1232g (if LMS training data)
- **URL:** https://www2.ed.gov/policy/gen/guid/fpco/ferpa/
- **Relevance:** If franchise management system includes LMS with training records, may require FERPA compliance for employee/franchisee training data.

## Similar Products — Developer Documentation & APIs

### FranConnect
- **Description:** Enterprise franchise management platform with comprehensive API for lead management, operations, and reporting.
- **API Documentation:** https://docs.franconnect.net/
- **SDKs/Libraries:** Limited; primarily REST API
- **Developer Guide:** REST API documentation with OAuth 2.0 authentication
- **Standards:** REST/JSON, OpenAPI-compatible
- **Authentication:** OAuth 2.0, API Key

### ClientTether
- **Description:** Unified franchise development and operations platform with multi-channel communication and automation.
- **API Documentation:** Not publicly documented; available to integration partners
- **SDKs/Libraries:** REST API with Zapier integration
- **Developer Guide:** Integration with third-party tools via REST endpoints
- **Standards:** REST/JSON
- **Authentication:** OAuth 2.0, API Key

### BrandWide
- **Description:** Simplified CRM and local marketing platform for growing franchise networks.
- **API Documentation:** Limited public API documentation
- **SDKs/Libraries:** REST API
- **Developer Guide:** Integration with QuickBooks and email platforms
- **Standards:** REST/JSON
- **Authentication:** API Key

### FranchiseSoft
- **Description:** Modular franchise management platform with CRM, LMS, and operations modules.
- **API Documentation:** Not extensively documented publicly
- **SDKs/Libraries:** REST API (Zapier integration available)
- **Developer Guide:** Available to enterprise customers
- **Standards:** REST/JSON
- **Authentication:** OAuth 2.0

### Claromentis
- **Description:** Enterprise digital workplace and intranet platform with workflow automation and LMS.
- **API Documentation:** https://help.claromentis.com/hc/en-us
- **SDKs/Libraries:** REST API, single sign-on (SSO) integration
- **Developer Guide:** Comprehensive developer portal with OAuth 2.0 and SSO examples
- **Standards:** REST/JSON, SAML 2.0 for SSO
- **Authentication:** OAuth 2.0, SAML 2.0

### Pipedrive (Franchise Module)
- **Description:** General-purpose CRM with franchise-specific module and 500+ app integrations.
- **API Documentation:** https://developers.pipedrive.com/docs/api/v1/
- **SDKs/Libraries:** REST API, Node.js SDK, Python SDK, Ruby SDK
- **Developer Guide:** Comprehensive developer documentation with examples
- **Standards:** REST/JSON, OpenAPI 3.0
- **Authentication:** OAuth 2.0, API Token

### QuickBooks Online (Financial Integration)
- **Description:** Cloud accounting software commonly integrated with franchise management systems.
- **API Documentation:** https://developer.intuit.com/app/developer/qbo/docs/api/accounting-api
- **SDKs/Libraries:** Node.js, Python, Java, C#/.NET
- **Developer Guide:** Comprehensive OAuth 2.0 flow and API reference
- **Standards:** REST/JSON, OpenAPI
- **Authentication:** OAuth 2.0

### Zapier (Integration Platform as a Service)
- **Description:** No-code integration platform connecting 7,000+ apps; widely used by franchise software providers.
- **API Documentation:** https://zapier.com/platform/
- **SDKs/Libraries:** JavaScript/Node.js for custom integrations
- **Developer Guide:** Zapier Platform API for creating custom integrations
- **Standards:** REST/JSON, webhooks
- **Authentication:** OAuth 2.0, API Key

### Stripe (Payment Processing)
- **Description:** Payment processor for collecting franchise fees, royalties, and subscriptions.
- **API Documentation:** https://stripe.com/docs/api
- **SDKs/Libraries:** Python, Node.js, Java, Go, Ruby, PHP, .NET
- **Developer Guide:** Comprehensive payment and billing API documentation
- **Standards:** REST/JSON, OpenAPI 3.0
- **Authentication:** API Key (live/test), OAuth 2.0

### Google Workspace APIs (Collaboration & Email)
- **Description:** Email, calendar, and document collaboration tools for franchisee communication.
- **API Documentation:** https://developers.google.com/workspace
- **SDKs/Libraries:** Python, Node.js, Java, C#/.NET, Go, Ruby
- **Developer Guide:** Gmail API, Calendar API, Drive API
- **Standards:** REST/JSON, OAuth 2.0
- **Authentication:** OAuth 2.0

### Microsoft Graph API (Microsoft 365/Office 365)
- **Description:** Unified API for Microsoft 365 services including email, calendar, and Outlook.
- **API Documentation:** https://docs.microsoft.com/en-us/graph/
- **SDKs/Libraries:** .NET, JavaScript, Python, Java, Go, Ruby, PHP
- **Developer Guide:** Comprehensive Microsoft Graph API reference
- **Standards:** REST/JSON, OData v4
- **Authentication:** OAuth 2.0

## Notes

### Gaps and Emerging Standards

1. **Franchise Data Interchange Standard:** While FDD content is standardized, no technical data model standard exists for electronic interchange of FDD data between franchisors and franchisees. Most systems rely on custom JSON/REST implementations or PDF uploads.

2. **Franchisee Performance Data:** No industry standard exists for performance metrics (sales, unit economics, customer satisfaction) across franchise networks, leading to proprietary dashboards in each system.

3. **Territory & Market Data Standards:** Franchise territory data and market analysis standards are not formally defined; most systems use proprietary geographic algorithms or integrate with third-party real estate/GIS services.

4. **Compliance Automation Standards:** Automated compliance monitoring is not yet standardized; regulations evolve by state and require manual template updates in most systems.

5. **Multi-Tenancy Standards:** Franchisors managing multiple brands or acquisitions require multi-tenancy support, but no standard architecture is widely adopted; each vendor uses proprietary approaches.

### Regulatory Evolution

- FTC modernizing Franchise Rule (2023-2024) with potential updates to electronic delivery, data reporting, and social media disclosure requirements
- State franchising laws increasingly require electronic filing and data standardization
- International franchisors face growing GDPR/CCPA compliance demands, driving need for privacy-by-design in franchise software

### Technical Recommendations

1. **Use REST/JSON APIs** with OAuth 2.0 for all third-party integrations to maintain industry alignment
2. **Adopt OpenAPI 3.1** for self-documenting APIs
3. **Implement JSON Schema** for FDD and franchise agreement structure validation
4. **Support SAML 2.0/OpenID Connect** for enterprise SSO integration with franchisee systems
5. **Build for GDPR/CCPA compliance** from the start with data residency and consent management
6. **Consider Model Context Protocol (MCP)** for AI-powered franchise insights and compliance recommendations
