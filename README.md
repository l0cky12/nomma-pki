# NOMMA PKI Infrastructure

**Organization:** NOMMA — New Orleans Military & Maritime Academy
**Location:** New Orleans, Louisiana, USA
**Repository:** `nomma-pki`

---

## Overview

This repository contains the complete Public Key Infrastructure (PKI) for NOMMA. It uses a two-tier CA hierarchy:

```
┌─────────────────────────────────────┐
│     Offline Root CA (root-ca/)      │
│   - Kept offline, powered off       │
│   - Signs Intermediate CA cert      │
│   - Signs CRLs only                 │
│   - Private key NEVER on network    │
└──────────────┬──────────────────────┘
               │ signs
               ▼
┌─────────────────────────────────────┐
│   Online Issuing CA (issuing-ca/)   │
│   - Always online, 24/7             │
│   - Issues server/client certs      │
│   - Generates CRLs                  │
│   - Optional OCSP responder         │
│   - Private key on server (secured) │
└─────────────────────────────────────┘
```

---

## Directory Structure

```
nomma-pki/
├── README.md                    ← This file
├── .gitignore
│
├── root-ca/                     ← Offline Root CA Ansible project
│   ├── ansible.cfg
│   ├── inventory/
│   ├── group_vars/
│   ├── playbooks/
│   ├── roles/
│   ├── templates/
│   └── artifacts/               ← Root CA certs (exported TO issuing-ca)
│       ├── .gitkeep
│       ├── nomma-root-ca.cert.pem
│       ├── nomma-issuing-ca.cert.pem
│       └── nomma-ca-chain.cert.pem
│
├── issuing-ca/                  ← Online Issuing CA Ansible project
│   ├── ansible.cfg
│   ├── inventory/
│   ├── group_vars/
│   ├── playbooks/
│   ├── roles/
│   ├── templates/
│   └── README.md
│
└── .gitignore
```

---

## CA Hierarchy Security Model

### Offline Root CA (`root-ca/`)

- **Purpose:** Establish the chain of trust. Signs exactly one Intermediate CA certificate.
- **Security:** The Root CA private key **never** touches a network. The Root CA system is air-gapped.
- **Operations:** Only powered on for:
  - Initial Intermediate CA signing
  - Intermediate CA certificate renewal (every 10 years)
  - Root CA CRL generation (if Intermediate CA is compromised)
  - Root CA key ceremony / backup

### Online Issuing CA (`issuing-ca/`)

- **Purpose:** Day-to-day certificate issuance, renewal, and revocation.
- **Security:** Online 24/7. Private key is on disk but protected with restrictive permissions (`0400`).
- **Operations:** Issues all server, client, device, and service certificates for `nomma.tech` and `nomma.lan`.

---

## File Transfer Matrix

### Files that MUST be transferred FROM Root CA TO Issuing CA

| File | Source | Destination | Purpose |
|------|--------|-------------|---------|
| `nomma-root-ca.cert.pem` | `root-ca/artifacts/` | `issuing-ca/` (via playbook) | Root CA certificate — needed to build trust chain |
| `nomma-issuing-ca.cert.pem` | `root-ca/artifacts/` | `issuing-ca/` (via playbook) | Signed Intermediate CA certificate |
| `nomma-ca-chain.cert.pem` | `root-ca/artifacts/` | `issuing-ca/` (via playbook) | Full chain: Intermediate + Root |

### Files that MUST NEVER leave the Root CA

| File | Reason |
|------|--------|
| `nomma-root-ca.key.pem` | Root CA private key — compromise destroys the entire PKI |
| Any Root CA private key backup | Same as above |
| Root CA password/passphrase | Protects the private key |

### Files that CAN be distributed freely

| File | Reason |
|------|--------|
| `nomma-root-ca.cert.pem` | Public certificate — needed for trust chain |
| `nomma-ca-chain.cert.pem` | Public certificate chain |
| `nomma-issuing-ca.cert.pem` | Public Intermediate CA certificate |
| CRL files | Public — needed for revocation checking |

---

## Initial Setup Workflow

### Step 1: Deploy the Offline Root CA

Run the `root-ca/` Ansible project on an air-gapped Debian system to create the Root CA.

### Step 2: Sign the Intermediate CA Certificate

On the offline Root CA system:
1. Generate the Intermediate CA key and CSR
2. Sign the Intermediate CA certificate with the Root CA
3. Build the certificate chain

### Step 3: Export Artifacts

Copy ONLY these files to `root-ca/artifacts/` in this repository:
- `nomma-root-ca.cert.pem`
- `nomma-issuing-ca.cert.pem`
- `nomma-ca-chain.cert.pem`

**DO NOT copy the Root CA private key.**

### Step 4: Deploy the Issuing CA

```bash
cd issuing-ca/

# Create vault password (one-time)
echo "your-strong-password" > ~/.ansible/vault-password.txt
chmod 600 ~/.ansible/vault-password.txt

# Create encrypted vault
cp group_vars/vault.example.yml group_vars/vault.yml
# Edit to set the real password hash
ansible-vault encrypt group_vars/vault.yml \
  --vault-password-file ~/.ansible/vault-password.txt

# Run the playbook
ansible-playbook -i inventory/hosts.yml playbooks/site.yml \
  --vault-password-file ~/.ansible/vault-password.txt
```

---

## Intermediate CA Certificate Renewal

When the Intermediate CA certificate approaches expiry (every 10 years):

1. Power on the offline Root CA
2. Generate a new Intermediate CA key/CSR
3. Sign with the Root CA
4. Export new certs to `root-ca/artifacts/`
5. Re-run the `issuing-ca/` playbook to import the new certs
6. Distribute the new chain to all relying parties

---

## Domains Covered

- `nomma.tech` — Public-facing services
- `nomma.lan` — Internal infrastructure

Example service certificates:
- `server.nomma.tech` / `server.nomma.lan`
- `pki.nomma.tech` / `pki.nomma.lan`
- `vpn.nomma.tech`
- `mail.nomma.tech`
- `ldap.nomma.lan`
- `radius.nomma.lan`

---

## Security Notes

- All private keys are stored with `0400` or `0600` permissions
- The Intermediate CA private key directory is mode `0700`
- SSH is hardened with key-only auth and Fail2Ban
- UFW denies all inbound except SSH, HTTP, HTTPS
- Secrets are encrypted with Ansible Vault
- `no_log: true` is used on all tasks that handle passwords

---

## Disaster Recovery

If the Issuing CA is compromised:
1. Revoke the Intermediate CA certificate from the Root CA
2. Generate a new Intermediate CA key and certificate
3. Re-deploy the Issuing CA
4. Re-issue all end-entity certificates

If the Root CA is compromised:
1. The entire PKI must be rebuilt from scratch
2. All certificates must be re-issued
3. This is why the Root CA stays offline