# Ansible Nexus Dashboard Software Image Management (SWIM)

Automates NX-OS upgrades for fabric switches managed by Cisco Nexus Dashboard (ND) using the ND unified REST API (introduced in ND 4.1). The playbook orchestrates the complete SWIM lifecycle end-to-end:

| Step | Task File | What It Does |
|---|---|---|
| 1 | `01_configure_remote_storage` | Ensure an SCP/SFTP remote storage entry exists on ND |
| 2 | `02_upload_image` | Import the target NOS image from remote storage into ND (skipped if already present) |
| 3 | `03_create_upgrade_group` | Create or reconcile an upgrade group with target switch serial numbers |
| 4 | `04_stage_upgrade` | Download image to switch bootflash, verify integrity, run optional pre-upgrade report |
| 5 | `05_start_upgrade` | Trigger the NX-OS upgrade and poll until completion |
| 6 | `06_post_upgrade_analysis` | Fetch and display a pre-vs-post comparison report (when reporting is enabled) |

Each step is idempotent — re-running the playbook checks current state before making any change.

---

## Prerequisites

| Requirement | Version / Notes |
|---|---|
| Cisco Nexus Dashboard | 4.1 or later (REST API shape changed in 4.1) |
| Python | 3.12 – 3.13 |
| [uv](https://docs.astral.sh/uv/) | Latest — used to manage the Python environment |
| SCP/SFTP server | Must be reachable from ND and hold the target NOS image |
| ND user account | Must have SWIM admin privileges on the target fabric |

### Install uv

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

---

## Installation

```bash
# 1. Clone the repository
git clone https://github.com/rjohnston6/ansible-nd-swim.git
cd ansible-nd-swim

# 2. Create the Python virtual environment and install all dependencies
uv sync
```

`uv sync` reads `pyproject.toml` and `uv.lock` and installs an exact, reproducible set of packages (ansible-core, ansible-lint, etc.) into `.venv/`. No manual `pip install` or `source .venv/bin/activate` is needed — all commands are run via `uv run`.

---

## Directory Structure

```text
ansible-nd-swim/
├── ansible.cfg                      # Ansible config (inventory path, roles path, collections)
├── pb_nd_fabric_upgrade.yml         # Main playbook — invokes the nd_swim_upgrade role
├── pyproject.toml                   # Python dependencies
├── uv.lock                          # Pinned dependency versions
│
├── group_vars/
│   └── all.yml                      # Global variables (empty by default)
│
├── host_vars/
│   └── nd-data/
│       ├── connection.yml           # ND connection parameters (reads from env vars)
│       └── swim_configuration.yml  # Target fabric, image, and upgrade group settings
│
├── inventories/
│   └── production/
│       └── hosts.yml                # Inventory — nd-data host using local connection
│
└── roles/
    └── nd_swim_upgrade/
        ├── defaults/main.yml        # Overridable role defaults
        ├── vars/main.yml            # Internal computed variables (API base URLs, timeouts)
        └── tasks/
            ├── main.yml             # Upgrade workflow entry point (includes tasks 01–06)
            ├── 01_configure_remote_storage.yml
            ├── 02_upload_image.yml
            ├── 03_create_upgrade_group.yml
            ├── 04_stage_upgrade.yml
            ├── 05_start_upgrade.yml
            └── 06_post_upgrade_analysis.yml
```

---

## Configuration

### Step 1 — Set connection credentials (environment variables)

`host_vars/nd-data/connection.yml` reads all credentials from shell environment variables. Export these before running the playbook:

```bash
export ND_HOST="nd.example.com"          # Hostname or IP of Nexus Dashboard
export ND_USERNAME="automation"          # ND username with SWIM privileges
export ND_API_KEY="<your-api-key>"       # ND API key (see below)
```

**How to create an ND API key:**

1. Log in to Nexus Dashboard.
2. Click your username (top-right) → **User Preferences**.
3. Under **API Keys**, click **Generate API Key**.
4. Copy the key — it is only shown once.

> **Security:** Never hard-code credentials in YAML files or commit them to version control. Use environment variables, a secrets manager, or [Ansible Vault](https://docs.ansible.com/ansible/latest/vault_guide/index.html).

---

### Step 2 — Configure the upgrade target (`host_vars/nd-data/swim_configuration.yml`)

Edit this file to match your environment:

```yaml
# Target fabric on Nexus Dashboard
nd_fabric_name: "MyFabric"

# ── Remote Storage ──────────────────────────────────────────────────────────
# Name of the SCP/SFTP remote storage entry on ND.
# If this name already exists on ND, the creation step is skipped automatically.
nd_remote_storage_name: "ansible-scp"
nd_remote_storage_hostname: "scp.example.com"
nd_remote_storage_path: "/home/sftpuser/nos_images/"
nd_remote_storage_username: "sftpuser"
nd_remote_storage_password: ""          # Leave empty if the entry already exists on ND
nd_remote_storage_type: scp
nd_remote_storage_ignore_host_key_validation: false
nd_remote_storage_description: "Remote storage managed by Ansible"

# ── Target NOS Image ─────────────────────────────────────────────────────────
# Filename must match exactly as it appears on the remote storage server.
nd_swim_image_name: "nxos64-cs.10.6.3.F.bin"
nd_swim_image_path: "nxos64-cs.10.6.3.F.bin"

# ── Upgrade Group ────────────────────────────────────────────────────────────
# IMPORTANT: Upgrade group names are GLOBALLY unique across all fabrics on ND.
# Use a name that includes the fabric identifier to avoid cross-fabric collisions.
nd_swim_upgrade_group_name: "ansible_upgrade_group_myfabric"

# Upgrade behaviour (see variable reference below for allowed values)
nd_swim_upgrade_group_contingency: continue   # continue | pause
nd_swim_upgrade_group_execution: parallel     # parallel | sequential
nd_swim_upgrade_group_report_selection: basic # noReport | basic | advanced

# List of switch serial numbers to include in this upgrade group.
# Find serial numbers in ND: Manage → Fabric → <fabric> → Switches tab.
nd_swim_upgrade_group_switches:
  - "SWITCH_SERIAL_1"
  - "SWITCH_SERIAL_2"
```

**Where to find switch serial numbers in ND UI:**
`Manage → Fabric Software → <fabric> → NX-OS/IOS-XE tab → Switches column`

**Where to verify remote storage in ND UI:**
`Admin → System Settings → General → Remote Locations`

**Where to verify uploaded images in ND UI:**
`Manage → Fabric Software → Images tab`

---

### Variable Reference

#### Required variables (no defaults)

| Variable | Description |
|---|---|
| `nd_host` | Hostname or IP address of the Nexus Dashboard instance |
| `nd_username` | ND username for API authentication |
| `nd_api_key` | ND API key |
| `nd_fabric_name` | Target fabric name on Nexus Dashboard |
| `nd_remote_storage_name` | Name of the remote storage entry (created if absent) |
| `nd_remote_storage_hostname` | Hostname or IP of the SCP/SFTP server |
| `nd_remote_storage_username` | Username for remote storage authentication |
| `nd_remote_storage_password` | Password for remote storage (leave `""` if entry already exists) |
| `nd_remote_storage_path` | Base directory path on the remote storage server |
| `nd_remote_storage_type` | Protocol: `scp` or `sftp` |
| `nd_swim_image_name` | Filename of the target NOS image |
| `nd_swim_image_path` | Path/filename of the image on the remote storage server |
| `nd_swim_upgrade_group_name` | Name of the upgrade group (must be globally unique across all fabrics) |
| `nd_swim_upgrade_group_switches` | List of switch serial numbers to upgrade |

#### Optional variables with defaults (`defaults/main.yml`)

| Variable | Default | Allowed Values | Description |
|---|---|---|---|
| `nd_swim_upgrade_group_analysis` | `noAnalysis` | `noAnalysis`, `snapshot`, `fullAnalysis`, `usePreExistingAnalysis` | Pre-upgrade analysis mode. `noAnalysis` skips the pre-upgrade report entirely |
| `nd_swim_upgrade_group_contingency` | `pause` | `continue`, `pause` | What to do if a switch fails during upgrade |
| `nd_swim_upgrade_group_execution` | `parallel` | `parallel`, `sequential` | Upgrade all switches simultaneously or one at a time |
| `nd_swim_upgrade_group_report_selection` | `noReport` | `noReport`, `basic`, `advanced` | Pre/post-upgrade report detail level. Any value other than `noReport` enables reporting |
| `nd_swim_upgrade_group_disruptive` | `true` | `true`, `false` | Allow disruptive upgrade (reload required) |
| `nd_swim_upgrade_group_maintenance` | `false` | `true`, `false` | Place switches in maintenance mode before upgrade |
| `nd_swim_upgrade_group_uninstall_package` | `true` | `true`, `false` | Uninstall the previous NOS package during upgrade |
| `nd_swim_upgrade_debug` | `false` | `true`, `false` | Print individual pre/post report test results in playbook output |
| `nd_swim_upgrade_nd_validate_certs` | `false` | `true`, `false` | Validate ND TLS certificate (set `true` with trusted certs) |
| `nd_swim_upgrade_nd_api_timeout` | `60` | integer (seconds) | Per-request HTTP timeout for ND API calls |

---

## Running the Playbook

All commands use `uv run` to ensure the project's isolated Python environment is used.

```bash
# Export credentials first (required every new shell session)
export ND_HOST="nd.example.com"
export ND_USERNAME="automation"
export ND_API_KEY="<your-api-key>"

# Run the full upgrade workflow
uv run ansible-playbook pb_nd_fabric_upgrade.yml

# Run with verbose output (shows API responses)
uv run ansible-playbook pb_nd_fabric_upgrade.yml -v

# Enable per-test debug output in the pre/post reports
uv run ansible-playbook pb_nd_fabric_upgrade.yml -e nd_swim_upgrade_debug=true

# Run only a specific phase using tags
uv run ansible-playbook pb_nd_fabric_upgrade.yml --tags setup          # steps 1–4
uv run ansible-playbook pb_nd_fabric_upgrade.yml --tags stage_upgrade  # step 4 only
uv run ansible-playbook pb_nd_fabric_upgrade.yml --tags start_upgrade  # step 5 only
```

### Available Tags

| Tag | Steps Executed |
|---|---|
| `setup` | 01 + 02 + 03 + 04 (remote storage, image upload, group, staging) |
| `remote_storage` | 01 — configure remote storage only |
| `image_upload` | 02 — upload NOS image only |
| `upgrade_group` | 03 — create/update upgrade group only |
| `stage_upgrade` | 04 — stage upgrade and optional pre-upgrade report |
| `start_upgrade` | 05 — execute the upgrade |
| `post_upgrade_analysis` | 06 — post-upgrade report only |

---

## How It Works

### Step 1 — Remote Storage (`01_configure_remote_storage.yml`)

Calls `GET /api/v1/infra/remoteStorage/{name}`. If the entry is absent, creates it via `POST`. If it already exists, this step is skipped with no changes made.

### Step 2 — Image Upload (`02_upload_image.yml`)

Calls `GET /api/v1/manage/fabricSoftware/images` and checks whether the target image filename is already present on ND. If absent, triggers an import from the remote storage server. Polls until the import completes.

### Step 3 — Upgrade Group (`03_create_upgrade_group.yml`)

Calls `GET /api/v1/manage/fabrics/{fabric}/updateGroups` (fabric-scoped — NOT the global endpoint) to check whether the group exists in this fabric. Creates the group via `POST` if absent; updates it via `PUT` if already present.

> **Important:** ND enforces globally unique upgrade group names across all fabrics. If a group with the same name exists in any other fabric, the fabric-scoped check correctly ignores it and creates a new group. Always include the fabric name in your group name (e.g. `ansible_upgrade_group_myfabric`) to avoid confusion.
>
> **ND API quirk:** The `POST` response is always `HTTP 207 Multi-Status`, even on failure. The playbook inspects the response body (`updateGroups[0].status`) in addition to the HTTP status to detect real errors.

### Step 4 — Stage Upgrade (`04_stage_upgrade.yml`)

Posts to `POST /api/v1/manage/fabrics/{fabric}/softwareUpdatePlan/actions/stage`. This downloads the target image to the switch bootflash and verifies its integrity. The operation is asynchronous — the playbook polls the plan summary every 60 seconds until the group reaches `stageSuccess` or `validateSuccess`.

If `nd_swim_upgrade_group_report_selection` is not `noReport`, the pre-upgrade report is fetched after staging and its results are displayed in the playbook output.

> **Note:** When `nd_swim_upgrade_group_analysis: noAnalysis` is set (the default), ND does not generate a pre-upgrade report. The `preReportInstanceId` field will be `null`. The playbook skips the report tasks cleanly in this case.

### Step 5 — Execute Upgrade (`05_start_upgrade.yml`)

Posts to `POST /api/v1/manage/fabrics/{fabric}/softwareUpdatePlan/actions/updateSoftware`. ND returns `HTTP 202 Accepted` and begins the upgrade asynchronously. The playbook polls the plan summary every 60 seconds (up to 60 retries = 60 minutes) until the upgrade group reaches a terminal status.

### Step 6 — Post-Upgrade Report (`06_post_upgrade_analysis.yml`)

Only runs when `nd_swim_upgrade_group_report_selection != noReport`. Fetches the post-upgrade report and displays a comparison against the pre-upgrade baseline (test counts, pass/fail/warning totals).

---

## Expected Output (successful run)

```
TASK [nd_swim_upgrade : Display Post-Upgrade Report Summary]
ok: [nd-data] => {
    "msg": [
        "==========================================",
        "Post-Upgrade Report Summary",
        "==========================================",
        "Total Tests: 12",
        "Tests Passed: 11",
        "",
        "Overall Totals:",
        "  Success: 11",
        "  Errors: 0",
        "  Warnings: 1",
        "  Info: 0",
        "=========================================="
    ]
}

PLAY RECAP
nd-data : ok=35  changed=0  unreachable=0  failed=0  skipped=12
```

---

## Troubleshooting

| Symptom | Likely Cause | Resolution |
|---|---|---|
| `FAILED - RETRYING: Poll Staging Progress` repeats indefinitely | Image import from remote storage stalled | Verify the SCP server is reachable from ND. Check `Manage → Fabric Software → Images` for import status |
| Task 03 fails with HTTP 404 on PUT | Upgrade group check found a false-positive from another fabric | Use the fabric-scoped endpoint (already fixed in this repo). Rename the group to include the fabric name |
| Task 03 creation fails with `status: failed` in body | Group name already exists in another fabric | Rename `nd_swim_upgrade_group_name` to include the fabric name |
| `nd_swim_upgrade_pre_upgrade_report_id` errors | `preReportInstanceId` is null — analysis mode is `noAnalysis` | Expected — pre-report tasks are skipped automatically. No action needed |
| Playbook shows `UNREACHABLE` for `nd-data` | `ND_HOST` environment variable is not set | Re-export `export ND_HOST=...` in the current shell session |
| API calls return HTML instead of JSON | Session expired or wrong URL — raw response is an ND error page | Verify `ND_HOST`, `ND_USERNAME`, and `ND_API_KEY` are all set and correct |
| Upgrade group stuck in `stageFailed` | Image file missing or corrupted on bootflash | Delete the upgrade group in ND UI, re-run the playbook from `--tags setup` |
| Post-upgrade report shows interface warnings | Port state changed from `init` to `notconnec` after reload | Expected on virtual/simulated switches with no physical cables — not an error |

---

## Requirements

| Dependency | Version |
|---|---|
| Cisco Nexus Dashboard | 4.1+ |
| Python | 3.12 – 3.13 |
| ansible-core | 2.15.0+ |
| uv | Latest |

---

## License

This project is licensed under the **Cisco Sample Code License, Version 1.1**. See the [LICENSE](LICENSE) file for details.

---

## Contributing

1. Set up pre-commit hooks (see [docs/development/pre-commit.md](docs/development/pre-commit.md)).
2. Create a feature branch: `git checkout -b feature/your-change`.
3. Ensure linting passes: `uv run ansible-lint`.
4. Open a Pull Request with a description of what was tested and against which ND version.

### Why `uv` instead of pip?

- **Speed:** Installs ansible-core and all dependencies up to 100× faster than pip.
- **Reproducibility:** `uv.lock` pins every transitive dependency so every team member runs identical versions.
- **No activation required:** `uv run` automatically uses the project environment — no `source .venv/bin/activate` needed.
