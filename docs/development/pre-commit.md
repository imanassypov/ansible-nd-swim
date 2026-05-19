# Setting Up Pre-Commit Hooks

To maintain high code quality and prevent common Ansible/Python errors from being committed, this project uses `pre-commit`.

## Prerequisites

Ensure you have `uv` installed on your system.

## 1. Install pre-commit

We recommend installing `pre-commit` as a global tool using `uv`. This version includes a plugin that makes hook execution significantly faster.

```bash
uv tool install pre-commit --with pre-commit-uv
```

## 2. Enable Hooks in the Repo

Once installed, you must register the hooks with Git in the root of this repository:

```bash
pre-commit install
```

This will create a script in .git/hooks/pre-commit that runs automatically every time you run git commit.

## 3. (Optional) Run against all files

If you are working on an existing project or want to verify everything immediately:

```bash
pre-commit run --all-files
```

## Common Commands

* **Update hooks:** pre-commit autoupdate (Updates the versions in .pre-commit-config.yaml)
* **Skip hooks:** git commit -m "msg" --no-verify (Use sparingly!)
* **Uninstall:** pre-commit uninstall
