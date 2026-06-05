# cyberark_group_management

[![CI](https://github.com/TobyAnscombe/cyberark-api-group-management/actions/workflows/ci.yml/badge.svg)](https://github.com/TobyAnscombe/cyberark-api-group-management/actions/workflows/ci.yml)

Ansible role that ensures CyberArk Privilege Cloud vault groups exist, creating any that are absent. Intended as a **one-time bootstrap** for groups required by [`tobyanscombe.cyberark_safe_management`](https://github.com/TobyAnscombe/cyberark-api-safe-management).

## When to run

Run this once when setting up a new Privilege Cloud tenant, before running any safe management playbooks. The two default groups it creates are prerequisites for `cyberark_safe_management`'s standard member and break glass configuration.

The three Privilege Cloud built-in groups (`Privilege Cloud Administrators`, `Privilege Cloud Auditors`, `Privilege Cloud Safe Managers`) are provisioned automatically by CyberArk — this role does not create or modify them.

## How it works

For each group in `cyberark_groups`, calls `POST /Roles/StoreRole` against the CyberArk Identity API. If the role already exists the API returns `ErrorCode 1409`, which is treated as a no-op. Any other failure is fatal.

All API calls run `delegate_to: localhost` / `run_once: true`. The role is idempotent — running it against a tenant where the groups already exist reports `ok` with no changes.

## Requirements

- Ansible 2.9+
- A CyberArk Privilege Cloud tenant
- `cyberark_token` set on the play — produced by [`tobyanscombe.cyberark_auth`](https://github.com/TobyAnscombe/cyberark-api-management) or supplied by any other means

## Install

Add both roles to your project's `requirements.yml`:

```yaml
# requirements.yml
roles:
  - name: cyberark_auth
    src: https://github.com/TobyAnscombe/cyberark-api-management
    version: main
  - name: cyberark_group_management
    src: https://github.com/TobyAnscombe/cyberark-api-group-management
    version: main
```

```bash
ansible-galaxy install -r requirements.yml
```

## Role variables

| Variable | Required | Default | Description |
|---|---|---|---|
| `cyberark_subdomain` | yes | `""` | Privilege Cloud subdomain — forms `<subdomain>.privilegecloud.cyberark.cloud` |
| `cyberark_validate_certs` | no | `true` | Validate TLS certificates |
| `cyberark_groups` | no | two default groups (see below) | List of groups to ensure exist |

Each item in `cyberark_groups`:

| Key | Required | Description |
|---|---|---|
| `name` | yes | Vault group name |
| `description` | no | Free-text description set on create; not updated if the group already exists |

### Default groups

```yaml
cyberark_groups:
  - name: "Privilege Cloud Access Approvers"
    description: "Approvers for Privilege Cloud dual-control access requests"
  - name: "CyberArk Break Glass Access"
    description: "Emergency break glass access — full safe permissions, bypasses dual-control"
```

These match the names expected by `cyberark_safe_management`. If you change the group names in that role, update `cyberark_groups` here to match.

## Usage

```yaml
- name: Bootstrap CyberArk Privilege Cloud groups
  hosts: localhost
  gather_facts: false

  vars:
    cyberark_identity_tenant: "YOUR_TENANT_ID"
    cyberark_subdomain: "YOUR_SUBDOMAIN"
    # cyberark_client_id and cyberark_client_secret from vault

  roles:
    - cyberark_auth
    - cyberark_group_management
```

See [`examples/bootstrap.yml`](examples/bootstrap.yml) for the full example.

## License

MIT
