# NOMMA Offline Root CA

This directory contains the offline Root CA Ansible workflow for the NOMMA PKI hierarchy.

Run this only on an offline or air-gapped Root CA host. The playbook creates:

- encrypted Root CA private key
- self-signed Root CA certificate
- encrypted Issuing CA private key
- Issuing CA CSR
- signed Issuing CA certificate
- public CA chain
- public artifacts for transfer to the online Issuing CA

## Important security rules

Never commit or transfer private CA material.

Safe to transfer to the online Issuing CA:

- `artifacts/nomma-root-ca.cert.pem`
- `artifacts/nomma-issuing-ca.cert.pem`
- `artifacts/nomma-ca-chain.cert.pem`

Never transfer or commit:

- `pki/root-ca/private/nomma-root-ca.key.pem`
- `pki/root-ca/private/nomma-issuing-ca.key.pem`
- `group_vars/vault.yml`
- vault password files
- CA database files under `pki/root-ca/db/`

## First run

Create and encrypt the vault file:

```bash
cd root-ca
cp group_vars/vault.example.yml group_vars/vault.yml
ansible-vault encrypt group_vars/vault.yml
```

Run the playbook on the offline Root CA host:

```bash
ansible-galaxy collection install -r requirements.yml
ansible-playbook playbooks/site.yml --vault-password-file ~/.ansible/vault-password.txt
```

## Output locations

Private CA state is created under:

```text
root-ca/pki/root-ca/
```

Public transfer files are exported under:

```text
root-ca/artifacts/
```

Copy only the public artifacts into the online Issuing CA workflow.
