# CRM Selection Research

> Status: **Research / Draft**
> Date: 2026-02-15

## 1. Requirements Summary

Derived from ADR-003 and project context:

| Requirement | Detail |
|---|---|
| **Self-hosted, free** | No per-seat licensing. Must run on our infrastructure |
| **SSO / SAML** | Must integrate with existing Keycloak (cozy realm, OIDC primary) |
| **Customer segmentation** | Group and filter customers by plan, region, lifecycle stage, etc. |
| **Email campaigns** | Send welcome emails, payment reminders, newsletters from the CRM |
| **Marketing campaigns** | Multi-channel campaign management with analytics |
| **Dynamic entity data** | Custom entities/fields to model tenant-specific data beyond standard CRM objects |
| **PostgreSQL preferred** | Co-location with Temporal + KillBill cluster enables LISTEN/NOTIFY for live updates |
| **REST or GraphQL API** | BFF reads CRM timeline data; Temporal workers write events as activities |
| **Reference-based payloads** | CRM must resolve Keycloak user IDs / KillBill refs to display names at read time |
| **GDPR erasure support** | CRM must handle erasure requests from the GDPR workflow |

---

## 2. Candidates Evaluated

### Tier 1: Full CRM Platforms

| | SuiteCRM | EspoCRM | Twenty CRM | Erxes |
|---|---|---|---|---|
| **License** | AGPL-3.0 | GPL-3.0 | AGPL-3.0 | AGPL-3.0 (core plugins free) |
| **GitHub Stars** | ~4.6k | ~1.8k | ~28k+ | ~3.5k |
| **Language** | PHP (Symfony) | PHP (custom framework) | TypeScript (NestJS + React) | TypeScript (Node.js, React) |
| **Database** | MySQL / MariaDB **only** | MySQL / MariaDB / PostgreSQL (experimental since v7.4) | **PostgreSQL** (native) | **MongoDB** |
| **SSO / SAML** | Native SAML 2.0 (Keycloak documented) | Native OIDC (Keycloak documented, minor config quirks) | OAuth (Google/Microsoft); OIDC/SAML immature | Enterprise-only (not in OSS) |
| **Email campaigns** | Built-in campaign module | Built-in (templates, automation) | None | Built-in (core plugin, multi-channel) |
| **Marketing automation** | Built-in (target lists, drip) | Basic (BPMN 2.0 workflows) | None | Strong (email, SMS, chat, in-app) |
| **Customer segmentation** | Target lists, dynamic groups | Saved filters, reports | Basic list views | Segments with tags and filters |
| **Dynamic entities** | Module Builder (code-heavy) | Entity Manager (no-code UI + code API) | Custom Objects via Metadata API + GraphQL | Plugin architecture (code-heavy) |
| **API** | REST (JSON API v8) | REST + auto-generated OpenAPI spec | REST + GraphQL (schema adapts to custom objects) | GraphQL Federation |
| **Maturity** | High (SugarCRM fork, 10+ years) | Medium-High (8+ years) | Growing (YC-backed, 2+ years) | Medium (5+ years) |
| **Resource footprint** | Heavy (PHP + MySQL + Elasticsearch recommended) | Light (PHP + MySQL/PG, runs on small VPS) | Medium (Node.js + PostgreSQL) | Heavy (MongoDB + Redis + microservices) |

### Tier 2: Marketing Automation (Complementary)

| | Mautic | Dittofeed |
|---|---|---|
| **License** | GPL-3.0 | MIT |
| **Focus** | Marketing automation + email | Customer engagement + journeys |
| **Database** | MySQL / MariaDB only | PostgreSQL / ClickHouse |
| **SSO / SAML** | Native SAML 2.0 (Keycloak has known bugs) | N/A (API-first) |
| **Segmentation** | Advanced (lead scoring, behavioural) | Advanced (event-driven, streaming) |
| **Email campaigns** | Full suite (drip, A/B, templates) | Journey-based messaging |
| **API** | REST | REST |
| **Use case** | Pair with a CRM that lacks marketing features | Pair with a CRM for event-driven engagement |

---

## 3. Analysis Against Architecture Constraints

### 3.1 Database Compatibility (PostgreSQL co-location)

ADR-003 specifies a shared PostgreSQL cluster for Temporal and KillBill. A PostgreSQL-backed CRM can join this cluster, enabling LISTEN/NOTIFY for live BFF updates without polling.

| CRM | PostgreSQL | Can co-locate |
|---|---|---|
| **Twenty** | Native (only option) | Yes |
| **EspoCRM** | Experimental (v7.4+) | Possible, but less battle-tested |
| **SuiteCRM** | No | No -- requires separate MySQL/MariaDB |
| **Erxes** | No (MongoDB) | No -- requires separate MongoDB + Redis |

**Finding:** Only **Twenty** natively uses PostgreSQL. EspoCRM has experimental support. SuiteCRM and Erxes require a separate database engine entirely.

### 3.2 Keycloak SSO Integration

The platform uses Keycloak with OIDC (cozy realm). The CRM must authenticate via Keycloak.

| CRM | Protocol | Keycloak Status |
|---|---|---|
| **SuiteCRM** | SAML 2.0 | Documented but community reports configuration pain, limited docs |
| **EspoCRM** | OIDC | Documented, known `client_secret_post` quirk, works with fix |
| **Twenty** | OAuth2 | Google/Microsoft only; generic OIDC is a GitHub feature request (#4328) |
| **Erxes** | N/A (Enterprise) | Not available in open-source edition |

**Finding:** **EspoCRM** has the most natural fit (OIDC, same as Keycloak primary protocol). SuiteCRM works via SAML (Keycloak supports both). Twenty lacks generic OIDC -- a blocker unless the feature request ships. Erxes is a non-starter for SSO in the free tier.

### 3.3 Dynamic Entity Data

ADR-003's Layer 2 (Audit/CRM Log) stores an append-only timeline with reference IDs. The CRM needs to model custom entities beyond contacts/companies (e.g. tenant events, workflow outcomes, infrastructure actions).

| CRM | Approach | Flexibility |
|---|---|---|
| **Twenty** | Custom Objects via Metadata API; GraphQL schema auto-regenerates | High -- developer-friendly, API-first |
| **EspoCRM** | Entity Manager (UI) + code-based custom entities; OpenAPI auto-generated | High -- no-code for simple cases, code for advanced |
| **SuiteCRM** | Module Builder (UI) + vardefs (code) | Medium -- functional but PHP-heavy, legacy patterns |
| **Erxes** | Plugin system (code-only) | Medium -- powerful but requires deep framework knowledge |

**Finding:** **Twenty** and **EspoCRM** both excel here. Twenty's metadata-driven GraphQL is particularly well-suited for the BFF pattern (query exactly what you need). EspoCRM's Entity Manager is more accessible for non-developers.

### 3.4 BFF Integration Pattern

The Go BFF reads CRM data for display. Temporal workers write events via CRM API calls.

| CRM | Read path (BFF) | Write path (Temporal workers) |
|---|---|---|
| **Twenty** | GraphQL or REST -- schema adapts to custom objects | REST/GraphQL API calls |
| **EspoCRM** | REST with auto-generated OpenAPI | REST API calls |
| **SuiteCRM** | REST (JSON API v8) | REST API calls |
| **Erxes** | GraphQL Federation | GraphQL mutations |

**Finding:** All four expose APIs. **Twenty's** adaptive GraphQL is strongest for a BFF that queries across custom objects. EspoCRM's OpenAPI spec generation is good for Go code generation.

### 3.5 Email and Marketing Campaigns

| CRM | Built-in email | Marketing automation | Alternative |
|---|---|---|---|
| **SuiteCRM** | Yes (campaigns, templates, target lists) | Yes | Standalone |
| **EspoCRM** | Yes (templates, mass email) | Basic (BPMN workflows) | Pair with Mautic |
| **Twenty** | No | No | Must pair with Mautic or Dittofeed |
| **Erxes** | Yes (email, SMS, chat, in-app) | Yes (strong) | Standalone |

**Finding:** **SuiteCRM** and **Erxes** are the only standalone solutions for marketing. EspoCRM covers basics. Twenty requires a separate marketing tool entirely.

---

## 4. Candidate Shortlist

Based on the analysis, three viable architectures emerge:

### Option A: EspoCRM (+ Mautic for marketing)

**Strengths:**
- OIDC support matches Keycloak natively
- Experimental PostgreSQL support enables co-location
- Entity Manager provides no-code custom entity creation
- Auto-generated OpenAPI spec simplifies Go BFF integration
- Lightest resource footprint -- runs on small VPS
- REST API is straightforward for Temporal activity calls

**Weaknesses:**
- PostgreSQL support is experimental (not production-hardened)
- Marketing is basic -- requires pairing with Mautic for campaigns
- Mautic adds operational complexity (separate PHP app, MySQL-only, own SAML config)
- Custom framework (not Symfony/Laravel) makes deep customization harder

**Operational cost:** 2 systems (EspoCRM + Mautic), mixed database backends

### Option B: Twenty CRM (+ Mautic/Dittofeed for marketing)

**Strengths:**
- Native PostgreSQL -- joins the shared cluster, LISTEN/NOTIFY works
- Modern TypeScript/NestJS stack aligns with industry direction
- Custom Objects via Metadata API + GraphQL is the strongest dynamic entity story
- GraphQL schema auto-adapts to custom objects -- ideal for BFF
- Active development, YC-backed, fast-growing community (28k+ stars)

**Weaknesses:**
- No built-in email/marketing campaigns -- must pair with another tool
- OIDC/SAML not yet supported for generic providers (Keycloak) -- GitHub issue #4328 open
- Younger project, less battle-tested in production
- SSO gap is a **hard blocker** until resolved (workaround: reverse proxy auth or Keycloak broker)

**Operational cost:** 2 systems (Twenty + marketing tool), but unified PostgreSQL

### Option C: SuiteCRM (standalone)

**Strengths:**
- Most feature-complete standalone: CRM + email + marketing + campaigns
- Native SAML 2.0 with Keycloak documentation
- Proven in enterprise environments (10+ year lineage)
- Single system covers all CRM requirements
- Largest community and ecosystem

**Weaknesses:**
- MySQL/MariaDB only -- cannot co-locate with PostgreSQL cluster
- No LISTEN/NOTIFY -- BFF must poll or use webhooks for live updates
- Legacy PHP codebase (SugarCRM heritage), outdated patterns
- Heavy resource footprint (PHP + MySQL + Elasticsearch recommended)
- Module Builder is functional but clunky for dynamic entity creation
- Community reports Keycloak SAML config is painful (limited docs, subtle bugs)

**Operational cost:** 1 system, but requires separate MySQL/MariaDB infrastructure

### ~~Option D: Erxes~~ (Eliminated)

- SSO only in Enterprise tier (not free) -- **violates requirements**
- MongoDB-only -- cannot co-locate with PostgreSQL
- Heavy microservices architecture (MongoDB + Redis + multiple services)
- Despite strong marketing features, the SSO and database constraints are disqualifying

### 4.1 Code-First Entity Management (GitOps Readiness)

A hard requirement is that entity structures are **fully managed in code** — defined in version-controlled files and deployed via CI/CD, not created through a UI.

| Dimension | EspoCRM | Twenty CRM | SuiteCRM | Mautic |
|---|---|---|---|---|
| **Definition format** | JSON files | TypeScript decorators (standard objects) / DB metadata (custom objects) | PHP vardef arrays | Doctrine PHP entities (core) / DB-only (custom fields) |
| **Definitions in files?** | Yes — fully file-based JSON | Standard: Yes. Custom: No (DB only) | Yes — PHP vardef files | Plugin entities: Yes. Custom fields: No |
| **Schema management API** | Yes (`/api/v1/Admin/fieldManager/`) | Yes (GraphQL metadata API) | No | Yes (REST `/api/fields/`) |
| **CLI for applying changes** | `php command.php rebuild` | `yarn command:prod workspace:sync-metadata -f` | Community `repair.php` scripts | `bin/console doctrine:schema:update` |
| **Extension packaging** | Yes — zip with `manifest.json`, CLI install via `php command.php extension` | N/A (modify source directly) | Yes — Module Loader zip | Yes — Composer plugins with migrations |
| **GitOps readiness** | **Strong** | **Strong** (standard), Moderate (custom) | **Moderate** | **Moderate** (plugins), Weak (custom fields) |

#### EspoCRM: How code-first works

Entity definitions live as JSON files in a layered directory structure:

```
custom/Espo/Modules/{YourModule}/Resources/
├── metadata/
│   ├── entityDefs/{EntityName}.json    # fields, links, indexes
│   ├── scopes/{EntityName}.json        # visibility, ACL, stream
│   └── clientDefs/{EntityName}.json    # frontend configuration
└── layouts/{EntityName}/
    ├── detail.json
    └── list.json
```

The recommended workflow:
1. Define entities as JSON files in a custom module directory
2. Version-control the module in Git
3. Package using [espocrm/ext-template](https://github.com/espocrm/ext-template) tooling
4. Deploy: `php command.php extension --file="package.zip" && php command.php rebuild`

This approach cleanly separates code-managed definitions (in `custom/Espo/Modules/`) from any UI-generated changes (in `custom/Espo/Custom/`). The `rebuild` command applies database schema changes from the JSON definitions.

#### Why EspoCRM wins for code-first

- **Twenty** has excellent code-first for *standard* objects (TypeScript decorators + migrations), but *custom* objects (the kind we'd create for tenant events, workflow outcomes) live only in DB metadata tables with no file export. We would need to drive custom object creation via GraphQL API calls in CI/CD — imperative, not declarative.
- **SuiteCRM** has version-controllable PHP vardefs, but no schema management API and relies on community-maintained CLI repair scripts. The dual source of truth (files + `fields_meta_data` table) creates drift risk.
- **Mautic** custom fields are DB-only with no file representation. Plugin entities use Doctrine and are code-first, but the primary extension mechanism (custom fields on contacts) cannot be version-controlled as files.

---

## 5. Evaluation Matrix

Scoring: 1 (poor) to 5 (excellent) against weighted requirements.

| Requirement (weight) | EspoCRM + Mautic | Twenty + Marketing | SuiteCRM |
|---|---|---|---|
| Self-hosted, free (must-have) | 5 | 5 | 5 |
| Keycloak SSO (must-have) | 4 (OIDC, documented quirk) | 1 (not supported yet) | 3 (SAML, config pain) |
| PostgreSQL co-location (high) | 3 (experimental) | 5 (native) | 1 (MySQL only) |
| Customer segmentation (high) | 4 | 3 | 4 |
| Email campaigns (high) | 4 (via Mautic) | 3 (via external tool) | 5 (built-in) |
| Marketing campaigns (medium) | 4 (via Mautic) | 3 (via external tool) | 4 |
| Dynamic entities (high) | 4 (Entity Manager + API) | 5 (Metadata API + GraphQL) | 3 (Module Builder) |
| API for BFF/Temporal (high) | 4 (REST + OpenAPI) | 5 (GraphQL + REST, adaptive) | 3 (JSON API) |
| GDPR erasure (medium) | 3 | 4 (API-driven) | 3 |
| Operational simplicity (medium) | 2 (two systems) | 2 (two systems) | 4 (one system) |
| Codebase quality (low) | 3 (custom PHP) | 5 (modern TS/NestJS) | 2 (legacy PHP) |
| **Weighted total** | **~3.5** | **~3.5** (blocked by SSO) | **~3.3** |

---

## 6. Open Questions for Discussion

1. **SSO: Hard blocker or solvable?**
   - Twenty's OIDC gap (#4328) -- is a reverse-proxy auth layer (e.g. oauth2-proxy in front of Twenty) acceptable as a workaround? Or must the CRM natively support Keycloak?

2. **PostgreSQL co-location: How important is LISTEN/NOTIFY?**
   - If polling/webhooks are acceptable for live BFF updates, the database constraint relaxes and SuiteCRM becomes more viable.

3. **Marketing scope for MVP:**
   - Do we need full marketing automation at launch, or just transactional email (welcome, payment reminders)? If transactional-only, even Twenty + a simple SMTP relay (via Temporal activities) could suffice initially.

4. **EspoCRM PostgreSQL experimental:**
   - Acceptable risk for a hosting platform? Or does "experimental" mean we should plan for MySQL fallback?

5. **Two-system vs one-system:**
   - Is the operational overhead of CRM + marketing tool acceptable, or is a single system (SuiteCRM) preferred despite its database and codebase trade-offs?

---

## 7. Sources

### General CRM Comparisons
- [CRM.org: Best Open Source CRM Software for 2026](https://crm.org/crmland/open-source-crm)
- [Marmelab: Best Open Source CRM for 2026](https://marmelab.com/blog/2026/01/09/open-source-crm-benchmark-2026.html)
- [Marmelab: Best Open Source CRM for 2025](https://marmelab.com/blog/2025/02/03/open-source-crm-benchmark-for-2025.html)
- [DevDiligent: Top 5 Best Open Source CRM Software (2026)](https://devdiligent.com/blog/best-open-source-crm-software/)
- [GrowCRM: Top 20 Open-Source Self-Hosted CRMs](https://growcrm.io/2026/01/04/top-20-open-source-self-hosted-crms-in-2025/)
- [NocoBase: 4 Open Source Alternatives to Salesforce](https://www.nocobase.com/en/blog/salesforce-open-source-crmalternative)

### SuiteCRM
- [SuiteCRM SAML Docs (8.7.0+)](https://docs.suitecrm.com/8.x/admin/configuration/saml/8.7.0-saml-configuration/)
- [SuiteCRM + Keycloak SAML Community Thread](https://community.suitecrm.com/t/how-to-integrate-keycloak-suitecrm-using-saml-for-sso/82082)
- [SuiteCRM Compatibility Matrix](https://docs.suitecrm.com/admin/compatibility-matrix/)
- [SuiteCRM PostgreSQL Feature Request](https://community.suitecrm.com/t/support-for-postgresql-as-db-backend/11740)

### EspoCRM
- [EspoCRM OIDC Documentation](https://docs.espocrm.com/administration/oidc/)
- [EspoCRM Keycloak SSO Forum Thread](https://forum.espocrm.com/forum/general/98352-keycloak-sso)
- [EspoCRM Entity Manager](https://docs.espocrm.com/administration/entity-manager/)
- [EspoCRM Custom Entity Types (Dev)](https://docs.espocrm.com/development/custom-entity-type/)
- [EspoCRM API Overview](https://docs.espocrm.com/development/api/)
- [EspoCRM PostgreSQL Experimental Support (#2605)](https://github.com/espocrm/espocrm/issues/2605)
- [Keycloak client_secret_post Issue (#2786)](https://github.com/espocrm/espocrm/issues/2786)

### Twenty CRM
- [Twenty CRM Official Site](https://twenty.com/)
- [Twenty Custom Objects](https://twenty.com/developers/section/backend-development/custom-objects)
- [Twenty APIs Documentation](https://docs.twenty.com/developers/extend/capabilities/apis)
- [Twenty OIDC Feature Request (#4328)](https://github.com/twentyhq/twenty/issues/4328)
- [Twenty SSO Configuration (#10049)](https://github.com/twentyhq/twenty/issues/10049)
- [Twenty GitHub Repository](https://github.com/twentyhq/twenty)

### Erxes
- [Erxes GitHub Repository](https://github.com/erxes/erxes)
- [Erxes Campaign Feature](https://erxes.io/campaign)
- [Erxes Official Site](https://erxes.io/)

### Marketing Automation (Complementary)
- [Mautic Official Site](https://mautic.org/)
- [Mautic SAML Authentication Docs](https://docs.mautic.org/en/authentication)
- [Mautic + Keycloak SAML Issue (#12304)](https://github.com/mautic/mautic/issues/12304)
- [Dittofeed Official Site](https://www.dittofeed.com)
