# BIG-IP Certificate Renewal Automation Template (ACME v2 + DigiCert)

This template automates certificate renewal for BIG-IP by requesting a certificate from DigiCert's ACME v2 endpoint and deploying it to an existing client SSL profile.

## Files

- `renew_bigip_certificate.yml` - ACME order + BIG-IP deployment playbook.

## Prerequisites

1. Ansible core (2.14+ recommended)
2. Ansible collections:
   ```bash
   ansible-galaxy collection install f5networks.f5_modules community.crypto
   ```
3. BIG-IP API access (management interface).
4. DigiCert ACME v2 account and account private key file.
5. DNS/API process to satisfy ACME challenge validation (for `dns-01`, etc.).

## Required variables

- `bigip_host`
- `bigip_username`
- `bigip_password`
- `digicert_acme_directory` (DigiCert ACME v2 directory URL)
- `acme_account_key_path` (path to ACME account private key)
- `certificate_common_name`
- `target_cert_name`
- `target_clientssl_profile`

## Optional variables

- `certificate_sans` (list, default `[]`)
- `acme_challenge_type` (default `dns-01`)
- `acme_order_mode` (`issue` or `finalize`, default `issue`)
- `prior_acme_order_data_path` (default: `<acme_work_dir>/<cn>-order-data.json`)
- `acme_validate_directory_certs` (default `true`)
- `acme_work_dir` (default `./artifacts/acme`)
- `domain_private_key_path` (auto-generated default under `acme_work_dir`)
- `csr_path` (auto-generated default under `acme_work_dir`)
- `issued_cert_path`, `issued_chain_path`, `issued_fullchain_path` (defaults under `acme_work_dir`)
- `target_key_name` (default: `target_cert_name`)
- `target_chain_name` (default: `<target_cert_name>-chain`)
- `target_partition` (default: `Common`)
- `bigip_port` (default: `443`)
- `bigip_validate_certs` (default: `false`)
- `set_sni_default` (default: `true`)

## Example vars file

```yaml
# BIG-IP connection
bigip_host: 10.0.0.245
bigip_username: admin
bigip_password: "{{ vault_bigip_password }}"

# DigiCert ACME v2
# Replace with your DigiCert-provided ACME directory endpoint.
digicert_acme_directory: https://acme.digicert.com/v2/acme/directory
acme_account_key_path: ./secrets/digicert-acme-account.key
acme_challenge_type: dns-01
acme_order_mode: issue

# Certificate request
certificate_common_name: app.example.com
certificate_sans:
  - www.app.example.com
  - api.app.example.com

# BIG-IP target objects
target_cert_name: app-example-com-2026
target_key_name: app-example-com-2026
target_chain_name: app-example-com-2026-chain
target_clientssl_profile: app-example-com-clientssl
target_partition: Common

# Optional behavior
bigip_validate_certs: false
set_sni_default: true
```

## Run

### 1) Issue mode (create ACME order)

```bash
ansible-playbook templates/certificate-renewal/renew_bigip_certificate.yml -e @vars.yml -e acme_order_mode=issue
```

If DigiCert returns challenge requirements, the playbook stops and writes order/challenge details to `prior_acme_order_data_path`.

### 2) Complete challenge externally

- For `dns-01`, create the required TXT records using your DNS automation/process.
- Wait for propagation and validation readiness.

### 3) Finalize mode (retrieve cert and deploy to BIG-IP)

```bash
ansible-playbook templates/certificate-renewal/renew_bigip_certificate.yml -e @vars.yml -e acme_order_mode=finalize
```

## Workflow summary

1. Validate required inputs.
2. Prepare local key/CSR artifacts.
3. Create/continue ACME order with DigiCert.
4. Retrieve certificate artifacts (cert + chain + fullchain).
5. Upload certificate, key, and chain into BIG-IP.
6. Update client SSL profile cert-key-chain and save config.
7. Print BIG-IP certificate expiration for verification.

## Security notes

- Store BIG-IP and ACME secrets in Ansible Vault or a secure secret store.
- Restrict filesystem permissions on account and private key files.
- Test in non-production before broad rollout.
