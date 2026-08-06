# BOS-0002 — Company Profile & Multi-Tenant Configuration

**Document Type:** Approved Specification (Architecture ruled by Chief Architect — ChatGPT)
**Milestone:** BOS-0002
**Status:** Ready for Development
**Priority:** Critical
**Dependencies:** ✅ BOS-0001 Repository Setup
**Chief Architect:** ChatGPT
**Implementer:** Claude Code (Principal Software Engineer)
**Product Owner:** Christopher Powell

> This document supersedes the Claude Code draft dated 2026-08-01. It is the canonical BOS-0002 specification, authored and approved by the Chief Architect. Claude Code implements against this document only.

---

## 1. Executive Summary

The Company Profile Module establishes the multi-tenant architecture for the AI Business Operating System (AI BOS). Every business supported by AI BOS — including KrispiClean, Aurelium, Farm Link Jamaica, Sun Sky Solar, and future companies — uses the exact same workflows. No workflow contains company-specific logic. Instead, workflows dynamically load configuration from a Company Profile. This design allows one automation to serve unlimited companies.

## 2. Purpose

Provide a centralized configuration service that defines every company operating within AI BOS. The Company Profile is the source of truth for: branding, business rules, pricing, AI providers, marketing, integrations, API key references, publishing destinations, and automation behavior.

## 3. Scope

- Company Profile Schema
- Company Loader
- Environment Configuration
- Secrets Management
- Multi-tenant Routing
- Validation
- Versioning

Out of scope for this milestone: AI Provider Manager internal routing logic, the Workflow Engine itself, billing/subscription metering, admin console UI (all separate milestones).

---

## 4. Functional Requirements

| ID | Requirement |
|----|-------------|
| FR-1 | Company Registry — support unlimited companies (e.g., KrispiClean, Farm Link Jamaica, Sun Sky Solar, Aurelium, future companies) |
| FR-2 | Unique Company ID per company (e.g., `KRISPI`, `FARMLINK`, `SUNSKY`, `AURELIUM`) |
| FR-3 | Company Metadata: `company_id`, `company_name`, `legal_name`, `industry`, `country`, `currency`, `timezone`, `language`, `website`, `logo`, `brand_colors`, `contact_email`, `phone`, `address` |
| FR-4 | AI Configuration: each company selects a `default_provider` and `fallback` list; routing may consider quality, cost, speed, context window |
| FR-5 | Marketing Configuration: `brand_voice`, `writing_style`, `hashtags`, `CTA`, `target_audience`, `social_platforms`, `SEO defaults` |
| FR-6 | Service Configuration: enable/disable modules (Marketing OS, Sales OS, Finance OS, Legal OS, Customer Service OS, Content Factory, Analytics) |
| FR-7 | Workflow Configuration: timezone, notifications, retry policy, approval required, publishing schedule, content limits |
| FR-8 | API Configuration: reference only, never secrets — covers OpenAI, Claude, Gemini, Stripe, WooCommerce, WordPress, TikTok, Facebook, Google, n8n, Twilio |
| FR-9 | Feature Flags: `video_generation`, `voice_generation`, `beta_features`, `experimental`, `AI_autopilot` |
| FR-10 | Pricing Configuration: company-specific (e.g., KrispiClean — `travel_fee`, `minimum_charge`, `service_levels`, `discounts`, `sales_tax`; Farm Link — `shipping`, `currency`, `commissions`) |

---

## 5. Non-Functional Requirements

Multi-tenant · Cloud-ready · Version controlled · Extensible · Backward compatible · Secure · Cached · Hot reload supported

---

## 6. Architecture (Approved)

```
AI BOS
   │
Company Profile API   (standalone service — see Decision 3)
   │
Company Loader Service
   │
Company Profile record   (PostgreSQL — see Decision 1)
   │
Dynamic Configuration Layer
   │
Shared Enterprise Workflows
   │
AI Provider Manager
```

Consumers of the Company Profile API include: Marketing OS, Sales OS, Finance OS, Legal OS, Content Factory, Analytics, AI Provider Manager, Publishing Service, Script Service, Voice Service, Image Service, and future customer portals. Consumers own nothing about company configuration — they resolve it from the API.

### Architecture Decision Record (ruled by Chief Architect)

**Decision 1 — Storage.** Use a dedicated **database-backed Company Profile Service** (PostgreSQL as system of record) as the canonical Company Registry. Do not use n8n Data Tables as the primary datastore — they remain acceptable for workflow cache, temporary runtime state, queue metadata, and execution snapshots only.

**Decision 2 — Secrets.** Use the **n8n Credential Store** for the initial production implementation, wrapped behind a `CredentialResolver` / Secrets Provider abstraction. The Company Profile stores only a `credential_ref` (e.g., `"credential_ref": "openai-production"`), never the secret itself. Migration path: Phase 1 n8n Credentials → Phase 2 Vault → Phase 3 cloud-specific providers, with no workflow changes required at any phase.

**Decision 3 — API shape.** Implement `/api/v1/company-profiles` as its **own standalone service**, not a module inside another service. It independently owns: company lookup, validation, schema enforcement, versioning, environment overrides, credential reference resolution, configuration caching, profile CRUD, and audit logging.

---

## 7. Directory Structure

```
AI-Business-Operating-System/
├── company-profiles/
│   ├── krispiclean/
│   │   ├── company.json
│   │   ├── branding.json
│   │   ├── marketing.json
│   │   ├── pricing.json
│   │   └── integrations.json
│   ├── farmlink/
│   ├── sunsky/
│   └── aurelium/
│
├── services/
│   └── company-loader/
│       ├── README.md
│       ├── schema/
│       ├── validators/
│       └── loader/
│
└── docs/
    └── architecture/
        └── BOS-0002-Company-Profile-Multi-Tenant-Configuration.md
```

---

## 8. Data Model

### Example Company Profile (`company.json`)

```json
{
  "company_id": "KRISPI",
  "company_name": "KrispiClean",
  "industry": "Residential Cleaning",
  "country": "USA",
  "currency": "USD",
  "timezone": "America/New_York",

  "branding": {
    "primary": "#0F7BFF",
    "secondary": "#00C853",
    "logo": "/branding/logo.png"
  },

  "marketing": {
    "tone": "Professional Friendly",
    "audience": "Homeowners"
  },

  "ai": {
    "default": "OpenAI",
    "fallback": ["Claude", "Gemini"]
  }
}
```

Secrets are referenced, never embedded:
```json
{
  "openai_secret": "OPENAI_API_KEY"
}
```

### Company Loader Interface

```typescript
interface CompanyProfile {
  load(companyId: string): Promise<CompanyProfileData>;
  validate(): ValidationResult;
  cache(): void;
  refresh(): Promise<void>;
}
```

---

## 9. API Contracts

```
GET    /api/v1/company-profiles/{companyId}
POST   /api/v1/company-profiles
PUT    /api/v1/company-profiles/{companyId}
DELETE /api/v1/company-profiles/{companyId}
GET    /api/v1/company-profiles/{companyId}/branding
GET    /api/v1/company-profiles/{companyId}/marketing
GET    /api/v1/company-profiles/{companyId}/pricing
```

---

## 10. Data Flow

```
Workflow Starts
   │
Read Company ID
   │
Load Company Profile
   │
Validate
   │
Merge Environment Settings
   │
Return Configuration
   │
Execute Workflow
```

---

## 11. Validation Rules

**Required:** Company ID, Name, Timezone, Currency, Industry
**Optional:** Logo, Branding, Marketing, SEO, Publishing

---

## 12. Security

Never store: API Keys, Passwords, OAuth Tokens, Private certificates. Reference secrets only via `credential_ref`, resolved through the Secrets Provider abstraction defined in Decision 2.

---

## 13. Error Handling

- Missing Company → `404 Company Not Found`
- Invalid Configuration → `Configuration Validation Failed`
- Missing Secret → `Secrets Manager Error`

---

## 14. Performance & Scalability

Supports 1 → 10 → 100 → 10,000 → unlimited companies with no workflow modifications required. Configuration is cached at the loader layer with hot-reload support.

---

## 15. Acceptance Criteria

- [ ] A new company can be onboarded via configuration files only — no code or workflow changes.
- [ ] KrispiClean, Farm Link Jamaica, Sun Sky Solar, and Aurelium each resolve as distinct, isolated profiles.
- [ ] No AI BOS workflow contains a hardcoded company name, ID, or credential.
- [ ] Company Profile Service is deployed standalone, backed by PostgreSQL.
- [ ] Secrets resolve through the Credential Resolver abstraction; no raw secrets appear in profile records or logs.
- [ ] All CRUD operations are audit-logged.

---

## 16. Future Enhancements

Company inheritance (parent → child profiles) · Regional overrides · White-label deployments · Multi-brand organizations · Customer-specific AI settings · Environment-specific profiles (Dev/Staging/Prod) · Dynamic pricing engines · Policy-as-code configuration

---

## 17. Version History

| Version | Date | Author | Change |
|---------|------|--------|--------|
| 1.0 (Draft) | 2026-08-01 | Claude Code | Initial specification submitted for architectural review |
| 2.0 (Approved) | 2026-08-01 | ChatGPT (Chief Architect) | Canonical spec issued; ADR rulings on storage, secrets, and API shape incorporated |

---

## 18. Handoff to Implementation

This document is approved for implementation. Next step per project governance (Documentation → Architecture → Specification → **Review** → Implementation): Product Owner review, then Claude Code begins implementation of the Company Profile Service, Company Loader, and schema/validators per §7–§9.
