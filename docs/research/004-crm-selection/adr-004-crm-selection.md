# ADR-004: CRM and Marketing Automation Selection

## Status

Accepted

## Date

2026-02-15

## Context

ADR-003 identifies the CRM as an external system that the portal reads from but does not own. The CRM choice determines how the BFF reads timeline data, how Temporal workers record events, whether live updates can use PostgreSQL LISTEN/NOTIFY, and how email/marketing campaigns are managed.

The platform will also host CRM instances for customers, so operational experience with the chosen stack has value beyond the portal itself.

### Requirements

- Self-hosted, free (no per-seat licensing)
- SSO via existing Keycloak (cozy realm, OIDC)
- Customer segmentation, email campaigns, marketing campaigns
- Dynamic entity data (custom entities/fields for tenant events, workflow outcomes)
- REST API for Go BFF reads and Temporal activity writes
- GDPR erasure support
- PostgreSQL preferred for co-location with Temporal + KillBill cluster

## Considered Options

1. **EspoCRM + Mautic** — CRM with OIDC + dedicated marketing automation
2. **Twenty CRM + marketing tool** — PostgreSQL-native CRM, no SSO yet
3. **SuiteCRM** — standalone CRM+marketing, MySQL-only
4. **Erxes** — SSO only in paid Enterprise tier, MongoDB-only

See `crm-landscape-research.md` for the full evaluation.

## Decision

**EspoCRM** for CRM. **Mautic** for marketing automation and email campaigns.

## Rationale

- **Keycloak SSO**: EspoCRM has native OIDC support with documented Keycloak integration. This matches the platform's OIDC-first authentication approach directly, unlike SuiteCRM (SAML with configuration pain) or Twenty (no generic OIDC yet).
- **Code-managed entity structures**: EspoCRM stores entity definitions as JSON files in `custom/Espo/Modules/{Module}/Resources/metadata/` (entityDefs, scopes, clientDefs, layouts). These are plain files that live in a Git repo and are applied via `php command.php rebuild`. The recommended pattern is to build a custom module using the [ext-template](https://github.com/espocrm/ext-template), package it as a zip, and deploy via `php command.php extension --file="package.zip"`. This gives us a fully code-managed, CI/CD-deployable entity schema — no reliance on the UI Entity Manager. The admin REST API (`/api/v1/Admin/fieldManager/`) provides an additional programmatic path for validation or drift detection.
- **Marketing via Mautic**: EspoCRM covers CRM fundamentals but lacks full marketing automation. Mautic provides email campaigns, drip sequences, lead scoring, and advanced segmentation. Both are established open-source projects with large communities.
- **Operational leverage**: The team will host EspoCRM and Mautic for customers as well. The operational burden of running two systems is justified because it becomes a shared capability across the business, not portal-specific overhead.
- **PostgreSQL**: EspoCRM has experimental PostgreSQL support (v7.4+). This can be evaluated for co-location with the shared cluster. If the experimental status proves problematic, MySQL/MariaDB fallback is straightforward with the BFF polling instead of LISTEN/NOTIFY.

### Trade-offs accepted

- **Two systems** instead of one (SuiteCRM standalone). Accepted because of operational reuse across customers and better architectural alignment.
- **EspoCRM PostgreSQL is experimental**. Accepted as a try-first approach with MySQL fallback available.
- **Mautic requires MySQL/MariaDB**. Accepted — Mautic runs its own database; it does not need to co-locate with the Temporal/KillBill cluster.

## Consequences

- Portal BFF reads CRM data via EspoCRM REST API (OpenAPI-generated Go client)
- Temporal workers write CRM events via EspoCRM REST API as activities
- Email campaigns (welcome, payment reminders, newsletters) are managed in Mautic
- Mautic syncs contacts from EspoCRM (standard integration)
- Both systems authenticate via Keycloak (EspoCRM: OIDC, Mautic: SAML)
- Infrastructure team provisions EspoCRM + Mautic alongside the portal stack
