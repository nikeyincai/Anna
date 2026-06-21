# Automated Certificate Rollover Procedure
## Windows Enterprise CA + Azure Key Vault for CI/CD Pipelines

*Companion implementation procedure for `certificate-rollover-guidelines_Azure_Keyvault.md`. Section references like (§5.2) point back to that document.*

This procedure covers three certificate classes end-to-end:

| Class | Issuer | Store | Target cadence (per guidelines §"Recommended rollover intervals") |
|---|---|---|---|
| Internal TLS leaf certs | Windows Enterprise CA | Key Vault Standard | Renew every 90 days, no later than 180 |
| Code signing certs | Windows Enterprise CA | Key Vault **Premium**, HSM-protected key | Rotate cert + key at least every 12 months |
| Intermediate (subordinate) CA cert | Windows Enterprise CA, signed by offline root | N/A — CA's own key, not Key Vault | Plan every 3–5 years, staged overlap |

**Important constraint that shapes everything below:** Azure Key Vault's *native* auto-renewal (`LifetimeAction: AutoRenew`, §7.3 of the guidelines) only works with CAs Key Vault is commercially partnered with (currently DigiCert and GlobalSign). A Windows Enterprise CA is **not** one of these, so Key Vault cannot call it directly. The supported pattern for any non-integrated CA — confirmed against current Microsoft documentation — is:

```
Key Vault creates a key pair + CSR (issuer = "Unknown")
        │
        ▼
CSR submitted to the Windows Enterprise CA (certreq, over RPC/DCOM)
        │
        ▼
CA auto-issues (template configured for no manual approval)
        │
        ▼
Signed certificate (+ chain) merged back into the same Key Vault
certificate object, creating a new version
```

Everything in Parts C and D below is this loop wrapped in orchestration, because Key Vault has no concept of "renew automatically" for this CA type — your pipeline has to drive it on a schedule.

---

## Prerequisites

- Windows Server with AD CS role installed, configured as an **Enterprise Subordinate CA** (chained to an offline root — do not run the issuing CA as the root).
- Domain-joined CA server reachable from wherever the orchestration runs (RPC/DCOM: TCP 135 + dynamic high port range, or the CA's web enrollment role if you're going that route).
- An Azure subscription, `Owner` or `User Access Administrator` rights to do initial RBAC setup.
- Azure CLI / Az PowerShell module, `dotnet` SDK (for AzureSignTool) on build agents.
- A CI/CD platform: examples below use Azure DevOps and GitHub Actions; the pattern is portable to others.
- Decide now: will the orchestration agent that talks to the CA be (a) a self-hosted CI/CD agent with network line-of-sight to the CA, or (b) an Azure Automation Hybrid Runbook Worker / Azure Function with a private network connection (ExpressRoute/VPN) to the CA. Public-cloud-hosted agents generally **cannot** reach an on-prem CA directly — pick (a) or (b) before building pipelines.

---

## Part A — Windows Enterprise CA Preparation

### A.1 Create a dedicated certificate template for short-lived leaf certificates

1. On the CA (or a management workstation with RSAT), open `certtmpl.msc`.
2. Duplicate the built-in **Web Server** template (or **Computer** template for mTLS client certs). Name it something explicit, e.g. `CI-CD-ShortLived-WebServer`.
3. **General tab:** set validity to a value comfortably inside your internal cap (guidelines recommend issuing no longer than 90 days, capped at 180 — set validity to **90 days**, renewal period 30 days). Set "Publish certificate in Active Directory" off for high-volume short-lived certs to avoid AD bloat.
4. **Subject Name tab:** set "Supply in the request" (CI/CD will populate CN/SAN per service) rather than "Build from Active Directory information."
5. **Extensions tab → Application Policies:** confirm "Server Authentication" (and "Client Authentication" if mTLS).
6. **Issuance Requirements tab:** **uncheck** "CA certificate manager approval." This is the step most guides skip and it silently breaks automation — if approval is required, every renewal blocks on a human and you've defeated the purpose of this procedure.
7. **Request Handling tab:** "Allow private key to be exported" — leave **unchecked**. The private key for leaf certs is generated inside Key Vault, never on the CA; the CA only ever sees the public key in the CSR.
8. **Security tab:** grant **Read** and **Enroll** to the dedicated automation service account/group created in A.3 below. Leave Authenticated Users with at least Read so the CA's own computer account can still read the template (removing it entirely causes the CA service to fail template lookups).
9. On the CA: `certutil -SetCATemplates +CI-CD-ShortLived-WebServer` (or use the "Certificate Templates to Issue" node in the CA MMC snap-in) to publish it.

### A.2 Create a dedicated template for code signing certificates

1. Duplicate the built-in **Code Signing** template, name it `CI-CD-CodeSigning`.
2. **General tab:** validity ≤ 12 months (guidelines §"Recommended rollover intervals": rotate cert + key at least every 12 months; this also keeps you under the public CA/Browser Forum 460-day cap if you ever migrate this template's logic to a publicly-trusted CA later).
3. **Extensions → Application Policies:** "Code Signing" only — remove unrelated EKUs.
4. **Request Handling:** "Allow private key to be exported" — **unchecked**. Same reasoning as A.1: the key is generated and stays inside Key Vault Premium as an HSM-protected key; the CA never possesses it.
5. **Issuance Requirements:** uncheck CA manager approval, same as A.1, for the same automation reason. If your policy requires human sign-off for code-signing specifically, gate that approval in the CI/CD pipeline (e.g., a manual approval stage before the signing job runs) rather than at the CA — keeps the CA itself fully automatable.
6. **Security tab:** Enroll permission limited to the dedicated code-signing automation account — keep this narrower than the TLS template's principal list, since a compromised code-signing credential is a worse outcome than a compromised TLS cert.
7. Publish: `certutil -SetCATemplates +CI-CD-CodeSigning`.

### A.3 Service account / automation identity on the CA side

1. Create a dedicated AD service account (e.g., `svc-cicd-pki`) or, if your orchestration runs as a gMSA, prefer that — no password to rotate.
2. Grant it **Read + Enroll** on both templates above (A.1, A.2 Security tabs) — nothing broader. Do not grant CA Administrator or Certificate Manager rights; this account only needs to submit requests.
3. If the orchestration host is domain-joined, prefer running the submission step under the host's machine identity / gMSA rather than a static password account — one fewer secret to manage.

---

## Part B — Azure Key Vault Setup

### B.1 Create the vaults

Use two vaults (or two resource configurations) matching the table at the top — don't put code-signing keys in the same tier as routine internal TLS leaf certs.

```bash
# Internal TLS leaf certs — Standard tier is sufficient (guidelines §7.1)
az keyvault create \
  --name kv-cicd-tls-leaf \
  --resource-group rg-pki \
  --location eastus \
  --sku standard \
  --enable-rbac-authorization true \
  --enable-soft-delete true \
  --retention-days 90 \
  --enable-purge-protection true

# Code signing — Premium tier required for HSM-protected keys (guidelines §7.1/§7.3)
az keyvault create \
  --name kv-cicd-codesign \
  --resource-group rg-pki \
  --location eastus \
  --sku premium \
  --enable-rbac-authorization true \
  --enable-soft-delete true \
  --retention-days 90 \
  --enable-purge-protection true
```

Soft delete and purge protection are **mandatory**, not optional, per guidelines §7.2 — without purge protection, an accidental or malicious delete permanently destroys key material needed during an overlap period.

### B.2 RBAC role assignments (not legacy access policies)

Per guidelines §7.2, use Azure RBAC. Minimum viable role split:

| Identity | Role | Scope |
|---|---|---|
| Pipeline identity that requests/renews leaf certs | `Key Vault Certificates Officer` | `kv-cicd-tls-leaf` |
| Application/runtime identity that reads the deployed cert | `Key Vault Secrets User` | `kv-cicd-tls-leaf` |
| Pipeline identity that requests/renews code-signing cert | `Key Vault Certificates Officer` | `kv-cicd-codesign` |
| Signing-stage identity (runs AzureSignTool) | `Key Vault Crypto User` | `kv-cicd-codesign` (key-only — does not need Certificates Officer) |
| PKI/security admins | `Key Vault Administrator` | both vaults |

```bash
az role assignment create \
  --role "Key Vault Certificates Officer" \
  --assignee <pipeline-identity-object-id> \
  --scope /subscriptions/<sub>/resourceGroups/rg-pki/providers/Microsoft.KeyVault/vaults/kv-cicd-tls-leaf
```

### B.3 Authenticate pipelines without long-lived secrets

Per guidelines §7.2: use **Managed Identity** (Azure-hosted agents) or **OIDC workload identity federation** (GitHub Actions / Azure DevOps) — never a service principal client secret, which just creates a second rotation problem on top of the certificate one.

**Azure DevOps (workload identity federation service connection):**
- Project Settings → Service connections → New → Azure Resource Manager → **Workload identity federation (automatic)**. This creates a federated credential on an app registration with no secret to store.

**GitHub Actions:** create a user-assigned managed identity, add a federated credential trusting your repo/branch:

```bash
az identity create -g rg-pki -n id-gha-cicd-pki

az identity federated-credential create \
  --name gha-main-branch \
  --identity-name id-gha-cicd-pki \
  --resource-group rg-pki \
  --issuer "https://token.actions.githubusercontent.com" \
  --subject "repo:myorg/myrepo:ref:refs/heads/main" \
  --audiences "api://AzureADTokenExchange"
```

Then assign that identity the RBAC roles from B.2.

### B.4 Network restriction

Per guidelines §7.2, scope access rather than leaving the vault open to all Azure services:

```bash
az keyvault network-rule add --name kv-cicd-tls-leaf --subnet <agent-subnet-resource-id>
az keyvault update --name kv-cicd-tls-leaf --default-action Deny
```

For hybrid scenarios (orchestration host needs to reach both the on-prem CA *and* the vault), use Private Endpoint / Private Link into the same VNet the orchestration host or hybrid runbook worker sits in.

### B.5 Certificate contacts and diagnostic logging

```bash
az keyvault certificate contact add \
  --vault-name kv-cicd-tls-leaf \
  --email pki-team@yourorg.com

az monitor diagnostic-settings create \
  --name kv-diagnostics \
  --resource /subscriptions/<sub>/resourceGroups/rg-pki/providers/Microsoft.KeyVault/vaults/kv-cicd-tls-leaf \
  --workspace <log-analytics-workspace-id> \
  --logs '[{"category":"AuditEvent","enabled":true}]'
```

Set the contact to an owned distribution list (guidelines §7.3), not an individual — it's the failure path if the orchestration job stops running.

---

## Part C — Automated Leaf Certificate Rollover

### C.1 The core script: request, submit to Enterprise CA, merge

This script implements the loop from the top of this document. It uses `Add-AzKeyVaultCertificate` with issuer `Unknown` to get a CSR, `certreq` to submit that CSR non-interactively to the Enterprise CA against the dedicated template, then merges the result back.

```powershell
# Request-LeafCertificate.ps1
# Run on an orchestration host that has both Az PowerShell context and
# network/RPC line-of-sight to the issuing CA.

param(
    [Parameter(Mandatory)] [string]$VaultName,
    [Parameter(Mandatory)] [string]$CertificateName,
    [Parameter(Mandatory)] [string]$Fqdn,
    [Parameter(Mandatory)] [string]$CaConfigString,     # e.g. "ca01.corp.contoso.com\Contoso-Issuing-CA"
    [Parameter(Mandatory)] [string]$CaTemplateName,      # e.g. "CI-CD-ShortLived-WebServer"
    [int]$ValidityDays = 90
)

$ErrorActionPreference = 'Stop'

# 1. Define the certificate policy: issuer "Unknown" tells Key Vault to
#    generate the key pair + CSR and wait for an external CA response.
$policy = New-AzKeyVaultCertificatePolicy `
    -SecretContentType 'application/x-pkcs12' `
    -SubjectName "CN=$Fqdn" `
    -DnsName $Fqdn `
    -IssuerName 'Unknown' `
    -ValidityInMonths ([math]::Ceiling($ValidityDays / 30)) `
    -KeyType 'RSA' -KeySize 2048 -KeyUsage 'DigitalSignature','KeyEncipherment' `
    -ReuseKeyOnRenewal:$false   # fresh key every rollover, not just a new cert over an old key

# 2. Create/refresh the CSR. If the certificate already exists, this starts
#    a new pending version without disturbing the currently-deployed one.
$csrOp = Add-AzKeyVaultCertificate -VaultName $VaultName -Name $CertificateName -CertificatePolicy $policy
$csrPem = "-----BEGIN CERTIFICATE REQUEST-----`n$($csrOp.CertificateSigningRequest)`n-----END CERTIFICATE REQUEST-----"
$csrPath = "$env:TEMP\$CertificateName.csr"
Set-Content -Path $csrPath -Value $csrPem -Encoding ascii

# 3. Submit the CSR to the Windows Enterprise CA non-interactively, pinned
#    to the dedicated short-lived template. The template's "no manager
#    approval" setting (A.1 step 6) is what makes this a single command
#    rather than a ticket queue.
$cerPath = "$env:TEMP\$CertificateName.cer"
certreq -submit -config $CaConfigString -attrib "CertificateTemplate:$CaTemplateName" $csrPath $cerPath

if ($LASTEXITCODE -ne 0) {
    throw "certreq submission failed for $CertificateName (exit $LASTEXITCODE) — check CA Issuance Requirements / template permissions"
}

# 4. Merge the signed certificate (and chain, if returned as a .p7b) back
#    into the pending Key Vault certificate object. This completes the
#    object and creates a new current version.
Import-AzKeyVaultCertificate -VaultName $VaultName -Name $CertificateName -FilePath $cerPath

Write-Host "Certificate '$CertificateName' renewed in vault '$VaultName', new version is now current."
```

Notes:
- `certreq -submit -config <CA>\<Name> -attrib "CertificateTemplate:<Template>"` is the standard non-interactive submission path against a named Enterprise CA and template (this requires the calling identity to have Enroll rights on that template, per A.3).
- If `certreq` returns a pending status instead of immediate issuance, your template's Issuance Requirements still has manager approval enabled somewhere — fix it at the CA, not in the script.
- `Import-AzKeyVaultCertificate` performs the merge step Key Vault expects to complete an "Unknown"-issuer pending certificate; it must be run against the *same* CSR/pending object, which is why this is one continuous job rather than two decoupled steps.

### C.2 Orchestrating the renewal schedule

You have two reasonable choices; pick based on where your CI/CD platform's agents already have network access.

**Option 1 — Scheduled CI/CD pipeline (simplest if a self-hosted agent already reaches the CA):**

```yaml
# Azure DevOps — azure-pipelines-leaf-renewal.yml
trigger: none
schedules:
  - cron: "0 3 * * *"        # daily check; script below only acts inside the renewal window
    branches: { include: [main] }
    always: true

pool: { name: 'OnPremPKIAgents' }   # self-hosted pool with CA network access

steps:
  - task: AzureCLI@2
    displayName: 'Check expiry and renew if inside window'
    inputs:
      azureSubscription: 'kv-cicd-tls-leaf-connection'   # workload identity federation
      scriptType: pscore
      scriptLocation: inlineScript
      inlineScript: |
        $cert = Get-AzKeyVaultCertificate -VaultName "kv-cicd-tls-leaf" -Name "svc-web-frontend"
        $daysLeft = ($cert.Certificate.NotAfter - (Get-Date)).Days
        if ($daysLeft -le 45) {   # guidelines §5.2: trigger with >=30-45 days remaining
          ./Request-LeafCertificate.ps1 -VaultName "kv-cicd-tls-leaf" `
            -CertificateName "svc-web-frontend" -Fqdn "frontend.corp.contoso.com" `
            -CaConfigString "ca01.corp.contoso.com\Contoso-Issuing-CA" `
            -CaTemplateName "CI-CD-ShortLived-WebServer" -ValidityDays 90
        } else {
          Write-Host "No renewal needed: $daysLeft days remaining."
        }
```

**Option 2 — Event Grid + Azure Automation (matches guidelines §7.3 for non-Key-Vault-partnered CAs):** Key Vault emits a `Microsoft.KeyVault.CertificateNearExpiry` event at 80% of lifetime by default. Route it through Event Grid to an Azure Automation Runbook (running on a **Hybrid Runbook Worker** with network access to the CA) that calls the same script:

```bash
az eventgrid event-subscription create \
  --name leaf-cert-renewal \
  --source-resource-id /subscriptions/<sub>/resourceGroups/rg-pki/providers/Microsoft.KeyVault/vaults/kv-cicd-tls-leaf \
  --endpoint-type webhook \
  --endpoint <automation-webhook-url> \
  --included-event-types Microsoft.KeyVault.CertificateNearExpiry
```

For short-lived certs the default 80%-elapsed trigger leaves a shrinking buffer — override the runbook logic to fire on an explicit days-before-expiry threshold (45 days for a 90-day internal cert) instead of relying on the percentage, consistent with guidelines §7.3.

### C.3 Consuming the renewed certificate

Reference the certificate **without a version** from app config/deployment manifests, so a renewal is picked up automatically (guidelines §7.3, mirrors the "no hard-coded intermediates" principle in §2.3/§4):

```
https://kv-cicd-tls-leaf.vault.azure.net/certificates/svc-web-frontend/
```

For services that read the cert as a file (e.g., an nginx ingress controller, not Key-Vault-aware), add a deployment-stage step that pulls the current secret value and writes it to the runtime's expected path/keystore as part of every deployment, not just at initial provisioning — this is what makes renewal "automatic" from the workload's point of view.

---

## Part D — Automated Code Signing Certificate Rollover

Same loop as Part C, with three differences driven by guidelines §5.1/§7.1: HSM-backed key, code-signing EKU, narrower access, and an actual signing step using the certificate.

### D.1 Request with an HSM-protected key in Key Vault Premium

```powershell
# Request-CodeSigningCertificate.ps1
param(
    [string]$VaultName = "kv-cicd-codesign",
    [string]$CertificateName = "release-codesign",
    [string]$Subject = "CN=Contoso Engineering, O=Contoso Inc, C=US",
    [string]$CaConfigString = "ca01.corp.contoso.com\Contoso-Issuing-CA",
    [string]$CaTemplateName = "CI-CD-CodeSigning",
    [int]$ValidityMonths = 12     # guidelines: rotate at least every 12 months
)

$ErrorActionPreference = 'Stop'

$policy = New-AzKeyVaultCertificatePolicy `
    -SubjectName $Subject `
    -IssuerName 'Unknown' `
    -ValidityInMonths $ValidityMonths `
    -KeyType 'RSA-HSM' -KeySize 3072 `        # -HSM key type is what makes this HSM-protected, not just imported
    -KeyUsage 'DigitalSignature' `
    -Exportable:$false `
    -ReuseKeyOnRenewal:$false

$csrOp = Add-AzKeyVaultCertificate -VaultName $VaultName -Name $CertificateName -CertificatePolicy $policy
# ... CSR write, certreq submit, Import-AzKeyVaultCertificate: identical to C.1 steps 2-4,
#     pointed at $CaTemplateName = "CI-CD-CodeSigning"
```

Guidelines §7.1 flags this explicitly: *importing* a `.pfx` does **not** by itself satisfy an HSM-backed-key requirement. Using `-KeyType RSA-HSM` so the key is generated inside the vault's HSM partition (and confirming the resulting key version's `hsmPlatform` attribute) is what actually satisfies the FIPS 140-2 Level 2+ obligation for EV/OV code signing.

### D.2 Signing stage — AzureSignTool against the Key Vault certificate

The signing job never touches the private key directly; AzureSignTool calls Key Vault's `sign` operation remotely.

```yaml
# Azure DevOps signing stage
steps:
  - task: UseDotNet@2
    inputs: { packageType: 'sdk', version: '8.0.x' }

  - script: dotnet tool install --global AzureSignTool
    displayName: 'Install AzureSignTool'

  - task: AzureCLI@2
    displayName: 'Sign release binaries'
    inputs:
      azureSubscription: 'kv-cicd-codesign-connection'   # workload identity federation, role: Key Vault Crypto User
      scriptType: pscore
      scriptLocation: inlineScript
      inlineScript: |
        $token = (az account get-access-token --resource https://vault.azure.net --query accessToken -o tsv)
        Get-ChildItem -Path "$(Build.ArtifactStagingDirectory)" -Include *.exe,*.dll -Recurse | ForEach-Object {
          azuresigntool sign `
            -kvu "https://kv-cicd-codesign.vault.azure.net" `
            -kvc "release-codesign" `
            -kva $token `
            -tr "http://timestamp.digicert.com" -td sha256 -fd sha256 `
            $_.FullName
        }
```

On a self-hosted Azure DevOps/GitHub Actions agent you can also use `-azure-key-vault-managed-identity` instead of pulling a token manually, if the agent VM itself carries the managed identity. Note `AzureSignTool` does not currently support Office VBA/macro Subject Interface Packages — for that file type, use it to sign the binary then a separate Office-aware tool, or revisit that requirement separately.

### D.3 Renewal cadence and key handling

- Run the same scheduled-pipeline or Event-Grid pattern as C.2, but on a far less frequent cadence (annual, well ahead of the 12-month target) — code-signing renewal should still be deliberate, not silent, given the blast radius of a compromised signing identity. Consider routing the renewal trigger to a manual-approval gate before the new cert goes live, even though issuance itself is automated.
- `-ReuseKeyOnRenewal:$false` is intentional: each rotation gets a fresh HSM key, not just a new certificate wrapping the old key — this is what "rotate certificate **and key**" in the guidelines table actually requires.
- Restrict the `Key Vault Crypto User` role (needed only for the `sign` operation) to the signing job's identity, separately from `Key Vault Certificates Officer` (needed only for the renewal job's identity) — these should usually be different pipeline identities.

---

## Part E — Intermediate (Subordinate) CA Rollover

This maps guidelines §6 onto a Windows Enterprise Subordinate CA specifically. Unlike leaf certs, the subordinate CA's own private key **stays on the CA** (ideally backed by an HSM via a Key Storage Provider on the CA server itself, or a dedicated `Managed HSM` if your CA software supports an external KSP) — it is not pushed into Key Vault as a "certificate" object, consistent with the guidelines §7.1 mapping table (root/intermediate signing keys → Managed HSM, single-tenant, not the multi-tenant Key Vault used for leaf/code-signing certs).

### E.1 Preparation (guidelines §6.1)

1. On a new or existing subordinate CA server, generate a new CA key pair and CSR:
   ```
   certreq -new CAPolicy.inf NewSubCA.req
   ```
   `CAPolicy.inf` should mirror your current subordinate CA's policy OIDs/CDP-AIA URL conventions so downstream chain-building logic doesn't have to change.
2. Submit `NewSubCA.req` to the offline root CA (out-of-band — root CAs should not be network-reachable) and obtain the signed subordinate CA certificate.
3. Install it: `certutil -installCert NewSubCA.cer`, then start the CA service against the new identity (or stand up a parallel CA server if you're doing a full server replacement rather than a same-server cert renewal).
4. **Do not point CI/CD issuance at the new intermediate yet** — this step only makes the new CA cert exist and be installable.

### E.2 Update trust bundles before switching issuance (guidelines §6.1 step 2, §2.3/§4)

1. Publish the new subordinate CA certificate to AD and to your AIA/CDP distribution points alongside the existing one (do not remove the old one yet):
   ```
   certutil -dspublish -f NewSubCA.cer SubCA
   certutil -addstore -f Root RootCA.cer        # only if the root also changed
   certutil -addstore -f CA NewSubCA.cer
   ```
2. Update the trust bundle artifact that CI/CD agents and runtime environments consume (per the central-source-of-truth principle in guidelines §3.2) to include **both** the old and new intermediate certs. Roll this out as a controlled change (PR + pipeline), not a manual copy to individual machines.
3. Push the updated bundle to: CI/CD build agents, container base images / runtime trust stores, any custom verification tooling — the same list as guidelines §3.2.
4. **Verify** before proceeding: from a representative CI/CD agent and a representative runtime environment, validate a test chain that includes the new intermediate. Don't skip this — it's the step that prevents an outage during the actual cutover.

### E.3 Switch issuance to the new intermediate (guidelines §6.2)

For a Windows Enterprise CA, "switching the issuance profile" is what happens automatically once the CA service is running under the new CA certificate — every certificate it signs from that point forward chains to the new intermediate, with **no changes required** in the templates from Parts A.1/A.2 or in the CI/CD scripts from Parts C/D. This is the practical benefit of the versionless-reference and dynamic-trust-bundle design: leaf-cert rollover code never has to know which intermediate is currently active.

During the overlap period:
- Certificates issued before the cutover continue chaining to the **old** intermediate and remain valid until they expire/are rotated under the normal 90-day (TLS) / 12-month (code signing) cadence from Parts C/D.
- Certificates issued after cutover chain to the **new** intermediate.
- Both validate correctly everywhere because trust bundles updated in E.2 contain both.

### E.4 Decommission the old intermediate (guidelines §6.3)

Only after the overlap period has fully elapsed (every leaf cert that could chain to the old intermediate has expired or rotated — for 90-day TLS certs this is bounded; confirm via the certificate inventory in Part F):

1. Remove the old subordinate CA's issuing capability (decommission the old CA service instance, or revoke/retire its CA certificate if running parallel servers).
2. Remove the old intermediate from trust bundles and roll out the update through the same controlled-change process as E.2.
3. Run an inventory query (Part F) confirming nothing still chains to the old intermediate before calling this complete, then update your PKI documentation to mark it decommissioned.

### E.5 Emergency / unplanned rollover readiness (guidelines §6.4)

Specific to a Windows Enterprise CA:

1. **Pre-stage the emergency path**: keep a tested runbook for publishing a new CDP/AIA entry and pushing an emergency trust-bundle update within hours — `certutil -dspublish` and the bundle-distribution pipeline from E.2 should already be exercised in non-prod (Part G) before you ever need them live.
2. **Pre-identify blast radius**: maintain the certificate inventory from Part F tagged by issuing intermediate, so if your root program distrusts an intermediate (or a key compromise forces emergency revocation) you can immediately enumerate every affected leaf cert and service.
3. **Decoupling**: if a given service is critical enough to need issuance continuity even during a CA outage, evaluate a secondary subordinate CA (or a path to a public CA for that one service) rather than relying on disaster recovery of the single subordinate CA.
4. **CRL/OCSP responsiveness**: an emergency revocation of the old intermediate is only effective if relying parties actually see the updated CRL/OCSP status promptly — confirm CRL publication interval and OCSP responder health as part of the same readiness check, not just trust-bundle distribution.

---

## Part F — Monitoring, Alerting, and Inventory

Implements guidelines §2.5 / §7.4 / §8.

### F.1 Key Vault side

```bash
# Renewal failures and unexpected access in Log Analytics (KQL)
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.KEYVAULT"
| where OperationName in ("CertificateNearExpiry", "CertificateNewVersionCreated")
| where ResultType != "Success"
```

Alert on this query, plus a separate alert on `CertificateNearExpiry` events that are **not** followed by a `CertificateNewVersionCreated` event within your renewal-window threshold — that gap means the orchestration in Part C/D silently failed.

### F.2 CA side

```powershell
# Inventory of issued certs by template and remaining validity, for the
# blast-radius mapping referenced in E.5
Get-CertificationAuthority | Get-IssuedRequest -Filter "CertificateTemplate -eq 'CI-CD-ShortLived-WebServer'" |
  Select-Object CommonName, NotAfter, "Certificate Template"
```

Watch CA health directly too: CRL publication success, AIA/CDP reachability, and certificate-template ACL drift (guidelines §8 — "issuance from non-approved intermediates or roots" applies here as "issuance under a template that shouldn't still be active").

### F.3 Tie it together

Maintain a single inventory (even a simple table: cert name, vault, type, issuing intermediate, expiry, owning service) covering both Key Vault and CA-side data, refreshed by a scheduled job that queries both sources from Parts F.1/F.2. This is what makes the emergency-rollover blast-radius step in E.5 actually executable in hours rather than days.

---

## Part G — Documentation and Testing Checklist (guidelines §9)

Before relying on this in production, and at least annually thereafter:

- [ ] Document: which vault holds which cert class, who has which RBAC role, and the CA template names/config strings each pipeline uses.
- [ ] Test in non-production: full leaf-cert renewal loop (C.1) end-to-end, including a forced near-expiry trigger.
- [ ] Test in non-production: code-signing renewal + a real AzureSignTool signing run against the test cert.
- [ ] Test in non-production: full intermediate rollover simulation (E.1–E.4) on a staging PKI hierarchy.
- [ ] Test: emergency rollover path (E.5) at least annually, timed end-to-end.
- [ ] Test: Key Vault backup/restore (`az keyvault backup`) actually recovers a working certificate object.
- [ ] Review and sign-off from security, PKI/operations, and the application teams consuming these certs.

---

## Quick Reference

| Task | Command/Tool |
|---|---|
| Create CSR in Key Vault (issuer Unknown) | `Add-AzKeyVaultCertificate` with `New-AzKeyVaultCertificatePolicy -IssuerName Unknown` |
| Submit CSR to Enterprise CA, no manual approval | `certreq -submit -config "<CA>\<Name>" -attrib "CertificateTemplate:<Template>"` |
| Merge signed cert back into Key Vault | `Import-AzKeyVaultCertificate` |
| HSM-backed key for code signing | `-KeyType RSA-HSM` in the certificate policy, Key Vault **Premium** tier |
| Sign a binary against a Key Vault cert | `azuresigntool sign -kvu <vault-uri> -kvc <cert-name> ...` |
| Publish new subordinate CA cert to AD | `certutil -dspublish -f <cert> SubCA` |
| Pipeline auth to Key Vault, no stored secret | Workload identity federation (GitHub Actions) / Workload identity federation service connection (Azure DevOps) |

---

*This procedure operationalizes guidelines §§2–9. Re-validate the cadences in Parts C/D annually against any change to internal policy or, if a leaf-cert template here is ever pointed at a publicly-trusted CA instead of the internal Enterprise CA, against the current CA/Browser Forum schedule referenced in the guidelines document's "Regulatory context" section.*
