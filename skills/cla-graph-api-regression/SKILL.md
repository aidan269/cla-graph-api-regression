on it. let's see what we can actually check here.

# Universal Print Graph API Regression Check Skill

## Overview

This skill assesses whether a tenant's Universal Print deployment is affected by the Microsoft Graph API code regression disclosed on 2026-04-22, which causes intermittent `Sharing Print Failed` errors when administrators attempt to create printer shares via the Universal Print portal or Graph endpoints. The issue is a service-side regression rather than a customer-deployable patch, so this skill focuses on detection, blast-radius scoping, and operational mitigation until Microsoft rolls the fix.

## Key Incident Details

- **Service affected**: Microsoft Universal Print (cloud print)
- **Root cause**: A code change in the Microsoft Graph API surface that handles printer-share creation
- **Observable symptom**: `Sharing Print Failed` errors in the Universal Print portal, intermittent
- **Failing operations**: `POST /print/shares` and related Graph calls under `/print/*`
- **Classification**: Availability / service regression (not a credential or data-exfil event)
- **First reported**: 2026-04-22
- **Fix location**: Microsoft-side; tenants cannot self-patch

Because this is a Graph-layer regression, any automation, ITSM connector, MDM flow, or provisioning script that calls `/print/shares` during the incident window should be considered suspect for silent failures even if no user raised a ticket.

## Procedure Overview

The skill executes seven sequential checks against the target tenant and supporting infrastructure:

1. **Tenant scoping** — enumerates the Entra ID tenant(s) in scope via `az account show` and `Get-MgContext`, confirms Universal Print is licensed (`Microsoft 365 E3/E5`, `Universal Print` standalone SKU) and that the `Print.ReadWrite.All` / `PrinterShare.ReadWrite.All` scopes are granted to any calling app registrations.
2. **Printer and share inventory** — queries `GET /print/printers` and `GET /print/shares` to snapshot current state, counts shares created in the 7 days preceding the incident window, and flags any printers with zero associated shares that *should* have them.
3. **Failure reproduction** — attempts a controlled `POST /print/shares` against a non-production printer to confirm whether the tenant is currently hitting the regression, capturing the exact HTTP status, `x-ms-ags-diagnostic` header, and correlation ID for Microsoft support.
4. **Audit log sweep** — pulls Entra ID audit logs and Universal Print activity logs for `Create printer share` events with `result = failure` since 2026-04-22, correlating against the Graph error taxonomy (`UnknownError`, `ServiceNotAvailable`, `generalException`).
5. **Automation exposure** — scans Logic Apps, Power Automate flows, Intune configuration profiles, Autopilot provisioning packages, and custom scripts (`Invoke-MgGraphRequest`, `msgraph-sdk`, `@microsoft/microsoft-graph-client`) for calls to `/print/shares` that may have silently failed without alerting.
6. **Downstream impact check** — reviews Intune printer-deployment policies and Endpoint Manager assignments for shares that failed to materialize, cross-referencing device-side printer enumeration to identify users who can no longer see expected printers.
7. **Microsoft service health correlation** — captures the current entry from the Microsoft 365 admin center Service Health dashboard and the Message Center advisory ID tied to this regression so that remediation timing aligns with Microsoft's published rollout.

## Exposure Classification

- **HIGH**: Reproduction step returns a failure, AND automated provisioning pipelines (Intune, Autopilot, Logic Apps) have attempted share creation during the incident window. Users are actively blocked from printing on newly deployed endpoints.
- **MEDIUM**: Reproduction fails intermittently OR audit logs show `Create printer share` failures, but provisioning is manual and the admin team is aware. No blocked end users confirmed yet.
- **LOW**: Tenant uses Universal Print but reproduction succeeds and no failure events appear in audit logs during the incident window.
- **NONE**: Universal Print not licensed or not in use; no Graph `/print/*` calls in the environment.

## Critical Remediation

For HIGH exposure: pause any automated printer-share provisioning (disable affected Logic Apps, Power Automate flows, and Intune printer-deployment policy assignments) to prevent a backlog of silently failed shares. Open a Microsoft support case referencing the captured correlation IDs and the Message Center advisory. Communicate to helpdesk that `Sharing Print Failed` is a known service-side issue — do not rotate credentials, recreate app registrations, or delete/recreate printers, as none of those actions will resolve a Graph regression and several will create cleanup work once Microsoft ships the fix. Once Microsoft confirms rollout completion, replay failed share creations from the audit log in a controlled batch and verify device-side printer visibility before re-enabling automation.

[PUSH_READY]