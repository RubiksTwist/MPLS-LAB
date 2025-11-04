Linting and pre-commit setup
=============================

This repository includes linters and a pre-commit configuration to keep Ansible
playbooks and YAML data consistent. The files added are:

- `.pre-commit-config.yaml` — pre-commit hooks for yamllint, basic checks and a
  local `ansible-lint` hook.
- `requirements-dev.txt` — Python packages for the development environment.

Quick start (local)
-------------------

1. Create and activate a Python virtual environment:

```bash
python3 -m venv .venv
source .venv/bin/activate
```

2. Install development dependencies:

```bash
pip install -r requirements-dev.txt
```

3. Install the pre-commit hooks into your git configuration:

```bash
pre-commit install
```

4. Run the hooks against all files once (optional but recommended):

```bash
pre-commit run --all-files
```

Run linters manually
--------------------

YAML linting (uses `.yamllint` config):

```bash
yamllint ansible/playbooks/*.yml ansible/playbooks/group_vars/*.yml ansible/playbooks/host_vars/*.yml
```

Ansible linting:

```bash
ansible-lint ansible/playbooks/*.yml
```

Notes
-----
- The `ansible-lint` pre-commit hook is configured as a `system` hook, which
  means it uses whatever `ansible-lint` binary is available in your environment
  (recommended: the `.venv` you created above).
- Secrets (for example `group_vars/vyos_secrets.yml`) should not be committed.
  Consider using Ansible Vault or an external secrets manager for sensitive
  data.
