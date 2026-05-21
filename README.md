# Two-Tier PKI on Windows Server

> MSc Cyber Security coursework: design and deploy a two-tier Public Key Infrastructure (PKI) on Windows Server, terminating a TLS-protected web service for a fictional organisation called **SELDOM**.
---

## What this project demonstrates

A working two-tier PKI built end-to-end in a virtualised Windows Server environment. The proof that everything works is a domain-joined browser loading `https://www.seldom.test` with a valid HTTPS padlock and the full certificate chain `seldom-ROOT-CA-CA → seldom-DOMAINCONTROLER-CA-1 → www.seldom.test`.

![Final HTTPS proof: www.seldom.test loaded with full certificate chain](screenshots/04-web-server/04-final-https-proof.png)

*The end-to-end deliverable: a green-padlocked TLS page whose certificate chain validates back to the offline Root CA built in the lab.*

---

## Architecture

Two-tier Microsoft PKI hierarchy, with the catastrophic-impact key (the Root) kept off the network except during ceremony events, and the high-volume key (the Issuing CA) integrated with Active Directory for day-to-day issuance.

```
                       ┌──────────────────────────────┐
                       │   Offline Root CA            │
                       │   (Standalone, powered-off)  │
                       │   RSA 4096 / SHA-256         │
                       │   seldom-ROOT-CA-CA          │
                       └──────────────┬───────────────┘
                                      │  signs Subordinate cert
                                      ▼
                       ┌──────────────────────────────┐
                       │   Subordinate / Issuing CA   │
                       │   (Enterprise CA on the DC)  │
                       │   DomainController.seldom.local
                       │   seldom-DOMAINCONTROLER-CA-1│
                       └──────────────┬───────────────┘
                                      │  issues TLS server cert
                                      ▼
                       ┌──────────────────────────────┐
                       │   IIS Web Server             │
                       │   https://www.seldom.test    │
                       │   Bound to port 443          │
                       └──────────────────────────────┘
```

All three roles ran as virtual machines on **AWS EC2** Windows Server, with the Root CA isolated and powered off after the bootstrap key ceremony.

---

## What was built — at a glance

| Stage | What was done | Evidence |
|-------|---------------|----------|
| 1. Root CA | Installed AD CS in standalone mode, configured RSA 4096 / SHA-256, published a Certificate Revocation List (CRL), exported the self-signed root certificate | [`screenshots/01-root-ca/`](screenshots/01-root-ca/) |
| 2. Domain Controller | Promoted a fresh Windows Server to a new `seldom.local` forest with integrated DNS | [`screenshots/02-domain-controller/`](screenshots/02-domain-controller/) |
| 3. Issuing CA | Installed AD CS in Enterprise Subordinate mode, generated a CSR, had the Root CA sign it, installed the chain, started the CA | [`screenshots/03-issuing-ca/`](screenshots/03-issuing-ca/) |
| 4. Web server | Used IIS Manager → Server Certificates → Create Domain Certificate to enrol an SSL/TLS cert, bound it to HTTPS on port 443, served the proof page | [`screenshots/04-web-server/`](screenshots/04-web-server/) |

---

## Screenshots — key moments

### Cryptographic choices on the Root CA

![Root CA cryptography settings: RSA 4096-bit, SHA-256](screenshots/01-root-ca/02-cryptography-rsa4096-sha256.png)

*RSA 4096-bit key with SHA-256 hash chosen for the Root CA — comfortably exceeds the CA/Browser Forum Baseline Requirements minimum and gives the root a long usable lifetime.*

### Subordinate CA wired into the Root's chain of trust

![AD CS Configuration confirmation page for the Subordinate Enterprise CA](screenshots/03-issuing-ca/02-subordinate-ca-confirmation.png)

*The Subordinate CA's Distinguished Name, parent CA reference, and crypto parameters at the moment of installation — the chain begins here.*

### TLS certificate bound to IIS

![IIS Add Site Binding dialog with the issued certificate selected](screenshots/04-web-server/02-iis-https-binding.png)

*The issued certificate selected for the HTTPS:443 binding on the IIS Default Web Site, completing the TLS termination configuration for SELDOM's web service.*

> For the full image gallery, browse [`/screenshots`](screenshots/).

---

## Tech stack and tools used

| Category | What was used |
|----------|---------------|
| Hosting | AWS EC2 (Windows Server VMs) |
| Operating system | Microsoft Windows Server (Server Manager UI consistent with the 2016 / 2019 timeframe) |
| Directory / DNS | Active Directory Domain Services (`seldom.local` forest, Windows Server 2016 functional level), integrated DNS |
| PKI | Active Directory Certificate Services (AD CS) — Certification Authority + Certification Authority Web Enrollment |
| Web | Internet Information Services (IIS) — Default Web Site with HTTPS binding |
| Tooling | Server Manager, AD CS Configuration Wizard, `certsrv.msc`, Certificates MMC snap-in, IIS Manager, Certificate Import / Export Wizards |
| Cryptography | RSA 4096-bit, SHA-256, X.509, PKCS#10 (`.req`), PKCS#7 (`.p7b`) |

> Everything listed above is something that was actually used in this lab. No frameworks, languages, or tools have been added "for show".

---

## Repository structure

```
two-tier-pki-windows-server/
├── README.md                           ← you are here
├── LICENSE                             ← see "License" section below
├── .gitignore
└── screenshots/
    ├── 01-root-ca/                     ← offline Root CA installation evidence
    ├── 02-domain-controller/           ← AD DS forest promotion evidence
    ├── 03-issuing-ca/                  ← Subordinate / Issuing CA evidence
    └── 04-web-server/                  ← IIS TLS binding and end-to-end browser proof
```

---

## How to read this repository

Start with the screenshots in the order numbered above; they tell the build story end-to-end in roughly 12 frames, from the empty Server Manager to the final green-padlocked HTTPS page.

---

## How to reproduce the lab

The lab is reproducible on any virtualisation platform that supports Windows Server (AWS EC2 was used; Hyper-V, VMware Workstation/ESXi, or VirtualBox would work equivalently). At a high level:

1. Provision three Windows Server VMs: `root-CA` (workgroup), `DomainController` (will host the Subordinate CA), and `webserver` (IIS).
2. Promote `DomainController` to a new forest (`seldom.local`); install DNS.
3. On `root-CA`: install AD CS as **Standalone Root CA**, RSA 4096, SHA-256; publish a CRL; export the root certificate.
4. On `DomainController`: install AD CS as **Enterprise Subordinate CA**; submit the CSR to the Root CA; install the signed chain.
5. Import the Root CA certificate into the **Trusted Root Certification Authorities** store of every relying-party machine (handled by Group Policy in production).
6. On the web server: use IIS Manager → **Server Certificates → Create Domain Certificate** to enrol a TLS cert from the Issuing CA, then add an **HTTPS** binding on port 443 for `www.seldom.test`.

Step-by-step screenshots for every wizard pane are in `/screenshots`.

---

## Scope of contribution

This was a **group coursework submission** under Northumbria University's Network Security module. This repository is deliberately scoped to the **implementation** portion of that submission, the practical PKI build, the certificate issuance, and the IIS TLS termination, which the author personally delivered. Other sections of the assessed group report (threat modelling, ethical and legal analysis) were authored by other group members and are intentionally not included here.

---

## Acknowledgements

- Northumbria University, MSc Cyber Security programme.
- The wider Microsoft / Active Directory Certificate Services documentation and community knowledge base that informed the implementation choices.
