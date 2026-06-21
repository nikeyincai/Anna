# Certificate Rollover Guidelines for CI/CD Pipelines

*Last reviewed: June 2026*

This document provides guidance for designing and implementing automatic leaf and intermediate certificate rollover in CI/CD environments. It is intended for security, platform, and DevOps teams.

## Regulatory context

The intervals below are no longer just internal best practice — they're increasingly forced by industry rules. The CA/Browser Forum (the body that sets requirements for publicly-trusted certificates) has approved phased reductions to both certificate lifetimes and the reuse window for the validation data that backs them:

- **Public TLS certificates (Ballot SC-081v3):** maximum validity drops from 398 days → 200 days (March 15, 2026) → 100 days (March 15, 2027) → 47 days (March 15, 2029). Domain Control Validation (DCV) reuse follows the same schedule, eventually capping at 10 days.
- **Code signing certificates (Ballot CSC-31):** maximum validity dropped from 39 months to 460 days (~15 months), effective March 1, 2026.

These changes apply to **publicly-trusted** certificates only — internal/private CAs aren't bound by them, but many organizations choose to track the same direction of travel for consistency and to avoid relearning automation under pressure later.

The practical effect: manual or semi-manual renewal processes that were merely inconvenient at 200–398 days become operationally untenable at 47–100 days. Automation is no longer optional for anything chaining to a public CA.

## Recommended certificate rollover intervals

| Certificate type | Current maximum validity (as of mid-2026) | Recommended rollover / renewal target in CI/CD |
|---|---|---|
| Public TLS leaf cert | 200 days (dropping to 100 days March 2027, 47 days March 2029) | Renew every 30–45 days now; tighten to ~20 days by 2027 and ~10 days by 2029 |
| Internal TLS leaf cert | Not governed by CA/Browser Forum; historically 1–2 years | Renew every 90 days (no more than 180 days) |
| Code signing certificate | 460 days (~15 months), down from a previous 39-month max; EV/OV signing keys must live in a FIPS 140-2 Level 2+ (or Common Criteria-equivalent) HSM | Rotate certificate and key at least every 12 months; manage as a CI/CD-issued credential rather than a manually procured, multi-year token |
| Intermediate CA certificate | Commonly 5–10 years | Plan rollover every 3–5 years, with a defined transition plan; maintain a separate contingency plan for unplanned rollover (see §6.4) |

**Also track separately:** DCV reuse windows (200 → 100 → 10 days through 2029) and OV/EV Subject Identity Information reuse (now 398 days, down from 825). Even if a leaf certificate hasn't expired, pipelines that re-request certificates from a public CA may need to re-run domain or organization validation on a shorter cycle than the certificate's own lifetime.

The rest of this document describes how to implement processes and automation so these rollover targets can be met without manual intervention.

---

## 1. Definitions and scope

- **Leaf certificate:** End entity certificate used directly by an application, service, or signing process (e.g., TLS server cert, mTLS client cert, code signing cert).
- **Intermediate CA certificate:** Certificate for a subordinate CA that issues leaf certificates and is signed by a root CA.
- **Rollover:** Planned (or, in some cases, emergency) transition from one certificate or CA to another, ensuring that both old and new certificates remain valid and trusted during an overlap period.
- **CI/CD pipeline:** Automated process that builds, tests, signs, and deploys software and infrastructure.

This document covers:
- Automated leaf certificate enrollment and renewal via CI/CD.
- Safe and predictable intermediate CA rollover — planned and unplanned — ensuring CI/CD pipelines and downstream environments remain functional during transitions.

---

## 2. Policy and design principles

1. **Short-lived leaf certificates**
   - Use shorter lifetimes than the maximum allowed by external or internal CAs.
   - Design pipelines so that renewing and deploying new certificates is routine and fully automated.

2. **Centralized trust management**
   - Use a central source of truth (PKI or certificate management platform) for roots and intermediate CAs.
   - Distribute trust bundles from this central source rather than embedding individual certificates directly in application repositories.

3. **No hard-coded intermediates**
   - Avoid pinning specific intermediate certificates in code, images, or configuration files.
   - Consume chains and trust bundles dynamically so that intermediate rollover does not require code changes.

4. **Staged intermediate rollover**
   - Treat intermediate CA changes as planned events with a defined overlap period.
   - Ensure all environments trust both old and new intermediates before switching issuance.

5. **Continuous monitoring and auditing**
   - Monitor certificate validity and chain composition.
   - Alert on approaching expiry, shrinking DCV/SII reuse windows, and non-compliant issuers or lifetimes.

6. **Automate for shrinking validity windows**
   - Treat ACME (RFC 8555) or an equivalent API-driven issuance protocol as the default for any certificate chaining to a public CA — manual issuance workflows will not keep pace with 100-day and 47-day renewal cycles.
   - Re-validate automation coverage annually against the current CA/Browser Forum schedule, since the targets in §"Recommended certificate rollover intervals" will continue to tighten through 2029.

---

## 3. Centralizing CA and trust distribution

### 3.1 CA and lifecycle system

Organizations should designate a primary CA or certificate lifecycle management system that CI/CD pipelines will use. This may be:
- An external Certificate Authority with APIs or ACME support.
- An enterprise certificate management platform.
- An internal PKI with a suitable API or automation gateway.

The chosen system should:
- Issue certificates and return full chains (leaf + intermediates + root if needed).
- Publish current root and intermediate bundles.
- Support automated renewal and revocation.
- Surface DCV/SII reuse status, not just certificate expiry, so pipelines can anticipate re-validation requirements ahead of issuance.

### 3.2 Trust bundle management

Define a process for building and distributing standardized trust bundles:
- Include all current roots and active intermediate CAs.
- Publish bundles through a single source of truth (e.g., configuration repository, artifact repository, or configuration management system).
- Consume these bundles from:
  - CI/CD build agents and runners.
  - Runtime environments (containers, VMs, appliances).
  - Any custom verification tools.

Trust bundle updates should be managed as a controlled change (e.g., via pull request and deployment pipeline).

---

## 4. Removing hard-coded intermediates

To support intermediate rollover, pipelines and applications must not depend on fixed intermediate certificates embedded directly in configuration.

Recommended actions:

1. **Inventory usage**
   - Identify repositories, base images, and configuration files that:
     - Contain PEM-encoded intermediate certificates.
     - Import custom Java keystores or similar that were manually curated.
     - Reference explicit chain files for web servers or proxies.

2. **Refactor usage**
   - Replace static intermediates with:
     - References to standard trust bundles.
     - Use of platform/system trust stores where appropriate.

3. **Deployment model**
   - Ensure container images and runtime environments retrieve trust bundles at build or deploy time, not hard-coded at image creation time (unless images are frequently refreshed).

This design allows intermediate rollover to be handled once in the trust bundle, rather than across many repositories.

---

## 5. Automated leaf certificate enrollment and renewal in CI/CD

The CI/CD pipeline should be responsible for obtaining and renewing leaf certificates, not intermediates.

### 5.1 High-level pattern

1. **Request certificates automatically**
   - A pipeline stage requests a new certificate from the CA for a given identity (domain, service name, or signing identity).
   - The request type is tied to a profile that determines which intermediate CA issues the certificate.

2. **Retrieve full chain**
   - The pipeline receives:
     - Leaf certificate
     - Intermediate chain (and, optionally, root)

3. **Store securely**
   - Certificates and private keys are placed in:
     - Short-lived workspace for immediate use, and/or
     - Secure key/certificate stores (e.g., vaults or HSMs) for longer-term access. Code-signing keys subject to EV/OV requirements must reside in a FIPS 140-2 Level 2+ or Common Criteria-equivalent HSM rather than a general-purpose secret store.

4. **Use in subsequent stages**
   - Certs are used by:
     - Signing stages (e.g., code signing).
     - Deployment stages that configure TLS for services or ingress controllers.
     - Integration tests that require mutual TLS.

5. **Renew automatically**
   - Scheduled or recurring pipelines regularly:
     - Evaluate expiry **and** remaining DCV/SII reuse window.
     - Request new leaf certificates prior to expiry.
     - Update secrets, keystores, and configurations automatically.

### 5.2 Rollover intervals

- Configure pipelines so that new leaf certificates are issued and deployed well before expiry — for current 200-day public TLS certificates, this means triggering renewal with at least 30–45 days of validity remaining; tighten this margin proportionally as the maximum validity shrinks to 100 and then 47 days.
- For short-lived certificates, renewal should run on a fixed, frequent cadence (e.g., every 30 days) rather than relying on someone noticing an expiry warning.
- Treat any pipeline still issuing certificates manually, or relying on multi-year code-signing tokens, as a migration priority — the 460-day code-signing cap and sub-100-day TLS caps make manual handling a near-term reliability risk, not just a future one.

---

## 6. Intermediate CA rollover process

Intermediate CA rollover should be implemented as a staged process — with a separate, faster path available for unplanned events.

### 6.1 Preparation

1. **New intermediate creation**
   - A new intermediate CA is created and signed by the existing root.
   - The certificate lifecycle system is configured to support issuing certificates under the new intermediate (e.g., via a new or updated profile).

2. **Trust store update**
   - Trust bundles are updated to include:
     - Existing root(s).
     - Existing intermediate(s).
     - New intermediate(s).
   - All CI/CD build agents and runtime environments are updated to consume the new bundle.

3. **Verification**
   - Connectivity tests verify that clients in all environments can validate chains that include the new intermediate.

### 6.2 Switching issuance to the new intermediate

1. **Profile update**
   - The CA's issuance profiles used by CI/CD are updated so that new leaf certificates chain to the new intermediate.

2. **Pipeline behavior**
   - CI/CD pipelines continue to request certificates in the same way.
   - Newly issued certificates automatically chain to the new intermediate without pipeline changes.

3. **Overlap period**
   - Both old and new leaf certificates co-exist:
     - Old certificates chained to the previous intermediate.
     - New certificates chained to the new intermediate.
   - Validation succeeds everywhere because both intermediates are present in trust bundles.

### 6.3 Decommissioning the old intermediate

After a defined overlap period and once all old certificates have expired or been rotated:

1. **Cease issuance**
   - Stop issuing certificates from the old intermediate.

2. **Update trust bundles**
   - Remove the old intermediate from trust bundles.
   - Roll out updated bundles to all environments through the standard configuration process.

3. **Verification and cleanup**
   - Confirm that no active certificates chain to the old intermediate.
   - Update documentation to mark the old intermediate as decommissioned.

### 6.4 Emergency / unplanned intermediate or root rollover

Planned rollover assumes weeks or months of lead time. Some events don't allow that — a root or intermediate may be distrusted by a browser/OS root program, or a CA may be compelled to revoke an intermediate due to compromise or compliance failure, with a much shorter notice window.

To reduce exposure when this happens:

1. **Maintain readiness, not just process**
   - Keep a tested, documented path for emergency trust-bundle updates that can be executed in hours, not the weeks assumed by §6.1–6.3.
   - Pre-identify which services and pipelines would be affected if a given root or intermediate were distrusted, so impact can be scoped immediately.

2. **Decouple issuance from a single CA where feasible**
   - For critical services, evaluate maintaining issuance capability with a secondary CA so that a single root/intermediate event doesn't block all certificate issuance.

3. **Monitor CA/root-program announcements**
   - Track root program changes (e.g., browser distrust notices) as an operational signal, not just a compliance one — these can force an intermediate rollover outside the normal 3–5 year planning cycle.

---

## 7. Detection, validation, and alerting

To ensure rollover happens predictably and safely:

1. **Policy checks in CI/CD**
   - Add validation steps in pipelines that:
     - Check certificate expiry and enforce minimum remaining lifetime.
     - Check remaining DCV/SII reuse window where applicable.
     - Confirm that issuers and chain composition comply with policy (e.g., only approved roots/intermediates).

2. **Certificate inventory and monitoring**
   - Maintain an inventory of certificates issued for critical services.
   - Monitor for:
     - Certificates nearing expiry.
     - Certificates chaining to deprecated intermediates.
     - Unexpected issuers.

3. **Alerting**
   - Configure alerts for certificate events, including:
     - Approaching expiry.
     - Issuance from non-approved intermediates or roots.
     - Failure to obtain or deploy updated certificates.

---

## 8. Documentation and testing

Before a live intermediate rollover:

1. **Document the process**
   - Source of truth for roots and intermediates.
   - How trust bundles are built and distributed.
   - How CI/CD requests certificates and which profiles are used.
   - The planned overlap and decommission timelines, plus the emergency path from §6.4.

2. **Test in non-production**
   - Use a staging or test PKI environment to simulate:
     - Introduction of a new intermediate.
     - Trust bundle updates.
     - Issuance switching from old to new intermediate.
     - Removal of the old intermediate.
     - An emergency/expedited rollover, on at least an annual basis.

3. **Review and sign off**
   - Ensure security, operations, and application teams understand and approve the rollover plan.

---

## Sources

- CA/Browser Forum, Ballot SC-081v3 — TLS certificate validity and data-reuse reduction schedule (398 → 200 → 100 → 47 days, March 2026–2029).
- CA/Browser Forum, Ballot CSC-31 — Code signing certificate maximum validity reduction (39 months → 460 days, effective March 1, 2026).

---

## Appendix A: Standards Mapping by Section

The table below maps each section of this document to the external standards, baseline requirements, or specifications it draws on. This is a guidance-level mapping, not a formal compliance crosswalk — teams subject to a specific audit framework (e.g., WebTrust for CAs) should verify against the current text of each standard directly.

| Section | Standards / references followed | Why it applies |
|---|---|---|
| 1. Definitions and scope | RFC 5280 (Internet X.509 Public Key Infrastructure Certificate and CRL Profile) | Defines the leaf / intermediate CA / chain terminology used throughout the document |
| 2. Policy and design principles | NIST SP 800-57 Part 1 (Recommendation for Key Management); RFC 8555 (ACME) | NIST SP 800-57 underpins the general short-lifetime / key-rotation principle; RFC 8555 is the basis for the automation principle in §2.6 |
| 3. Centralizing CA and trust distribution | RFC 5280 (certification path and trust anchor handling); CA/Browser Forum Baseline Requirements for the Issuance and Management of Publicly-Trusted TLS Server Certificates, §6.1–6.3 | Governs how roots/intermediates are issued, published, and expected to be distributed by CAs and consumed by relying parties |
| 4. Removing hard-coded intermediates | RFC 5280 §6 (certification path validation); CA/Browser Forum Baseline Requirements | Path-validation behavior depends on dynamic chain discovery rather than pinned intermediates, consistent with how the BRs expect chains to be built |
| 5. Automated leaf certificate enrollment and renewal | RFC 8555 (ACME); CA/Browser Forum TLS Baseline Requirements, Ballot SC-081v3 (validity and DCV-reuse schedule); CA/Browser Forum Code Signing Baseline Requirements, Ballot CSC-31 (code-signing validity cap); FIPS 140-2 / Common Criteria (HSM key-storage requirement for EV/OV code signing) | Defines both the automated issuance protocol and the regulatory validity/reuse limits the renewal cadence must satisfy |
| 6. Intermediate CA rollover process | CA/Browser Forum Baseline Requirements (intermediate CA issuance and decommissioning); Mozilla Root Store Policy, Chrome Root Program, Microsoft Trusted Root Program, Apple Root Program | Root program policies govern planned intermediate lifecycles and are the source of unplanned distrust/sunset events referenced in §6.4 |
| 7. Detection, validation, and alerting | RFC 6960 (Online Certificate Status Protocol); RFC 5280 §6 (path validation); CA/Browser Forum Baseline Requirements (revocation and issuer compliance) | Basis for expiry, revocation, and chain-compliance checks pipelines should run |
| 8. Documentation and testing | RFC 3647 (Certificate Policy and Certification Practices Framework) | Standard structure for documenting CA-facing practices, used as the model for what the rollover runbook should capture |
