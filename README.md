# Ansible Nexus Dashboard Software Image Management

This project manages [Describe your Infrastructure, e.g., Cisco IOS-XE Core Switches] using Ansible. It follows a modular directory structure and uses `uv` for lightning-fast Python dependency management.

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

### 3. Install Ansible Dependencies

Download the required Cisco collections and community roles:

```bash
uv run ansible-galaxy install -r data/requirements.yml
```

---

## 📂 Project Structure

The repository is split into a Python root (for tooling) and an `data/` subdirectory (for logic).

```text
.
├── pyproject.toml          # Python dependencies (ansible-core, ansible-lint)
├── uv.lock                 # Pinned dependency versions
├── .ansible-lint           # Linting configuration
├── .markdownlint-cli2.yaml # Markdown formatting rules
├── data/                # Ansible Root
│   ├── ansible.cfg         # Project-specific Ansible config
│   ├── site.yml            # Master playbook
│   ├── inventories/        # Environment-specific data (prod/stage)
│   ├── playbooks/          # Workflow playbooks
│   └── roles/              # Reusable automation logic
└── scripts/                # Helper Python/Bash scripts
```

---

## 🛠 Usage

All Ansible commands should be run using `uv run` to ensure they use the isolated project environment.

### Run a Playbook

To run the master playbook against the **Staging** environment:

```bash
cd data
uv run ansible-playbook -i inventories/staging/hosts.yml site.yml
```

### Check for Syntax Errors (Linting)

We use `ansible-lint` to ensure code quality and `markdownlint-cli2` for documentation.

```bash
# Lint Ansible code
uv run ansible-lint data/

# Lint Markdown files
uv run markdownlint-cli2 "**/*.md"
```

### Encrypting Secrets

Use Ansible Vault to manage sensitive data like Cisco enable passwords:

```bash
uv run ansible-vault encrypt data/inventories/production/group_vars/all.yml
```

---

## 📋 Requirements

| Dependency | Version |
| :--- | :--- |
| Python | ^3.10 |
| Ansible-Core | ^2.16 |
| uv | Latest |

---

## ⚖️ License

This project is licensed under the **MIT License**. See the [LICENSE](LICENSE) file for details.

---

## 🤝 Contributing

1. Create a feature branch (`git checkout -b feature/new-automation`).
2. Ensure all tests pass (`uv run ansible-lint`).
3. Open a Pull Request for review.

### Why use `uv` with Ansible?

1. **Speed**: `uv` installs `ansible-core` and its dependencies (like `cryptography` and `jinja2`) up to 100x faster than `pip`.
2. **Reproducibility**: The `uv.lock` file ensures that every team member is using the exact same version of Ansible and its Python dependencies, preventing "it works on my machine" bugs.
3. **No Activation Needed**: By using `uv run`, you don't have to remember to `source .venv/bin/activate`. It automatically finds the right environment for you.
