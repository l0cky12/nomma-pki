# NOMMA Offline Root CA

This directory contains the offline Root CA Ansible workflow for the NOMMA PKI hierarchy.

The configured Root CA host is:

```text
liam@192.168.122.58
```

Run this only against the dedicated offline or isolated Root CA host. The playbook creates:

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

- `/home/liam/nomma-root-ca/pki/root-ca/private/nomma-root-ca.key.pem`
- `/home/liam/nomma-root-ca/pki/root-ca/private/nomma-issuing-ca.key.pem`
- `inventory/group_vars/vault.yml`
- vault password files
- CA database files under `/home/liam/nomma-root-ca/pki/root-ca/db/`

## First run

Create and encrypt the vault file on the control node:

```bash
cd root-ca
cp inventory/group_vars/vault.example.yml inventory/group_vars/vault.yml
ansible-vault encrypt inventory/group_vars/vault.yml
```

Confirm SSH works:

```bash
ssh liam@192.168.122.58 'hostname && openssl version'
```

Install any Ansible requirements:

```bash
ansible-galaxy collection install -r requirements.yml
```

Run the playbook:

```bash
ansible-playbook playbooks/site.yml --vault-password-file ~/.ansible/vault-password.txt
```

## Output locations

Private CA state stays on the remote Root CA host under:

```text
/home/liam/nomma-root-ca/pki/root-ca/
```

Public transfer files are staged on the remote host under:

```text
/home/liam/nomma-root-ca/artifacts/
```

The playbook also fetches those public files back to this repo on the control node under:

```text
root-ca/artifacts/
```

Copy only the public artifacts into the online Issuing CA workflow.
