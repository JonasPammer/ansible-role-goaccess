# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a production-grade Ansible role that automates the installation and configuration of GoAccess, a real-time web log analyzer. The role is published to Ansible Galaxy as `jonaspammer.goaccess` and supports multiple Linux distributions with extensive testing across different Ansible versions.

**Key Capabilities:**
- Installs GoAccess from either system package manager or source compilation
- Generates fully customizable GoAccess configuration files
- Optionally creates systemd services for real-time HTML report generation
- Manages file permissions for log files and directories
- Supports Ubuntu, Debian, Rocky Linux, and Fedora

## Common Commands

### Testing

```bash
# Run all tests (lint + molecule across all Ansible versions)
tox

# Run tests with a specific distribution
MOLECULE_DISTRO=ubuntu2204 tox

# Run tests for a specific Ansible version
tox -e py3-ansible-9

# Debug by keeping container alive after failure
MOLECULE_DESTROY=never MOLECULE_DISTRO=ubuntu2204 tox -e py3-ansible-9

# Connect to a molecule-created container for debugging
docker ps  # Find container name
docker exec -it <container-id> /bin/bash
```

**Available test environments:**
- `py3-ansible-6` (Ansible 6.x / core 2.13)
- `py3-ansible-7` (Ansible 7.x / core 2.14)
- `py3-ansible-8` (Ansible 8.x / core 2.15)
- `py3-ansible-9` (Ansible 9.x / core 2.16)

**Tested distributions:** Ubuntu 20.04/22.04, Debian 11/12, Rocky Linux 8/9, Fedora 39

### Linting

```bash
# Run all pre-commit hooks
pre-commit run --all-files --show-diff-on-failure

# Install pre-commit hooks (runs on every commit)
pre-commit install

# Specific linters
yamllint . -f standard
ansible-lint
```

### Development Setup

```bash
# Create and activate virtual environment
python3 -m venv venv
source venv/bin/activate

# Install development dependencies
pip3 install -r requirements-dev.txt

# Install pre-commit hooks
pre-commit install
```

## Architecture

### Task Execution Flow

The role follows this high-level execution flow (see `tasks/main.yml`):

1. **Variable Validation** (`tasks/assert.yml`): Validates 80+ role variables with type checking and range validation. This prevents cryptic failures later.

2. **Version Check**: Queries installed GoAccess version to determine if installation/upgrade is needed.

3. **Installation** (conditional):
   - **System Method** (`tasks/install-from_system.yml`): Uses package manager
     - For Debian/Ubuntu: Optionally adds official GoAccess APT repository
     - For RedHat/Fedora: Uses distro repositories (default for these families)
   - **Source Method** (`tasks/install-from_source.yml`): Compiles from GitHub source
     - Installs build tools (gcc, autoconf, gettext)
     - Installs system dependencies (ncurses, geoip2, openssl)
     - Downloads and extracts source
     - Runs `autoreconf → configure → make → make install`
     - Default for Debian/Ubuntu for newer versions

4. **Configuration Generation** (`templates/goaccess.conf.jinja2`): Creates configuration file with 100+ possible settings using macro-driven Jinja2 template.

5. **File Permissions**: Sets ownership/permissions for log files and directories.

6. **Systemd Service** (optional): When `goaccess_systemd: true`:
   - Creates HTML output file
   - Generates systemd unit file from template
   - Starts and enables service
   - Validates service is running

### Variable Resolution System

The role uses a sophisticated **5-tier variable resolution** pattern for OS-specific values (see `defaults/main.yml:26-31` and `defaults/main.yml:64-69`):

```yaml
# Priority order (highest to lowest):
1. Distribution+Version specific (e.g., _packages[Ubuntu_22])
2. OS Family+Version specific (e.g., _packages[Debian_11])
3. Distribution specific (e.g., _packages[Ubuntu])
4. OS Family specific (e.g., _packages[Debian])
5. Default (e.g., _packages[default])
```

**Example application:**
```yaml
_goaccess_source_system_packages:
  Debian:
    - libncursesw5-dev
    - libmaxminddb-dev
  RedHat_9:  # Override for Rocky/Fedora 9
    - ncurses-devel
    - libmaxminddb  # No -devel suffix
```

This allows precise control over package names that vary across distributions while maintaining DRY principles.

### Key Architectural Patterns

**Dual Installation Strategy:**
- System package manager for stability (RedHat family default)
- Source compilation for latest features (Debian/Ubuntu default)
- Automatic method selection based on `ansible_os_family`

**Template-Driven Configuration:**
- Single Jinja2 template with conditional rendering
- Variables with `~` (None) or empty arrays `[]` are omitted from output
- Supports all 100+ GoAccess configuration options

**Comprehensive Variable Validation:**
- Early failure prevents hard-to-debug issues
- Type checking (boolean, string, integer, array)
- Range validation for numeric values
- See `tasks/assert.yml` for validation patterns

**Idempotent Installation:**
- Version checking prevents unnecessary reinstalls
- Source installation only runs if version mismatches or binary missing

## Important Development Conventions

### CookieCutter Synchronization

This project is templated from [cookiecutter-ansible-role](https://github.com/JonasPammer/cookiecutter-ansible-role). Before making changes to infrastructure files (CI workflows, pre-commit config, tox.ini, etc.), check if the change should be made to the template instead. Use `cruft` to sync template updates.

### Commit Message Convention

**Core contributors must follow Conventional Commits** for automatic versioning and changelog generation. Casual contributors don't need to worry as PRs are squash-merged.

Format: `type(scope): description`
- Types: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`
- Scope: optional, e.g., `(systemd)`, `(install)`

### Versioning

- Versions are defined using Git tags (no `v` prefix)
- Tags automatically trigger Galaxy release via GitHub Actions
- Follows Semantic Versioning

### Testing Strategy

**Multi-dimensional matrix:**
- 4 Ansible versions (6-9)
- 7 distributions (Ubuntu 20/22, Debian 11/12, Rocky 8/9, Fedora 39)
- Tests run in parallel (max 4 concurrent) on GitHub Actions

**Molecule test flow:**
1. Create Docker container
2. Run `ansible-lint`
3. Prepare: Install bootstrap + apache2 roles
4. Converge: Apply role twice (default scenario + systemd scenario)
5. Verify: Check binary, service status, HTML generation
6. Collect debug artifacts (`/var/tmp/vars.yml`, `/var/tmp/environment.yml`)

**Debug artifacts** are attached to GitHub CI runs for comparing test runs and debugging failures.

### Dev Container Support

The repository includes a VS Code devcontainer definition (`.devcontainer/`) with:
- Docker-in-Docker (dind) support for molecule testing
- Pre-configured Python environment
- SSH/GPG key forwarding capability

Use `Remote-Containers: Open Folder in Container` in VS Code for a complete development environment.

## File Organization

```
ansible-role-goaccess/
├── tasks/
│   ├── main.yml                  # Orchestrates role execution flow
│   ├── assert.yml                # Variable validation (80+ assertions)
│   ├── install-from_system.yml   # Package manager installation
│   ├── install-from_source.yml   # Source compilation workflow
│   └── install_deb_repo.yml      # Debian repository setup
├── defaults/main.yml             # Default variables (100+ options)
├── templates/
│   ├── goaccess.conf.jinja2      # GoAccess config template
│   └── goaccess.service.jinja2   # Systemd unit template
├── molecule/
│   ├── default/
│   │   ├── molecule.yml          # Test configuration
│   │   ├── converge.yml          # Role application playbook
│   │   └── verify.yml            # Post-run validation
│   └── resources/
│       └── prepare.yml           # Pre-test setup (bootstrap, apache2)
├── meta/main.yml                 # Galaxy metadata (platforms, dependencies)
└── vars/main.yml                 # Non-overrideable platform variables
```

## Important Notes for Development

### Installation Method Limitation

**RedHat source installation is not supported** due to `autopoint` requiring `find` command during `autoreconf`. This is why `goaccess_install_method` defaults to `'system'` for RedHat family. See README.adoc line 93-125 for details.

### Common Pitfalls

1. **Variable Validation**: Always run full tests after changing variables in `defaults/main.yml`. The strict validation in `assert.yml` will catch issues early.

2. **Template Changes**: When modifying `templates/goaccess.conf.jinja2`, ensure variables set to `~` (None) or `[]` don't render empty lines. Use conditional blocks.

3. **Systemd Testing**: Systemd tests fail on CentOS 7 due to old GoAccess version. This is expected.

4. **CI vs Local**: To see installed package versions during local testing (as CI does), run: `CI=true tox`

5. **Molecule Debugging**: When `MOLECULE_DESTROY=never` is set, remember to manually clean up containers afterward with `docker container prune`.

### Pre-commit Hooks

15 hooks are configured:
- **Commit validation**: commitlint (conventional commits)
- **YAML**: yamllint, prettier
- **Ansible**: ansible-lint
- **Python**: pyupgrade, black, flake8, mypy, reorder-python-imports, docformatter
- **General**: check-case-conflict, check-symlinks, detect-secrets, trailing-whitespace

Hooks run automatically on commits (if installed) and on PRs via pre-commit.ci.

## Common Development Tasks

### Adding a New GoAccess Configuration Option

1. Add variable to `defaults/main.yml` with sensible default
2. Document in README.adoc under "Role Variables" section
3. Add to `templates/goaccess.conf.jinja2` with conditional rendering
4. Add validation to `tasks/assert.yml` if needed
5. Test with `MOLECULE_DISTRO=ubuntu2204 tox -e py3-ansible-9`

### Adding Support for a New Distribution

1. Add platform to `meta/main.yml`
2. Update OS-specific variable dictionaries in `defaults/main.yml` (package names)
3. Add to CI matrix in `.github/workflows/ci.yml`
4. Test locally with `MOLECULE_DISTRO=<new-distro> tox`

### Debugging Test Failures

1. Run with `MOLECULE_DESTROY=never MOLECULE_DISTRO=<distro> tox -e py3-ansible-X`
2. Find container: `docker ps`
3. Connect: `docker exec -it <container-id> /bin/bash`
4. Check debug files: `cat /var/tmp/vars.yml` and `/var/tmp/environment.yml`
5. Manually test commands from verify.yml
6. Clean up: `docker stop <container-id> && docker container rm <container-id>`

For CI failures, download the debug artifacts from the GitHub Actions run to compare with successful runs.
