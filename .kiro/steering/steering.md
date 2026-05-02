---
inclusion: always
---

# Diffusion Ansible Tests Role — Technical Steering Document

## Project Overview

An Ansible role (`polar.diffusion_tests`) that provides a DRY, reusable set of infrastructure verification tasks for Molecule-based testing. It validates Docker containers, network ports, shell command outputs, HTTP/HTTPS endpoints, and PostgreSQL databases inside CI/CD pipelines using the Diffusion framework.

- **Role name**: `diffusion_tests`
- **Namespace**: `polar`
- **Galaxy name**: `polar.diffusion_tests`
- **Framework**: [Diffusion](https://github.com/Polar-Team/diffusion)
- **Min Ansible**: 2.15
- **Platforms**: Ubuntu (jammy, noble), Debian (bookworm)
- **License**: MIT
- **Author**: Daniel Dalavurak / Polar Team

## Development Rule — dev-new-features

All agent changes, new feature implementation, refactoring, and experimental work MUST be done inside the `dev-new-features/` directory. This directory mirrors the main project structure and acts as the active development branch within the repo.

```
dev-new-features/
├── .github/workflows/
├── defaults/main.yml
├── meta/main.yml
├── scenarios/default/
├── tasks/
│   ├── main.yml
│   ├── test_cmd.yml
│   ├── test_docker.yml
│   ├── test_docker_rootless.yml
│   ├── test_ports.yml
│   ├── test_postgres.yml
│   └── test_uri.yml
├── tests/
│   ├── cmd/
│   ├── docker/
│   ├── ports/
│   ├── postgres/
│   └── uri/
├── diffusion.toml
├── diffusion.lock
├── CHANGELOG.md
└── README.md
```

**Do NOT modify files in the root-level project directories directly.** Always work in `dev-new-features/` unless explicitly told otherwise.

## Project Structure

```
diffusion-ansible-tests-role/
├── defaults/main.yml              # Default variables (all test types optional)
├── meta/main.yml                  # Galaxy metadata, collections
├── tasks/
│   ├── main.yml                   # Entry point — dispatches to test modules
│   ├── test_ports.yml             # Port state verification (wait_for)
│   ├── test_docker.yml            # Docker container state/image/health checks
│   ├── test_docker_rootless.yml   # Rootless Docker socket auto-detection
│   ├── test_cmd.yml               # Shell command output validation
│   ├── test_postgres.yml          # PostgreSQL DB/roles/tables/records
│   └── test_uri.yml               # HTTP/HTTPS endpoint + SNI testing
├── scenarios/
│   └── default/
│       ├── molecule.yml           # Molecule config (Docker driver, Ubuntu 22.04)
│       ├── converge.yml           # Environment setup playbook
│       ├── verify.yml             # Verification playbook (includes tests/)
│       └── requirements.yml       # Ansible collection deps (pinned versions)
├── tests/                         # Test cases for the role itself
│   ├── cmd/test_cmd.yml
│   ├── docker/{setup,test_docker}.yml
│   ├── ports/{setup,test_ports}.yml
│   ├── postgres/{setup,test_postgres}.yml
│   ├── uri/{setup,test_uri}.yml
│   └── README.md
├── .github/workflows/
│   └── test-and-release.yml       # CI: test on push/PR, release on tag
├── diffusion.toml                 # Diffusion config
├── diffusion.lock                 # Pinned dependency versions
├── CHANGELOG.md
└── README.md
```

## Test Modules (tasks/)

The role entry point `tasks/main.yml` dispatches to individual test modules based on which variables are defined:

| Module | Variable | What it validates |
|---|---|---|
| `test_ports.yml` | `verify_ports` | Network port states (started/stopped) via `wait_for` |
| `test_docker.yml` | `verify_docker_containers` | Container state, image, health check status |
| `test_docker_rootless.yml` | `verify_docker_rootless` | Auto-detects rootless Docker socket, sets env |
| `test_cmd.yml` | `verify_output_in_cmd` | Shell/command output against expected patterns |
| `test_postgres.yml` | `postgres_db` + `postgres_expected_*` | DB existence, roles, tables, record queries |
| `test_uri.yml` | `verify_uris` | HTTP/HTTPS endpoints with SNI, redirects, status codes |

## Role Variables (defaults/main.yml)

### Port Tests

```yaml
verify_ports: []
# Each item: { port, host?, timeout?, state?, description?, delay?, sleep? }
# state: "started" (listening) or "stopped" (not listening)
```

### Docker Tests

```yaml
verify_docker_containers: []
# Each item: { name, state, image?, health? }
# state: running, stopped, paused, restarting, created

verify_docker_user: "{{ docker_user | default('root') }}"
verify_docker_rootless: true
verify_docker_env_override: []
```

### Command Tests

```yaml
verify_output_in_cmd: []
# Each item: { cmd, output, exact_match?, use_shell? }
```

### URI Tests

```yaml
verify_uris: []                              # List of URL strings
uri_test_ip: "127.0.0.1"                    # IP for SNI /etc/hosts entries
uri_validate_certs: false                    # SSL cert validation
uri_timeout: 10                              # Request timeout (seconds)
uri_expected_status: [200, 301, 302]         # Accepted HTTP status codes
uri_follow_redirects: "safe"                 # Redirect behavior
```

### PostgreSQL Tests

```yaml
postgres_host: "127.0.0.1"
postgres_port: 5432
postgres_db: ""
postgres_user: ""
postgres_password: ""                        # Protected with no_log
postgres_expected_tables: []                 # Schema-qualified table names
postgres_expected_records: []                # { query, description? }
postgres_expected_roles: []                  # Role names that must exist
```

## Required Ansible Collections

| Collection | Min Version | Purpose |
|---|---|---|
| `community.general` | 7.4.0 | General utilities |
| `community.docker` | 3.4.0 | Docker container inspection |
| `community.postgresql` | 2.4.0 | PostgreSQL queries and validation |

Pinned versions in `scenarios/default/requirements.yml`:
- `community.general` 12.5.0
- `community.postgresql` 4.2.0
- `community.docker` 5.1.0

## Diffusion Configuration (diffusion.toml)

```toml
[container_registry]
  registry_server = "ghcr.io"
  registry_provider = "Public"
  molecule_container_name = "polar-team/diffusion-molecule-container"
  molecule_container_tag = "latest-amd64"

[tests]
  type = "local"

[cache]
  enabled = true
  docker_cache = true
  uv_cache = true
```

Dependencies locked in `diffusion.lock` — Python 3.13 pinned, ansible ≥13.0.0, molecule ≥24.0.0.

## Test Execution

### With Diffusion CLI

```bash
diffusion molecule --ci                          # Converge (CI mode)
diffusion molecule --ci --verify                 # Run all verification tests
diffusion molecule --ci --lint                   # Linting
diffusion molecule --ci --idempotence            # Idempotence check
diffusion molecule --ci --wipe                   # Cleanup

# Tag-based selective execution
diffusion molecule --ci --verify -- --tags ports
diffusion molecule --ci --verify -- --tags docker
diffusion molecule --ci --verify -- --tags cmd
diffusion molecule --ci --verify -- --tags postgres
diffusion molecule --ci --verify -- --tags uri
diffusion molecule --ci --verify -- --tags "ports,docker"
```

### Test Tags

| Tag | Test Cases | Description |
|---|---|---|
| `ports` | 4 | Network port state verification |
| `docker` | 6 | Container state, image, health checks |
| `cmd` | 9 | Shell command output validation |
| `postgres` | 7 | Database, roles, tables, records |
| `uri` | 8 | HTTP/HTTPS endpoints, SNI, redirects |
| `all` | 34 | All test categories |

## CI/CD (GitHub Actions)

Workflow: `.github/workflows/test-and-release.yml`

- **Trigger**: push to `main`, PRs to `main`, tag pushes (`v*`)
- **Test job**: Uses `Polar-Team/diffusion/diffusion-test@main` composite action with caching
- **Release job**: On tag push only — generates changelog, creates GitHub release

## Molecule Scenario (scenarios/default/)

- **Driver**: Docker
- **Platform**: `geerlingguy/docker-ubuntu2204-ansible` (privileged, systemd, cgroups)
- **Provisioner**: Ansible
- **Verifier**: Ansible (tag-based test inclusion from `tests/` directory)

## Usage in Other Roles

Include in a Molecule `verify.yml`:

```yaml
- name: Verify
  hosts: all
  tasks:
    - name: Run tests
      ansible.builtin.include_role:
        name: tests/diffusion_tests
      vars:
        verify_ports:
          - port: 80
            description: "HTTP"
        verify_docker_containers:
          - name: "my-app"
            state: "running"
            health: "healthy"
```
