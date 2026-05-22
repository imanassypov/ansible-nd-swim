# Ansible Nexus Dashboard Software Image Management

This project automates fabric switch upgrades in Nexus Dashboard (ND) using the unified REST API introduced in ND 4.1. It orchestrates the full SWIM (Software Image Management) lifecycle: configuring remote storage, uploading NOS images, creating upgrade groups, staging, executing, and post-upgrade reporting.

## 🚀 Quick Start

### 1. Prerequisites

Ensure you have [uv](https://docs.astral.sh/uv/) installed:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### 2. Setup Environment

This command will create a virtual environment, install `ansible-core`, and all required Python libraries defined in `pyproject.toml`.

```bash
uv sync
```

---

## 📂 Project Structure

```text
.
├── ansible.cfg                  # Ansible configuration (inventory, roles, collections paths)
├── pb_nd_fabric_upgrade.yml     # Main playbook — runs the nd_swim_upgrade role
├── pyproject.toml               # Python dependencies (ansible-core, ansible-lint)
├── uv.lock                      # Pinned dependency versions
├── .ansible-lint                # Linting configuration
├── .markdownlint-cli2.yaml      # Markdown formatting rules
├── group_vars/
│   └── all.yml                  # Global Ansible variables
├── host_vars/
│   └── nd-data/
│       ├── connection.yml       # ND host, username, and API key
│       └── swim_configuration.yml  # SWIM-specific variables (fabric, image, upgrade group)
├── inventories/
│   ├── production/
│   │   └── hosts.yml
│   └── staging/
├── roles/
│   └── nd_swim_upgrade/         # SWIM upgrade role (see roles/nd_swim_upgrade/README.md)
│       ├── defaults/main.yml    # Overridable defaults
│       ├── tasks/               # Ordered upgrade workflow tasks (01–06)
│       └── vars/main.yml        # Computed/internal variables
├── data/
│   └── site.yml                 # Master playbook entry point (stub)
└── docs/
    └── development/
        └── pre-commit.md        # Pre-commit hook setup guide
```

---

## ⚙️ Configuration

### Connection (`host_vars/nd-data/connection.yml`)

```yaml
nd_host: "nd.example.com"
nd_username: "admin"
nd_api_key: "{{ vault_nd_api_key }}"   # Store in Ansible Vault
```

### SWIM Configuration (`host_vars/nd-data/swim_configuration.yml`)

```yaml
nd_fabric_name: "MyFabric"

nd_remote_storage_name: "ansible-scp"
nd_remote_storage_hostname: "scp.example.com"
nd_remote_storage_path: "/home/sftpuser/nos_images/"
nd_remote_storage_username: "sftpuser"
nd_remote_storage_password: "{{ vault_scp_password }}"  # Store in Ansible Vault
nd_remote_storage_type: "scp"

nd_swim_image_name: "nxos64-cs.10.6.2.F.bin"
nd_swim_image_path: "nxos64-cs.10.6.2.F.bin"

nd_swim_upgrade_group_name: "ansible_upgrade_group"
nd_swim_upgrade_group_switches:
  - "SWITCH_SERIAL_NUMBER"
```

See [roles/nd_swim_upgrade/README.md](roles/nd_swim_upgrade/README.md) for the full variable reference.

> **Security:** `nd_api_key` and `nd_remote_storage_password` are sensitive credentials. Store them in [Ansible Vault](https://docs.ansible.com/ansible/latest/vault_guide/index.html) and never commit plaintext values to version control.

---

## 🛠 Usage

All Ansible commands should be run using `uv run` to ensure they use the isolated project environment.

### Run the Upgrade Playbook

```bash
# Against production inventory
uv run ansible-playbook -i inventories/production pb_nd_fabric_upgrade.yml

# Against staging inventory
uv run ansible-playbook -i inventories/staging pb_nd_fabric_upgrade.yml
```

### Run Selective Stages with Tags

```bash
# Run only setup (remote storage, image upload, upgrade group creation, staging)
uv run ansible-playbook -i inventories/production pb_nd_fabric_upgrade.yml --tags setup

# Run only the upgrade execution
uv run ansible-playbook -i inventories/production pb_nd_fabric_upgrade.yml --tags start_upgrade
```

| Tag | Tasks Executed |
|---|---|
| `setup` | Remote storage + image upload + upgrade group + staging |
| `remote_storage` | Configure remote storage only |
| `image_upload` | Upload NOS image only |
| `upgrade_group` | Create/update upgrade group only |
| `stage_upgrade` | Stage upgrade and optional pre-upgrade report |
| `start_upgrade` | Execute the upgrade |
| `post_upgrade_analysis` | Post-upgrade report generation |

### Linting

```bash
# Lint Ansible code
uv run ansible-lint

# Lint Markdown files
uv run markdownlint-cli2 "**/*.md"
```

### Encrypting Secrets

```bash
uv run ansible-vault encrypt host_vars/nd-data/connection.yml
```

---

## 📋 Requirements

| Dependency | Version |
| :--- | :--- |
| Python | `>=3.12,<3.14` |
| ansible-core | `>=2.15.0` |
| uv | Latest |

---

## ⚖️ License

This project is licensed under the **MIT License**. See the [LICENSE](LICENSE) file for details.

---

## 🤝 Contributing

1. Set up pre-commit hooks (see [docs/development/pre-commit.md](docs/development/pre-commit.md)).
2. Create a feature branch (`git checkout -b feature/new-automation`).
3. Ensure linting passes (`uv run ansible-lint`).
4. Open a Pull Request for review.

### Why use `uv` with Ansible?

1. **Speed**: `uv` installs `ansible-core` and its dependencies up to 100x faster than `pip`.
2. **Reproducibility**: `uv.lock` ensures every team member uses the exact same dependency versions.
3. **No Activation Needed**: `uv run` automatically uses the project environment without requiring `source .venv/bin/activate`.
