# cyberark_group_management

[![CI](https://github.com/TobyAnscombe/cyberark-api-group-management/actions/workflows/ci.yml/badge.svg)](https://github.com/TobyAnscombe/cyberark-api-group-management/actions/workflows/ci.yml)

Ansible role that ensures CyberArk Identity roles exist, creating any that are absent. Intended as a **one-time bootstrap** for roles required by [`tobyanscombe.cyberark_safe_management`](https://github.com/TobyAnscombe/cyberark-api-safe-management).

## When to run

Run this once when setting up a new Privilege Cloud tenant, before running any safe management playbooks. The two default roles it creates are prerequisites for `cyberark_safe_management`'s standard member and break glass configuration.

The three Privilege Cloud built-in groups (`Privilege Cloud Administrators`, `Privilege Cloud Auditors`, `Privilege Cloud Safe Managers`) are provisioned automatically by CyberArk — this role does not create or modify them.

## How it works

For each group in `cyberark_groups`, calls `POST /Roles/StoreRole` against the CyberArk Identity API. If the role already exists the API returns `ErrorCode 1409`, which is treated as a no-op. Any other failure is fatal.

All API calls run `delegate_to: localhost` / `run_once: true`. The role is idempotent — running it against a tenant where the roles already exist reports `ok` with no changes.

## Requirements

- Ansible 2.9+
- A CyberArk Privilege Cloud tenant
- `cyberark_token` set on the play — produced by [`tobyanscombe.cyberark_api_authentication`](https://github.com/TobyAnscombe/cyberark-api-management) or supplied by any other means
- The service account associated with `cyberark_token` must have **Role Management** rights in CyberArk Identity (Admin Portal → Roles → Administrative Rights)

## Install

Add both roles to your project's `requirements.yml`:

```yaml
# requirements.yml
roles:
  - name: cyberark_api_authentication
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
| `cyberark_identity_tenant` | yes | `""` | CyberArk Identity tenant ID — forms `<tenant>.id.cyberark.cloud` |
| `cyberark_validate_certs` | no | `true` | Validate TLS certificates |
| `cyberark_groups` | no | two default roles (see below) | List of Identity roles to ensure exist |

Each item in `cyberark_groups`:

| Key | Required | Description |
|---|---|---|
| `name` | yes | Identity role name |
| `description` | no | Free-text description set on create; not updated if the role already exists |

### Default roles

```yaml
cyberark_groups:
  - name: "Privilege Cloud Access Approvers"
    description: "Approvers for Privilege Cloud dual-control access requests"
  - name: "CyberArk Break Glass Access"
    description: "Emergency break glass access — full safe permissions, bypasses dual-control"
```

These match the names expected by `cyberark_safe_management`. If you change the role names in that role, update `cyberark_groups` here to match.

## Usage

```yaml
- name: Bootstrap CyberArk Privilege Cloud roles
  hosts: localhost
  gather_facts: false

  vars:
    cyberark_identity_tenant: "YOUR_TENANT_ID"
    # cyberark_client_id and cyberark_client_secret from vault

  roles:
    - cyberark_api_authentication
    - cyberark_group_management
```

See [`examples/bootstrap.yml`](examples/bootstrap.yml) for the full example.

## License

MIT
