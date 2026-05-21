# SSL PKI Threat Model — SELDOM

> Extracted from §3 of the group coursework report for LD7007 Networks Security. This document is the GitHub-renderable companion to the assessed `.docx`.

**Methodology:** STRIDE for identification (Spoofing · Tampering · Repudiation · Information disclosure · Denial of service · Elevation of privilege), DREAD for ranking (Damage · Reproducibility · Exploitability · Affected users · Discoverability, each 1–10, mean is the risk score).

**Scope:** identity-spoofing threats (T1–T5) and Certification-Authority threats (T6–T12), as required by the brief.

---

## 1. Threat Taxonomy

Threats T1–T5 concern impersonation of identities certified by the SELDOM PKI; threats T6–T12 target the CA infrastructure itself.

| ID | Threat | STRIDE |
|----|--------|--------|
| T1 | Phishing using a look-alike domain certificate (e.g. `www.seldom-test.com` obtained from a public CA) targeting SELDOM staff. | S |
| T2 | Subject-name forgery exploiting weak request validation on the Issuing CA, allowing an internal attacker to obtain a certificate for someone else's identity. | S, E |
| T3 | Subject Alternative Name (SAN) abuse via the `EDITF_ATTRIBUTESUBJECTALTNAME2` flag or vulnerable templates, enabling cross-identity impersonation. | S, E |
| T4 | Theft of an end-entity private key from the web server (file-system access, backup compromise, memory disclosure) used to impersonate `www.seldom.test`. | S, I |
| T5 | Insider with Enrol rights obtaining a certificate in the name of another principal (user or service). | S, R |
| T6 | Compromise of the offline Root CA private key — catastrophic; undermines every certificate ever issued by the PKI. | T, I, E |
| T7 | Compromise of the online Issuing CA private key — enables wholesale rogue issuance until the CA is revoked. | T, I, E |
| T8 | Rogue / unauthorised certificate issuance via a compromised CA administrator account or supply-chain attack against the CA software (cf. DigiNotar 2011). | S, T, E |
| T9 | Denial of service against CRL Distribution Points or OCSP responders, rendering revocation status unverifiable. | D |
| T10 | Cryptographic weakening through deprecated algorithm or short-key templates (SHA-1, RSA-1024) leaving certificates open to forgery. | T, I |
| T11 | Phishing or credential stuffing against CA administrator accounts; lateral movement onto the Issuing CA. | S, E |
| T12 | Physical or virtualisation-layer tampering with the Root CA host (hypervisor escape, disk image theft) bypassing offline assumptions. | T, I, E |

---

## 2. DREAD Ranking

Each threat scored 1–10 across the five DREAD dimensions; mean rounded to one decimal place. Scores reflect SELDOM's context (small-to-medium internal PKI, Issuing CA highly available because it runs on a domain controller).

| Rank | Threat | D | R | E | A | Di | Mean |
|------|--------|---|---|---|---|----|------|
| 1 | T1 — Look-alike domain phishing | 6 | 9 | 9 | 8 | 8 | **8.0** |
| 2 | T11 — Admin credential compromise | 9 | 6 | 7 | 9 | 5 | **7.2** |
| 3 | T3 — SAN abuse / vulnerable template | 9 | 6 | 7 | 8 | 5 | **7.0** |
| 4 | T8 — Rogue issuance via admin / supply chain | 9 | 5 | 6 | 9 | 5 | **6.8** |
| 5 | T7 — Issuing CA key compromise | 9 | 4 | 5 | 10 | 5 | **6.6** |
| 6 | T9 — DoS on CRL/OCSP | 5 | 6 | 6 | 7 | 7 | **6.2** |
| 7 | T2 — Subject-name forgery (weak validation) | 8 | 5 | 6 | 7 | 4 | **6.0** |
| 7 | T12 — Root CA host tampering | 10 | 3 | 4 | 10 | 3 | **6.0** |
| 9 | T6 — Root CA key compromise | 10 | 2 | 3 | 10 | 4 | **5.8** |
| 10 | T4 — End-entity key theft | 7 | 4 | 5 | 6 | 5 | **5.4** |
| 11 | T10 — Weak crypto template | 7 | 3 | 4 | 7 | 5 | **5.2** |
| 12 | T5 — Insider mis-enrolment | 6 | 4 | 5 | 5 | 4 | **4.8** |

**Interpretation.** The top of the ranking is dominated by external-facing, low-friction attacks (T1 phishing) and high-privilege internal attacks (T11, T3, T8). The catastrophic but low-likelihood Root CA key compromise (T6) ranks below several "everyday" risks — consistent with current industry practice: keep the Root extremely difficult to reach (low R and E), and spend more day-to-day defensive effort on the threats that are easier to execute.

---

## 3. Mitigation Plan

Controls are layered (defence in depth) and where possible draw on industry standards — CA/Browser Forum Baseline Requirements, NIST SP 800-57 / SP 800-152, RFC 5280, and ETSI EN 319 411-1 — so they can be evidenced in an audit.

| Threat | Primary mitigations |
|--------|---------------------|
| T1 | Continuous Certificate Transparency (CT) monitoring of look-alike domains; user security-awareness training; DNS protective filtering; HSTS preloading and CAA records pinning issuance to SELDOM's internal CA. |
| T2 | Enterprise-only enrolment (no Web Enrollment for the Web Server template); template ACLs restricted to dedicated security groups; manual approval for templates marked "CA Certificate Manager Approval"; auditing of every Issue / Deny event. |
| T3 | Verify `EDITF_ATTRIBUTESUBJECTALTNAME2` is unset on the CA; duplicate templates so SAN is supplied in the request rather than added as an attribute; disable vulnerable v1 templates; quarterly review against the SpecterOps ESC1–ESC8 checklist. |
| T4 | Store TLS private keys in a hardware-backed store (TPM, Windows Platform Crypto Provider) or HSM; restrict NTFS permissions on the private-key folder; encrypt server backups; enable BitLocker on web-server volumes. |
| T5 | Separation of duties between enrolment agents and CA administrators; just-in-time PAM access; periodic re-attestation of who holds Enrol rights; SIEM correlation between Active Directory authentication events and certificate issuance. |
| T6 | Root CA powered off and stored in a locked virtual-machine snapshot on encrypted media; ceremony procedures requiring multi-person attendance; HSM with FIPS 140-2 Level 3 backing in production; periodic offline CRL refresh with an established cadence. |
| T7 | Issuing CA on a hardened domain controller with restricted RDP; Tier 0 administration model; private key in a TPM/HSM; continuous monitoring of certificate database growth; periodic re-key with longer-lived Root re-signing. |
| T8 | Two-person rule for sensitive template changes; protected-users / privileged access workstations for CA administration; supply-chain controls on AD CS patching; immutable audit logging shipped off-host; tabletop incident-response exercises modelled on DigiNotar. |
| T9 | Publish CRLs to a redundant HTTP distribution point fronted by a CDN-style cache; deploy OCSP stapling on the web server so the client does not depend on direct OCSP availability; monitor revocation endpoint uptime as a first-class SLI. |
| T10 | Codify a minimum cryptographic baseline (RSA ≥ 3072 or ECDSA P-256, SHA-256+); annual algorithm review; enforce via certificate template settings and CA policy module. |
| T11 | MFA for all CA administrators; conditional-access policies; just-in-time elevation via PIM; password-length requirements; phishing-resistant authentication (FIDO2) where feasible. |
| T12 | Encrypted root volumes; segregated hypervisor cluster with restricted administrator access; integrity monitoring of Root CA snapshots; physical access controls if hosted on premises; cloud-provider attestation evidence if hosted on EC2. |

---

## 4. Privacy linkage (in brief)

Two significant PKI risks are discussed at length in the report:

- **Risk A — Compromise of a trusted CA (T6 / T7 / T8).** Models the DigiNotar (2011) class of incident. Confidentiality and integrity are both destroyed; the relying party sees no warning. Under UK GDPR, this would trigger Article 33 notification obligations.
- **Risk B — End-entity key theft and lack of forward secrecy (T4).** If RSA key-exchange cipher suites are still permitted, recorded TLS sessions become retrospectively decryptable once the key is stolen. Mitigated by enforcing PFS-only cipher suites (ECDHE) and TLS 1.3.

Full discussion in `docs/SELDOM_PKI_Group_Report.docx`, §3.6.

---

## 5. Validation of the threat-identification approach

- STRIDE entries cross-checked against the MITRE ATT&CK Enterprise matrix (e.g., T11 ↔ T1110 Brute Force / T1078 Valid Accounts; T8 ↔ T1649 Steal or Forge Authentication Certificates).
- DREAD scores sanity-checked by reverse-engineering a published incident (DigiNotar) and confirming the model places T7/T8 in the top three for that scenario.
- Mitigation table cross-checked against CA/Browser Forum Baseline Requirements clauses.

Full discussion in `docs/SELDOM_PKI_Group_Report.docx`, §3.5.
