nd_swim_upgrade
===============

Automates the Cisco Nexus Dashboard (ND) Software Image Management (SWIM) upgrade workflow using the ND REST API. The role orchestrates the full upgrade lifecycle: configuring remote storage, uploading a NOS image, creating or updating an upgrade group, staging the upgrade with optional pre-upgrade reporting, executing the upgrade, and optionally generating a post-upgrade comparison report.

The role is idempotent — each step checks current state before making changes.

Upgrade Workflow
----------------

```text
01_configure_remote_storage  →  Ensure the SCP/SFTP remote storage entry exists on ND
02_upload_image              →  Upload the NOS image from remote storage to ND (skipped if already present)
03_create_upgrade_group      →  Create or update the upgrade group with target switches
04_stage_upgrade             →  Stage the upgrade; optionally retrieve and display a pre-upgrade report
05_start_upgrade             →  Execute the upgrade and poll until completion
06_post_upgrade_analysis     →  Generate and display a post-upgrade report (only when reporting is enabled)
```

Requirements
------------

- `ansible-core >= 2.15.0`
- No additional Ansible collections are required. All tasks use `ansible.builtin.uri` and `ansible.builtin.set_fact`.
- A Nexus Dashboard instance reachable from the Ansible control node.
- A remote storage server (SCP) accessible by Nexus Dashboard that holds the target NOS image.
- An ND API key with sufficient privileges to manage SWIM operations.

Role Variables
--------------

### Required Variables

These variables have no defaults and must be supplied by the caller.

| Variable | Description |
|---|---|
| `nd_host` | Hostname or IP address of the Nexus Dashboard instance |
| `nd_username` | ND username used for API authentication |
| `nd_api_key` | ND API key for authentication — **store in Ansible Vault** |
| `nd_fabric_name` | Target fabric name on Nexus Dashboard |
| `nd_remote_storage_name` | Name to assign (or reference) for the remote storage entry |
| `nd_remote_storage_hostname` | Hostname or IP of the SCP server |
| `nd_remote_storage_username` | Username to authenticate to the remote storage server |
| `nd_remote_storage_password` | Password for remote storage authentication — **store in Ansible Vault** |
| `nd_remote_storage_path` | Base path on the remote storage server |
| `nd_remote_storage_type` | Remote storage protocol type (e.g., `SCP`) |
| `nd_swim_image_name` | Filename of the NOS image (used for existence check) |
| `nd_swim_image_path` | Full path to the NOS image on the remote storage server |
| `nd_swim_upgrade_group_name` | Name of the upgrade group to create or update |
| `nd_swim_upgrade_group_switches` | List of switch serial numbers or identifiers to include in the upgrade group |

### Defaults (`defaults/main.yml`)

These variables have sensible defaults and can be overridden.

| Variable | Default | Description |
|---|---|---|
| `nd_swim_upgrade_debug` | `false` | Enable verbose debug output (individual test result loops) |
| `nd_swim_upgrade_nd_validate_certs` | `{{ nd_validate_certs \| default(false) }}` | Validate TLS certificates when calling the ND API |
| `nd_swim_upgrade_nd_api_timeout` | `{{ nd_api_timeout \| default(60) }}` | HTTP request timeout (seconds) for API calls |
| `nd_swim_upgrade_api_retry_count` | `{{ api_retry_count \| default(3) }}` | Number of retries for API requests |
| `nd_swim_upgrade_api_retry_delay` | `{{ api_retry_delay \| default(5) }}` | Delay (seconds) between API retries |
| `nd_swim_upgrade_group_analysis` | `noAnalysis` | Pre-upgrade analysis type: `snapshot`, `noAnalysis`, `fullAnalysis`, `usePreExistingAnalysis` |
| `nd_swim_upgrade_group_contingency` | `pause` | Contingency action on failure: `continue`, `pause` |
| `nd_swim_upgrade_group_execution` | `parallel` | Switch upgrade execution mode: `parallel`, `sequential` |
| `nd_swim_upgrade_group_uninstall_package` | `true` | Uninstall previous NOS package during upgrade |
| `nd_swim_upgrade_group_disruptive` | `true` | Allow disruptive upgrade |
| `nd_swim_upgrade_group_maintenance` | `false` | Place switches in maintenance mode during upgrade |
| `nd_swim_upgrade_group_report_selection` | `noReport` | Report type: `noReport`, `basic`, `advanced`. Setting to anything other than `noReport` enables pre/post upgrade reports |

### Optional Variables (no defaults)

These variables are consumed with Ansible's `default()` filter and are only required in specific scenarios.

| Variable | Description |
|---|---|
| `nd_remote_storage_description` | Human-readable description for the remote storage entry (default: `Ansible Configured Remote Storage`) |
| `nd_remote_storage_ignore_host_key_validation` | Skip SSH host key validation for the remote storage server (default: `false`) |
| `nd_swim_upgrade_group_nos_image_name` | Override the NOS image name used by the upgrade group (falls back to `nd_swim_image_name`) |
| `nd_swim_upgrade_group_epld_image_name` | EPLD image filename to include in the upgrade (default: `''`) |
| `nd_swim_upgrade_group_install_package_names` | List of additional package names to install (default: `[]`) |
| `nd_swim_upgrade_group_installation_order_devices` | Ordered list of devices for sequential upgrades (default: `[]`) |
| `nd_swim_upgrade_group_reports` | Reports field passed to the upgrade group payload (default: `noReport`) |
| `nd_swim_upgrade_group_report_checks` | List of report check objects for the upgrade group (default: `[]`) |

### Internal Variables (`vars/main.yml`)

These are computed from `nd_host` and should not be overridden.

| Variable | Value | Description |
|---|---|---|
| `nd_swim_upgrade_infra_base_url` | `https://{{ nd_host }}/api/v1/infra` | Base URL for ND infrastructure API endpoints |
| `nd_swim_upgrade_manage_base_url` | `https://{{ nd_host }}/api/v1/manage` | Base URL for ND management API endpoints |
| `nd_swim_upgrade_polling_max_attempts` | `60` | Maximum polling attempts for long-running operations |
| `nd_swim_upgrade_nd_polling_timeout` | `3600` | Total polling timeout in seconds |

Tags
----

Tasks are tagged to allow selective execution.

| Tag | Tasks |
|---|---|
| `setup` | All setup tasks (remote storage, image upload, upgrade group creation, staging) |
| `remote_storage` | Configure remote storage |
| `image_upload` | Upload NOS image |
| `upgrade_group` | Create/update upgrade group |
| `stage_upgrade` | Stage upgrade and pre-upgrade report |
| `start_upgrade` | Execute the upgrade |
| `post_upgrade_analysis` | Post-upgrade report generation |

Dependencies
------------

None. This role has no dependencies on other Ansible roles.

Example Playbook
----------------

```yaml
- name: Upgrade Fabric Switches via Nexus Dashboard SWIM
  hosts: localhost
  gather_facts: false

  vars:
    nd_host: "nd.example.com"
    nd_username: "admin"
    nd_api_key: "{{ vault_nd_api_key }}"
    nd_fabric_name: "MyFabric"

    nd_remote_storage_name: "scp-server-01"
    nd_remote_storage_hostname: "scp.example.com"
    nd_remote_storage_username: "scp_user"
    nd_remote_storage_password: "{{ vault_scp_password }}"
    nd_remote_storage_path: "/images"
    nd_remote_storage_type: "SCP"

    nd_swim_image_name: "nxos.10.4.2.F.bin"
    nd_swim_image_path: "/images/nxos.10.4.2.F.bin"

    nd_swim_upgrade_group_name: "upgrade-group-01"
    nd_swim_upgrade_group_switches:
      - "FD12345678A"
      - "FD12345678B"

    # Optional overrides
    nd_swim_upgrade_group_report_selection: "basic"
    nd_swim_upgrade_group_execution: "sequential"
    nd_swim_upgrade_group_maintenance: true

  roles:
    - role: nd_swim_upgrade
```

Run only the setup phase (remote storage + image upload + group creation + staging):

```bash
ansible-playbook site.yml --tags setup
```

Run the full upgrade including post-upgrade analysis:

```bash
ansible-playbook site.yml
```

Security Notes
--------------

- `nd_api_key` and `nd_remote_storage_password` are sensitive credentials. Store them in [Ansible Vault](https://docs.ansible.com/ansible/latest/vault_guide/index.html) and never commit plaintext secrets to version control.
- `nd_swim_upgrade_nd_validate_certs` defaults to `false` for lab convenience. Set to `true` in production environments.

License
-------

Cisco Sample Code License, Version 1.1

Author Information
------------------

Russell Johnston — rujohns2@cisco.com
